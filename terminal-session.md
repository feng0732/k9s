# K9s Exec/Attach 终端会话代码路径分析

## 概述

K9s 的 exec/attach 终端会话采用「外挂 kubectl 二进制 + TUI 挂起移交」的设计模式。整个流程包括：会话创建、TUI 挂起、终端移交、尺寸同步、异常清理五个阶段。

---

## 1. 会话创建流程

### 1.1 入口点（键盘事件）

| 视图 | 按键 | 处理函数 |
|------|------|----------|
| Pod 视图 | `s` (Shell) | [pod.go:shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L222-L238) |
| Pod 视图 | `a` (Attach) | [pod.go:attachCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L240-L256) |
| Container 视图 | `s` (Shell) | [container.go:shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L158-L181) |
| Container 视图 | `a` (Attach) | [container.go:attachCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L183-L194) |

### 1.2 前置检查

在 [pod.go:shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L222-L238) 中：
```go
if !podIsRunning(p.App().factory, path) {
    p.App().Flash().Errf("%s is not in a running state", path)
    return nil
}
```

### 1.3 容器选择逻辑

[pod.go:containerShellIn](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L358-L386)：
1. 若指定了容器名，直接使用
2. 检查 `kubectl.kubernetes.io/default-container` annotation
3. 若只有一个容器，直接使用
4. 多容器时弹出选择器

### 1.4 命令参数构建

[pod.go:buildShellArgs](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L478-L503) 构建 kubectl 命令参数：
```go
args := []string{"exec", "-it"}  // 或 "attach"
args = append(args, "-n", namespace)
args = append(args, podName)
args = append(args, "-c", containerName)
// 追加用户命令: sh -c "command -v bash >/dev/null && exec bash || exec sh"
```

[pod.go:computeShellArgs](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L461-L468) 会根据 pod OS（Linux/Windows）选择不同的 shell 命令。

---

## 2. TUI 挂起与终端移交

### 2.1 核心调用链

```
resumeShellIn
    ↓
[pod.go] shellIn → runK
    ↓
[exec.go:runK](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L57-L97)
    ↓
[exec.go:run](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L99-L122)
    ├─ a.Halt()        // 停止后台任务
    ├─ defer a.Resume() // 恢复后台任务
    └─ a.Suspend(func() {
           execute(...) // 执行 kubectl 命令
       })
```

### 2.2 Halt/Resume 机制

[app.go:Halt](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go#L334-L339) 停止后台任务：
```go
func (a *App) Halt() {
    if a.cancelFn != nil {
        a.cancelFn()      // 取消集群更新、文件监听等后台 goroutine
        a.cancelFn = nil
    }
}
```

[app.go:Resume](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go#L342-L362) 恢复后台任务：
```go
func (a *App) Resume() {
    ctx, a.cancelFn = context.WithCancel(context.Background())
    go a.clusterUpdater(ctx)           // 集群信息更新
    go a.ConfigWatcher(ctx, a)         // 配置文件监听
    // ... 其他监听器
}
```

### 2.3 tview.Application.Suspend 实现

来自 [derailed/tview v0.8.5](https://raw.githubusercontent.com/derailed/tview/v0.8.5/application.go)：

```go
func (a *Application) Suspend(f func()) bool {
    a.RLock()
    screen := a.screen
    a.RUnlock()
    if screen == nil {
        return false
    }
    // 1. 挂起屏幕：退出终端原始模式
    if err := screen.Suspend(); err != nil {
        return false
    }
    // 2. 执行用户函数（此时终端控制权已移交）
    f()
    // 3. 恢复屏幕：重新进入终端原始模式
    a.RLock()
    defer a.RUnlock()
    if a.screen != screen {
        screen.Fini()
        if a.screen == nil {
            return true
        }
    } else {
        screen.Resume() // 重新初始化终端原始模式
    }
    return true
}
```

**tcell Screen 挂起原理：**
- `Screen.Suspend()` → 调用 `tcsetattr` 恢复终端标准模式（canonical mode + echo）
- `Screen.Resume()` → 调用 `tcsetattr` 设置终端原始模式（raw mode + no echo），并重新初始化屏幕缓冲区

### 2.4 kubectl 执行上下文

[exec.go:execute](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L172-L239) 中的关键设置：
```go
cmd := exec.CommandContext(ctx, opts.binary, opts.args...)
cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
_, _ = cmd.Stdout.Write([]byte(opts.banner)) // 打印 banner
err := cmd.Run()
```

此时 kubectl 进程完全接管标准输入输出，直接与用户交互。

---

## 3. 终端尺寸同步机制

K9s 采用「完全移交」策略：TUI 挂起后，终端尺寸同步完全由 kubectl 负责。

### 3.1 kubectl 内部尺寸同步流程

```
用户调整终端窗口大小
    ↓
内核发送 SIGWINCH 信号给前台进程组（kubectl）
    ↓
kubectl 的信号处理器被触发
    ↓
ioctl(STDIN_FILENO, TIOCGWINSZ, &winsize) 获取新尺寸
    ↓
通过 SPDY/WebSocket 的 resize 子通道发送尺寸到 API Server
    ↓
API Server 转发给 kubelet
    ↓
kubelet 调用容器运行时 ResizeTTY 接口
    ↓
容器运行时更新 PTY 窗口大小
    ↓
内核发送 SIGWINCH 给容器内前台进程（如 bash）
    ↓
shell 重新计算并调整布局
```

### 3.2 关键技术点

| 技术 | 说明 |
|------|------|
| `SIGWINCH` | 窗口尺寸变化信号，发送给终端前台进程组 |
| `TIOCGWINSZ` | ioctl 命令，从内核读取终端窗口大小（rows/cols） |
| `TIOCSWINSZ` | ioctl 命令，设置终端窗口大小 |
| `struct winsize` | 定义终端尺寸的内核结构体 |

**winsize 结构体：**
```c
struct winsize {
    unsigned short ws_row;     // 行数
    unsigned short ws_col;     // 列数
    unsigned short ws_xpixel;  // 水平像素（未使用）
    unsigned short ws_ypixel;  // 垂直像素（未使用）
};
```

### 3.3 K9s 为何不处理 resize？

因为 K9s 挂起 TUI 后：
1. tcell 不再监听终端事件（包括 `EventResize`）
2. 终端的标准输入输出完全由 kubectl 进程接管
3. kubectl 有自己成熟的 SIGWINCH 处理逻辑

---

## 4. 异常清理与会话收尾

采用**多层 defer 防御式编程**，确保各种异常路径下资源都能正确释放。

### 4.1 清理层级（从内到外）

**第一层：execute 函数内部清理** [exec.go:execute](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L176-L197)
```go
ctx, cancel := context.WithCancel(context.Background())
defer func() {
    if !opts.background {
        cancel()      // 取消命令上下文，终止子进程
        clearScreen() // 清屏，准备恢复 TUI
    }
}()

// 信号监听 goroutine
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
go func(cancel context.CancelFunc) {
    select {
    case sig := <-sigChan:
        slog.Debug("Command canceled with signal", slogs.Sig, sig)
        cancel() // 用户按 Ctrl+C 时取消命令
    case <-ctx.Done():
        slog.Debug("Signal context canceled!")
    }
    interrupted = true
}(cancel)
```

**第二层：run 函数恢复后台任务** [exec.go:run](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L112-L113)
```go
a.Halt()       // 停止后台集群更新、文件监听等
defer a.Resume() // 恢复后台任务
```

**第三层：视图刷新控制** [pod.go:resumeShellIn](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L388-L401)
```go
c.Stop()       // 停止当前视图的数据刷新
defer c.Start() // 恢复视图刷新
```

### 4.2 Node Shell 特殊清理

Node Shell 会启动一个特权 pod 访问节点，需要额外的清理机制防止残留。

**清理触发点：**

1. **启动前清理** [exec.go:launchNodeShell](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L301-L304)
   - 防止之前的 k9s-shell pod 残留

2. **使用后清理** [exec.go:launchPodShell](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L332-L337)
   ```go
   defer func() {
       if err := nukeK9sShell(a); err != nil { ... }
   }()
   ```

3. **应用退出清理** [app.go:BailOut](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go#L540-L542)
   ```go
   if err := nukeK9sShell(a); err != nil {
       slog.Error("Unable to nuke k9s shell pod", ...)
   }
   ```

**nukeK9sShell 实现** [exec.go:nukeK9sShell](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L381-L405)：
- 检查 `NodeShell` feature gate 是否启用
- 删除名为 `k9s-shell-{pid}` 的 pod（pid 是 k9s 进程 ID）
- 500ms 超时避免阻塞

### 4.3 tview 内置 Panic 恢复

[tview Run 函数](https://raw.githubusercontent.com/derailed/tview/v0.8.5/application.go) 中的防御性编程：
```go
defer func() {
    if p := recover(); p != nil {
        if a.screen != nil {
            a.screen.Fini() // panic 时确保终端状态恢复，避免终端乱码
        }
        panic(p)
    }
}()
```

---

## 5. 完整调用链图

```
用户按键 (s/a)
    ↓
[pod.go] shellCmd/attachCmd
    ├─ podIsRunning() 检查 Pod 状态
    └─ containerShellIn/containerAttachIn
        ├─ 多容器时显示 Picker 选择器
        └─ resumeShellIn/resumeAttachIn
            ├─ c.Stop()               // 停止视图刷新
            ├─ [pod.go] shellIn/attachIn
            │   └─ [exec.go] runK
            │       ├─ exec.LookPath("kubectl")  查找 kubectl
            │       ├─ 追加 --as/--as-group/--context 等参数
            │       └─ [exec.go] run
            │           ├─ a.Halt()             // 停止后台任务
            │           ├─ defer a.Resume()      // 恢复后台任务
            │           └─ a.Suspend(f)          // 挂起 TUI
            │               ├─ screen.Suspend()   // 终端恢复标准模式
            │               ├─ f() → [exec.go] execute
            │               │   ├─ clearScreen()
            │               │   ├─ ctx, cancel := context.WithCancel()
            │               │   ├─ 启动 SIGINT/SIGTERM 监听 goroutine
            │               │   ├─ exec.CommandContext(ctx, "kubectl", ...)
            │               │   ├─ cmd.Stdin/Stdout/Stderr = os.Stdin/os.Stdout/os.Stderr
            │               │   ├─ cmd.Run()          // kubectl 完全接管终端
            │               │   │   └─ kubectl 内部处理:
            │               │   │       ├─ SPDY/WebSocket 连接 API Server
            │               │   │       ├─ 分配 PTY
            │               │   │       ├─ 监听 SIGWINCH 信号
            │               │   │       ├─ ioctl(TIOCGWINSZ) 读尺寸
            │               │   │       └─ SPDY resize 子通道同步
            │               │   └─ defer: cancel() + clearScreen()
            │               └─ screen.Resume()    // 终端恢复原始模式
            └─ defer c.Start()                  // 恢复视图刷新
```

---

## 6. 关键设计决策分析

### 6.1 为什么调用 kubectl 而不是直接用 client-go API？

| 优点 | 缺点 |
|------|------|
| 复用 kubectl 成熟的 exec/attach 实现（TTY 分配、信号处理、resize 同步） | 需要用户 PATH 中有 kubectl 二进制 |
| 自动兼容不同 k8s 版本和容器运行时 | 多了一层进程开销 |
| 命令会经过 k8s API Server 的审计日志 | 无法细粒度控制终端行为 |
| 代码量大幅减少，维护成本低 | |

### 6.2 为什么需要多层 Halt/Resume？

- **`a.Halt()`/`a.Resume()`**：停止集群信息更新、配置文件监听等后台任务，避免在终端会话期间产生干扰输出或竞争条件
- **`c.Stop()`/`c.Start()`**：停止当前视图的数据刷新，避免在终端会话期间更新 UI 导致屏幕混乱

### 6.3 异常恢复保障层级

1. **kubectl 进程崩溃** → `execute` defer 清理 + `run` defer 恢复 TUI
2. **用户 Ctrl+C 中断** → 信号 goroutine 取消 context → defer 链正常执行
3. **k9s 内部 panic** → tview Run 函数的 defer 恢复终端状态
4. **k9s 进程被强制杀死** → Node Shell pod 可能残留，但命名带 pid 可识别
5. **应用正常退出** → `BailOut()` 主动清理 k9s-shell pod

---

## 7. 核心代码文件速查表

| 文件 | 核心函数 | 职责 |
|------|----------|------|
| [exec.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go) | `runK`, `run`, `execute`, `pipe`, `nukeK9sShell` | kubectl 命令执行、TUI 挂起恢复、Node Shell 清理 |
| [pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go) | `shellCmd`, `attachCmd`, `containerShellIn`, `shellIn`, `buildShellArgs` | Pod 视图的 exec/attach 入口、命令参数构建 |
| [container.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go) | `shellCmd`, `attachCmd` | Container 视图的 exec/attach 入口 |
| [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go) | `Halt`, `Resume`, `BailOut` | 后台任务启停、应用退出清理 |
| [ui/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/ui/app.go) | `Suspend` (继承自 tview) | TUI 挂起（tview 提供） |
