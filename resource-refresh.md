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

## 五、界面重绘：三层机制的本质差异

界面重绘涉及 **三种完全不同层级的机制**，之前容易混淆的是"丢弃"和"排队"语义，本节逐一对齐源码讲清各自的边界。

### 5.1 三者定位总览

先从代码位置和目的两个维度区分：

| 机制 | 所在文件 | 作用层 | 目的 | 语义 |
|------|---------|-------|------|------|
| ① **刷新并发控制**（CAS `inUpdate`） | `internal/model/table.go#L229-L247` | Model 层 | 防止同一份数据同时被拉取 + 计算 Delta | 尝试并行 → **直接丢弃** |
| ② **界面回调入队**（`QueueUpdateDraw`） | tview.Application 方法 | 框架层 | 跨线程安全地把 UI 操作送到主循环 | 无论多少，**都进队列串行执行** |
| ③ **绘制期间互斥**（Browser `updating`） | `internal/view/browser.go#L36-L65` | View 层 | 防止多条独立触发路径重复执行 `UpdateUI` | 有回调在跑 → **本轮跳过** |

三者是**逐层传递**关系：① 通过后才会触发 listener；listener 里调用 ② 入队；队列回调里用 ③ 判断是否跳过。

### 5.2 机制① 刷新并发控制（CAS inUpdate）：数据层防重入

`internal/model/table.go#L229-L247`
```go
func (t *Table) refresh(ctx context.Context) error {
    // 在数据拉取 + Delta 计算的入口处，用原子锁防并发
    if !atomic.CompareAndSwapInt32(&t.inUpdate, 0, 1) {
        slog.Debug("Dropping update...")   // ★ 直接丢弃，什么都不做
        return nil
    }
    defer atomic.StoreInt32(&t.inUpdate, 0)

    if err := t.reconcile(ctx); err != nil {  // DAO.List + Render + Update(Delta)
        return err
    }
    data := t.Peek()
    if data.RowCount() == 0 {
        t.fireNoData(data)   // 通知 listener（可能不止一个 Browser）
    } else {
        t.fireTableChanged(data)
    }
    return nil
}
```

**边界和语义：**
- 位置：**在 goroutine 中、数据拉取之前**
- 语义：尝试并行调用 `refresh()` → **静默丢弃后到的请求**（不等，不落队列，直接返回 nil）
- 场景：用户在定时刷新 tick 之前按 Ctrl+R 重跑 Start，刚好 2s 的 tick 也到了，就会出现两次 refresh 同时竞争 CAS

### 5.3 机制② 界面回调入队（QueueUpdateDraw）：线程安全 + 严格串行

`QueueUpdateDraw` 是 **tview.Application** 的标准方法（k9s 不自己实现），语义是：

> 把传入的闭包**压入 UI 主循环的事件队列尾部**，tview 下一次处理事件时会从队头依次取出**一个一个执行**。

调用点有三类独立路径（都在 `internal/view/browser.go`）：

| 触发源 | 位置 | 何时发生 |
|--------|------|---------|
| A. `TableDataChanged` | `#L338-L365` | 定时 refresh 成功，CAS 通过后 fire → listener |
| B. `TableNoData` | `#L298-L335` | refresh 结果为空，同上路径 |
| C. `BufferActive` | `#L228-L251` | 用户输入过滤器后回车，退出搜索模式时 |
| D. `TableLoadFailed` | `#L368-L373` | refresh 失败超过退避阈值 |

**关键：每一路径独立调用 QueueUpdateDraw，每次调用都产生一个新的闭包入队。入队本身永不丢弃，一定执行（除非 App 退出）。**

### 5.4 机制③ 绘制期间互斥（Browser.updating）：串行回调的逻辑去重

`internal/view/browser.go#L55-L65` — 读写锁保护的标志：
```go
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

三条独立路径的回调内部结构都相同（以 TableDataChanged 为例）：

`internal/view/browser.go#L338-L365`
```go
func (b *Browser) TableDataChanged(mdata *model1.TableData) {
    // 前置计算（在 fire 的 goroutine 里执行，不占主线程）
    cdata := b.Update(mdata, b.app.Conn().HasMetrics())

    b.app.QueueUpdateDraw(func() {
        // ★ 在主线程真正开始执行回调的第一时间检查
        if b.getUpdating() {
            return   // 已有另一条路径的绘制正在进行 → 本轮跳过
        }
        b.setUpdating(true)
        defer b.setUpdating(false)

        b.refreshActions()
        b.UpdateUI(cdata, mdata)   // Clear + 逐行 buildRow
    })
}
```

`BufferActive`（路径 C）的回调同样做了这件事（`#L240-L250`）：
```go
b.app.QueueUpdateDraw(func() {
    if b.getUpdating() { return }
    b.setUpdating(true)
    defer b.setUpdating(false)
    b.UpdateUI(cdata, mdata)
})
```

### 5.5 时序例证：三条路径在 1ms 内先后触发的场景

用一个最容易混淆的场景（用户刚输完 filter 回车，同时定时 tick 也到了）说明三者协作：

```
时间轴（T 为毫秒）：
  T=0   updater goroutine 到达 2s tick，调 refresh() → CAS 通过（inUpdate=1）
  T=0~5 refresh() 内部：DAO.List → Render → Delta 计算（假设耗时 5ms，此时 inUpdate=1）
  T=2   用户按回车退出 filter，BufferActive() 被调用：
          ├─ 先调 model.Refresh(ctx) → refresh() 检查 CAS → inUpdate 已经是 1
          │     → 直接 return nil（被机制① 丢弃） ← 此时没任何 UI 变动
          ├─ 再调 model.Peek() 拿当前数据（仍是 T=0 开始前的旧快照）
          └─ b.Update(mdata) 算过滤
          └─ QueueUpdateDraw(闭包 C1) 入队 ← 机制② 已接收，稍后会执行
  T=5   refresh() 完成，Peek() 拿新快照，fireTableChanged：
          └─ listener TableDataChanged 被调：
              ├─ b.Update(mdata) 算过滤（基于新数据）
              └─ QueueUpdateDraw(闭包 A1) 入队 ← 机制② 又接收一个
  T=6   tview 主循环取出队头 → 执行 闭包 C1：
          ├─ getUpdating() → false
          ├─ setUpdating(true)
          ├─ UpdateUI(旧快照)  →  用旧数据画出界面（filter 已生效，基于旧 snapshot）
          └─ defer setUpdating(false)  ← 退出时重置
  T=12  tview 主循环取下一个 → 执行 闭包 A1：
          ├─ getUpdating() → false（C1 已经跑完，释放了）
          ├─ setUpdating(true)
          └─ UpdateUI(新快照)  →  画出最终正确界面，filter + 新数据同时生效
```

**如果没有 `updating` 标志**，在更快的时序下：C1 正在执行 `UpdateUI`（主线程里跑 Clear + buildRow），此时 tview 同一个主线程不可能同时执行 A1，所以其实不会"并发操作 tview 控件"。那 `updating` 到底防什么？

它防的是下面这种 **逻辑上的重复绘制浪费**（注意 tview 回调虽然串行，但可以连续排多个）：

```
T=0   BufferActive 入队 C1（基于旧 snapshot）
T=1   TableDataChanged 入队 A1（基于新 snapshot）
T=2   TableNoData 又入队 B1（另一个命名空间切换触发）
```

三个闭包会被 tview **一个接一个地连续执行**。如果没有 updating，就是 3 次 Clear + 重建。有了 updating：
- C1 执行时 `updating=true`
- 紧跟着执行 A1 时，检查发现 `updating=true` → **return，跳过这次重绘**
- B1 同理也会被跳过
- 最终用户只看到一次绘制（C1 的结果），但 C1 之后 A1 的数据其实是更新的——等等，这里被跳过不是会丢数据吗？

这就是为什么 **`updating` 的语义不是"保留最新"，而是"避免短时间内连续重绘"**。它要求业务上接受"第一次绘制的结果在一段时间内就是最终展示"。对于 2 秒刷新周期来说，这个牺牲是可接受的（下一个周期会补回最新数据）。

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
    ComputeMaxColumns(pads, ..., cdata)    // 算列宽
    cdata.RowsRange(func(row int, re model1.RowEvent) bool {
        ore, _ := data.FindRow(re.Row.ID)  // 从原表找 Delta 信息
        t.buildRow(row+1, re, ore, cdata.Header(), pads)
        return true
    })
    t.updateSelection(true)                // 重定位选中行
    t.UpdateTitle()                        // 刷新标题栏计数
}
```

> **关键点**：
> - `cdata` = 过滤后展示用的 TableData（经过 Browser.filtered），`data` = 原始数据（用于查 Delta）
> - 每次都是 **Clear 后全量重建**，没有 DOM diff；但由于是 tcell 内存单元格操作 + tview 双缓冲 Draw，用户不会感知"清空→重建"的过程
> - `cdata.Sort()` 是就地排序，`RowsRange` 按排序后顺序遍历

### 5.7 三者语义差异对比表

| 对比维度 | ① 刷新并发控制 | ② 回调入队 | ③ 绘制互斥 |
|---------|---------------|-----------|-----------|
| **处理对象** | `refresh()` 函数调用 | 任意 UI 操作闭包 | `UpdateUI()` 调用 |
| **所在线程** | 任意 goroutine（通常是 updater） | 任意 goroutine 入队 → 主线程出队执行 | 主线程 |
| **冲突策略** | **丢**（后到者直接 return nil） | **排队**（都进队列，一个一个跑） | **跳**（当前正在跑就 return，本轮不跑） |
| **是否保存最新** | 不保存，靠下一轮 tick 补 | 全部保存，严格按入队顺序 | 不保存，被跳过的 UpdateUI 不会再执行 |
| **代码位置** | model.Table.refresh 首行 | app.QueueUpdateDraw 调用点 | Browser 各 QueueUpdateDraw 闭包首行 |
| **可重入释放时机** | defer atomic.Store（reconcile 全跑完） | 闭包 return（队列 FIFO 下一个） | defer setUpdating(false)（当前 UpdateUI 跑完） |
| **典型冲突场景** | 用户 Ctrl+R 与 2s tick 撞车 | 三条独立路径各自调用 | 回调连续串行执行时的逻辑重复绘制 |

---

## 六、边界关系总表

| 阶段 | 边界位置 | 保障手段 | 失败/异常处理 |
|------|---------|---------|--------------|
| **刷新发起频率** | `updater time.After(rate)` + `refreshRate` | 定时器间隔（拉模式） | 指数退避，最长 2min 退出 |
| **刷新并发控制（丢）** | `refresh()` 首行 | CAS 原子锁 `inUpdate` | 后到的 refresh 直接 return nil 丢弃 |
| **DAO 数据源选择** | `resourceMeta(gvr)` | Registry 三层 fallback（明确 DAO → nil→Resource → Table） | 未注册资源默认 HTTP Table |
| **Inform 缓存未就绪** | `Browser.TableNoData` | `HasSynced()` 检查 | 显示 Synchronizing 而非误报 |
| **增量 Delta 计算** | `TableData.Update` | 按 ID 匹配 + DeltaRow | 无则 Add、有变 Update、无变 Unchanged、缺失 Delete |
| **时间列防抖动** | `NewDeltaRow` | `h.IsTimeCol(i)` 跳过 | AGE 等变化不计入 Delta |
| **界面回调入队（排）** | Browser 三处独立调用点 | `app.QueueUpdateDraw` 入 FIFO 队列 | 永远入队，tview 主线程按序出队 |
| **绘制期间互斥（跳）** | 各 QueueUpdateDraw 回调首行 | `b.updating` 读写锁标志 | 已有绘制在执行时本轮 return 跳过 |
| **UI 渲染串行** | `UpdateUI` 整体 | 仅在 QueueUpdateDraw 回调主线程中执行 | tview 双缓冲，操作控件线程安全 |

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
- tview 的 Draw 是双缓冲，更新完统一 `Sync()` 到终端；外层还有 `updating` 标志 + 2s 刷新间隔确保不会每秒重建。

**Q6：三层 UI 机制（CAS/入队/互斥）合并成一层不够吗？为什么要分层？**
- 不够，因为三者解决的问题域不同：
  - **CAS 不能替代 QueueUpdateDraw**：前者在数据层，不知道 UI 线程的存在；tview 控件不能在任意 goroutine 操作，所以必须有 QueueUpdateDraw 切到主线程。
  - **QueueUpdateDraw 不能替代 updating**：入队是"全部保留按顺序跑"，如果 1ms 内三条路径各入队一次，就是 3 次连续重绘，浪费 CPU。updating 相当于在回调里再叠加一层"节流"——保留第一次，后两次跳过。
  - **仅用 updating 又不能替代 CAS**：refresh() 的瓶颈在 DAO.List + Delta 计算（可能跨网络），不等这一步结束就直接丢弃，比"算完了进了队列再跳过"省得多。
- 三者合起来是一个 **丢弃→排队→跳过** 的多级漏斗，越在前面丢弃，越节省后面的算力。
