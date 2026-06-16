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

## 五、界面重绘：从通知到像素，两层串行保障

### 5.1 三层 Listener 回调链

从 Model 到 UI 的通知链路：

```
model.Table.fireTableChanged(data)           （clone 后的 TableData 快照）
    ↓ 遍历 listeners 调用 TableDataChanged()
view.Browser.TableDataChanged(mdata)          （internal/view/browser.go#L338-L365）
    ↓ 1. b.Update() → filtered + doUpdate()   （过滤、排序列计算）
    ↓ 2. app.QueueUpdateDraw(func(){...})     （投递到 tview 主循环）
        ui.Table.UpdateUI(cdata, data)        （internal/ui/table.go#L472-L505）
            ↓ t.Clear() + 逐行 buildRow()     （全量重建 tview 单元格）
```

### 5.2 第一层串行：Browser.updating 互斥标志

`internal/view/browser.go#L338-L365`
```go
func (b *Browser) TableDataChanged(mdata *model1.TableData) {
    cdata := b.Update(mdata, b.app.Conn().HasMetrics())  // 在 goroutine 内先算过滤排序

    // 投递到 tview 主线程（QueueUpdateDraw 是线程安全的入队）
    b.app.QueueUpdateDraw(func() {
        // 边界：如果上一次 QueueUpdateDraw 的回调尚未跑完，跳过
        if b.getUpdating() {
            return
        }
        b.setUpdating(true)
        defer b.setUpdating(false)

        // 真正操作 tview 控件（必须在主线程）
        b.refreshActions()
        b.UpdateUI(cdata, mdata)
    })
}
```

> **要点**：`QueueUpdateDraw` 本身是入队操作（串行排队执行），`updating` 又提供了一层"去重"——如果队列中还有没跑完的绘制，新的就丢弃。这两层叠加确保 UI 不会并发操作 tview 树。

### 5.3 第二层串行：UpdateUI 全量重建单元格

`internal/ui/table.go#L472-L505`
```go
func (t *Table) UpdateUI(cdata, data *model1.TableData) {
    t.Clear()                              // 清空所有单元格
    // 重建 header
    col := 0
    for _, h := range cdata.Header() {
        if t.shouldExcludeColumn(h) { continue }
        t.AddHeaderCell(col, h)
        col++
    }
    cdata.Sort(t.getSortCol())             // 先排序
    ComputeMaxColumns(pads, ..., cdata)    // 算对齐用的列宽
    // 逐行构建
    cdata.RowsRange(func(row int, re model1.RowEvent) bool {
        ore, _ := data.FindRow(re.Row.ID)  // 找原表（未过滤）中的 delta 信息
        t.buildRow(row+1, re, ore, cdata.Header(), pads)
        return true
    })
    t.updateSelection(true)                // 重新定位选中行
    t.UpdateTitle()                        // 更新标题计数
}
```

> **关键边界**：
> - `cdata` 是过滤后的数据（用于展示行），`data` 是原始数据（用于找 Delta）
> - `UpdateUI` 每次都是 **Clear 后全量重建**，没有 diff-dom；但因为是本地重建 tcell 单元格且有前面的节奏控制，性能可接受
> - `cdata.Sort()` 是就地排序，会影响 `rowEvents.index`，`RowsRange` 以排序后顺序遍历

---

## 六、边界关系总表

| 阶段 | 边界位置 | 保障手段 | 失败/异常处理 |
|------|---------|---------|--------------|
| **刷新发起频率** | `updater time.After(rate)` + `refreshRate` | 定时器间隔（拉模式） | 指数退避，最长 2min 退出 |
| **refresh 并发控制** | `refresh()` 首行 | CAS 原子锁 `inUpdate` | 并发请求静默丢弃 |
| **DAO 数据源选择** | `resourceMeta(gvr)` | Registry 三层 fallback（明确 DAO → nil→Resource → Table） | 未注册资源默认 HTTP Table |
| **Inform 缓存未就绪** | `Browser.TableNoData` | `HasSynced()` 检查 | 显示 Synchronizing 而非误报 |
| **增量 Delta 计算** | `TableData.Update` | 按 ID 匹配 + DeltaRow | 无则 Add、有变 Update、无变 Unchanged、缺失 Delete |
| **时间列防抖动** | `NewDeltaRow` | `h.IsTimeCol(i)` 跳过 | AGE 等变化不计入 Delta |
| **UI 回调并发** | `Browser.TableDataChanged` | `QueueUpdateDraw` 入队 + `updating` 标志 | 队列中已有绘制时跳过 |
| **UI 渲染线程** | `UpdateUI` 整体 | 全在 `QueueUpdateDraw` 回调内 | tview 主线程安全 |

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
