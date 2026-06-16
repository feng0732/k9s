# K9s Pod 日志流机制深度解析

本文档从代码层面分析 K9s 中 Pod 日志拉取的持续展示、多容器切换和缓冲窗口机制。

---

## 一、整体架构概览

日志系统采用典型的 **Model-View 分层架构**，分为三层：

| 层级 | 职责 | 核心文件 |
|------|------|----------|
| DAO 层 | 与 Kubernetes API 交互，负责真正的流式日志读取和重试 | [pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go) |
| Model 层 | 日志缓冲、过滤、事件通知，作为 View 和 DAO 之间的桥梁 | [log.go](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go) |
| View 层 | UI 展示、用户交互、自动滚动控制 | [log.go](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go) |

数据流向：
```
Kubernetes API Stream
    ↓ (bufio.ReadLine)
DAO: readLogs() → LogChan channel
    ↓ (goroutine 消费)
Model: Log.lines (LogItems 环形缓冲)
    ↓ (fireLogChanged 事件通知)
View: Log.Flush() → ansiWriter 写入 TextView
```

---

## 二、流式读取：从 K8s API 到日志通道

### 2.1 入口：TailLogs

定义在 [pod.go#L213-L256](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go#L213-L256)，是整个日志流的起点。

核心逻辑：
1. 先获取 Pod 实例，统计容器数量（Init + 普通 + Ephemeral）
2. 若只有 1 个容器，标记 `SingleContainer = true`
3. 根据条件决定启动几个日志流 goroutine：
   - 有默认容器注解且非 AllContainers 模式 → 只拉默认容器
   - 指定了 Container 且非 AllContainers 模式 → 只拉指定容器
   - 否则 → 遍历 InitContainers / Containers / EphemeralContainers，每个容器调用一次 `tailLogs()`

```go
// 关键判断分支
if co, ok := GetDefaultContainer(&po.ObjectMeta, &po.Spec); ok && !opts.AllContainers {
    // 只有默认容器
    return append(outs, tailLogs(ctx, p, opts)), nil
}
if opts.HasContainer() && !opts.AllContainers {
    // 指定了容器
    return append(outs, tailLogs(ctx, p, opts)), nil
}
// 否则所有容器都起一个 goroutine
for i := range po.Spec.InitContainers { ... }
for i := range po.Spec.Containers { ... }
for i := range po.Spec.EphemeralContainers { ... }
```

### 2.2 带重试的流式读取：tailLogs

定义在 [pod.go#L357-L474](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go#L357-L474)。

这是最核心的函数，包含 **指数退避重试机制**：

- 最多重试 `logRetryCount = 20` 次
- 初始退避 `500ms`，最大 `30s`，使用 `cenkalti/backoff/v4` 指数退避
- 每次失败后先检查 Pod 状态（是否 Terminating / Succeeded / Failed），若已终止则停止重试

每个容器对应一个 goroutine，执行流程：

```
for 重试次数 < 20:
    1. 调用 logger.Logs() 构造 restclient.Request
    2. 调用 req.Stream(ctx) 获取 io.ReadCloser
    3. 调用 readLogs() 持续读取直到 EOF/错误/取消
    4. 根据 readLogs 返回值决定下一步：
        - streamEOF: 正常结束，直接 return
        - streamError: 检查 Pod 状态，退避后重试
        - streamCanceled: 上下文取消，直接 return
```

### 2.3 实际读取：readLogs

定义在 [pod.go#L476-L533](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go#L476-L533)。

使用 `bufio.NewReader(stream)` 按行读取，每行以 `\n` 分隔。

**通道背压处理（关键设计）**：

```go
select {
case <-ctx.Done():
    return streamCanceled
case out <- item:
    // 正常写入
default:
    // 通道满了，丢弃日志行
    droppedLines++
    if droppedLines == 1 || droppedLines%100 == 0 {
        slog.Warn("Dropping log lines due to slow consumer", ...)
    }
}
```

- 通道缓冲区大小：`logChannelBuffer = 50`，可通过 `LogOptions.LogBufferSize` 自定义
- 使用 `select + default` 非阻塞写入，消费者慢时主动丢弃而非阻塞生产者
- 丢弃第 1 条和每 100 条记录警告日志，避免日志刷屏

**EOF 处理**：遇到 `io.EOF` 时，如果还有残余字节（尾部不完整行），先 emit 出去，再发送一条错误 LogItem，最后返回 `streamEOF`。

### 2.4 Head 模式 vs Tail 模式

`ToPodLogOptions()`（[log_options.go#L83-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_options.go#L83-L115)）根据 `LogOptions` 生成 Kubernetes 实际的 `PodLogOptions`，是回看逻辑的核心转换：

| 模式 | Head | SinceSeconds | Follow | TailLines | LimitBytes | 含义 |
|------|------|--------------|--------|-----------|------------|------|
| Tail | false | -1 | true | &Lines | nil | 持续跟随最新日志（默认） |
| Head | true | 0 | false | nil | 5000 | 只拉日志开头 5000 字节，不拉新日志 |
| 时间范围 | false | >0 | true | &Lines | nil | 拉最近 N 秒的历史日志，之后继续跟随 |
| 断点续传 | false | 0 | true | &Lines | nil | 从 SinceTime 时间戳开始拉 |

```go
// Head 模式关键转换
if o.Head {
    var maxBytes int64 = 5000
    opts.Follow = false
    opts.TailLines, opts.SinceSeconds, opts.SinceTime = nil, nil, nil
    opts.LimitBytes = &maxBytes
    return &opts
}

// SinceSeconds > 0 时间范围
if o.SinceSeconds != 0 {
    opts.SinceSeconds, opts.SinceTime = &o.SinceSeconds, nil
    return &opts
}

// SinceTime 断点续传
if o.SinceTime != "" {
    if t, err := time.Parse(time.RFC3339, o.SinceTime); err == nil {
        opts.SinceTime = &metav1.Time{Time: t.Add(time.Second)}
    }
}
```

**Head 模式流自动停止机制**：
- `Follow=false` 时，Kubernetes API 不会保持连接，读取完历史数据后立即返回 EOF
- `readLogs()` 收到 `io.EOF` → emit 残余行 → 返回 `streamEOF`
- `tailLogs()` goroutine 收到 `streamEOF` 后直接 return，不重试
- 最终 LogChan 被 close，Model 层 `updateLogs()` 读到 channel 关闭后退出循环

**⚠️ 代码事实校准点 1：Head 结束显示的是错误项而非结束提示**

真实事件链：
1. `readLogs()`（[pod.go#L516-L524](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go#L516-L524)）遇到 io.EOF 时，除了 emit 残余行，还会主动发送一条 **错误 LogItem**：
   ```go
   out <- opts.ToErrLogItem(fmt.Errorf("stream closed: %w for %s", err, opts.Info()))
   ```
   这条 LogItem 的 `IsError=true`，渲染时会显示橙色文字（如：`2026-06-17Txx:xx:xx stream closed: EOF for ns/pod (container1)`）

2. `tailLogs()`（[pod.go#L430-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/pod.go#L430-L434)）收到 `streamEOF` 后直接 return，不重试，也**不会向 channel 写入 `dao.ItemEOF` 哨兵值**

3. `wg.Wait()` → `close(out)` 关闭 LogChan

4. Model 层 `updateLogs()`（[model/log.go#L284-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L284-L288)）读 channel 时：
   - 消费完正常日志行 + 那条错误 LogItem
   - 最后读到 `ok=false`（channel 关闭），`item` 为 `nil`
   - 调用 `l.Append(nil)` → 因 `line==nil` 被 `IsEmpty()` 过滤掉，不写入缓冲
   - 调用 `l.Notify()` 最后一次刷新
   - goroutine return

**最终显示内容：** 日志末尾是一条**橙色错误提示行**（如 `stream closed: EOF for ...`），而不是 "🏁 Stream exited! No more logs..."。原因：
- `dao.ItemEOF` 哨兵值在代码中仅被声明（[log_item.go#L13](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_item.go#L13)），**没有任何地方将其写入 channel**
- 因此 `item == dao.ItemEOF` 判断永远为 false，`fireCanceled()`（[model/log.go#L289-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L289-L293)）永远不会被触发
- 对应的 View 层 `LogCanceled()`（显示"🏁 Stream exited!"）是死代码路径，实际运行中不会被调用

---

## 三、多容器切换机制

### 3.1 默认容器判定

`GetDefaultContainer` 定义在 [helpers.go#L30-L47](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/helpers.go#L30-L47)。

判断逻辑：
1. 检查 Pod 注解 `kubectl.kubernetes.io/default-container`
2. 注解中指定的容器名必须真实存在于 `spec.containers` 中
3. 否则返回 `("", false)`，表示无默认容器

### 3.2 LogOptions 状态管理

定义在 [log_options.go#L15-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_options.go#L15-L81)。

关键字段：
| 字段 | 含义 |
|------|------|
| `Container` | 当前正在查看的容器名，空表示 AllContainers |
| `DefaultContainer` | Pod 注解中指定的默认容器，用于切换回单容器时恢复 |
| `AllContainers` | 是否显示所有容器日志 |
| `SingleContainer` | Pod 是否只有一个容器（决定是否允许切换） |

### 3.3 ToggleAllContainers 切换流程

View 层按键 A 触发 `toggleAllContainers`（[log.go#L397-L406](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L397-L406)），调用 Model 层 `Log.ToggleAllContainers()` → `LogOptions.ToggleAllContainers()`。

`LogOptions.ToggleAllContainers()` 定义在 [log_options.go#L67-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_options.go#L67-L80)：

```go
func (o *LogOptions) ToggleAllContainers() {
    if o.SingleContainer {
        return  // 单容器 Pod 不允许切换
    }
    o.AllContainers = !o.AllContainers
    if o.AllContainers {
        // 进入多容器模式：保存当前 Container 到 DefaultContainer，清空 Container
        o.DefaultContainer, o.Container = o.Container, ""
        return
    }
    // 切回单容器：若有默认容器则恢复
    if o.DefaultContainer != "" {
        o.Container = o.DefaultContainer
    }
}
```

切换后调用 `Log.Restart(ctx)`：Stop → Clear → fireLogResume → Start，完全重建所有日志流。

### 3.4 多容器日志的渲染区分

每条 LogItem 携带 Pod/Container 信息。渲染时（[log_item.go#L70-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_item.go#L70-L100)）：
- 多 Pod 模式下显示 Pod 名前缀
- 多容器（`!SingleContainer`）模式下显示容器名前缀，并用颜色区分不同容器

容器颜色分配：[log_items.go#L109-L121](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_items.go#L109-L121)，对容器/ Pod ID 做哈希取模，从 8 色调色板中挑选稳定颜色。

---

## 四、缓冲窗口与增量刷新

### 4.1 环形缓冲：LogItems

定义在 [log_items.go#L31-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/dao/log_items.go#L31-L158)。

`Log.lines` 是一个 `*dao.LogItems`，内部是固定容量的切片，超过上限时采用 **Shift 策略**（先进先出，丢弃最旧行）：

```go
// Log.Append 定义在 model/log.go#L246-L262
func (l *Log) Append(line *dao.LogItem) {
    l.mx.Lock()
    defer l.mx.Unlock()
    l.logOptions.SinceTime = line.GetTimestamp()  // 更新断点时间，用于重连续传
    if l.lines.Len() < int(l.logOptions.Lines) {
        l.lines.Add(line)       // 缓冲未满：追加
        return
    }
    l.lines.Shift(line)         // 缓冲已满：移除第一条，追加到末尾
    l.lastSent--                // 已发送指针同步前移
    if l.lastSent < 0 {
        l.lastSent = 0
    }
}
```

- 缓冲容量由 `LogOptions.Lines` 控制（默认来自 config.Logger.TailCount）
- `SinceTime` 记录最新日志时间戳，用于流中断重连时从断点续传（见 `ToPodLogOptions` 中 `SinceTime` 转换）

### 4.2 增量刷新指针：lastSent

Model 层维护 `lastSent` 字段，记录上一次推送给 View 的行索引。每次 Notify 时只发送增量：

```go
// Log.Notify 定义在 model/log.go#L265-L273
func (l *Log) Notify() {
    l.mx.Lock()
    defer l.mx.Unlock()
    if l.lastSent < l.lines.Len() {
        l.fireLogBuffChanged(l.lastSent)  // 从 lastSent 开始渲染
        l.lastSent = l.lines.Len()        // 更新指针
    }
}
```

### 4.3 触发 Notify 的两种时机

`updateLogs` 循环（[model/log.go#L281-L308](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L281-L308)）中有两个条件会触发 Notify：

1. **溢出触发**：当未发送的积压行数超过 `Lines` 阈值时立即刷新
   ```go
   overflow = int64(l.lines.Len()-l.lastSent) > l.logOptions.Lines
   ```

2. **定时触发**：`flushTimeout`（默认 50ms）超时后刷新，保证日志延迟在 50ms 以内

这种双条件策略平衡了 **吞吐量**（批量刷新减少 UI 重绘）和 **实时性**（超时兜底保证用户看到最新日志）。

### 4.4 续传重连：SinceTime

当 Kubernetes API 流异常断开后，`tailLogs` 会触发重试。新的 `PodLogOptions` 会带上 `SinceTime`：

```go
// log_options.go#L110-L112
if t, err := time.Parse(time.RFC3339, o.SinceTime); err == nil {
    opts.SinceTime = &metav1.Time{Time: t.Add(time.Second)}
}
```

`SinceTime` 取自 Model 层 Append 时记录的最后一条日志时间戳，并加 1 秒避免重复。这样实现了 **断点续传**，避免重连后日志丢失或重复。

---

## 五、View 层持续展示机制

### 5.1 观察者模式

View 层 `Log` 实现了 `LogsListener` 接口（[model/log.go#L22-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L22-L40)），通过 `AddListener` 订阅 Model 事件。

Model 层通过 `fireLogChanged / fireLogCleared / fireLogFailed / fireLogResume / fireLogCanceled` 广播事件。

### 5.2 Flush：写入 TextView

`Log.Flush` 定义在 [view/log.go#L349-L376](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L349-L376)。

关键控制变量：
- `cancelUpdates`：暂停/恢复更新标志，用于 Restart 期间丢弃旧数据
- `follow`：是否自动滚动到底部（按键 S 切换）
- `columnLock`：列锁定模式，滚动时保持横向位置不变
- `requestOneRefresh`：切换时间范围等操作后强制刷新一次

写入使用 `tview.ANSIWriter`，支持 ANSI 颜色转义序列。

### 5.3 指示器面板：LogIndicator

定义在 [log_indicator.go](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log_indicator.go)，显示在日志视图顶部一行，展示当前状态：
- AllContainers / Autoscroll / ColumnLock / FullScreen / Timestamps / Wrap

使用 `atomic.Int32` 保证 `scrollStatus` 的并发读写安全。

---

## 七、历史输出入口与时间范围回看

### 7.1 初始入口：LogOptions 构建

有两个入口构建初始 `LogOptions`：

**1. LogsExtender.buildLogOpts()**（[logs_extender.go#L81-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/logs_extender.go#L81-L96)）

从资源列表页（如 Deployment、StatefulSet 列表）按 L/P 键查看日志时调用：

```go
func (l *LogsExtender) buildLogOpts(path, co string, prevLogs bool) *dao.LogOptions {
    cfg := l.App().Config.K9s.Logger
    opts := dao.LogOptions{
        Path:          path,
        Container:     co,
        Lines:         cfg.TailCount,
        Previous:      prevLogs,
        ShowTimestamp: cfg.ShowTime,
        LogBufferSize: cfg.LogBufferSize,
    }
    if opts.Container == "" {
        opts.AllContainers = true  // 未指定容器 → 默认为所有容器
    }
    return &opts
}
```

**2. podLogOptions()**（[logs_extender.go#L98-L121](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/logs_extender.go#L98-L121)）

从 Pod 详情页查看日志时调用，会优先使用默认容器注解：

```go
func podLogOptions(app *App, fqn string, prev bool, m *metav1.ObjectMeta, spec *v1.PodSpec) *dao.LogOptions {
    opts := dao.LogOptions{
        Path:            fqn,
        Lines:           cfg.TailCount,
        SinceSeconds:    cfg.SinceSeconds,  // 配置中的默认时间范围
        SingleContainer: len(cc) == 1,
        ShowTimestamp:   cfg.ShowTime,
        Previous:        prev,
    }
    if c, ok := dao.GetDefaultContainer(m, spec); ok {
        opts.Container, opts.DefaultContainer = c, c
    } else if len(cc) == 1 {
        opts.Container = cc[0]
    } else {
        opts.AllContainers = true  // 多容器且无默认 → 所有容器
    }
    return &opts
}
```

### 7.2 按键与时间范围映射

View 层初始化时绑定数字键 0~6（[log.go#L250-L256](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L250-L256)）：

| 按键 | 调用 | 参数 | 模式 |
|------|------|------|------|
| `0` | `sinceCmd(-1)` | `SinceSeconds=-1, Head=false` | Tail 模式，跟随最新 |
| `1` | `sinceCmd(0)` | `Head=true` | Head 模式，只拉开头 5000 字节 |
| `2` | `sinceCmd(60)` | `SinceSeconds=60` | 最近 1 分钟 + 持续跟随 |
| `3` | `sinceCmd(300)` | `SinceSeconds=300` | 最近 5 分钟 + 持续跟随 |
| `4` | `sinceCmd(900)` | `SinceSeconds=900` | 最近 15 分钟 + 持续跟随 |
| `5` | `sinceCmd(1800)` | `SinceSeconds=1800` | 最近 30 分钟 + 持续跟随 |
| `6` | `sinceCmd(3600)` | `SinceSeconds=3600` | 最近 1 小时 + 持续跟随 |

### 7.3 Model 层切换入口

**Head()**（[model/log.go#L96-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L96-L101)）：
```go
func (l *Log) Head(ctx context.Context) {
    l.mx.Lock()
    l.logOptions.Head = true
    l.mx.Unlock()
    l.Restart(ctx)  // 重建流
}
```

**SetSinceSeconds()**（[model/log.go#L104-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L104-L107)）：
```go
func (l *Log) SetSinceSeconds(ctx context.Context, i int64) {
    l.logOptions.SinceSeconds, l.logOptions.Head = i, false  // 互斥设置
    l.Restart(ctx)  // 重建流
}
```

---

## 八、Restart 流程与持续流衔接

### 8.1 Restart 四步曲（真实事件链校准）

`Restart()`（[model/log.go#L155-L160](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L155-L160)）是所有时间范围/容器切换的统一入口：

```go
func (l *Log) Restart(ctx context.Context) {
    l.Stop()          // 1. 停止旧流：调用 cancel() 取消旧 ctx
    l.Clear()         // 2. 清空缓冲：lines 清空，lastSent=0，通知 View Clear
    l.fireLogResume() // 3. 通知 View：cancelUpdates = false
    l.Start(ctx)      // 4. 启动新流：重新 load → TailLogs → 建新 goroutine
}
```

**Step 1 - Stop()**：
- 调用 `cancel()` 设置 `cancelFn()` 取消旧的 context（[model/log.go#L209-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L209-L216)）
- **⚠️ 代码事实校准点 2：没有 fireLogStop() 方法！** 虽然 `LogsListener` 接口定义了 `LogStop()`（[model/log.go#L32-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L32-L33)），View 层也实现了 `cancelUpdates=true` 的逻辑（[view/log.go#L125-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L125-L132)），但 **Model 层没有 fireLogStop()，LogStop 永远不会被调用**。这意味着 `cancelUpdates` 在 Restart 期间永远不会被设置为 true

**Step 2 - Clear()（真正的隔离机制）**：
- `l.lines.Clear()` 清空缓冲切片
- `l.lastSent = 0` 重置增量指针
- 调用 `fireLogCleared()` 通知 View 清空 TextView（[view/log.go#L143-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L143-L147)）
- **这是新旧流数据真正隔离的关键步骤：Model 缓冲和 View 显示被同时清空**

**Step 3 - fireLogResume()**：
- 通知 View 层 `LogResume()` → 设置 `cancelUpdates = false`
- 由于 Step 1 从未设置 cancelUpdates=true，这一步在当前实现中是冗余的

**Step 4 - Start()**：
- 调用 `load(ctx)` → `Pod.TailLogs()` 启动新的日志流 goroutine
- 使用新的 `LogOptions`（Head/ SinceSeconds 已更新）

---

### 8.2 旧流残余数据的真实处理（竞态分析）

由于 `LogStop()` 未被触发，`cancelUpdates` 闸门机制实际不生效，旧流数据隔离依赖于以下时序保证：

```
时间线（从 sinceCmd 被调用开始）：

t0: sinceCmd() 执行
      ├── l.logs.Clear()             // 先清空 TextView（View 层同步操作）
      ├── l.requestOneRefresh = true // 强制刷新标志
      └── 调用 model.Head() / SetSinceSeconds()

t1: Restart() 开始执行

t2: Stop() → cancel() 取消旧 ctx
      ├── 旧 tailLogs goroutine 下一次 select ctx.Done() 时退出
      └── 旧 updateLogs goroutine 下一次 select ctx.Done() 时退出
      * 注意：如果旧 goroutine 正在 Append() 中间，它可能完成这次 Append

t3: Clear() 执行
      ├── l.lines.Clear()            // 清空 Model 层缓冲（关键隔离点）
      ├── l.lastSent = 0
      └── fireLogCleared()           // 通知 View 再次清空 TextView

t4: fireLogResume() → cancelUpdates = false（无效，本来就是 false）

t5: Start() → load() 启动新流 goroutine
      * 此时新流开始写入已清空的 lines 缓冲
```

**真实隔离机制总结：**

| 机制 | 实际是否生效 | 说明 |
|------|-------------|------|
| `cancel()` 取消旧 ctx | ✅ 生效 | 旧 goroutine 在下一次 select 时快速退出 |
| `Clear()` 清空 Model 缓冲 | ✅ 生效 | **核心隔离点**，清除所有旧数据 |
| `fireLogCleared()` 清空 View | ✅ 生效 | 与 Model 清空同步 |
| `cancelUpdates` 闸门 | ❌ 不生效 | fireLogStop 未实现，标志位永远为 false |
| `LogStop()` 暂停更新 | ❌ 不生效 | Model 层没有对应的 fire 方法 |

**残余数据竞态窗口：** 极端情况下，`t2` 时刻取消 ctx 后，如果某个旧 `updateLogs` goroutine 恰好在 `Clear()` 之后、新流数据写入之前完成一次 `Append()`，这条旧数据可能会短暂混入缓冲。但由于：
1. ctx 取消后 goroutine 很快退出
2. channel 关闭后剩余数据量非常有限（最多 LogBufferSize=50 条）
3. `Clear()` 之后旧 goroutine 被调度的时间窗口极短

因此在实际运行中极少出现新旧数据混杂。`cancelUpdates` 闸门机制可以看作是**设计意图存在但实现未完成**的代码路径。

---

### 8.3 requestOneRefresh：强制刷新标志

`requestOneRefresh` 是 View 层的关键标志（[log.go#L53](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L53)），解决 head 模式下的显示问题。

在 `sinceCmd()` 中设置：
```go
func (l *Log) sinceCmd(n int) func(...) {
    return func(...) {
        l.logs.Clear()
        // ... 调用 model.Head/SetSinceSeconds
        l.requestOneRefresh = true  // 标记强制刷新一次
        l.updateTitle()
    }
}
```

在 `Flush()` 中使用：
```go
func (l *Log) Flush(lines [][]byte) {
    // 关键判断：即使 AutoScroll=false，只要 requestOneRefresh=true 也显示
    if len(lines) == 0 || (!l.requestOneRefresh && !l.indicator.AutoScroll()) || l.cancelUpdates {
        return
    }
    if l.requestOneRefresh {
        l.requestOneRefresh = false  // 消费掉标志
    }
    // ... 写入 TextView
}
```

**为什么需要这个标志？**
- head 模式下 `Follow=false`，数据一次性拉取完成后流就结束了
- 如果用户之前关闭了 AutoScroll，正常情况下数据不会被 Flush 出来
- `requestOneRefresh` 保证切换时间范围后，即使 AutoScroll=off，至少把拉到的历史数据显示一次

### 8.4 Head → Tail 切换的完整衔接

从 head（按键 1）切回 tail（按键 0）的完整流程（按真实事件链校准）：

```
用户按键 0
  ↓
view.sinceCmd(-1)
  → l.logs.Clear()                      // t0: 先同步清空 TextView
  → l.requestOneRefresh = true          // t0: 强制刷新一次（解决 AutoScroll=off）
  → model.SetSinceSeconds(ctx, -1)
    → opts.SinceSeconds = -1, opts.Head = false   // 互斥设置
    → Restart(ctx)
        ├── Stop() → cancel()
        │     └── 旧 head 流 ctx 被取消，goroutine 即将退出
        │     └── ⚠️ 不会触发 fireLogStop → cancelUpdates 仍为 false
        ├── Clear()
        │     ├── lines.Clear()         // t3: Model 缓冲清空（核心隔离点）
        │     ├── lastSent = 0
        │     └── fireLogCleared()      // t3: TextView 再次清空
        ├── fireLogResume()             // t4: cancelUpdates = false（冗余）
        └── Start(ctx)
              └── load() → Pod.TailLogs()
                    ├── ToPodLogOptions(): Follow=true, TailLines=&Lines
                    └── 每个容器启动新 tailLogs goroutine
                          ├── req.Stream() 建立新 Follow 连接
                          └── readLogs() 循环读取 → 持续写入新 LogChan
  ↓
model.updateLogs()（新 goroutine）
  → Append() 写入已清空的环形缓冲
  → 50ms 超时或溢出 → Notify()
  → fireLogBuffChanged(lastSent=0) 全量渲染
  ↓
view.LogChanged(lines)
  → QueueUpdateDraw 排队到 UI 线程
  → Flush(lines)
       ├── requestOneRefresh=true → 绕过 AutoScroll 判断
       ├── requestOneRefresh=false → 消费标志
       ├── ansiWriter.Write(lines)
       └── follow=true → ScrollToEnd()
```

### 8.5 cancelUpdates：设计意图存在但未完成的闸门

View 层 `cancelUpdates` 标志（[log.go#L49](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L49)）虽然在代码中存在，但其完整的触发链路是**断裂**的：

```go
// View 层 LogStop（存在但永远不会被 Model 调用）
func (l *Log) LogStop() {
    l.cancelUpdates = true   // 闸门关闭（设计意图）
}

// View 层 LogResume（Restart 时会被调用）
func (l *Log) LogResume() {
    l.cancelUpdates = false  // 闸门打开（实际会执行，但前提从未为 true）
}

// Flush 中的闸门判断（永远不会命中）
if l.cancelUpdates {
    return  // 设计意图：闸门关闭时丢弃旧流数据
}
```

**链路断裂点**：
- `LogsListener` 接口定义了 `LogStop()`（[model/log.go#L32-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L32-L33)）
- View 层实现了 `LogStop()`，会设置 `cancelUpdates = true`
- **但 Model 层从未实现 `fireLogStop()` 方法**，也没有任何地方调用 `listener.LogStop()`
- 全代码库搜索：不存在 `fireLogStop` 字符串

**Flush 的 defer 副作用**（[view/log.go#L350-L354](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/view/log.go#L350-L354)）：
```go
defer func() {
    if l.cancelUpdates {
        l.cancelUpdates = false  // 设计意图：闸门关闭后，至多丢弃一次 Flush
    }
}()
```
这段逻辑也印证了设计意图：闸门关闭后，允许丢弃至多一次 Flush 的数据（旧流残余），然后自动复位避免卡死。但由于上游触发点缺失，这部分逻辑同样无法生效。

**结论**：`cancelUpdates` / `LogStop` 是一套**设计意图存在但实现未完成**的闸门机制，当前版本中真实的隔离依赖于 `Clear()` 清空缓冲 + `cancel()` 取消 goroutine 的组合。

---

## 九、完整调用链总结

### 9.1 初始查看日志

```
用户按 L/P 键查看日志
  ↓
LogsExtender.logsCmd(prev=true/false)
  → buildLogOpts() 或 podLogOptions() 构造 LogOptions
    → 未指定 Container 时 AllContainers=true
    → 有默认容器注解时 Container/DefaultContainer=注解值
  → NewLog(gvr, opts) 创建 View
  → App.inject() 注入组件
    → Log.Init(ctx) 初始化 UI、Model、绑定 0~6 数字键
    → Log.Start()
      → model.Start(ctx)
        → load(ctx)
          → Pod.TailLogs(ctx, opts)
            → 根据容器数量启动 N 个 tailLogs() goroutine
              → 指数退避循环重试：req.Stream() → readLogs() → LogChan
      → model.AddListener(l) 注册观察者
```

### 9.2 时间范围切换（用户按键 0~6）（已校准）

```
用户按数字键 N (0~6)
  ↓
view.sinceCmd(n)  // n=-1/0/60/300/900/1800/3600
  → l.logs.Clear()  // 清空 TextView
  → l.requestOneRefresh = true  // 强制刷新一次
  → n==0 ? model.Head(ctx) : model.SetSinceSeconds(ctx, n)
    → 更新 logOptions.Head / SinceSeconds，互斥设置
    → Restart(ctx) 四步曲
      ├── Stop() → cancel() 取消旧 ctx
      │     ⚠️ 没有 fireLogStop → view.LogStop() 不会被调用
      │     ⚠️ cancelUpdates 不会被设置为 true，闸门机制不生效
      ├── Clear() → 清空 lines、lastSent=0 → fireLogCleared() → UI Clear
      │     ✅ 这是新旧数据真正隔离的关键步骤
      ├── fireLogResume() → view.LogResume() → cancelUpdates=false（冗余）
      └── Start(ctx) → load() → 用新参数重建 tailLogs goroutine
  → l.updateTitle() 更新标题显示当前模式 (tail/head/1m/5m...)
```

### 9.3 Head 模式流结束（已校准：错误项 vs 结束提示）

```
head 模式 (Follow=false)
  ↓
Kubernetes API 返回 5000 字节后主动关闭连接
  ↓
readLogs() 收到 io.EOF
  → emit 残余行（如果有）
  → out <- ToErrLogItem("stream closed: EOF for ns/pod (cont1)")   // ⚠️ 是错误项！
  → return streamEOF
  ↓
tailLogs() 收到 streamEOF → 直接 return，不重试
  ⚠️ 不会发送 dao.ItemEOF 到 channel
  ↓
wg.Wait() → close(out) 关闭 LogChan
  ↓
model.updateLogs() 收到 channel 关闭 ok=false
  → 先消费完所有已写入 channel 的数据（含错误项）
  → Append(错误项) ✅ → IsError=true 渲染为橙色文字
  → 再读时 ok=false，item=nil → Append(nil) ❌ 被 IsEmpty 过滤
  → Notify() 最后一次刷新
  → 退出 for 循环，goroutine 结束
  ⚠️ ItemEOF 从未写入 channel，item == dao.ItemEOF 判断永远 false
  ⚠️ fireCanceled() 不会被调用
  ↓
最终显示：
  ...正常 Head 日志...
  [橙色] 2026-06-17Txx:xx:xx stream closed: EOF for ns/pod (container1)
  ⚠️ 不会显示 "🏁 Stream exited! No more logs..."（死代码路径）
```

### 9.4 持续流缓冲展示（Tail 模式）（已校准）

```
readLogs() 按行读取 → 写入 LogChan (带背压丢弃)
    ↓
  model.updateLogs() goroutine 消费 channel
    ├── Append(line) 写入环形缓冲 (Shift 策略)
    │   └── 更新 SinceTime = 最新日志时间戳
    ├── 检查 overflow = 未发送行数 > Lines 阈值 ?
    ├── 或 flushTimeout (50ms) 超时 ?
    └── 满足任一 → Notify()
        ├── fireLogBuffChanged(lastSent) 渲染增量
        └── lastSent = lines.Len() 更新指针
    ↓
  view.LogChanged(lines)
    → QueueUpdateDraw 排队到 UI 线程
    → Flush(lines)
        ├── 检查 !requestOneRefresh && !AutoScroll ?
        │     ⚠️ cancelUpdates 永远为 false，不构成实际判断条件
        ├── 是 → return 丢弃
        └── 否 → ansiWriter.Write(lines) 写入 TextView
            → follow=true ? ScrollToEnd() : 停留在当前位置
```

### 9.5 多容器切换（按键 A）（已校准）

```
用户按 A 键
  ↓
view.toggleAllContainers()
  → indicator.ToggleAllContainers() 更新 UI 显示
  → model.ToggleAllContainers(ctx)
    → LogOptions.ToggleAllContainers()
        ├── SingleContainer ? return 不允许切换
        ├── AllContainers = !AllContainers
        ├── 进入多容器：DefaultContainer, Container = Container, ""
        └── 退出多容器：有 DefaultContainer ? Container = DefaultContainer
    → Restart(ctx) 四步曲，重建所有日志流
        ├── Stop() → cancel() 取消旧 ctx（无 fireLogStop）
        ├── Clear() → 真正清空 Model 和 View 层数据
        ├── fireLogResume() → cancelUpdates=false（冗余）
        └── Start(ctx) → 按新 AllContainers 模式启动 N/M 个 tailLogs goroutine
  → updateTitle()
```
