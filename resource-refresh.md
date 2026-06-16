# K9s 资源表格刷新机制代码分析

本文档基于仓库源码逐行梳理资源表格的**刷新节奏控制、Informer 缓存读取、差异标记、界面重绘**四个环节的边界关系。

---

## 一、模块分层与文件映射

```
┌──────────────────────────────────────────────────────────────────┐
│  View 层：UI 展示与事件分发                                        │
│  ├─ internal/view/browser.go     — 浏览器，实现 TableListener    │
│  ├─ internal/view/table.go       — 视图表格封装                  │
│  └─ internal/ui/table.go         — tview 表格渲染，含 UpdateUI() │
├──────────────────────────────────────────────────────────────────┤
│  Model 层：数据刷新调度与增量计算                                   │
│  ├─ internal/model/table.go      — Table：updater goroutine      │
│  ├─ internal/model/helpers.go    — resourceMeta() 选择 DAO      │
│  ├─ internal/model1/table_data.go — TableData.Update() 算 Delta │
│  ├─ internal/model1/delta.go     — DeltaRow 计算                │
│  └─ internal/model1/row_event.go — RowEvent 标记 Add/Update 等  │
├──────────────────────────────────────────────────────────────────┤
│  DAO 层：数据来源策略（三种获取方式并存）                            │
│  ├─ internal/dao/resource.go     — Resource.List → 读 Informer  │
│  ├─ internal/dao/generic.go      — Generic.List → 直连 APIServer│
│  ├─ internal/dao/table.go        — Table.List → HTTP Table 格式 │
│  ├─ internal/dao/pod.go          — Pod.List → Informer + 指标  │
│  └─ internal/dao/accessor.go     — AccessorFor() DAO 注册       │
├──────────────────────────────────────────────────────────────────┤
│  Informer 层：Kubernetes 缓存                                      │
│  └─ internal/watch/factory.go    — Factory.List/Get → Lister    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、刷新节奏：什么时候刷新？刷新频率如何控制？

### 2.1 定时刷新循环（核心路径）

启动入口：`Browser.Start()` → `GetModel().Watch(ctx)`

`internal/model/table.go#L120-L128`
```go
func (t *Table) Watch(ctx context.Context) error {
    if err := t.refresh(ctx); err != nil {
        return err
    }
    go t.updater(ctx)     // 启动后台定时刷新 goroutine
    return nil
}
```

`internal/model/table.go#L203-L227` — `updater()` 定时循环
```go
func (t *Table) updater(ctx context.Context) {
    bf := backoff.NewExponentialBackOff()
    bf.InitialInterval = initRefreshRate  // 300ms
    bf.MaxElapsedTime    = maxReaderRetryInterval  // 2 分钟
    rate := initRefreshRate  // 首次 300ms 快速刷新

    for {
        select {
        case <-ctx.Done():
            return
        case <-time.After(rate):
            rate = t.refreshRate  // 之后切换为配置的刷新间隔
            err := backoff.Retry(func() error {
                if err := t.refresh(ctx); err != nil {
                    return err  // 失败时按指数退避重试
                }
                return nil
            }, backoff.WithContext(bf, ctx))
            if err != nil {
                t.fireTableLoadFailed(err)  // 超过 2 分钟持续失败就退出
                return
            }
        }
    }
}
```

**刷新节奏要点：**

| 阶段 | 间隔值 | 定义位置 |
|------|--------|---------|
| 首次刷新后首循环 | 300ms | `initRefreshRate` in `internal/model/table.go#L26` |
| 稳定期循环 | 默认 2s，可配置 | `defaultRefreshRate=2` in `internal/config/types.go#L7` |
| 失败退避 | 指数递增，最长 2min 总时长 | `internal/model/types.go#L21` |

### 2.2 刷新频率配置链路

配置值最终注入到 `Table.refreshRate`：

```
internal/config/flags.go#L8       DefaultRefreshRate = 2.0 (秒)
    ↓ (用户可通过 --refresh-rate 覆盖)
internal/config/k9s.go#L396-L419   GetRefreshRate() → RefreshDuration()
    ↓ (最低限制不低于 DefaultRefreshRate)
internal/view/browser.go#L116      SetRefreshRate(Config.K9s.RefreshDuration())
```

### 2.3 防重入：CAS 原子锁丢弃并发刷新

`internal/model/table.go#L229-L247` — `refresh()`
```go
func (t *Table) refresh(ctx context.Context) error {
    // 边界 1：如果上一次 refresh 尚未结束，直接丢弃本次请求
    if !atomic.CompareAndSwapInt32(&t.inUpdate, 0, 1) {
        slog.Debug("Dropping update...")
        return nil
    }
    defer atomic.StoreInt32(&t.inUpdate, 0)

    // 边界 2：CAS 成功之后才进入实际数据拉取与计算
    if err := t.reconcile(ctx); err != nil {
        return err
    }
    // 边界 3：数据计算完成后，Peek() 克隆一份快照再通知
    data := t.Peek()
    if data.RowCount() == 0 {
        t.fireNoData(data)
    } else {
        t.fireTableChanged(data)
    }
    return nil
}
```

> **关键边界**：`refresh()` 期间的 CAS 锁保证"同一时刻只有一个 refresh 在执行"，即使用户触发多次也会被合并。

### 2.4 手动刷新触发点

除了定时循环，下列操作也会触发立即刷新：

| 操作 | 入口代码 | 实际效果 |
|------|---------|---------|
| Ctrl+R | `internal/view/browser.go#L478-L483` `refreshCmd` | 调用 `Start()` 重连 Watcher |
| 切换命名空间 | `internal/view/browser.go#L587` | `b.refresh()` → `Start()` |
| 退出搜索模式 | `internal/view/browser.go#L228-L251` `BufferActive()` | `model.Refresh()` |
| 排序、标记、列切换等 | `internal/ui/table.go` 内各 Cmd | 直接走本地 `Refresh()` |

---

## 三、数据读取：从哪里取数据？三种 DAO 策略并存

### 3.1 DAO 选择逻辑

`internal/model/helpers.go#L32-L45` — `resourceMeta(gvr)`
```go
func resourceMeta(gvr *client.GVR) ResourceMeta {
    // 优先级 1：在 model.Registry 中查是否有明确配置
    meta, ok := Registry[gvr]
    if !ok {
        // 完全未注册：默认走 HTTP Table 格式 API
        meta = ResourceMeta{
            DAO:      new(dao.Table),
            Renderer: new(render.Table),
        }
    }
    // 优先级 2：Registry 有注册但 DAO 为 nil → 走 Informer 缓存
    if meta.DAO == nil {
        meta.DAO = new(dao.Resource)
    }
    return meta
}
```

### 3.2 三种 DAO 的数据源与性能特征

#### A. 走 Informer 缓存（dao.Resource）

`internal/dao/resource.go#L27-L34`
```go
func (r *Resource) List(ctx context.Context, ns string) ([]runtime.Object, error) {
    lsel := labels.Everything()
    if sel, ok := ctx.Value(internal.KeyLabels).(labels.Selector); ok {
        lsel = sel
    }
    // wait=false：不等待缓存同步，直接读当前状态
    return r.getFactory().List(r.gvr, ns, false, lsel)
}
```

→ 最终到 `internal/watch/factory.go#L75-L99` `Factory.List()`：
```go
inf.Lister().List(lbls)       // 读本地内存缓存，无网络开销
// wait=false 时不等 HasSynced，直接返回
```

**使用该 DAO 的资源**：Registry 中注册但未指定 DAO 的资源，如：
`EpsGVR, EpGVR, SaGVR, PvGVR, PvcGVR, NpGVR, ScGVR, PdbGVR, CrbGVR, RoGVR, RobGVR` 等（见 `internal/model/registry.go`）

#### B. 直连 HTTP Table 格式（dao.Table）

`internal/dao/table.go#L59-L100`
```go
func (t *Table) List(ctx context.Context, ns string) ([]runtime.Object, error) {
    // 直接走 REST 接口，Accept: application/json;as=Table
    o, err := c.Get().
        SetHeader("Accept", header).
        Param("includeObject", includeObject).
        Namespace(ns).Resource(t.gvr.R()).
        VersionedParams(&metav1.ListOptions{...}, ...).
        Do(ctx).Get()
    ...
    return []runtime.Object{ta}, nil  // 包装成单个 metav1.Table 对象
}
```

**使用该 DAO 的资源**：
- 完全未在 Registry 中注册的 CRD（走默认）
- `EvGVR (Events)`：`Registry` 中明确指定 `DAO: new(dao.Table)`

#### C. 直连动态 client（dao.Generic）

`internal/dao/generic.go#L41-L72`
```go
func (g *Generic) List(ctx context.Context, ns string) ([]runtime.Object, error) {
    dial, _ := g.dynClient()
    // dynamic client 直连 API Server
    ll, err = dial.Namespace(ns).List(ctx, opts)
    ...
}
```

**实际使用极少**，只有当 ResourceMeta 中的 DAO 既不是 nil 也不是明确配置时才会出现。注册过的 Pod/Deployment/Node 等都覆盖了 List 方法（见下）。

#### D. 扩展型 DAO（Informer + 额外数据合并）

如 Pod：`internal/dao/pod.go#L110-L149`
```go
func (p *Pod) List(ctx context.Context, ns string) ([]runtime.Object, error) {
    oo, err := p.Resource.List(ctx, ns)  // 第一步：走 Informer 缓存
    if err != nil { return oo, err }

    var pmx client.PodsMetricsMap
    if withMx, ok := ctx.Value(internal.KeyWithMetrics).(bool); ok && withMx {
        pmx, _ = client.DialMetrics(p.Client()).FetchPodsMetricsMap(ctx, ns)  // 第二步：合并 Metrics API
    }
    // 包装成 PodWithMetrics
}
```

同类：`Node.List, Deployment.List` 等都继承自 Resource 并做二次加工。

### 3.3 Informer 缓存未同步时的处理

在 View 层 `TableNoData` 中做了保护（`internal/view/browser.go#L298-L335`）：
```go
func (b *Browser) TableNoData(mdata *model1.TableData) {
    // 如果 informer 还没同步完，只提示"Synchronizing..."不显示"无资源"警告
    if synced, _ := b.app.factory.HasSynced(b.GVR(), b.GetNamespace()); !synced {
        b.app.QueueUpdateDraw(func() {
            b.app.Flash().Infof("Synchronizing %s in %q namespace...", ...)
        })
        return
    }
    // 同步完了还没数据才真正提示 No resources found
    ...
}
```

> **关键边界**：缓存同步检查只在"无数据"时才做；有数据时直接展示，避免误报。

---

## 四、差异标记：增量如何计算？时间列为何不抖动？

完整链路：`reconcile` → `data.Render` → `data.Update(rows)` → `DeltaRow` 计算

### 4.1 Render：runtime.Object → Rows

`internal/model1/table_data.go#L252-L279`
```go
func (t *TableData) Render(_ context.Context, r Renderer, oo []runtime.Object) error {
    var rows Rows
    if len(oo) > 0 {
        if r.IsGeneric() {  // 对应 dao.Table 的 HTTP Table 格式
            table, _ := oo[0].(*metav1.Table)
            rows = make(Rows, len(table.Rows))
            GenericHydrate(t.namespace, table, rows, r)
        } else {            // 对应 unstructured 对象列表
            rows = make(Rows, len(oo))
            Hydrate(t.namespace, oo, rows, r)
        }
    }
    t.Update(rows)                    // 计算 Delta 的入口
    t.SetHeader(t.namespace, r.Header(t.namespace))
    return nil
}
```

### 4.2 Update：新旧 Rows 对比，标记事件类型

`internal/model1/table_data.go#L425-L457`
```go
func (t *TableData) Update(rows Rows) {
    empty := t.Empty()
    kk := sets.New[string]()
    for _, row := range rows {
        kk.Insert(row.ID)

        // 情况 1：空表首次填充 → 全量 EventAdd
        if empty {
            t.rowEvents.Add(NewRowEvent(EventAdd, row))
            continue
        }

        // 情况 2：ID 已存在 → 算 Delta
        if index, ok := t.rowEvents.FindIndex(row.ID); ok {
            ev, _ := t.rowEvents.At(index)
            delta := NewDeltaRow(ev.Row, row, t.header)  // ★ 关键：算差异
            if delta.IsBlank() {
                ev.Kind = EventUnchanged                  // 无变化
            } else {
                t.rowEvents.Set(index, NewRowEventWithDeltas(row, delta))  // EventUpdate
            }
            continue
        }

        // 情况 3：新 ID → EventAdd
        t.rowEvents.Add(NewRowEvent(EventAdd, row))
    }

    // 情况 4：旧数据中出现但新集合没出现的 ID → 做 Delete
    if !empty {
        t.Delete(kk)
    }
}
```

### 4.3 DeltaRow：时间列被排除，避免 AGE 每秒变导致整行闪

`internal/model1/delta.go#L12-L24`
```go
func NewDeltaRow(o, n Row, h Header) DeltaRow {
    deltas := make(DeltaRow, len(o.Fields))
    for i, old := range o.Fields {
        if i >= len(n.Fields) {
            continue
        }
        // ★ 边界核心：h.IsTimeCol(i) == true 的列（AGE、Last Seen 等）不参与差异
        if old != "" && old != n.Fields[i] && !h.IsTimeCol(i) {
            deltas[i] = old   // 只保存旧值，用于 UI 层拼接"old→new"提示
        }
    }
    return deltas
}
```

时间列的定义在各 Renderer.Header 中（如 `ageCols = {"Last Seen", "First Seen", "Age"}`，见 `internal/render/table.go#L22`），HeaderColumn.Attrs.Time = true。

### 4.4 Delta 在 UI 层的消费

`internal/ui/table.go#L507-L561` `buildRow()`
```go
func (t *Table) buildRow(r int, re, ore model1.RowEvent, h model1.Header, pads MaxyPad) {
    ...
    for c, field := range re.Row.Fields {
        ...
        // 只在 Delta 非空且非时间列时拼"旧值→新值"的视觉提示
        if !re.Deltas.IsBlank() && !h.IsTimeCol(c) {
            var old string
            if c < len(re.Deltas) {
                old = re.Deltas[c]
            }
            field += Deltas(old, field)  // 拼高亮符号（如 ←xxx）
        }
        ...
    }
}
```

---

## 五、界面重绘：三层机制的代码证据与真实边界

本节的所有结论都基于可复核的代码路径，不做推断。先讲清楚两个容易混淆的核心事实：

1. **哪些请求会被丢弃？** — 只有 Model 层 `refresh()` 的并发请求会被 CAS 原子锁丢弃，其余一律排队执行。
2. **`updating` 标志真的能跳过 UpdateUI 吗？** — 在当前代码实现中**不能**。所有进入 draw 队列的回调都会完整执行 UpdateUI，下文会给出逐行证据。

### 5.1 三层机制定位（代码可复核）

| 机制 | 所在文件 | 冲突语义 | 代码证据 |
|------|---------|---------|---------|
| ① **刷新并发控制（丢）** | `internal/model/table.go#L229-L247` | **直接丢弃** refresh() 并行调用 | `CompareAndSwapInt32(&t.inUpdate, 0, 1)`失败就 `return nil` |
| ② **界面回调入队（排）** | `internal/ui/app.go#L75-L82` 封装 + tview 队列 | **绝不丢弃，一定入队串行执行** | 每次调用启动新 goroutine，调 `tview.Application.QueueUpdateDraw(f)` |
| ③ **绘制期间互斥（跳）** | `internal/view/browser.go#L36-L65` | 设计意图为跳过，但**当前实现下实际不发生跳过** | 下文 5.4 节逐行证明 |

三者的传递路径：`① 通过 → fire listener → listener 调 ② 入队 → tview 取出 f 执行 → 在 f() 首行执行 ③ 的检查`

---

### 5.2 机制① 刷新并发控制（CAS inUpdate）：唯一会丢弃请求的位置

`internal/model/table.go#L229-L247`
```go
func (t *Table) refresh(ctx context.Context) error {
    // 代码证据：唯一会"丢"的位置
    if !atomic.CompareAndSwapInt32(&t.inUpdate, 0, 1) {
        slog.Debug("Dropping update...")
        return nil   // 直接 return，不会 fire 任何 listener
    }
    defer atomic.StoreInt32(&t.inUpdate, 0)

    if err := t.reconcile(ctx); err != nil {
        return err
    }
    // 以下只有 CAS 成功才会执行
    data := t.Peek()
    if data.RowCount() == 0 {
        t.fireNoData(data)
    } else {
        t.fireTableChanged(data)  // 同步遍历 listeners 调 TableDataChanged
    }
    return nil
}
```

**代码定位事实：**
- CAS 成功才能进入 `reconcile`（DAO.List + Delta 计算）和 `fireXxx`
- CAS 失败时直接 `return nil`，**完全不触发任何 View 层回调**
- 释放时机：`defer`，`fireTableChanged` / `fireNoData` **同步执行完毕** 之后才释放锁（因为 `fire` 是同步遍历 listeners）

---

### 5.3 机制② 界面回调入队（QueueUpdateDraw）：所有回调一定入队并串行执行

#### 5.3.1 k9s 的关键封装：每次调用都启动新 goroutine

`internal/ui/app.go#L75-L82`
```go
// 代码证据：k9s 在 tview 之上又套了一层 goroutine
func (a *App) QueueUpdateDraw(f func()) {
    if a.Application == nil {
        return
    }
    go func() {
        a.Application.QueueUpdateDraw(f)  // 交给 tview 的 draw 队列
    }()
}
```

> 为什么要多套一层 goroutine？
> tview.QueueUpdateDraw 是把 f 发到 `drawChan`（无缓冲 channel）。如果 tview Run 主循环消费较慢，send 操作会**阻塞调用方 goroutine**（比如 updater goroutine）。k9s 加一层 goroutine 就是为了避免调用方被 channel send 阻塞，把 send 移到一个临时 goroutine 里。

#### 5.3.2 四条独立调用路径（每条都独立调一次 QueueUpdateDraw）

| 触发源 | 入口位置 | 在哪个 goroutine 中被调用 | 什么时间发生 |
|--------|---------|---------------------------|-------------|
| A. `TableDataChanged` | `internal/view/browser.go#L338-L365` | **updater goroutine**（`fireTableChanged` 同步调用 listener） | 定时 refresh 成功 + CAS 通过 → reconcile 完成 → fire |
| B. `TableNoData` | `internal/view/browser.go#L298-L335` | **updater goroutine**（同上） | refresh 后数据为空 + CAS 通过 |
| C. `BufferActive` | `internal/view/browser.go#L228-L251` | **tview 主循环 goroutine**（用户按键事件 → CmdBuff.SetActive → fireActive 同步调用 listener） | 用户按回车退出 filter 模式时 |
| D. `TableLoadFailed` | `internal/view/browser.go#L368-L373` | **updater goroutine**（`fireTableLoadFailed` 同步调用） | backoff 超过 2 分钟，持续失败 |

**代码证据 A：TableDataChanged 在 updater goroutine（fire 是同步遍历）**

`internal/model/table.go#L293-L299`
```go
func (t *Table) fireTableChanged(data *model1.TableData) {
    t.listeners.Range(func(key, value any) bool {
        value.(TableListener).TableDataChanged(data)  // 同步调，不是 go func
        return true
    })
}
```
所以整个 `TableDataChanged() → b.Update() → b.app.QueueUpdateDraw()` 这一串都在 **updater goroutine** 中执行。

**代码证据 C：BufferActive 在 tview 主循环 goroutine**

`internal/model/cmd_buff.go#L83-L89`（SetActive）→ `#L241-L244`（fireActive）都是同步调 listener：
```go
func (c *CmdBuff) SetActive(b bool) {
    c.mx.Lock()
    c.active = b
    c.mx.Unlock()
    c.fireActive(c.active)  // 同步遍历 listeners 调 BufferActive
}
```

而 `SetActive(false)` 的触发链：用户按回车 → tview 主循环处理按键 → Prompt 组件 → FishBuff/CmdBuff.Reset → SetActive(false) → fireActive。所以整个 `BufferActive() → model.Refresh() → model.Peek() → b.app.QueueUpdateDraw()` 这一串都在 **tview 主循环 goroutine** 中执行。

#### 5.3.3 tview 的执行保证：所有 f() 在同一个 goroutine 串行执行

无论哪个 goroutine 调 k9s.QueueUpdateDraw，最终所有 `f()` 闭包都在 **tview Run 主循环 goroutine** 中按进入队列的顺序**一个一个串行执行**。这是 tview 框架的核心线程安全保证。

**这是下文分析 `updating` 行为的关键前提。**

---

### 5.4 机制③ 绘制期间互斥（Browser.updating）：当前实现下实际不跳过任何 UpdateUI

#### 5.4.1 updating 标志和访问器的代码

`internal/view/browser.go#L36-L65`
```go
type Browser struct {
    ...
    mx        sync.RWMutex  // 保护 updating
    updating  bool          // 标志：是否"正在绘制"
}

func (b *Browser) setUpdating(f bool) {
    b.mx.Lock()
    defer b.mx.Unlock()
    b.updating = f
}

func (b *Browser) getUpdating() bool {
    b.mx.RLock()
    defer b.mx.RUnlock()
    return b.updating
}
```

#### 5.4.2 updating 在回调中的使用（以 TableDataChanged 为例）

`internal/view/browser.go#L338-L365`
```go
func (b *Browser) TableDataChanged(mdata *model1.TableData) {
    // 前置计算在当前调用方 goroutine 中执行（通常是 updater goroutine）
    cdata := b.Update(mdata, b.app.Conn().HasMetrics())

    b.app.QueueUpdateDraw(func() {
        // ★ 关键：这部分闭包在 tview 主循环 goroutine 中串行执行
        if b.getUpdating() {   // 第 1 步：读 updating
            return
        }
        b.setUpdating(true)    // 第 2 步：写 updating=true
        defer b.setUpdating(false)  // 第 N 步：闭包 return 前重置

        b.refreshActions()
        b.UpdateUI(cdata, mdata)
    })
}
```

所有三条路径（TableDataChanged / TableNoData / BufferActive）的 QueueUpdateDraw 闭包结构完全相同。

#### 5.4.3 逐行证明"当前实现下不会发生跳过"

下面基于代码事实逐步推导：

**前提事实（已由代码证明）：**
- P1：所有闭包 f() 在**同一个 goroutine**（tview 主循环 goroutine）中**按队列顺序串行执行**。
- P2：每个闭包内部的 `getUpdating() → setUpdating(true) → ... → defer setUpdating(false)` 这一整段代码在 goroutine 中**原子地顺序执行**，中间不会被其他闭包插入（因为是串行执行）。
- P3：`defer setUpdating(false)` 在**当前闭包 return 之前**一定已执行完毕。

**逐行推导：**
1. tview 取出队列第一个闭包 `f(C1)`，在主循环 goroutine 中执行。
2. `f(C1)` 第 1 步：`getUpdating()` → `updating=false`（初始状态 / 上一个闭包已 defer 重置）。
3. `f(C1)` 第 2 步：`setUpdating(true)` → `updating=true`。
4. `f(C1)` 执行 `UpdateUI(...)`。
5. `f(C1)` return，触发 defer：`setUpdating(false)` → `updating=false`。**此时 C1 已经完全退出。**
6. tview 取出队列下一个闭包 `f(A1)`，**在同一个 goroutine 中继续串行执行。**
7. `f(A1)` 第 1 步：`getUpdating()` → **此时 updating 一定是 false**（由 P2、P3：C1 的 defer setUpdating(false) 在**第 5 步就已经完成了**，在 A1 开始执行之前）。
8. 所以 `f(A1)` 通过检查，执行 `setUpdating(true)` → UpdateUI → defer → false。

**结论：** 在当前实现中，**所有进入 tview 队列的闭包都会通过 updating 检查并完整执行 UpdateUI**。`updating` 标志不会跳过任何一次 UpdateUI 调用。

> 那 `updating` + RWMutex 有什么意义？
> 这是**防御性编程**：
> - RWMutex 保证即使未来有**其他 goroutine**（非 tview 主循环）直接读写 `updating` 也是线程安全的。
> - updating 标志代表了"设计意图"——期望"同一时刻只允许一个绘制流程进行"。当前代码因为所有 f() 在同一个 goroutine 串行，这个意图天然被 tview 满足；但如果未来重构（比如把 QueueUpdateDraw 的 goroutine 嵌套去掉、或者引入并行渲染路径），这层保护就会真正生效。

---

### 5.5 完整时序例证（基于代码事实推演）

场景：用户输入 filter 后按回车（BufferActive），刚好 2s tick 也触发 refresh，两条路径几乎同时发生。

```
    updater goroutine          tview 主循环 goroutine          临时 goroutine（k9s 额外启动）
   ──────────────────        ────────────────────────        ──────────────────────────────────
T=0
  │ time.After(2s)到
  │ refresh()调用
  │ → CAS 通过(inUpdate=1)
  │ → reconcile()开始（耗时 T0-T5）
  │                                │
T=2 │                                │ 用户按回车：
  │                                │   CmdBuff.SetActive(false)
  │                                │   → fireActive 同步调
  │                                │   → BufferActive() 在主循环中执行
  │                                │     ├─ 调 model.Refresh() → refresh() CAS 失败
  │                                │     │     → return nil（被机制①丢弃）
  │                                │     ├─ model.Peek() 拿旧快照
  │                                │     ├─ b.Update(mdata) 算过滤
  │                                │     └─ b.app.QueueUpdateDraw(f_C1)
  │                                │           → 启动 goroutine X1 ────────┐
  │                                │                                        │
  │                                │                               X1: tview.QueueUpdateDraw(f_C1)
  │                                │                                    (send to drawChan，阻塞直到消费)
T=5 │                                │                                        │
  │ reconcile()完成                  │                                        │
  │ → fireTableChanged()同步         │                                        │
  │   → TableDataChanged()执行       │                                        │
  │     ├─ b.Update(mdata)算过滤     │                                        │
  │     └─ b.app.QueueUpdateDraw(f_A1)                                        │
  │           → 启动 goroutine X2 ────────────────────────────────────┐        │
  │                                                                  │        │
  │                                                      X2: tview.QueueUpdateDraw(f_A1)
  │                                                          (send to drawChan)
  │                                                                  ↓        ↓
T=6 │                               ┌──────────────────────────────────────────┘
  │                               │ Run 循环 select 取到 f_C1
  │                               │ 执行 f_C1（主循环 goroutine）：
  │                               │   getUpdating()=false
  │                               │   setUpdating(true)
  │                               │   UpdateUI(旧快照+新filter)
  │                               │   defer → setUpdating(false)  ← ★ 此时已变 false
  │                               │   Draw() 刷到终端
  │                               │
T=12│                               │ Run 循环下一轮 select 取到 f_A1
  │                               │ 执行 f_A1（同一 goroutine 继续）：
  │                               │   getUpdating()=false  ← ★ C1 已经设为 false
  │                               │   setUpdating(true)    ← ★ 通过检查，不被跳过
  │                               │   UpdateUI(新快照+新filter)
  │                               │   defer → setUpdating(false)
  │                               │   Draw() 刷到终端
```

**本时序可复核的代码点：**
- T=2 的 `model.Refresh()` 被 CAS 丢弃 → `internal/model/table.go#L229-L247`
- BufferActive 在主循环 goroutine → `internal/model/cmd_buff.go#L83-L89` + `#L241-L244`
- QueueUpdateDraw 启动临时 goroutine → `internal/ui/app.go#L75-L82`
- C1 defer 结束后 A1 才开始（串行）→ tview Run 主循环单线程执行保证
- A1 的 getUpdating() 看到 false → 不跳过 → 上文 5.4.3 的推导

---

### 5.6 UpdateUI：全量重建单元格（Clear + 逐行构建）

`internal/ui/table.go#L472-L505`
```go
func (t *Table) UpdateUI(cdata, data *model1.TableData) {
    t.Clear()                              // 清空所有单元格
    col := 0
    for _, h := range cdata.Header() {
        if t.shouldExcludeColumn(h) { continue }
        t.AddHeaderCell(col, h)
        col++
    }
    cdata.Sort(t.getSortCol())             // 就地排序
    ComputeMaxColumns(pads, ..., cdata)    // 算对齐列宽
    cdata.RowsRange(func(row int, re model1.RowEvent) bool {
        ore, _ := data.FindRow(re.Row.ID)  // 从原表查 Delta 信息
        t.buildRow(row+1, re, ore, cdata.Header(), pads)
        return true
    })
    t.updateSelection(true)                // 重定位选中行
    t.UpdateTitle()                        // 刷新标题栏计数
}
```

> **代码事实：**
> - `cdata` = Browser 过滤后的 TableData（行可能少），`data` = 原始 TableData（查 Delta 用）
> - 每次 `UpdateUI` 都是 `Clear` 后**全量重建所有单元格**，没有增量 diff；但由于是 tcell 内存操作 + tview 双缓冲 Draw，用户不会看到"清空→重绘"过程
> - `cdata.Sort()` 是就地排序，`RowsRange` 按排序后顺序遍历

---

### 5.7 三者语义差异对比（基于代码证据）

| 对比维度 | ① 刷新并发控制 | ② 界面回调入队 | ③ 绘制互斥 |
|---------|---------------|-------------|-----------|
| **处理对象** | `refresh()` 调用 | `QueueUpdateDraw(f)` 中的 UI 闭包 f | UpdateUI 调用 |
| **冲突策略** | **丢**（CAS 失败直接 return nil，不 fire listener） | **排**（全部入 tview 队列，按序串行执行，绝不丢） | 设计意图为"跳"，但**当前实现不会跳过**（见 5.4.3） |
| **所在线程** | 任意 goroutine（通常 updater） | 任意 goroutine 调 k9s.QueueUpdateDraw → 启动临时 goroutine → tview 主循环执行 f | f() 全部在**同一个 goroutine**（tview 主循环）串行执行 |
| **代码位置** | `internal/model/table.go#L229` 首行 | `internal/ui/app.go#L75-L82`（k9s 封装）+ 4 个 browser 调用点 | `internal/view/browser.go#L55-L65` + 3 处调用点 |
| **释放时机** | defer atomic.Store（fireXxx 同步执行完毕后） | f() return → tview 取队列下一个 | defer setUpdating(false)（当前 f return 前） |
| **典型场景** | Ctrl+R 与 2s tick 撞车 → 丢 | A/B/C/D 四条路径各自独立入队 → 排 | 连续排 2+ 个 f → 设计跳 实际全跑 |

---

## 六、边界关系总表

| 阶段 | 边界位置 | 保障手段 | 失败/异常处理 |
|------|---------|---------|--------------|
| **刷新发起频率** | `updater time.After(rate)` + `refreshRate` | 定时器间隔（拉模式） | 指数退避，最长 2min 退出 |
| **refresh 并发控制（丢）** | `internal/model/table.go#L229-L247` 首行 | CAS 原子锁 `inUpdate` | 后到的 refresh 直接 return nil，**不 fire 任何 listener** |
| **QueueUpdateDraw 封装（防阻塞）** | `internal/ui/app.go#L75-L82` | 每次调启动临时 goroutine，send drawChan 不阻塞调用方 | drawChan send 在临时 goroutine 里等待 |
| **界面回调入队（排）** | `internal/view/browser.go` 4 处调用点 | tview draw 队列 FIFO + 主循环 goroutine 串行执行 | **全部入队，不会丢弃**（除非 App 退出） |
| **DAO 数据源选择** | `resourceMeta(gvr)` | Registry 三层 fallback（明确 DAO → nil→Resource → Table） | 未注册资源默认 HTTP Table |
| **Inform 缓存未就绪** | `Browser.TableNoData` | `HasSynced()` 检查 | 显示 Synchronizing 而非误报 |
| **增量 Delta 计算** | `TableData.Update` | 按 ID 匹配 + DeltaRow | 无则 Add、有变 Update、无变 Unchanged、缺失 Delete |
| **时间列防抖动** | `NewDeltaRow` | `h.IsTimeCol(i)` 跳过 | AGE 等变化不计入 Delta |
| **绘制期间互斥（updating）** | Browser 3 个 QueueUpdateDraw 回调首行 | RWMutex + bool | 当前实现下**实际不跳过**（见 5.4.3），为防御性代码 |
| **UpdateUI 全量重建** | `ui/table.go#L472-L505` | Clear + AddHeader + buildRow + 双缓冲 Draw | tview 主线程安全，用户看不到中间态 |

---

## 七、常见疑问的代码定位

**Q1：刷新时是不是每次都打 API Server？**
- 不是。走 `dao.Resource` 的资源读 Informer 本地缓存（`watch/factory.go#L75-L99`）。只有 `dao.Table`（HTTP Table）和 `dao.Generic`（dynamic client）才发网络请求。

**Q2：AGE 列每秒都变，为什么不会整行高亮闪烁？**
- Delta 计算时显式跳过了时间列（`model1/delta.go#L18` 的 `!h.IsTimeCol(i)` 条件）。时间相关列定义在各 Renderer.Header 的 `Attrs.Time=true`。

**Q3：用户快速切命名空间为什么不会卡？**
- `Browser.Stop()` 调 `cancelFn()` 终止旧 Watcher goroutine（`view/browser.go#L190-L200`），然后 `Start()` 启动新的。旧 goroutine 因为 `ctx.Done()` 会退出，不会继续刷新。

**Q4：手动 refresh（Ctrl+R）和定时 refresh 有什么不同？**
- 手动走 `refreshCmd → b.refresh() → b.Start()`，是重走整个 Watch 流程（会重建 context/cancel），定时只在 updater goroutine 内循环调 `t.refresh(ctx)`。

**Q5：UpdateUI 全量重建会不会导致闪烁？**
- tview 的 Draw 是双缓冲，更新完统一 `Sync()` 到终端；实际只有 C1 和 A1 两次独立绘制，不是每秒 1 次，用户感知不到"清空→重建"过程。

**Q6：`updating` 标志当前实现不跳过任何 UpdateUI，那它是不是多余代码？**
- 不是多余代码，而是**防御性编程**：
  - RWMutex 保证即使未来有**其他 goroutine**（非 tview 主循环）直接读写 `updating` 也是线程安全的。
  - 标志本身代表了"设计意图"——期望同一时刻只进行一次绘制流程。当前代码因为所有 f() 在同一个 goroutine 串行，意图天然被满足；但如果未来重构（比如去掉 k9s 的 `go func()` 嵌套、或者引入并行渲染路径），这层保护就会真正生效。

**Q7：真正防重/节流的机制是哪个？**
- 主要靠两层：
  1. **CAS 原子锁 `inUpdate`**（`internal/model/table.go#L229`）：在数据层丢弃并行的 `refresh()` 请求，省掉了 DAO.List + Delta 计算的浪费。
  2. **定时器 `time.After(rate)`**（`internal/model/table.go#L66`）：在最源头控制刷新频率（默认 2s），从根本上限制进入 `refresh()` 的次数。
- `updating` 是最外层的"设计意图表达"，在当前代码路径上实际上不产生节流效果。
