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
- 最终 LogChan 被关闭，Model 层 `updateLogs()` 循环结束

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

### 8.1 Restart 四步曲

`Restart()`（[model/log.go#L155-L160](file:///d:/fz/0601-2/solo-dogfeeding/code/2-k9s/internal/model/log.go#L155-L160)）是所有时间范围/容器切换的统一入口：

```go
func (l *Log) Restart(ctx context.Context) {
    l.Stop()          // 1. 停止旧流：调用 cancel() 取消旧 ctx
    l.Clear()         // 2. 清空缓冲：lines 清空，lastSent=0，通知 View Clear
    l.fireLogResume() // 3. 通知 View 恢复更新：cancelUpdates = false
    l.Start(ctx)      // 4. 启动新流：重新 load → TailLogs → 建新 goroutine
}
```

**Step 1 - Stop()**：
- 调用 `cancel()` 设置 `cancelFn()` 取消旧的 context
- 所有正在运行的 `tailLogs` / `updateLogs` goroutine 收到 `ctx.Done()` 后退出

**Step 2 - Clear()**：
- `l.lines.Clear()` 清空缓冲切片
- `l.lastSent = 0` 重置增量指针
- 调用 `fireLogCleared()` 通知 View 清空 TextView

**Step 3 - fireLogResume()**：
- 通知 View 层 `LogResume()` → 设置 `cancelUpdates = false`
- 确保新流的数据不会被丢弃

**Step 4 - Start()**：
- 调用 `load(ctx)` → `Pod.TailLogs()` 启动新的日志流 goroutine
- 使用新的 `LogOptions`（Head/ SinceSeconds 已更新）

### 8.2 requestOneRefresh：强制刷新标志

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

### 8.3 Head → Tail 切换的完整衔接

从 head（按键 1）切回 tail（按键 0）的完整流程：

```
用户按键 0
  ↓
view.sinceCmd(-1)
  → l.logs.Clear()  // 清空 UI
  → l.requestOneRefresh = true
  → model.SetSinceSeconds(ctx, -1)
    → opts.SinceSeconds = -1, opts.Head = false
    → Restart(ctx)
      → Stop() 取消旧 head 流 ctx
      → Clear() 清空缓冲，通知 View Clear
      → fireLogResume() 恢复更新
      → Start(ctx)
        → load(ctx) → Pod.TailLogs()
          → ToPodLogOptions() 转换：Follow=true, TailLines=&Lines
          → 每个容器启动 tailLogs goroutine
            → req.Stream(ctx) 建立新的 Follow=true 连接
            → readLogs() 循环读取，持续写入 LogChan
  ↓
model.updateLogs() 消费新流
  → Append() 写入环形缓冲
  → 50ms 超时或溢出 → Notify()
  → fireLogChanged(lines)
  ↓
view.LogChanged(lines)
  → QueueUpdateDraw
  → Flush(lines)：requestOneRefresh=true，即使 AutoScroll=off 也显示
  → 写入 TextView，follow=true 时 ScrollToEnd()
```

### 8.4 cancelUpdates：暂停更新机制

View 层 `cancelUpdates` 标志用于 Restart 期间丢弃旧流的残余数据：

```go
// Model 层 Stop 时触发
func (l *Log) LogStop() {
    l.mx.Lock()
    defer l.mx.Unlock()
    l.cancelUpdates = true  // 暂停更新
}

// Model 层 Restart 第三步 fireLogResume 触发
func (l *Log) LogResume() {
    l.mx.Lock()
    defer l.mx.Unlock()
    l.cancelUpdates = false  // 恢复更新
}

// Flush 时判断
if l.cancelUpdates {
    return  // 丢弃数据
}
```

**为什么需要这个机制？**
- 旧流被 cancel 后，可能还有一些数据在 channel 中未消费
- 如果不暂停，这些旧数据可能在新流启动后混杂进来
- `cancelUpdates` 作为一道闸门，确保新旧流数据完全隔离

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

### 9.2 时间范围切换（用户按键 0~6）

```
用户按数字键 N (0~6)
  ↓
view.sinceCmd(n)  // n=-1/0/60/300/900/1800/3600
  → l.logs.Clear()  // 清空 TextView
  → l.requestOneRefresh = true  // 强制刷新一次
  → n==0 ? model.Head(ctx) : model.SetSinceSeconds(ctx, n)
    → 更新 logOptions.Head / SinceSeconds，互斥设置
    → Restart(ctx) 四步曲
      ├── Stop() → cancel() 取消旧 ctx → view.LogStop() → cancelUpdates=true
      ├── Clear() → 清空 lines、lastSent=0 → fireLogCleared() → UI Clear
      ├── fireLogResume() → view.LogResume() → cancelUpdates=false
      └── Start(ctx) → load() → 用新参数重建 tailLogs goroutine
  → l.updateTitle() 更新标题显示当前模式 (tail/head/1m/5m...)
```

### 9.3 Head 模式流结束

```
head 模式 (Follow=false)
  ↓
Kubernetes API 返回 5000 字节后主动关闭连接
  ↓
readLogs() 收到 io.EOF
  → emit 残余行（如果有）
  → out <- ToErrLogItem("stream closed")
  → return streamEOF
  ↓
tailLogs() 收到 streamEOF → 直接 return，不重试
  ↓
wg.Wait() → close(out) 关闭 LogChan
  ↓
model.updateLogs() 收到 channel 关闭
  → Append(itemEOF) → Notify()
  → 退出 for 循环，goroutine 结束
  ↓
model.fireCanceled() → view.LogCanceled()
  → 显示 "🏁 Stream exited! No more logs..."
```

### 9.4 持续流缓冲展示（Tail 模式）

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
        ├── 检查 !requestOneRefresh && !AutoScroll && cancelUpdates ?
        ├── 是 → return 丢弃
        └── 否 → ansiWriter.Write(lines) 写入 TextView
            → follow=true ? ScrollToEnd() : 停留在当前位置
```

### 9.5 多容器切换（按键 A）

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
  → updateTitle()
```
