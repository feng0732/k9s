# K9s 应用退出资源释放分析（对照代码事实）

> 本文所有代码位置采用仓库相对路径表示，格式：`相对路径:行号范围`（例如 `internal/view/app.go:L185-L193`），可在任意本地克隆或 GitHub/GitLab Web 界面中定位。

---

## 一、应用退出代码路径总览

### 1.1 正常退出路径（Ctrl+C）

```
tcell.KeyCtrlC 键盘事件
  ↓
quitCmd()  internal/view/app.go:L698-L712
  ↓
BailOut(0)  internal/view/app.go:L533-L547
  ├─ nukeK9sShell()       internal/view/exec.go:L381-L405   删除 k9s-shell pod（500ms 超时）
  ├─ stopImgScanner()     internal/view/app.go:L146-L150    vul.ImgScanner.Stop()
  ├─ factory.Terminate()  internal/watch/factory.go:L60-L72
  │   ├─ close(f.stopChan)                    通知所有 informer 停止
  │   ├─ delete(f.factories, k)               从 map 移除引用（无等待）
  │   └─ forwarders.DeleteAll()  internal/watch/forwarders.go:L85-L92
  │       └─ 遍历 f.Stop() → close(pf.stopChan)
  └─ ui.App.BailOut(exitCode)  internal/ui/app.go:L155-L162
      ├─ Config.Save()
      ├─ Application.Stop() → tcell.Screen.Fini() 恢复终端
      └─ os.Exit(exitCode)
```

### 1.2 连接丢失退出路径

```
clusterUpdater()  internal/view/app.go:L364-L395   每 15s 轮询
  ↓
refreshCluster()  internal/view/app.go:L397-L446
  ↓
conRetry >= MaxConnRetry
  ↓
ExitStatus = "Lost K8s connection..."
  ↓
BailOut(1)   （同上清理流程）
```

### 1.3 SIGHUP 信号退出（无清理）

```
syscall.SIGHUP
  ↓
initSignals() goroutine  internal/view/app.go:L185-L193
  ↓
os.Exit(0)   ⚠️  完全跳过所有清理
```

### 1.4 致命错误直接退出（无清理）

以下位置直接调用 `os.Exit()`，完全跳过清理流程：

| 代码位置 | 触发条件 |
|----------|----------|
| `internal/view/app.go:L278` | toggleHeader flex view 类型断言失败 |
| `internal/view/app.go:L294` | toggleCrumbs flex view 类型断言失败 |
| `cmd/root.go:L72` | cobra rootCmd.Execute() 返回错误 |
| `internal/ui/table_helper.go:L46` | 屏幕转储目录创建失败 |
| `internal/render/context.go:L93` | 上下文 YAML 加载失败 |
| `internal/config/helpers.go:L65` | 配置数据目录创建失败 |
| `internal/client/helpers.go:L95` | kubeconfig 加载失败 |
| `main.go:L27-L40` | klog flag 设置失败（panic） |

---

## 二、关闭信号处理机制

### 2.1 应用级信号注册

`internal/view/app.go:L185-L193`

```go
func (*App) initSignals() {
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGHUP)  // 只注册了 SIGHUP

    go func(sig chan os.Signal) {
        <-sig
        os.Exit(0)  // 直接退出，无任何清理
    }(sig)
}
```

**代码事实**：
- 只监听 `SIGHUP`，**不监听 `SIGTERM` 和 `SIGINT`**
- `SIGINT`（Ctrl+C）不是通过系统信号处理，而是通过 tcell 键盘事件 `tcell.KeyCtrlC`
- 收到 `SIGHUP` 直接 `os.Exit(0)`，完全跳过 BailOut 清理流程

### 2.2 Ctrl+C 键盘事件处理

键盘事件绑定 `internal/view/app.go:L254-L266`：

```go
func (a *App) bindKeys() {
    a.AddActions(ui.NewKeyActionsFromMap(ui.KeyMap{
        // ...
        tcell.KeyCtrlC: ui.NewKeyAction("Quit", a.quitCmd, false),
    }))
}
```

quitCmd 实现 `internal/view/app.go:L698-L712`：

```go
func (a *App) quitCmd(evt *tcell.EventKey) *tcell.EventKey {
    noExit := a.Config.K9s.NoExitOnCtrlC
    if a.InCmdMode() {
        if isBailoutEvt(evt) && noExit {
            return nil
        }
        return evt
    }
    if !noExit {
        a.BailOut(0)  // 走完整清理流程
    }
    return nil
}
```

### 2.3 Exec 期间的临时信号处理

在执行外部命令期间（kubectl exec/edit 等），会临时接管信号：

`internal/view/exec.go:L184-L197`

```go
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)  // 临时接管
go func(cancel context.CancelFunc) {
    defer slog.Debug("Got signal canceled")
    select {
    case sig := <-sigChan:
        slog.Debug("Command canceled with signal", slogs.Sig, sig)
        cancel()  // 只取消命令 context，不退出应用
    case <-ctx.Done():
        slog.Debug("Signal context canceled!")
    }
    interrupted = true
}(cancel)
```

**代码事实**：
- Exec 期间用 `signal.Notify` 全局接管 `os.Interrupt` 和 `syscall.SIGTERM`
- 命令执行完后**没有调用 `signal.Reset` 恢复默认处理**
- 这两个信号在 exec 期间只用于取消当前命令，不会触发应用退出

---

## 三、核心清理流程分析

### 3.1 BailOut 主清理流程

`internal/view/app.go:L533-L547`

```go
func (a *App) BailOut(exitCode int) {
    defer func() {
        if err := recover(); err != nil {  // 防御性 recover，防止清理过程 panic
            slog.Error("Bailout failed", slogs.Error, err)
        }
    }()

    if err := nukeK9sShell(a); err != nil {  // 1. 删除 k9s-shell pod
        slog.Error("Unable to nuke k9s shell pod", slogs.Error, err)
    }

    a.stopImgScanner()       // 2. 停止镜像扫描协程
    a.factory.Terminate()    // 3. 终止 informer 和端口转发
    a.App.BailOut(exitCode)  // 4. UI 层保存配置 + 恢复终端 + os.Exit
}
```

**注意顺序**：先清理 k8s 资源（pod/端口转发），再恢复终端退出。BailOut 自身的 `defer recover` 在函数返回时才执行，如果 `nukeK9sShell` panic，后面三行不会被执行。

**运行时退出 vs 启动失败 —— defer 执行完全不同**：

`a.App.BailOut(exitCode)` 内部最终调用 `os.Exit(exitCode)`（`internal/ui/app.go:L155-L162`），直接终止进程。但 `cmd/root.go` 中 `run()` 的 defer 是否执行，取决于退出发生在 `app.Run()` 之前还是之后：

| 退出场景 | 退出方式 | run() 正常返回？ | defer logFile.Close() | defer recover() |
|----------|---------|-----------------|----------------------|----------------|
| 启动阶段 `config.InitLocs()` 失败 | `run()` 返回 err → `Execute()` 中 `os.Exit(1)` | ✅ 是 | ✅ 执行 | ✅ 执行 |
| 启动阶段 `app.Init()` 失败 | `run()` 返回 err → `Execute()` 中 `os.Exit(1)` | ✅ 是 | ✅ 执行 | ✅ 执行 |
| 运行时 Ctrl+C → `BailOut(0)` | `BailOut` 内部 `os.Exit(0)` | ❌ 否 | ❌ 不执行 | ❌ 不执行 |
| 运行时连接丢失 → `BailOut(1)` | `BailOut` 内部 `os.Exit(1)` | ❌ 否 | ❌ 不执行 | ❌ 不执行 |
| 运行时 SIGHUP | 信号 goroutine 中 `os.Exit(0)` | ❌ 否 | ❌ 不执行 | ❌ 不执行 |
| 运行时 panic | `run()` 的 defer recover 捕获 → 返回 err → `os.Exit(1)` | ✅ 是 | ✅ 执行 | ✅ 执行并打印堆栈 |
| `app.Run()` 正常返回 + ExitStatus | `run()` 返回 err → `Execute()` 中 `os.Exit(1)` | ✅ 是 | ✅ 执行 | ✅ 执行 |

关键区别：启动阶段的错误通过 `return err` 从 `run()` 正常返回，defer 会执行；运行时退出通过 `os.Exit` 直接终止，defer 不会执行。两者虽然最终都到达 `os.Exit`，但 `run()` 的 defer 在 `Execute()` 的 `os.Exit` 之前已经执行完毕。

### 3.2 Halt / Resume 事件循环控制

`internal/view/app.go:L333-L362`

```go
func (a *App) Halt() {
    if a.cancelFn != nil {
        a.cancelFn()  // 只取消 context
        a.cancelFn = nil
    }
    // ⚠️  没有 WaitGroup，不等待任何 goroutine 退出
}

func (a *App) Resume() {
    var ctx context.Context
    ctx, a.cancelFn = context.WithCancel(context.Background())

    go a.clusterUpdater(ctx)  // 启动集群健康检查协程

    if a.Config.K9s.UI.Reactive {
        a.ConfigWatcher(ctx, a)       // 启动配置文件 watcher（内部起 goroutine）
        a.SkinsDirWatcher(ctx, a)     // 启动皮肤目录 watcher
        a.CustomViewsWatcher(ctx, a)  // 启动自定义视图 watcher
        a.CustomJumpsWatcher(ctx, a)  // 启动自定义跳转 watcher
    }
}
```

**代码事实**：
- `Halt()` 只调用 `cancelFn()`，**没有任何等待机制**
- 所有 watcher 内部都是 `go func() { for { select { ... case <-ctx.Done(): w.Close(); return } } }` 模式
- cancel context 后，这些 goroutine 会在各自 select 中收到 `ctx.Done()` 然后退出，但 **Halt 不等待它们确认退出**

### 3.3 clusterUpdater 协程退出方式

`internal/view/app.go:L364-L395`

```go
func (a *App) clusterUpdater(ctx context.Context) {
    // ...前置检查...
    bf := model.NewExpBackOff(ctx, clusterRefresh, 2*time.Minute)
    delay := clusterRefresh
    for {
        select {
        case <-ctx.Done():          // Halt() 取消 context 后走这里
            slog.Debug("ClusterInfo updater canceled!")
            return
        case <-time.After(delay):
            // ...执行 refreshCluster，失败重试...
            if delay = bf.NextBackOff(); delay == backoff.Stop {
                a.BailOut(1)        // 重试耗尽，主动退出应用
                return
            }
        }
    }
}
```

**代码事实**：
- clusterUpdater 本身退出就是 return，没有 cleanup
- 但 `backoff.Stop` 分支会调用 `a.BailOut(1)` 触发完整退出流程
- 这个协程在 Halt 时通过 ctx.Done() 干净退出

---

## 四、资源释放细节（对照代码事实）

### 4.1 Informer 释放

#### 4.1.1 Factory 结构与 stopChan

`internal/watch/factory.go:L28-L35`

```go
type Factory struct {
    factories  map[string]di.DynamicSharedInformerFactory
    client     client.Connection
    stopChan   chan struct{}        // 所有 informer 共用同一个停止 channel
    forwarders Forwarders
    mx         sync.RWMutex
}
```

#### 4.1.2 Terminate 实际行为（与 WaitForCacheSync 无关）

`internal/watch/factory.go:L60-L72`

```go
func (f *Factory) Terminate() {
    f.mx.Lock()
    defer f.mx.Unlock()

    if f.stopChan != nil {
        close(f.stopChan)   // 1. 关闭 stopChan，通知 client-go informer 库停止
        f.stopChan = nil
    }
    for k := range f.factories {
        delete(f.factories, k)  // 2. 从 map 中删除引用，交由 GC 清理
    }
    f.forwarders.DeleteAll()    // 3. 停止所有端口转发
}
```

**关键代码事实 —— Terminate 没有调用任何 WaitForCacheSync**：
- 关闭 `stopChan` 后，client-go informer 内部的 goroutine 会检测到 channel 关闭而自行退出
- k9s **不等待**这些 goroutine 实际退出，只是从 map 中删除引用
- 没有 `sync.WaitGroup`、没有 `time.Sleep`、没有任何同步原语确认 informer 已停止

#### 4.1.3 WaitForCacheSync 的真实用途

`WaitForCacheSync` **不是终止时用的**，它是**启动时**用的：

公有方法 `internal/watch/factory.go:L162-L173`：
```go
func (f *Factory) WaitForCacheSync() {
    for ns, fac := range f.factories {
        m := fac.WaitForCacheSync(f.stopChan)  // 用 stopChan 做超时信号
        // stopChan 被关闭的话 WaitForCacheSync 会立即返回 map[*]false
        for k, v := range m {
            slog.Debug("CACHE `%q Loaded %t:%s", ...)
        }
    }
}
```

私有方法 `waitForCacheSync(ns)` `internal/watch/factory.go:L140-L159`：
```go
func (f *Factory) waitForCacheSync(ns string) {
    // ...
    c := make(chan struct{})
    go func(c chan struct{}) {
        <-time.After(defaultWaitTime)  // 500ms 超时
        close(c)
    }(c)
    _ = fac.WaitForCacheSync(c)  // List/Get 时最多等 500ms 缓存同步
}
```

**区分要点**：

| 方法 | 用途 | 传入 channel | 何时被调用 |
|------|------|-------------|-----------|
| `Terminate()` | 终止所有 informer | 关闭 `f.stopChan` | 应用退出时 |
| `WaitForCacheSync()` | 等待所有 informer 初始同步完成 | `f.stopChan`（作为停止等待的信号） | 启动时（实际代码中未被调用） |
| `waitForCacheSync(ns)` | 单个 namespace 缓存同步（500ms 超时） | 500ms 后关闭的临时 channel | List/Get 数据时 |

**重要**：如果在调用 `WaitForCacheSync()` 之前 `stopChan` 已经被关闭，`fac.WaitForCacheSync` 会立即返回一个所有 value 为 `false` 的 map，因为 client-go 库中 `WaitForCacheSync` 会检测 stop channel。

#### 4.1.4 Informer 启动方式

`internal/watch/factory.go:L46-L57` 和 `internal/watch/factory.go:L250-L270`

```go
func (f *Factory) Start(ns string) {
    f.stopChan = make(chan struct{})  // Start 时创建新的 stopChan
    for ns, fac := range f.factories {
        fac.Start(f.stopChan)  // 每个 factory 用同一个 stopChan
    }
}

// ForResource 懒加载时也会启动
func (f *Factory) ForResource(ns string, gvr *client.GVR) (informers.GenericInformer, error) {
    fact, err := f.ensureFactory(ns)
    // ...
    fact.Start(f.stopChan)  // 重复调用 Start 是安全的（client-go 内部 sync.Once）
    return inf, nil
}
```

### 4.2 端口转发释放

#### 4.2.1 PortForwarder Stop

`internal/dao/port_forwarder.go:L102-L108`

```go
func (p *PortForwarder) Stop() {
    p.active = false
    if p.stopChan != nil {
        close(p.stopChan)  // 通知 k8s portforward 库停止
        p.stopChan = nil
    }
}
```

每个端口转发有自己独立的 `stopChan`，与 informer 共用的那个无关。k8s `portforward.NewOnAddresses` 内部会监听这个 channel，关闭后停止转发并关闭 SPDY 连接。

#### 4.2.2 批量释放 DeleteAll

`internal/watch/forwarders.go:L85-L92`

```go
func (ff Forwarders) DeleteAll() {
    for k, f := range ff {
        slog.Debug("Deleting forwarder", slogs.ID, f.ID())
        f.Stop()           // close(pf.stopChan)
        delete(ff, k)      // 从 map 移除
    }
}
```

**代码事实**：
- 同步遍历，逐个 Stop，逐个 delete
- Stop 只是 close channel，**不等待** k8s 库实际关闭连接
- 没有错误返回，无法确认转发是否成功关闭

### 4.3 终端状态恢复

#### 4.3.1 tview Application.Stop()

`internal/ui/app.go:L155-L162`

```go
func (a *App) BailOut(exitCode int) {
    if err := a.Config.Save(true); err != nil {
        slog.Error("Config save failed!", slogs.Error, err)
    }

    a.Stop()        // tview.Application.Stop()
    os.Exit(exitCode)
}
```

`tview.Application.Stop()` → `tcell.Screen.Fini()` 负责：
- 关闭 alternate screen buffer
- 恢复光标显示
- 重置终端 termios 属性（回显、规范模式等）
- 关闭 tty 文件描述符

#### 4.3.2 Exec 期间的 Suspend/Resume

`internal/view/exec.go:L99-L122`

```go
func run(a *App, opts *shellOpts) (ok bool, errC chan error, outC chan string) {
    // ...
    a.Halt()           // 取消 clusterUpdater 和文件 watcher 的 context
    defer a.Resume()   // 执行完后重建 context 重启

    return a.Suspend(func() {  // tview.Application.Suspend
        if err := execute(opts, statusChan); err != nil { ... }
        close(errChan)
    }), errChan, statusChan
}
```

Suspend 工作流程（tview/tcell 库内部）：
1. `tcell.Screen.Suspend()` 保存当前 termios 设置
2. 关闭 alternate screen，恢复正常终端模式
3. 执行用户回调（外部命令）
4. `tcell.Screen.Resume()` 恢复 termios 和 alternate screen

### 4.4 K9s Shell Pod 清理

`internal/view/exec.go:L381-L405`

```go
func nukeK9sShell(a *App) error {
    ct, err := a.Config.K9s.ActiveContext()
    if err != nil {
        return err
    }
    if !ct.FeatureGates.NodeShell || a.Config.K9s.ShellPod == nil {
        return nil  // 功能未开启，直接返回
    }

    ns := a.Config.K9s.ShellPod.Namespace
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()

    dial, err := a.Conn().Dial()
    if err != nil {
        return err
    }

    err = dial.CoreV1().Pods(ns).Delete(ctx, k9sShellPodName(), metav1.DeleteOptions{})
    if kerrors.IsNotFound(err) {
        return nil  // 已经不存在了，算成功
    }
    return err
}
```

**代码事实**：
- 超时只有 **500ms**，网络延迟高时删除请求可能超时
- 删除失败只返回 error，调用方 `BailOut` 只打日志不重试
- Pod 名字格式 `k9s-shell-<pid>`，如果 k9s 被 SIGKILL，这个 pod 永久残留

---

## 五、协程等待机制分析（对照代码事实）

### 5.1 WorkerPool —— 唯一有 Drain/Wait 的地方

`internal/pool.go`

```go
type WorkerPool struct {
    semC     chan struct{}
    errC     chan error
    ctx      context.Context
    cancelFn context.CancelFunc
    wg       sync.WaitGroup  // 工作协程
    wge      sync.WaitGroup  // 错误收集协程
}

func (p *WorkerPool) Drain() []error {
    if p.cancelFn != nil {
        p.cancelFn()
        p.cancelFn = nil
    }
    p.wg.Wait()    // ✅ 等待所有 worker 完成
    close(p.semC)
    close(p.errC)
    p.wge.Wait()   // ✅ 等待错误收集协程退出
    // ...返回收集到的错误
}
```

**关键事实**：`WorkerPool` 本身有完整的等待逻辑，但 **BailOut 退出流程中没有调用任何 Drain**。WorkerPool 的生命周期由使用它的业务代码自行管理。

### 5.2 组件 Start/Stop 的实际模式

以 Browser 组件为例：

`internal/view/browser.go:L164-L200`

```go
func (b *Browser) Start() {
    // ...
    b.Stop()                          // 先清理前一次的
    b.firstView.Store(0)
    b.GetModel().AddListener(b)
    b.Table.Start()
    b.CmdBuff().AddListener(b)
    if err := b.GetModel().Watch(b.prepareContext()); err != nil { ... }
}

func (b *Browser) Stop() {
    b.mx.Lock()
    if b.cancelFn != nil {
        b.cancelFn()                  // 只 cancel context
        b.cancelFn = nil
    }
    b.mx.Unlock()
    b.GetModel().RemoveListener(b)    // 移除 listener
    b.CmdBuff().RemoveListener(b)
    b.Table.Stop()                    // 链式调用子组件 Stop
}
```

Table model Watch 启动协程 `internal/model/table.go:L121-L128`：

```go
func (t *Table) Watch(ctx context.Context) error {
    if err := t.refresh(ctx); err != nil {
        return err
    }
    go t.updater(ctx)  // 启动轮询协程，没有 WaitGroup 追踪
    return nil
}
```

Table model updater `internal/model/table.go:L203-L221`：

```go
func (t *Table) updater(ctx context.Context) {
    bf := backoff.NewExponentialBackOff()
    // ...
    for {
        select {
        case <-ctx.Done():  // Stop() cancel context 后走这里
            return
        case <-time.After(rate):
            // ...刷新数据...
        }
    }
}
```

**代码事实**：
- `Stop()` 只做两件事：`cancelFn()` + 移除各种 Listener
- **没有任何 sync.WaitGroup 等待 goroutine 实际 return**
- 依赖 goroutine 在下一次 select 循环中检测到 `ctx.Done()` 自行退出
- 组件 Stop 返回时，updater goroutine 可能还在执行 `refresh()` 过程中

### 5.3 文件 Watcher 的退出

以 CustomViewsWatcher 为例 `internal/ui/config.go:L63-L99`：

```go
func (c *Configurator) CustomViewsWatcher(ctx context.Context, s synchronizer) error {
    w, err := fsnotify.NewWatcher()
    // ...
    go func() {
        for {
            select {
            case evt := <-w.Events:
                // ...处理事件...
            case err := <-w.Errors:
                slog.Warn(...)
                return
            case <-ctx.Done():
                slog.Debug("CustomViewWatcher canceled", ...)
                if err := w.Close(); err != nil {  // 关闭 fsnotify watcher
                    slog.Error("Closing CustomView watcher", slogs.Error, err)
                }
                return
            }
        }
    }()
    // ...
}
```

**代码事实**：
- 4 个 watcher（Config/Skins/CustomViews/CustomJumps）都是这个模式
- `ctx.Done()` 时会 `w.Close()` 然后 return
- 调用方（Halt）只 cancel context，**不等待 goroutine 完成 w.Close() 和 return**

### 5.4 PageStack 页面切换时的 Stop

`internal/view/page_stack.go:L44-L47`

```go
func (p *PageStack) StackPopped(o, top model.Component) {
    o.Stop()        // 被弹出的页面调用 Stop
    p.StackTop(top)
}
```

**注意**：这只是页面切换时的 Stop，应用退出时 `BailOut` 并没有遍历 PageStack 调 Stop——直接 factory.Terminate + os.Exit 了。各组件的 goroutine 通过各自持有的 context cancel 来感知。

### 5.5 未被任何方式等待的协程清单

| 协程 | 启动位置 | 停止触发 | 有 WaitGroup？ |
|------|----------|----------|---------------|
| clusterUpdater | `internal/view/app.go:L346` | Halt cancel ctx | ❌ |
| SIGHUP handler | `internal/view/app.go:L189` | os.Exit | ❌ |
| flash.Watch | `internal/view/app.go:L168` | ctx.Done() | ❌ |
| clusterModel.Refresh | `internal/view/app.go:L121` | 自行退出 | ❌ |
| vul.ImgScanner.Init | `internal/view/app.go:L163` | stopImgScanner() | ❌ |
| command.Reset | `internal/view/app.go:L435` | 自行退出 | ❌ |
| Table.updater | `internal/model/table.go:L125` | component Stop cancel ctx | ❌ |
| Log.updateLogs | `internal/model/log.go:L239` | Log.Stop() cancel ctx | ❌ |
| Tree.updater | `internal/model/tree.go:L88` | component Stop cancel ctx | ❌ |
| ConfigWatcher goroutine | `internal/ui/config.go:L197` | Halt cancel ctx | ❌ |
| SkinsDirWatcher goroutine | `internal/ui/config.go:L163` | Halt cancel ctx | ❌ |
| CustomViewsWatcher goroutine | `internal/ui/config.go:L69` | Halt cancel ctx | ❌ |
| CustomJumpsWatcher goroutine | `internal/ui/config.go:L115` | Halt cancel ctx | ❌ |
| Exec 信号监听 goroutine | `internal/view/exec.go:L187` | exec ctx cancel | ❌ |
| Exec 后台命令 goroutine | `internal/view/exec.go:L557` | 自行退出 | ❌ |
| client-go informer goroutines | internal/watch/factory.go Start / ForResource | close(f.stopChan) | ❌（由 client-go 内部管理） |
| k8s port-forward goroutines | internal/dao/port_forwarder.go Start | close(pf.stopChan) | ❌（由 k8s.io/client-go 内部管理） |

---

## 六、异常退出副作用（对照代码事实）

### 6.1 不同退出路径对清理的影响

| 清理动作 | 启动失败 (return err) | 运行时 Ctrl+C BailOut | 运行时连接丢失 BailOut | SIGHUP os.Exit | 其他 os.Exit(1) | SIGKILL |
|----------|---------------------|----------------------|----------------------|----------------|-----------------|---------|
| `nukeK9sShell()` 删除 pod | —（未到运行时） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `stopImgScanner()` | —（未到运行时） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `factory.Terminate()` 关 informer stopChan | —（未到运行时） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `forwarders.DeleteAll()` 关端口转发 stopChan | —（未到运行时） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `Config.Save()` 保存配置 | —（未到运行时） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `tcell.Screen.Fini()` 恢复终端 | —（未进入 TUI） | ✅ | ✅ | ❌ | ❌ | ❌ |
| `defer logFile.Close()` (cmd/root.go) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `defer recover()` (cmd/root.go) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

说明：
- 启动失败指 `app.Run()` 之前的错误（`config.InitLocs`、`app.Init` 等），此时 `run()` 正常返回，defer 执行
- 运行时退出指 `app.Run()` 已进入事件循环后的退出，`BailOut` 内部 `os.Exit` 直接终止，defer 不执行

### 6.2 资源残留的实际后果

**k9s-shell Pod 残留（最严重）**：
- 退出路径没走 `nukeK9sShell` 时，Pod 永久留在集群中
- Pod 名字包含 PID：`k9s-shell-<pid>`，重启 k9s 后 PID 变了，旧的不会被自动清理
- Pod 特权模式（`privileged: true`, `HostPID: true`, `HostNetwork: true`），有安全隐患
- 需手动清理：`kubectl delete pod -n <ns> k9s-shell-<pid>`

**终端状态损坏（用户可见）**：
- 不走 `tcell.Screen.Fini()` 时，termios 属性没恢复：
  - `ECHO` 位关闭 → 输入不显示
  - `ICANON` 规范模式可能被改 → 换行异常
  - alternate screen 没退出 → 之前的终端内容看不见
- 手动恢复：`reset` 或 `stty sane`

**Informer / 端口转发不主动关闭**：
- 进程退出后，OS 会关闭所有文件描述符（包括 K8s API Server 的 TCP 连接、本地端口转发监听 socket）
- **实际上不会有端口泄漏**，因为进程死了所有资源都被内核回收
- 但 K8s API Server 那边要等 TCP keepalive 超时才会清理对应的 SPDY 会话

**配置更改丢失**：
- 没走 `Config.Save()`，当前 session 对 context/namespace/view 的变更不会写盘

**日志文件可能不完整（运行时退出路径）**：
- `cmd/root.go:L88-L92` 中的 `defer logFile.Close()` 是 `run()` 函数的 defer
- 运行时退出（Ctrl+C、连接丢失等）通过 `BailOut` 内的 `os.Exit` 直接终止进程，`run()` 的 defer 不执行，`logFile.Close()` 不会被调用
- 启动阶段失败则走 `return err`，`logFile.Close()` 正常执行
- 运行时退出时日志是否真正丢失取决于写入链路的缓冲行为：
  - `tint.NewHandler` 每条日志调用 `Write()` 写入 `*os.File`
  - Go 的 `*os.File` 写操作直接走 `syscall.Write`，没有 Go 层面的用户态 buffer
  - 已通过 `syscall.Write` 写入的数据在内核 page cache 中，进程退出后内核会负责落盘
  - 因此已完成的日志行通常不会丢失；实际风险主要在 `logFile.Close()` 本身会做的收尾工作（如更新文件元数据）被跳过

### 6.3 关于 goroutine 的说明

进程退出时所有 goroutine 都会被操作系统强制终止，不存在 goroutine 泄漏。goroutine 是用户态调度单元，进程退出后其地址空间不再存在。所以"不等待 goroutine"在进程退出这个维度上没有副作用——但这也意味着 goroutine 中正在执行的写操作（如写文件、发网络请求）可能被中途截断。

---

## 七、完整代码路径追踪

### 7.1 Ctrl+C 正常退出完整调用链

```
1. 用户按键 Ctrl+C
   └─ tcell 从 stdin 读取到 ^C，转换为 tcell.KeyCtrlC 事件
      └─ tview Application 事件分发  internal/view/app.go:L246-L252
         └─ internal/view/app.go:L264 KeyMap 匹配 → 调用 quitCmd
            └─ internal/view/app.go:L698-L712 quitCmd
               └─ NoExitOnCtrlC=false 时 → a.BailOut(0)
                  └─ internal/view/app.go:L533-L547 BailOut
                     ├─ internal/view/exec.go:L381-L405 nukeK9sShell
                     │   └─ K8s API DELETE /api/v1/namespaces/<ns>/pods/k9s-shell-<pid>
                     │      timeout=500ms
                     ├─ internal/view/app.go:L146-L150 stopImgScanner
                     │   └─ vul.ImgScanner.Stop()
                     ├─ internal/watch/factory.go:L60-L72 factory.Terminate
                     │   ├─ close(f.stopChan)
                     │   │   └─ client-go informer 内部所有 reflector 检测到 channel 关闭
                     │   │      停止 ListWatch，退出 goroutine
                     │   ├─ for k := range f.factories { delete }
                     │   └─ internal/watch/forwarders.go:L85-L92 forwarders.DeleteAll
                     │       └─ for each: f.Stop() → close(pf.stopChan)
                     │          └─ k8s portforward 库检测到 stopChan 关闭
                     │             关闭 SPDY 连接，停止本地监听
                     └─ internal/ui/app.go:L155-L162 ui.App.BailOut
                        ├─ Config.Save(true) 写 YAML 到磁盘
                        ├─ a.Stop() → tview.Application.Stop()
                        │   └─ tcell.Screen.Fini()
                        │      ├─ 写入终端恢复序列（\033[?1049l 等）
                        │      ├─ tcsetattr 恢复 termios
                        │      └─ Close(tty fd)
                        └─ os.Exit(exitCode) → 进程终止
                           ⚠️  运行时退出：os.Exit 直接终止进程，不返回到 cmd/root.go 的 run()
                                因此 run() 中的 defer logFile.Close() 和 defer recover() 都不执行
                                （注意：启动阶段的错误走 return err，defer 会正常执行，两者不同）
```

### 7.2 连接丢失退出链

```
1. internal/view/app.go:L346 Resume() 中启动 go a.clusterUpdater(ctx)
   └─ for 循环每 15s  internal/view/app.go:L377-L394
      └─ internal/view/app.go:L397-L446 refreshCluster
         └─ a.Conn().CheckConnectivity() 失败
            └─ atomic.AddInt32(&a.conRetry, 1)
               └─ count >= MaxConnRetry 时
                  ├─ ExitStatus = "Lost K8s connection (N). Bailing out!"
                  └─ a.BailOut(1)
                     └─ (同上清理流程)
```

### 7.3 SIGHUP 退出链

```
1. 终端关闭 / kill -HUP <pid>
   └─ 内核发送 SIGHUP 给进程
      └─ Go runtime 分发信号
         └─ internal/view/app.go:L187 signal.Notify(sig, syscall.SIGHUP)
            └─ goroutine internal/view/app.go:L189-L192 从 sig channel 读出
               └─ os.Exit(0)
                  ├─ ⚠️  不恢复终端
                  └─ ⚠️  不删除 k9s-shell pod
                  （注：其他走 os.Exit 的退出路径也不会执行 cmd/root.go 的 defer logFile.Close()）
```

---

## 八、总结

### 8.1 已实现的清理（正常路径）

- ✅ 端口转发：每个转发器独立的 `stopChan` 被 close，通知 k8s 库停止
- ✅ Informer：所有 factory 共享的 `stopChan` 被 close，通知 client-go 停止 reflector
- ✅ 终端：tcell Screen.Fini() 恢复 termios 和正常屏幕
- ✅ k9s-shell pod：尝试删除（500ms 超时，失败只打日志）
- ✅ 配置：调用 Config.Save(true) 持久化 YAML
- ⚠️ 日志文件：运行时退出时 `defer logFile.Close()` 不执行（`os.Exit` 跳过 `run()` 的 defer）；启动失败时正常执行。实际数据丢失风险较低（见 6.2 节分析）

### 8.2 缺失与不足

- ❌ **没有任何协程等待**：所有 goroutine 通过 context cancel 通知退出，但从不等待确认
- ❌ **信号处理不完整**：只处理 SIGHUP（且直接 os.Exit 无清理），不处理 SIGTERM
- ❌ **多处硬退出**：8+ 处 os.Exit 跳过清理流程，其中 SIGHUP 最常见
- ❌ **Terminal restore 缺失**：异常退出后终端状态损坏概率高
- ❌ **Shell pod 无兜底**：异常退出 pod 永久残留，没有 finalizer 或二次清理
- ❌ **WaitForCacheSync 与 Terminate 语义混淆**：前者是启动时等缓存，后者只关 channel + 删引用，二者无任何关系

### 8.3 WaitForCacheSync vs Terminate 对照总结

| 维度 | WaitForCacheSync | Terminate |
|------|-----------------|-----------|
| **功能** | 等待 informer 完成初次 LIST 同步 | 关闭 stopChan 通知所有 informer 停止运行 |
| **时机** | 应用启动阶段（代码中实际未调用） | 应用退出阶段 |
| **对 stopChan 操作** | 把 stopChan 作为参数传给 client-go，用于"停止等待" | 直接 close(stopChan) |
| **是否阻塞** | 是，阻塞到所有 informer HasSynced 或 stopChan 关闭 | 否，close channel 后立即返回 |
| **与 goroutine 的关系** | 不关心 goroutine，只等缓存状态 | 不等待 goroutine，只发停止通知 |
| **在退出流程中被调用？** | ❌ 从未 | ✅ 是 |
