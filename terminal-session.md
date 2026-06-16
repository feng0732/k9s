# K9s Exec/Attach 终端会话代码路径分析

## 概述

K9s 的 exec/attach 终端会话采用「外挂 kubectl 二进制 + TUI 挂起移交」的设计模式。整个流程包括：多视图入口、会话创建、TUI 挂起、标准 IO 接管、尺寸同步、异常清理六个阶段。

---

## 1. 资源详情视图入口全景

exec/attach 并非只能从 Pod/Container 列表视图触发，K9s 在多个视图中都暴露了相关入口。

### 1.1 主路径入口矩阵

> **⚠️ 热键/插件不属于 exec/attach 主路径**，详见 1.4 节分析。

| 视图类型 | 视图文件 | 按键 | 节点 GVR 类型 | 处理函数 | Feature Gate 依赖 |
|----------|----------|------|--------------|----------|------------------|
| **Pod 列表** | [pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L126-L135) | `s` | PodGVR | [shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L222-L238) | 非只读 |
| **Pod 列表** | [pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L126-L135) | `a` | PodGVR | [attachCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L240-L256) | 非只读 |
| **Container 列表** | [container.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L83-L94) | `s` | CoGVR | [shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L158-L181) | 非只读 |
| **Container 列表** | [container.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L83-L94) | `a` | CoGVR | [attachCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go#L183-L194) | 非只读 |
| **Xray 拓扑图（Pod 节点）** | [xray.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L214-L232) | `s` | PodGVR | [xray.shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L340-L362) | 非只读 |
| **Xray 拓扑图（Pod 节点）** | [xray.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L214-L232) | `a` | PodGVR | [xray.attachCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L364-L385) | 非只读 |
| **Xray 拓扑图（Container 节点）** | [xray.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L201-L213) | `s` | CoGVR | [xray.shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L340-L362) | 非只读 |
| **Xray 拓扑图（Container 节点）** | - | `a` | CoGVR | - | **未绑定** |
| **Node 列表** | [node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/node.go#L75-L77) | `s` | NodeGVR | [sshCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/node.go#L179-L191) | NodeShell + ShellPod 配置 |

### 1.2 Xray 拓扑图 Pod/Container 节点按键区别（资源详情视图）

**⚠️ 关键区别：Container 节点只有 `s` 键，没有 `a` 键（Attach 根本没绑定）。**

在 Xray 视图中，按键绑定根据当前选中节点的 GVR 类型动态设置：

**快捷键绑定逻辑** [xray.go:198-L233](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L198-L233)：
```go
switch gvr {
case client.CoGVR:  // 选中 Container 节点时
    x.Actions().Delete(tcell.KeyEnter)
    aa.Bulk(ui.KeyMap{
        ui.KeyL: ui.NewKeyAction("Logs", x.logsCmd(false), true),
        ui.KeyP: ui.NewKeyAction("Logs Previous", x.logsCmd(true), true),
    })
    if !x.app.Config.IsReadOnly() {
        // ⚠️ 只有 s 键（Shell），没有 a 键（Attach）！
        aa.Add(ui.KeyS, ui.NewKeyActionWithOpts("Shell", x.shellCmd,
            ui.ActionOpts{Visible: true, Dangerous: true}))
    }
case client.PodGVR:  // 选中 Pod 节点时
    aa.Bulk(ui.KeyMap{
        ui.KeyL: ui.NewKeyAction("Logs", x.logsCmd(false), true),
        ui.KeyP: ui.NewKeyAction("Logs Previous", x.logsCmd(true), true),
    })
    if !x.app.Config.IsReadOnly() {
        // ✅ s 键和 a 键都有
        aa.Bulk(ui.KeyMap{
            ui.KeyS: ui.NewKeyActionWithOpts("Shell", x.shellCmd, ...),
            ui.KeyA: ui.NewKeyActionWithOpts("Attach", x.attachCmd, ...),
        })
    }
}
```

---

### 1.3 Xray shellCmd/attachCmd 处理逻辑区别

**shellCmd 对 Pod/Container 节点的不同处理** [xray.go:340-L362](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L340-L362)：
```go
func (x *Xray) shellCmd(*tcell.EventKey) *tcell.EventKey {
    spec := x.selectedSpec()
    if spec.Status() != "ok" {
        x.app.Flash().Errf("%s is not in a running state", spec.Path())
        return nil
    }

    path, co := spec.Path(), ""
    if spec.GVR() == client.CoGVR {
        // ✅ Container 节点：正确提取 Pod 路径和容器名
        _, co = client.Namespaced(spec.Path())  // co = 容器名
        path = *spec.ParentPath()                // path = 父节点 Pod 路径
    }
    // Pod 节点：path = Pod 路径，co = ""（将弹出容器选择器）
    if err := containerShellIn(x.app, x, path, co); err != nil {
        x.app.Flash().Err(err)
    }
    return nil
}
```

**attachCmd 对 Pod/Container 节点的不同处理** [xray.go:364-L385](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go#L364-L385)：
```go
func (x *Xray) attachCmd(*tcell.EventKey) *tcell.EventKey {
    spec := x.selectedSpec()
    if spec.Status() != "ok" { return nil }

    path, co := spec.Path(), ""
    if spec.GVR() == client.CoGVR {
        // ⚠️ 这段代码实际上是死代码！
        // CoGVR 节点根本没有绑定 a 键，永远不会执行到这里
        // 而且代码有缺陷：只设置 path，没有提取 co！
        path = *spec.ParentPath()
        // 缺少：_, co = client.Namespaced(spec.Path())
    }
    // Pod 节点：path = Pod 路径，co = ""（将弹出容器选择器）
    if err := containerAttachIn(x.app, x, path, co); err != nil {
        x.app.Flash().Err(err)
    }
    return nil
}
```

**Xray 节点类型与按键行为对比表：**

| 节点类型 | `s` 键（Shell） | `a` 键（Attach） | shellCmd 处理 | attachCmd 处理 |
|----------|-----------------|-----------------|--------------|----------------|
| **PodGVR** | ✅ 有绑定 | ✅ 有绑定 | path=Pod路径, co="" → 弹出容器选择器 | path=Pod路径, co="" → 弹出容器选择器 |
| **CoGVR** | ✅ 有绑定 | ❌ **未绑定** | path=ParentPath, co=容器名 → 直接进入指定容器 | 代码为死代码（且有缺陷） |

---

### 1.4 热键与插件入口分析：不属于 exec/attach 主路径

#### 1.4.1 热键（HotKey）：纯导航，不执行 exec/attach

**代码路径** [actions.go:60-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/actions.go#L60-L106)：
```go
func hotKeyActions(r Runner, aa *ui.KeyActions) error {
    // ... 加载 hotkey 配置
    for k, hk := range hh.HotKey {
        // ... 环境变量替换
        command, err := r.EnvFn()().Substitute(hk.Command)
        aa.Add(key, ui.NewKeyActionWithOpts(
            hk.Description,
            gotoCmd(r, command, "", !hk.KeepHistory),  // ⚠️ 只调用 gotoCmd
            ui.ActionOpts{Shared: true, HotKey: true},
        ))
    }
}

func gotoCmd(r Runner, cmd, path string, clearStack bool) ui.ActionHandler {
    return func(*tcell.EventKey) *tcell.EventKey {
        r.App().gotoResource(cmd, path, clearStack, true)  // 纯视图跳转
        return nil
    }
}
```

**结论**：热键只是调用 `gotoResource()` 进行视图跳转，不直接执行 exec/attach 命令。

#### 1.4.2 插件（Plugin）：通用命令执行，exec/attach 只是其中一种可能

**代码路径** [actions.go:115-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/actions.go#L115-L265)：

插件确实会调用 `run()` 进入终端执行流程：
```go
func executePlugin(r Runner, p *config.Plugin, inputValues dialog.PluginInputValues) {
    // ... 参数替换
    cb := func() {
        opts := shellOpts{
            binary:     p.Command,     // ⚠️ 可以是任意命令，不只是 kubectl
            background: p.Background,
            pipes:      p.Pipes,
            args:       args,
        }
        suspend, errChan, statusChan := run(r.App(), &opts)  // 调用 run()
    }
    // ... 确认对话框
}
```

**但插件不属于 exec/attach 主路径的原因：**

| 原因 | 说明 |
|------|------|
| **命令不固定** | `p.Command` 可以是 `kubectl`，也可以是 `helm`、`curl` 等任意命令，exec/attach 只是众多可能之一 |
| **用户自定义** | 插件完全由用户配置，不是 k9s 内置的 exec/attach 流程 |
| **缺少上下文** | 插件没有 Pod/Container 选择逻辑，完全依赖用户在配置中通过环境变量（如 `$NAMESPACE`、`$NAME`、`$CONTAINER`）构建参数 |
| **定位不同** | 插件是通用扩展机制，而非专门的 exec/attach 入口 |

**主路径定义**：k9s 内置的、从资源视图按键触发的、包含完整 Pod/Container 上下文选择的 exec/attach 流程才是主路径。

---

### 1.5 Node Shell 入口（特殊场景）

Node 列表的 `s` 键不会直接 exec 到 Node（Node 不是 Pod），而是启动一个**特权 Pod** 来访问节点宿主机。

**入口检查** [node.go:75-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/node.go#L75-L77)：
```go
if ct.FeatureGates.NodeShell && n.App().Config.K9s.ShellPod != nil {
    aa.Add(ui.KeyS, ui.NewKeyAction("Shell", n.sshCmd, true))
}
```

**执行流程** [node.go:179-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/node.go#L179-L191)：
```go
func (n *Node) sshCmd(evt *tcell.EventKey) *tcell.EventKey {
    n.Stop()
    defer n.Start()
    _, node := client.Namespaced(path)
    launchNodeShell(n, n.App(), node)  // 进入 Node Shell 启动流程
    return nil
}
```

---

## 2. 标准输入输出接管位置与机制

K9s 不使用 client-go 的 `remotecommand.Stream()` API，而是通过 `exec.Command` 启动 kubectl 子进程，并**直接将其标准 IO 映射到操作系统终端**。

### 2.1 接管链路全景

```
用户键盘输入 → 终端设备驱动 (/dev/tty)
    ↓
os.Stdin (k9s 进程继承的文件描述符 0)
    ↓  tview Application.Suspend() 挂起 tcell 原始模式后
cmd.Stdin = os.Stdin  ← [exec.go:pipe 574行]
cmd.Stdout = os.Stdout ← [exec.go:pipe 574行]
cmd.Stderr = os.Stderr ← [exec.go:pipe 574行]
    ↓  cmd.Run() 启动 kubectl 子进程
kubectl exec -it ...
    ↓  kubectl 内部使用 client-go remotecommand
remotecommand.SPDYExecutor.Stream()
    ↓  SPDY 多路复用 5 个子通道
├─ stdin 通道 (channel 0)
├─ stdout 通道 (channel 1)
├─ stderr 通道 (channel 2)
├─ error 通道 (channel 3)
└─ resize 通道 (channel 4) ← 终端尺寸同步专用
```

### 2.2 关键接管代码位置

**核心函数** [exec.go:pipe 549-L615行](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L549-L615)

**单命令模式（exec/attach 使用）** [exec.go:554-L594](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L554-L594)：
```go
if len(cmds) == 1 {
    cmd := cmds[0]
    if opts.background {
        // 后台模式：输出写入 buffer，不直接交互
        go func() {
            cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, w, e
            if err := cmd.Run(); err != nil { ... }
        }()
        return nil
    }
    // ═══════════ 前台交互式模式（exec/attach 实际走这里）═══════════
    // 关键行：直接将 os 层面的标准 IO 赋值给子进程
    cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
    // 打印 banner 信息（如 Pod 名称提示）
    _, _ = cmd.Stdout.Write([]byte(opts.banner))
    // 同步执行：阻塞在此，直到 kubectl 退出
    err := cmd.Run()
    // 处理信号导致的异常退出（如 Ctrl+C）
    var ex *exec.ExitError
    if errors.As(err, &ex) && !ex.Exited() {
        return nil  // 信号终止不算错误
    }
    if err == nil {
        statusChan <- fmt.Sprintf("Command completed successfully: %q", cmd.String())
    }
    close(statusChan)
    return err
}
```

**管道命令模式（多命令组合，如 kubectl | grep）** [exec.go:597-L614](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L597-L614)：
```go
last := len(cmds) - 1
for i := range cmds {
    cmds[i].Stderr = os.Stderr  // 所有命令的 stderr 直接输出
    if i+1 < len(cmds) {
        // 相邻命令之间通过 io.Pipe 连接
        r, w := io.Pipe()
        cmds[i].Stdout, cmds[i+1].Stdin = w, r
    }
}
cmds[last].Stdout = os.Stdout  // 最后一条命令输出到终端
```

### 2.3 execute 函数包装层

[exec.go:execute 172-L239行](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L172-L239) 负责在 pipe 之前做信号和清理准备：

```go
func execute(opts *shellOpts, statusChan chan<- string) error {
    if opts.clear {
        clearScreen()  // 清屏：打印 ANSI 转义序列 \033[H\033[2J
    }
    ctx, cancel := context.WithCancel(context.Background())
    defer func() {
        if !opts.background {
            cancel()      // 取消命令上下文，可能终止 kubectl
            clearScreen() // 恢复 TUI 前再次清屏
        }
    }()

    // 信号监听：捕获 Ctrl+C (SIGINT) 和终止信号
    var interrupted bool
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    go func(cancel context.CancelFunc) {
        defer slog.Debug("Got signal canceled")
        select {
        case sig := <-sigChan:
            slog.Debug("Command canceled with signal", slogs.Sig, sig)
            cancel()  // 用户按 Ctrl+C 时取消 context
        case <-ctx.Done():
            slog.Debug("Signal context canceled!")
        }
        interrupted = true
    }(cancel)

    // 构建命令
    cmds := make([]*exec.Cmd, 0, 1)
    cmd := exec.CommandContext(ctx, opts.binary, opts.args...)
    // KUBE_EDITOR 环境变量注入（用于 kubectl edit 等场景）
    if env := os.Getenv("K9S_EDITOR"); env != "" {
        cmd.Env = append(os.Environ(), fmt.Sprintf("KUBE_EDITOR=%s", ...))
    }
    cmds = append(cmds, cmd)
    // 支持 opts.pipes 管道命令追加
    for _, p := range opts.pipes {
        tokens := strings.Split(p, " ")
        cmds = append(cmds, exec.CommandContext(ctx, tokens[0], tokens[1:]...))
    }

    // 调用 pipe 完成真正的 IO 接管
    var o, e bytes.Buffer
    err := pipe(ctx, opts, statusChan, &o, &e, cmds...)
    if err != nil && !interrupted {
        return errors.Join(err, fmt.Errorf("%s", e.String()))
    }
    return nil
}
```

### 2.4 TUI 挂起：移交终端控制权的前提

在 `execute` 被调用前，必须先通过 `tview.Application.Suspend()` 挂起 TUI，释放对终端的控制。

**调用顺序** [exec.go:run 99-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L99-L122)：
```go
func run(a *App, opts *shellOpts) (ok bool, errC chan error, outC chan string) {
    errChan := make(chan error, 1)
    statusChan := make(chan string, 1)

    if opts.background {
        // 后台模式：不挂起 TUI，直接执行
        if err := execute(opts, statusChan); err != nil {
            errChan <- err
            a.Flash().Errf("Exec failed %q: %s", opts, err)
        }
        close(errChan)
        return true, errChan, statusChan
    }

    // ═══════════ 前台模式（exec/attach 走这里）═══════════
    a.Halt()         // 停止集群更新、文件监听等后台 goroutine
    defer a.Resume() // Suspend 返回后恢复（Suspend 是同步阻塞的）

    // a.Suspend() 同步阻塞，直到闭包执行完毕
    // 返回值顺序: suspend成功与否, errChan, statusChan
    return a.Suspend(func() {
        if err := execute(opts, statusChan); err != nil {
            errChan <- err                              // 错误写入 channel
            a.Flash().Errf("Exec failed %q: %s", opts, err)  // Flash 排队显示
        }
        close(errChan)
    }), errChan, statusChan
}
```

**关键注意事项：**
- `a.Suspend()` 是**同步阻塞**的，闭包执行完才返回
- `defer a.Resume()` 在 `Suspend` 返回后才执行（因为 Suspend 是最后一条 return 语句，defer 在函数返回前执行）
- `a.Flash().Errf()` 在 Suspend 闭包**内部**调用，但此时 TUI 已挂起，消息通过 `QueueUpdateDraw` 排队，TUI 恢复后才显示

**tview Suspend 底层实现**（来自 derailed/tview v0.8.5）：
```go
func (a *Application) Suspend(f func()) bool {
    screen := a.screen
    // 1. 挂起 tcell：tcsetattr 恢复终端标准模式（canonical + echo）
    if err := screen.Suspend(); err != nil { return false }
    // 2. 执行用户函数（同步阻塞，直到 f 返回）
    f()
    // 3. 恢复 tcell：tcsetattr 重新设置原始模式（raw + noecho）
    screen.Resume()
    return true
}
```

---

### 2.5 TUI 挂起状态下的执行状态回传机制

这是最容易误解的部分：**TUI 挂起期间没有实时状态回传到 UI，所有状态消息都在 TUI 恢复后才显示。**

#### 2.5.1 三条回传通道

| 通道 | 类型 | 传递内容 | 前台模式行为 | 后台模式行为 |
|------|------|----------|-------------|-------------|
| **errChan** | `chan error` (buffer=1) | 执行错误 | 闭包内写入，闭包返回后 `runK` 读取 | 同步写入，调用方读取 |
| **statusChan** | `chan string` (buffer=1) | 状态消息 | 命令执行完毕后写入 1 条成功消息 | 逐行写入命令输出 |
| **Flash** | UI 组件 | 用户可见的提示消息 | `QueueUpdateDraw` 排队，TUI 恢复后显示 | 直接显示 |

#### 2.5.2 前台模式（exec/attach）时序图

```
用户按 s 键
    ↓
runK() 调用 run()
    ↓
a.Halt() → 停止后台 goroutine
    ↓
a.Suspend(func() { ... }) 开始
    ├─ screen.Suspend() → 终端切到标准模式
    ├─ execute(opts, statusChan)
    │   ├─ clearScreen() → 清屏（ANSI 转义序列）
    │   ├─ 启动 SIGINT/SIGTERM 监听 goroutine
    │   ├─ cmd.Stdin/Stdout/Stderr = os.Stdin/os.Stdout/os.Stderr
    │   ├─ cmd.Run() ───────────────────────┐
    │   │                                    │ kubectl 接管终端
    │   │                                    │ （用户直接与 kubectl 交互）
    │   │  ← 用户退出 kubectl，cmd.Run() 返回
    │   ├─ 成功: statusChan <- "Command completed successfully: ..."
    │   ├─ 失败: 返回 error → errChan <- err
    │   │          + a.Flash().Errf(...) → QueueUpdateDraw 排队 ⚠️
    │   ├─ close(statusChan)
    │   └─ defer: cancel() + clearScreen()
    ├─ close(errChan)
    └─ [闭包结束，Suspend 继续执行]
    ↓
screen.Resume() → 终端切回原始模式，重绘 TUI
    ↓ （此时 Flash 队列中的消息才被绘制出来）
defer a.Resume() → 恢复后台 goroutine
    ↓
run() 返回 (suspended, errChan, statusChan)
    ↓
runK() 读取 errChan 和 statusChan
    ├─ statusChan 内容 → slog.Debug("stdout", ...) （仅日志，不显示）
    └─ errChan 内容 → 收集错误并 return errs
    ↓
回到视图按键处理函数
```

#### 2.5.3 statusChan 的真实作用

**前台模式（exec/attach）** [exec.go:pipe 574-L594](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L574-L594)：
```go
cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
_, _ = cmd.Stdout.Write([]byte(opts.banner))  // 打印 banner
err := cmd.Run()                               // 阻塞等待 kubectl 退出
if err == nil {
    // ⚠️ 只有执行成功时才写 1 条成功消息到 statusChan
    statusChan <- fmt.Sprintf("Command completed successfully: %q", cmd.String())
}
close(statusChan)  // 关闭 channel
```

**runK 中对 statusChan 的处理** [exec.go:runK 88-L90](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L88-L90)：
```go
for v := range stChan {
    slog.Debug("stdout", slogs.Line, v)  // ⚠️ 仅输出到 debug 日志！不显示在 UI！
}
```

> **重要修正**：statusChan 的成功消息**不会显示给用户**，只用于 debug 日志。用户看到的成功/失败提示完全来自 Flash 组件。

#### 2.5.4 Flash 消息的排队机制

**Flash.SetMessage 实现** [flash.go:70-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/ui/flash.go#L70-L85)：
```go
func (f *Flash) SetMessage(m model.LevelMessage) {
    fn := func() {
        if m.Text == "" { f.Clear(); return }
        f.SetTextColor(flashColor(m.Level))
        f.SetText(f.flashEmoji(m.Level) + " " + m.Text)
    }
    if f.testMode {
        fn()
    } else {
        f.app.QueueUpdateDraw(fn)  // ⚠️ 放入更新队列，不立即执行
    }
}
```

**关键机制：`QueueUpdateDraw`**
- 将 UI 更新函数放入 tview 的事件队列
- 在 Suspend 期间，screen 被挂起，主 event loop 暂停，队列中的操作不会执行
- Suspend 结束、screen.Resume() 后，event loop 恢复，队列中的 Flash 更新被一次性执行
- 用户看到的效果：TUI 恢复后，Flash 区域显示错误消息

#### 2.5.5 错误回传的双重机制

| 机制 | 作用对象 | 显示时机 | 用途 |
|------|----------|----------|------|
| **Flash 消息** | 用户（UI 可见） | TUI 恢复后立即显示 | 用户感知操作结果 |
| **errChan** | 调用方代码（UI 不可见） | Suspend 返回后读取 | 上层函数判断执行是否成功 |

**为什么需要双重机制？**
- `errChan` 供代码逻辑判断（如 `runK` 返回错误给 `shellIn`）
- `Flash` 供用户感知，且必须在 Suspend 闭包内触发才能保证消息在 TUI 恢复后第一时间显示
- 如果在 Suspend 返回后再调用 Flash，用户可能会看到短暂的空白再显示消息，体验不好

---

## 3. 终端尺寸同步功能深度分析

尺寸同步是 exec/attach 最复杂的部分。K9s 采取「完全移交」策略：TUI 挂起后，终端尺寸同步**完全由 kubectl 子进程自行处理**，K9s 不参与。

### 3.1 完整尺寸同步链路

```
用户调整终端窗口大小（拖拽、字体变化等）
    ↓
操作系统内核（TTY 层）检测到窗口变化
    ↓
内核向前台进程组发送 SIGWINCH 信号
    ↓  （此时前台进程组是 kubectl，因为 k9s 的 Suspend 没有切换进程组）
kubectl 进程的信号处理 goroutine 捕获 SIGWINCH
    ↓  [kubectl util/term/resizeevents.go]
signal.Notify(winch, unix.SIGWINCH) → winch channel 收到信号
    ↓
ioctl(STDOUT_FILENO, TIOCGWINSZ, &winsize) 读取新终端尺寸
    ↓  [kubectl util/term/resize.go monitorSize]
resizeEvents chan 接收新尺寸 → 发送到 sizeQueue.resizeChan
    ↓  [kubectl util/term/resize.go sizeQueue.Next]
client-go remotecommand.TerminalSizeQueue.Next() 阻塞读取新尺寸
    ↓  [client-go tools/remotecommand/v3.go handleResizes]
streamProtocolV3.handleResizes() goroutine 获取到 TerminalSize
    ↓
JSON 编码后写入 SPDY resize 子通道（channel 4）
    ↓  → API Server → kubelet → CRI Runtime
kubelet 收到 resize 请求 → 调用容器运行时 ResizeTTY 接口
    ↓
runtime (containerd/CRI-O) 调用 ioctl(TIOCSWINSZ) 设置容器 PTY 尺寸
    ↓
内核向容器内前台进程组发送 SIGWINCH 信号
    ↓
容器内 bash/vim 等程序收到信号 → 重新布局终端界面
```

### 3.2 kubectl 端尺寸同步源码实现

#### 3.2.1 SIGWINCH 信号监听

文件：`k8s.io/kubectl/pkg/util/term/resizeevents.go`（非 Windows 平台）
```go
// monitorResizeEvents 监听 SIGWINCH 信号并读取终端尺寸
func monitorResizeEvents(fd uintptr, resizeEvents chan<- TerminalSize, stop chan struct{}) {
    go func() {
        defer runtime.HandleCrash()
        winch := make(chan os.Signal, 1)
        signal.Notify(winch, unix.SIGWINCH)   // 注册 SIGWINCH 监听
        defer signal.Stop(winch)

        for {
            select {
            case <-winch:                      // 收到窗口变化信号
                size := GetSize(fd)            // ioctl 读新尺寸
                if size == nil { return }
                select {
                case resizeEvents <- *size:    // 非阻塞发送
                default:                       // 消费者慢则丢弃
                }
            case <-stop:
                return
            }
        }
    }()
}
```

#### 3.2.2 TerminalSizeQueue 实现

文件：`k8s.io/kubectl/pkg/util/term/resize.go`
```go
type sizeQueue struct {
    t            TTY
    resizeChan   chan TerminalSize    // client-go 从此读取
    stopResizing chan struct{}        // 停止信号
}

// 被 client-go 的 handleResizes goroutine 循环调用，阻塞等待新尺寸
func (s *sizeQueue) Next() *TerminalSize {
    size, ok := <-s.resizeChan
    if !ok { return nil }
    return &size
}

// TTY.MonitorSize() 启动整个尺寸监控流程
func (t *TTY) MonitorSize(initialSizes ...*TerminalSize) TerminalSizeQueue {
    outFd, isTerminal := term.GetFdInfo(t.Out)
    if !isTerminal { return nil }
    t.sizeQueue = &sizeQueue{
        t: *t,
        resizeChan:   make(chan TerminalSize, len(initialSizes)),
        stopResizing: make(chan struct{}),
    }
    t.sizeQueue.monitorSize(outFd, initialSizes...)
    return t.sizeQueue
}

// monitorSize 后台 goroutine 转发信号事件
func (s *sizeQueue) monitorSize(outFd uintptr, initialSizes ...*TerminalSize) {
    for i := range initialSizes {
        if initialSizes[i] != nil {
            s.resizeChan <- *initialSizes[i]  // 先发送初始尺寸
        }
    }
    resizeEvents := make(chan TerminalSize, 1)
    monitorResizeEvents(outFd, resizeEvents, s.stopResizing)  // 启动 SIGWINCH 监听

    go func() {
        defer runtime.HandleCrash()
        for {
            select {
            case size, ok := <-resizeEvents:
                if !ok { return }
                select {
                case s.resizeChan <- size:  // 转发给 client-go
                default:                     // 消费不及时则丢弃
                }
            case <-s.stopResizing:
                return
            }
        }
    }()
}
```

#### 3.2.3 TIOCGWINSZ 读取尺寸

文件：`k8s.io/kubectl/pkg/util/term/resize.go`
```go
func (t TTY) GetSize() *TerminalSize {
    outFd, isTerminal := term.GetFdInfo(t.Out)
    if !isTerminal { return nil }
    return GetSize(outFd)
}

// GetSize 通过 ioctl 系统调用读取终端窗口大小
func GetSize(fd uintptr) *TerminalSize {
    // 底层调用 term.GetWinsize(fd)
    // → syscall.Syscall(syscall.SYS_IOCTL, fd, uintptr(syscall.TIOCGWINSZ), uintptr(unsafe.Pointer(&winsize)))
    winsize, err := term.GetWinsize(fd)
    if err != nil {
        runtime.HandleError(fmt.Errorf("unable to get terminal size: %v", err))
        return nil
    }
    return &TerminalSize{Width: winsize.Width, Height: winsize.Height}
}
```

#### 3.2.4 client-go handleResizes 发送尺寸

文件：`k8s.io/client-go/tools/remotecommand/v3.go`
```go
// 由 SPDYExecutor 在建立流连接后启动的后台 goroutine
func (p *streamProtocolV3) handleResizes() {
    if p.resizeStream == nil || p.TerminalSizeQueue == nil {
        return
    }
    go func() {
        defer runtime.HandleCrash()
        encoder := json.NewEncoder(p.resizeStream)  // SPDY resize 子通道
        for {
            // 阻塞调用 Next()，等待 kubectl 的 SIGWINCH 处理器推送新尺寸
            size := p.TerminalSizeQueue.Next()
            if size == nil {
                return  // sizeQueue.stop() 被调用时返回 nil，退出循环
            }
            // JSON 编码后写入 resizeStream → API Server
            if err := encoder.Encode(&size); err != nil {
                runtime.HandleError(err)
            }
        }
    }()
}
```

### 3.3 TTY Safe 包装：确保终端状态恢复

kubectl 在整个 exec/attach 过程中使用 `TTY.Safe()` 包装，确保即使 panic 也能恢复终端状态：

文件：`k8s.io/kubectl/pkg/util/term/term.go`
```go
func (t TTY) Safe(fn SafeFunc) error {
    inFd, isTerminal := term.GetFdInfo(t.In)
    if !isTerminal && t.TryDev {
        if f, err := os.Open("/dev/tty"); err == nil {
            defer f.Close()
            inFd = f.Fd()
            isTerminal = term.IsTerminal(inFd)
        }
    }
    if !isTerminal { return fn() }

    // 保存终端当前状态（tcgetattr）
    var state *term.State
    var err error
    if t.Raw {
        state, err = term.MakeRaw(inFd)  // 设置原始模式（关行缓冲、关回显、关信号处理字符）
    } else {
        state, err = term.SaveState(inFd)
    }
    if err != nil { return err }

    // interrupt.Chain 确保：信号 → 先停止 resize 监控 → 再恢复终端状态
    return interrupt.Chain(t.Parent, func() {
        if t.sizeQueue != nil {
            t.sizeQueue.stop()         // 关闭 SIGWINCH 监听 goroutine
        }
        term.RestoreTerminal(inFd, state)  // tcsetattr 恢复终端设置
    }).Run(fn)
}
```

### 3.4 K9s 为何不处理 resize？

| 原因 | 说明 |
|------|------|
| **TUI 已挂起** | `screen.Suspend()` 后，tcell 不再读取 `tcell.EventResize` 事件，主 event loop 停止 |
| **进程组不变** | k9s 启动 kubectl 时未调用 `syscall.Setsid()`，kubectl 与 k9s 同属一个前台进程组，SIGWINCH 同时发送给两者，但 k9s 的 signal handler 在 TUI 挂起时已不处理 resize |
| **移交原则** | kubectl 有成熟的 SIGWINCH + TIOCGWINSZ 方案，重复实现易出错 |

---

## 4. 会话创建流程（补充）

### 4.1 前置检查（Pod Running 验证）

在 [pod.go:shellCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L222-L238) 中：
```go
if !podIsRunning(p.App().factory, path) {
    p.App().Flash().Errf("%s is not in a running state", path)
    return nil
}
```

### 4.2 容器选择逻辑

[pod.go:containerShellIn](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L358-L386)：
1. 若指定了容器名，直接使用
2. 检查 `kubectl.kubernetes.io/default-container` annotation
3. 若只有一个容器，直接使用
4. 多容器时弹出 Picker 选择器

### 4.3 命令参数构建

[pod.go:buildShellArgs](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L478-L503) 构建 kubectl 命令参数：
```go
args := []string{"exec", "-it"}  // 或 "attach"
args = append(args, "-n", namespace)
args = append(args, podName)
args = append(args, "-c", containerName)
// 追加用户命令: sh -c "command -v bash >/dev/null && exec bash || exec sh"
```

[pod.go:computeShellArgs](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L461-L468) 会根据 pod OS（Linux/Windows）选择不同的 shell 命令。

### 4.4 kubectl 命令增强参数

[exec.go:runK](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L57-L97) 会在执行前追加 K9s 配置参数：
```go
// 追加用户仿冒身份
if u, err := a.Conn().Config().ImpersonateUser(); err == nil {
    args = append(args, "--as", u)
}
if g, err := a.Conn().Config().ImpersonateGroups(); err == nil {
    args = append(args, "--as-group", g)
}
// 追加当前 context 名
args = append(args, "--context", a.Config.K9s.ActiveContextName())
// 追加 kubeconfig 路径
if cfg := a.Conn().Config().Flags().KubeConfig; cfg != nil && *cfg != "" {
    args = append(args, "--kubeconfig", *cfg)
}
```

---

## 5. 异常清理与会话收尾

采用**多层 defer 防御式编程**，确保各种异常路径下资源都能正确释放。

### 5.1 清理层级（从内到外）

| 层级 | 清理内容 | 代码位置 |
|------|----------|----------|
| L1 | `cancel()` 终止 kubectl + `clearScreen()` 清屏 | [exec.go:execute L177-L182](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L177-L182) defer |
| L2 | SIGINT/SIGTERM 信号监听 → 取消 context | [exec.go:execute L184-L197](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L184-L197) goroutine |
| L3 | `a.Resume()` 恢复后台任务（集群更新、文件监听） | [exec.go:run L113](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L113) defer |
| L4 | `c.Start()` 恢复视图刷新 | [pod.go:resumeShellIn L388-L401](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go#L388-L401) defer |
| L5 | `nukeK9sShell()` 删除 Node Shell 特权 Pod | [exec.go:launchPodShell L332-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L332-L337) defer |
| L6 | tview Run 函数 panic recover → `screen.Fini()` 恢复终端 | tview Application.Run defer |
| L7 | `App.BailOut()` → `nukeK9sShell()` 退出清理 | [app.go:BailOut L540-L542](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go#L540-L542) |

### 5.2 Node Shell 清理机制

Node Shell 会启动名为 `k9s-shell-{k9s_pid}` 的特权 Pod，通过命名带 pid 实现多实例隔离。

**三重清理保障**：
1. **启动前清理** [exec.go:launchNodeShell L301-L304](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L301-L304)：防止上次崩溃残留
2. **使用后清理** [exec.go:launchPodShell L332-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L332-L337) defer：正常退出路径
3. **应用退出清理** [app.go:BailOut L540-L542](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go#L540-L542)：整个应用退出时

**nukeK9sShell 实现** [exec.go:381-L405](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go#L381-L405)：
```go
func nukeK9sShell(a *App) error {
    // 检查 Feature Gate
    ct, err := a.Config.K9s.ActiveContext()
    if err != nil || !ct.FeatureGates.NodeShell { return nil }
    // 使用 k9s 进程 pid 构造 pod 名
    podName := fmt.Sprintf("k9s-shell-%d", os.Getpid())
    // 删除 pod，500ms 超时避免阻塞
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    err = a.factory.Client().CoreV1().Pods(ns).Delete(ctx, podName, metav1.DeleteOptions{})
    // 404 不算错误（pod 可能已不存在）
    if errors.IsNotFound(err) { return nil }
    return err
}
```

---

## 6. 完整调用链图

```
用户按键 (s/a)
    ↓
┌───────────────────────────────────────────────────────┐
│  多视图入口层（统一调用 containerShellIn/containerAttachIn） │
│  Pod 列表 → shellCmd                                   │
│  Container 列表 → shellCmd                             │
│  Xray 拓扑 → xray.shellCmd (提取 parent path)          │
│    ⚠️ Xray Container 节点: 只有 s 键，无 a 键         │
│    ⚠️ xray.attachCmd 中 CoGVR 分支为死代码              │
│  Node 列表 → sshCmd → launchNodeShell (启动特权 Pod)   │
└───────────────────────────────────────────────────────┘
    ↓
containerShellIn
    ├─ podIsRunning() 检查
    ├─ 多容器时弹出 Picker
    └─ resumeShellIn
        ├─ c.Stop()                 // L4: 停止视图刷新
        ├─ shellIn
        │   └─ runK
        │       ├─ exec.LookPath("kubectl")
        │       ├─ 追加 --as/--as-group/--context/--kubeconfig
        │       └─ run
        │           ├─ a.Halt()    // L3: 停止后台任务
        │           ├─ defer a.Resume()   // L3 defer
        │           └─ a.Suspend(f)       // ★ 同步阻塞！
        │               ├─ screen.Suspend()   // tcsetattr 恢复标准模式
        │               ├─ f() → execute      // L1 & L2: execute 函数
        │               │   ├─ clearScreen()
        │               │   ├─ ctx, cancel := WithCancel()
        │               │   ├─ defer: cancel() + clearScreen()  // L1 defer
        │               │   ├─ goroutine: SIGINT/SIGTERM → cancel  // L2
        │               │   ├─ exec.CommandContext(ctx, "kubectl", ...)
        │               │   └─ pipe(ctx, opts, statusChan, ..., cmds)
        │               │       └── 单命令模式:
        │               │           cmd.Stdin = os.Stdin   ★ IO 接管点
        │               │           cmd.Stdout = os.Stdout ★ IO 接管点
        │               │           cmd.Stderr = os.Stderr ★ IO 接管点
        │               │           cmd.Run() ───────────────┐
        │               │           ┌────────────────────────┘
        │               │           │ kubectl 子进程内部:
        │               │           │ ├─ SetupTTY: MakeRaw + SaveState
        │               │           │ ├─ TTY.MonitorSize():
        │               │           │ │   ├─ signal.Notify(SIGWINCH) ★ 尺寸监听点
        │               │           │ │   ├─ monitorResizeEvents goroutine
        │               │           │ │   └─ sizeQueue.Next() ← TerminalSizeQueue
        │               │           │ ├─ SPDYExecutor.Stream():
        │               │           │ │   └─ handleResizes goroutine:
        │               │           │ │       Next() → JSON → resizeStream ★ 尺寸发送点
        │               │           │ └─ TTY.Safe defer: stopResize + RestoreTerminal
        │               │           │
        │               │           ↓  命令退出，回到 pipe 函数
        │               │   成功: statusChan <- "Command completed successfully..."
        │               │   失败: return error → errChan <- err
        │               │           + a.Flash().Errf() → QueueUpdateDraw 排队 ⚠️
        │               │   close(statusChan)
        │               ├─ close(errChan)
        │               └─ [闭包结束，Suspend 继续]
        │               ↓
        │       screen.Resume()   // tcsetattr 原始模式 + 重绘（Flash 消息此时显示）
        │       ↓
        │   defer a.Resume()   // L3 defer: 恢复后台任务
        │   ↓
        └─ run() 返回 (suspended, errChan, statusChan)
            │
            ├─ runK 读取:
            │   ├─ statusChan → slog.Debug (仅日志，不显示！)
            │   └─ errChan → 收集错误 return errs
            │
        defer c.Start()   // L4 defer: 恢复视图刷新
```

---

## 7. 核心代码文件速查表

| 文件 | 核心函数/代码行 | 职责 |
|------|----------------|------|
| [pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/pod.go) | `shellCmd`(222-238), `attachCmd`(240-256), `containerShellIn`(358-386), `buildShellArgs`(478-503) | Pod 视图入口、容器选择、命令参数构建 |
| [container.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/container.go) | `shellCmd`(158-181), `attachCmd`(183-194) | Container 视图入口 |
| [xray.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/xray.go) | `shellCmd`(340-362), `attachCmd`(364-385), **key binding**(198-233) | Xray 拓扑图视图入口（CoGVR 无 a 键） |
| [node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/node.go) | `bindDangerousKeys`(75-77), `sshCmd`(179-191) | Node Shell 入口，Feature Gate 检查 |
| [exec.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/exec.go) | **`runK`**(57-97), **`run`**(99-122), `execute`(172-239), **`pipe`**(549-615), `nukeK9sShell`(381-405) | ★ 执行核心：TUI 挂起、IO 接管、状态回传、清理 |
| [flash.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/ui/flash.go) | `SetMessage`(70-85), `Watch`(57-67) | ★ 状态显示：`QueueUpdateDraw` 排队机制 |
| [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/app.go) | `Halt`(334-339), `Resume`(342-362), `BailOut`(540-542) | 后台任务启停、应用退出清理 |
| [actions.go](file:///d:/fz/0601-2/solo-dogfeeding/code/4-k9s/internal/view/actions.go) | `hotKeyActions`(60-106), `pluginActions`(115-174), `executePlugin`(206-265) | 热键（纯导航）、插件（通用命令执行） |
| *kubectl* util/term/term.go | `TTY.Safe()`, `MakeRaw`, `RestoreTerminal` | 终端原始模式设置、状态恢复 |
| *kubectl* util/term/resize.go | `TTY.MonitorSize()`, `sizeQueue.Next()`, `GetSize()` | ★ 尺寸同步核心、SIGWINCH 到 TerminalSizeQueue 的桥接 |
| *kubectl* util/term/resizeevents.go | `monitorResizeEvents()` | ★ SIGWINCH 信号监听、TIOCGWINSZ 读尺寸 |
| *client-go* remotecommand/v3.go | `handleResizes()` | SPDY resize 子通道尺寸发送 |
