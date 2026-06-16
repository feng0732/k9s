# K9s 资源表格刷新与节流机制分析

## 整体架构概览

K9s 的资源表格刷新机制采用 **Model-View** 分层架构，由三个核心层协作完成：

```
┌─────────────────────────────────────────────────────────┐
│  View 层 (Browser → view.Table → ui.Table)              │
│  负责接收数据变更通知，在主线程安全地更新 UI               │
├─────────────────────────────────────────────────────────┤
│  Model 层 (model.Table → model1.TableData)              │
│  负责定时拉取数据、计算行级增量差异、分发变更事件          │
├─────────────────────────────────────────────────────────┤
│  Data 层 (watch.Factory → Informer → Kubernetes API)    │
│  负责通过 Informer 缓存提供数据源                        │
└─────────────────────────────────────────────────────────┘
```

---

## 一、刷新时机：什么时候刷新？

### 1.1 定时轮询（核心刷新路径）

表格的核心刷新由 [model.Table.updater](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model/table.go#L203-L227) 中的 goroutine 驱动：

```go
func (t *Table) updater(ctx context.Context) {
    bf := backoff.NewExponentialBackOff()
    bf.InitialInterval, bf.MaxElapsedTime = initRefreshRate, maxReaderRetryInterval
    rate := initRefreshRate  // 首次 300ms 快速刷新
    for {
        select {
        case <-ctx.Done():
            return
        case <-time.After(rate):
            rate = t.refreshRate  // 之后切换为用户配置的刷新间隔
            err := backoff.Retry(func() error {
                if err := t.refresh(ctx); err != nil {
                    return err
                }
                return nil
            }, backoff.WithContext(bf, ctx))
            // ...
        }
    }
}
```

**关键点：**
- **首次刷新间隔**：`initRefreshRate = 300ms`（[table.go:26](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model/table.go#L26)），让用户尽快看到数据
- **后续刷新间隔**：`t.refreshRate`，由配置决定，默认 **2 秒**（[types.go:7](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/config/types.go#L7)）
- **刷新失败退避**：使用指数退避（Exponential Backoff），`MaxElapsedTime = 2 分钟`（[model/types.go:21](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model/types.go#L21)）

### 1.2 刷新速率的配置来源

刷新速率 `refreshRate` 由以下路径注入：

1. **配置文件**：`k9s.yaml` 中的 `refreshRate` 字段，默认 `2`（秒）
2. **命令行参数**：`--refresh-rate` 覆盖配置文件
3. **最低限制**：不允许低于 `DefaultRefreshRate = 2.0` 秒（[flags.go:8](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/config/flags.go#L8)）

配置链路：
```
K9s.RefreshRate (yaml)
  → K9s.GetRefreshRate()         // 合并命令行覆盖
    → K9s.RefreshDuration()      // 转为 time.Duration
      → model.Table.SetRefreshRate()  // 设置到 model
```

在 [view/table.go:66](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/table.go#L66) 和 [browser.go:116](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/browser.go#L116) 中初始化：

```go
t.GetModel().SetRefreshRate(t.app.Config.K9s.RefreshDuration())
```

### 1.3 手动刷新触发

除定时刷新外，用户可通过以下方式主动触发：

- **Ctrl+R**：调用 `refreshCmd`（[browser.go:478](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/browser.go#L478)），实质是调用 `b.refresh()` → `b.Start()`，重新启动整个 Watch 循环
- **切换命名空间**：`switchNamespaceCmd`（[browser.go:587](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/browser.go#L587)）
- **过滤器变更**：`BufferActive` 回调（[browser.go:228](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/browser.go#L228)），当用户退出搜索模式时触发 `model.Refresh()`

---

## 二、节流防抖：怎样避免高频抖动？

K9s 采用 **四层防抖机制** 来避免 UI 高频抖动：

### 2.1 第一层：原子锁 — 丢弃并发刷新请求

[model.Table.refresh](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model/table.go#L229-L247) 使用 CAS 原子操作防止并发刷新：

```go
func (t *Table) refresh(ctx context.Context) error {
    if !atomic.CompareAndSwapInt32(&t.inUpdate, 0, 1) {
        slog.Debug("Dropping update...")
        return nil  // 正在更新中，直接丢弃本次请求
    }
    defer atomic.StoreInt32(&t.inUpdate, 0)
    // ... 执行实际刷新逻辑
}
```

- `inUpdate` 是 `int32` 类型原子变量
- 如果上一次 `refresh` 尚未完成，新的刷新请求会被**直接丢弃**
- 这保证同一时刻只有一个 `refresh` 在执行，避免数据竞争和重复渲染

### 2.2 第二层：定时器间隔 — 控制刷新频率

在 `updater` 的 `select` 循环中：

```go
case <-time.After(rate):
    rate = t.refreshRate  // 切换到配置的刷新间隔
```

- 每次循环结束后等待 `refreshRate` 时间（默认 2 秒）才进行下一次刷新
- 不是"数据一变就刷"，而是"每隔 N 秒拉一次"
- 这是一种 **拉模式（Pull）**，天然具有节流效果

### 2.3 第三层：指数退避 — 失败时逐步降低刷新压力

```go
bf := backoff.NewExponentialBackOff()
bf.InitialInterval = initRefreshRate     // 300ms
bf.MaxElapsedTime = maxReaderRetryInterval  // 2 分钟
```

当 `refresh` 失败时：
- 不立即重试，而是按指数间隔逐步增加重试间隔
- 避免在 API Server 不可用时产生大量无效请求
- 如果持续失败超过 2 分钟，`updater` 退出并通知 `TableLoadFailed`

### 2.4 第四层：UI 更新互斥锁 — 防止并发渲染

在 [Browser](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/view/browser.go#L36-L46) 层，`TableDataChanged` 和 `TableNoData` 都使用 `updating` 互斥标志：

```go
func (b *Browser) TableDataChanged(mdata *model1.TableData) {
    cdata := b.Update(mdata, b.app.Conn().HasMetrics())
    b.app.QueueUpdateDraw(func() {
        if b.getUpdating() {  // 如果正在更新，跳过
            return
        }
        b.setUpdating(true)
        defer b.setUpdating(false)
        b.refreshActions()
        b.UpdateUI(cdata, mdata)
    })
}
```

- `QueueUpdateDraw` 确保渲染逻辑在 tview 主循环中执行（线程安全）
- `updating` 标志防止多次 `QueueUpdateDraw` 回调并发执行 UI 更新
- 即使 model 频繁通知，UI 也只会一个一个地串行处理

---

## 三、数据流转全链路

### 3.1 完整刷新流程

```
1. updater goroutine 定时触发
       ↓
2. refresh() — CAS 获取锁（防并发）
       ↓
3. reconcile() — 调用 DAO.List() 从 Informer 缓存读取数据
       ↓
4. TableData.Render() — 将 []runtime.Object 渲染为 Rows
       ↓
5. TableData.Update(rows) — 计算行级增量 Delta
       ↓
6. fireTableChanged(data) / fireNoData(data) — 通知 Listener
       ↓
7. Browser.TableDataChanged() — View 层回调
       ↓
8. b.Update(data) → filtered() → doUpdate() — 过滤、排序
       ↓
9. app.QueueUpdateDraw() — 投递到主线程
       ↓
10. UpdateUI(cdata, mdata) — 实际渲染到 tview.Table
```

### 3.2 增量计算机制

[model1.TableData.Update](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model1/table_data.go#L425-L457) 负责计算新旧数据的差异：

```go
func (t *TableData) Update(rows Rows) {
    empty := t.Empty()
    kk := sets.New[string]()
    for _, row := range rows {
        kk.Insert(row.ID)
        if empty {
            t.rowEvents.Add(NewRowEvent(EventAdd, row))  // 首次全量添加
            continue
        }
        if index, ok := t.rowEvents.FindIndex(row.ID); ok {
            // 已存在：计算 Delta
            ev, _ := t.rowEvents.At(index)
            delta := NewDeltaRow(ev.Row, row, t.header)
            if delta.IsBlank() {
                ev.Kind = EventUnchanged  // 无变化
            } else {
                t.rowEvents.Set(index, NewRowEventWithDeltas(row, delta))  // 有更新
            }
        } else {
            t.rowEvents.Add(NewRowEvent(EventAdd, row))  // 新增行
        }
    }
    if !empty {
        t.Delete(kk)  // 删除不再存在的行
    }
}
```

**DeltaRow**（[delta.go:12-24](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/model1/delta.go#L12-L24)）只记录真正变化的字段：

```go
func NewDeltaRow(o, n Row, h Header) DeltaRow {
    deltas := make(DeltaRow, len(o.Fields))
    for i, old := range o.Fields {
        if old != "" && old != n.Fields[i] && !h.IsTimeCol(i) {
            deltas[i] = old  // 仅记录旧值（非时间列变化）
        }
    }
    return deltas
}
```

- **时间列不参与 Delta**：`AGE` 等时间列每秒都在变，如果纳入 Delta 会导致整行被标记为"已更新"，产生视觉抖动
- Delta 信息用于 UI 层显示变化高亮（如 `Deltas()` 函数添加变化标记符号）

---

## 四、Informer 缓存与数据源

### 4.1 数据不直接来自 API Server

[watch.Factory](file:///d:/f:/0601-2/solo-dogfeeding/code/1-k9s/internal/watch/factory.go#L29-L35) 使用 Kubernetes Dynamic SharedInformerFactory：

```go
f.factories[ns] = di.NewFilteredDynamicSharedInformerFactory(
    dial,
    defaultResync,  // 10 分钟全量重同步
    ns,
    nil,
)
```

- `defaultResync = 10 * time.Minute`：Informer 每 10 分钟全量重同步一次
- `refresh` 操作实际是读取 Informer 的本地缓存，不是直接请求 API Server
- 这意味着 K9s 的刷新频率（2 秒）与 API Server 的负载无关

### 4.2 缓存同步等待

在首次获取数据时，[Factory.List](file:///d:/fz/0601-2/solo-dogfeeding/code/1-k9s/internal/watch/factory.go#L75-L99) 会等待缓存同步：

```go
if !wait || (wait && inf.Informer().HasSynced()) {
    return oo, err
}
f.waitForCacheSync(ns)  // 最多等 500ms
```

- `defaultWaitTime = 500ms`：缓存同步的最长等待时间
- 如果缓存尚未同步完成，`HasSynced` 检查会在 `TableNoData` 中用于避免误报"无资源"警告

---

## 五、总结：四层防抖的协作关系

| 层次 | 机制 | 位置 | 效果 |
|------|------|------|------|
| 1 | 原子锁 `inUpdate` | model.Table.refresh | 丢弃并发刷新请求 |
| 2 | 定时间隔 `refreshRate` | model.Table.updater | 控制拉取频率（默认 2s） |
| 3 | 指数退避 Backoff | model.Table.updater | 失败时逐步降频 |
| 4 | UI 互斥 `updating` | Browser.TableDataChanged | 防止并发渲染 |

**核心设计理念**：K9s 不使用 Watch 事件驱动的推送模式，而是采用 **定时拉取 + 增量计算** 的模式。这种设计简化了数据流，同时通过多层防抖机制确保即使在高频变更场景下，UI 也不会产生抖动。Informer 缓存作为中间层，使得 2 秒一次的拉取操作成本极低（本地内存读取），不会对 API Server 造成压力。
