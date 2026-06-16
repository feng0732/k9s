# K9s 上下文切换：客户端与缓存重建机制

本文档对照源码，详细讲解 K9s 在 kubeconfig 上下文切换时，客户端（Client）和缓存（Cache）如何重建，以及默认命名空间和并发命令的处理逻辑。

---

## 一、整体调用链

上下文切换从 UI 触发到客户端重建，经过以下调用链：

```
用户操作 (Enter键/命令行)
  → view.Context.useCtx()           // UI层入口
    → view.useContext()              // 编排函数
      → dao.Context.Switch()        // DAO层委托
        → client.APIClient.SwitchContext()  // 核心重建逻辑
      → App.switchContext()         // 应用层重建（Config、Factory、别名等）
```

---

## 二、UI 层入口

### 2.1 用户触发

在上下文列表视图中，用户按下 Enter 键选中一个上下文，触发 `useCtx` 回调。

文件：[context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/context.go#L132-L144)

```go
func (c *Context) useCtx(app *App, _ ui.Tabular, gvr *client.GVR, path string) {
    if err := useContext(app, path); err != nil {
        app.Flash().Err(err)
        return
    }
    c.App().clearHistory()
    c.Refresh()
    c.GetTable().Select(1, 0)
}
```

### 2.2 `useContext` 编排函数

这是上下文切换的编排入口，依次完成：停止当前视图 → 获取 Switchable DAO → 保存配置 → 执行底层切换 → 应用层重建。

文件：[context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/context.go#L146-L173)

```go
func useContext(app *App, name string) error {
    // 1. 停止当前顶层视图的刷新
    if app.Content.Top() != nil {
        app.Content.Top().Stop()
    }

    // 2. 通过 DAO 注册表获取 Context 访问器，转型为 Switchable
    res, err := dao.AccessorFor(app.factory, client.CtGVR)
    switcher, ok := res.(dao.Switchable)

    // 3. 设置切换标志，防止 Reload 覆盖配置
    app.Config.K9s.ToggleContextSwitch(true)
    defer app.Config.K9s.ToggleContextSwitch(false)

    // 4. 保存当前配置到磁盘（切换前的快照）
    app.Config.Save(true)

    // 5. 执行底层上下文切换（Client + Cache 重建）
    switcher.Switch(name)

    // 6. 应用层上下文重建（Config、Factory、别名等）
    return app.switchContext(cmd.NewInterpreter("ctx "+name), true)
}
```

**关键点：`ToggleContextSwitch(true)`** — 这是一个互斥锁语义的标志位，用于告诉 `K9s.Reload()` 在上下文切换期间不要重新加载配置，避免覆盖正在切换的中间状态。

文件：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L87-L99)

```go
func (k *K9s) ToggleContextSwitch(b bool) {
    k.mx.Lock()
    defer k.mx.Unlock()
    k.contextSwitch = b
}
```

---

## 三、DAO 层委托

`dao.Context` 实现了 `Switchable` 接口，仅做一层简单委托：

文件：[context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/dao/context.go#L63-L65)

```go
func (c *Context) Switch(ctx string) error {
    return c.getFactory().Client().SwitchContext(ctx)
}
```

`Switchable` 接口定义：

文件：[types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/dao/types.go#L139-L143)

```go
type Switchable interface {
    Switch(ctx string) error
}
```

---

## 四、核心重建：`APIClient.SwitchContext`

这是整个上下文切换最关键的方法，负责客户端和缓存的彻底重建。

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L570-L586)

```go
func (a *APIClient) SwitchContext(name string) error {
    slog.Debug("Switching context", slogs.Context, name)

    // 1. 切换 Config 的 flags（底层 kubeconfig 指针）
    if err := a.config.SwitchContext(name); err != nil {
        return err
    }

    // 2. 重置所有内部客户端和缓存
    a.reset()

    // 3. 重置全局 MetricsServer 单例
    ResetMetrics()

    // 4. 用新 flags 重建 Config 对象
    a.config = NewConfig(a.config.flags)

    // 5. 连通性检查（同时预热 Dial 客户端）
    if !a.CheckConnectivity() {
        slog.Warn("SwitchContext: connectivity check failed", slogs.Context, name)
    }

    // 6. 预热 Dynamic Client
    if _, err := a.DynDial(); err != nil {
        slog.Warn("SwitchContext: DynDial pre-warm failed", slogs.Error, err)
    }

    // 7. 失效 Discovery 缓存
    return a.invalidateCache()
}
```

### 4.1 `Config.SwitchContext` — 重建 ConfigFlags

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L91-L111)

```go
func (c *Config) SwitchContext(name string) error {
    ct, err := c.GetContext(name)
    if err != nil {
        return fmt.Errorf("context %q does not exist", name)
    }

    // 创建全新的 ConfigFlags，设置新的 Context 和 ClusterName
    flags := genericclioptions.NewConfigFlags(UsePersistentConfig)
    flags.Context, flags.ClusterName = &name, &ct.Cluster

    // 保留原有的 Namespace、Timeout、KubeConfig、Impersonate 等设置
    flags.Namespace = c.flags.Namespace
    flags.Timeout = c.flags.Timeout
    flags.KubeConfig = c.flags.KubeConfig
    flags.Impersonate = c.flags.Impersonate
    flags.ImpersonateGroup = c.flags.ImpersonateGroup
    flags.ImpersonateUID = c.flags.ImpersonateUID
    flags.Insecure = c.flags.Insecure
    flags.BearerToken = c.flags.BearerToken

    c.flags = flags
    return nil
}
```

**关键设计**：不是修改现有 flags，而是 `NewConfigFlags(UsePersistentConfig=true)` 创建全新实例。`UsePersistentConfig=true` 表示使用持久化配置缓存，避免反复加载 kubeconfig 文件。

### 4.2 `APIClient.reset` — 彻底清空

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L588-L599)

```go
func (a *APIClient) reset() {
    a.config.reset()                         // Config.reset() 是空操作
    a.cache = cache.NewLRUExpireCache(cacheSize)  // 重建 LRU 缓存
    a.nsClient = nil                         // 清空 Namespace 客户端

    a.setDClient(nil)                        // 清空 Dynamic 客户端
    a.setMxsClient(nil)                      // 清空 Metrics 客户端
    a.setCachedClient(nil)                   // 清空 Discovery 缓存客户端
    a.setClient(nil)                         // 清空 Kubernetes 客户端
    a.setLogClient(nil)                      // 清空 Log 客户端
    a.setConnOK(true)                        // 重置连接状态
}
```

**重建的对象一览**：

| 字段 | 类型 | 作用 | reset 处理 |
|------|------|------|------------|
| `client` | `kubernetes.Interface` | 标准 K8s 客户端 | 置 nil |
| `logClient` | `kubernetes.Interface` | 日志专用客户端（无超时） | 置 nil |
| `dClient` | `dynamic.Interface` | 动态客户端（CRD等） | 置 nil |
| `nsClient` | `dynamic.NamespaceableResourceInterface` | NS 资源客户端 | 置 nil |
| `mxsClient` | `*versioned.Clientset` | Metrics 客户端 | 置 nil |
| `cachedClient` | `*disk.CachedDiscoveryClient` | Discovery 缓存客户端 | 置 nil |
| `cache` | `*cache.LRUExpireCache` | LRU 缓存（CanI、NS等） | 重建新实例 |

所有客户端均采用**懒初始化**模式：reset 时只清空，首次 Dial 时才重建。`reset()` 后所有字段为 nil/空，下一次 `Dial()`/`DynDial()`/`CachedDiscovery()` 等调用会根据新的 `config.flags` 重新创建。

### 4.3 `CheckConnectivity` — 连通性检查与客户端预热

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L307-L342)

```go
func (a *APIClient) CheckConnectivity() bool {
    defer func() {
        if err := recover(); err != nil {
            a.setConnOK(false)
        }
        if !a.getConnOK() {
            a.clearCache()
        }
    }()

    // 用新 Config 创建 REST 配置和 Client
    cfg, err := a.config.RESTConfig()
    cfg.Timeout = a.config.CallTimeout()
    client, err := kubernetes.NewForConfig(cfg)

    // 连通性验证：调用 ServerVersion
    if _, err := client.ServerVersion(); err == nil {
        a.setClient(client)    // 成功则缓存客户端，后续 Dial() 复用
        if !a.getConnOK() {
            a.reset()
        }
    } else {
        a.setConnOK(false)
    }

    return a.getConnOK()
}
```

**关键**：`CheckConnectivity` 不仅检查连通性，还会将成功创建的 client 缓存到 `a.client` 中，这样后续 `Dial()` 调用可以直接复用，无需再创建。

### 4.4 `invalidateCache` — Discovery 缓存失效

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L559-L567)

```go
func (a *APIClient) invalidateCache() error {
    dial, err := a.CachedDiscovery()
    if err != nil {
        return err
    }
    dial.Invalidate()   // 清除磁盘上的 Discovery 缓存
    return nil
}
```

`CachedDiscovery` 客户端的缓存路径基于 API Server 地址：

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L489-L518)

```go
func (a *APIClient) CachedDiscovery() (*disk.CachedDiscoveryClient, error) {
    // ...
    baseCacheDir := filepath.Join(mustHomeDir(), ".kube", "cache")
    httpCacheDir := filepath.Join(baseCacheDir, "http")
    discCacheDir := filepath.Join(baseCacheDir, "discovery", toHostDir(cfg.Host))
    // ...
}
```

不同集群的 API Server 地址不同，所以 Discovery 缓存天然按集群隔离。`Invalidate()` 确保切换后不会使用旧集群的 API 资源信息。

### 4.5 `ResetMetrics` — 全局 Metrics 单例重置

文件：[metrics.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/metrics.go#L39-L41)

```go
func ResetMetrics() {
    MetricsDial = nil
}
```

`MetricsServer` 是一个全局单例（`var MetricsDial *MetricsServer`），内部持有 `Connection` 和独立的 `cache.LRUExpireCache`。切换上下文时置 nil，下一次 `DialMetrics()` 调用会用新的 Connection 重建。

---

## 五、应用层重建：`App.switchContext`

`APIClient.SwitchContext` 完成底层重建后，`App.switchContext` 负责应用层的全面重建。

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L462-L522)

```go
func (a *App) switchContext(ci *cmd.Interpreter, force bool) error {
    contextName, ok := ci.HasContext()
    if (!ok || a.Config.ActiveContextName() == contextName) && !force {
        return nil  // 幂等：相同上下文不重复切换
    }

    // 1. 暂停应用事件循环
    a.Halt()
    defer a.Resume()

    {
        // 2. 重置 K9s 配置的活跃上下文
        a.Config.Reset()

        // 3. 激活新上下文配置
        ct, err := a.Config.ActivateContext(contextName)

        // 4. 如果命令行指定了命名空间，覆盖上下文的默认命名空间
        if cns, ok := ci.NSArg(); ok {
            ct.Namespace.Active = cns
        }

        // 5. 确定活跃视图
        p := cmd.NewInterpreter(a.Config.ActiveView())
        p.ResetContextArg()
        if p.IsContextCmd() {
            a.Config.SetActiveView(client.PodGVR.String())
        }

        // 6. 验证并设置活跃命名空间
        ns := a.Config.ActiveNamespace()
        if !a.Conn().IsValidNamespace(ns) {
            a.Config.SetActiveNamespace(ns)
        }

        // 7. 保存配置
        a.Config.Save(true)

        // 8. 初始化/重建 Factory
        if a.factory == nil && a.Conn() != nil {
            a.factory = watch.NewFactory(a.Conn())
            a.clusterModel = model.NewClusterInfo(a.factory, a.version, a.Config.K9s)
            a.clusterModel.AddListener(a.clusterInfo())
            a.clusterModel.AddListener(a.statusIndicator())
        }
        if a.factory != nil {
            a.initFactory(ns)
        }

        // 9. 重置命令别名
        a.command.Reset(a.Config.ContextAliasesPath(), true)

        // 10. 导航到活跃视图
        a.ReloadStyles()
        a.gotoResource(a.Config.ActiveView(), "", true, true)

        // 11. 异步重置集群模型
        if a.clusterModel != nil {
            go a.clusterModel.Reset(a.factory)
        }
    }
    return nil
}
```

### 5.1 `Halt` / `Resume` — 事件循环暂停与恢复

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L334-L362)

```go
func (a *App) Halt() {
    if a.cancelFn != nil {
        a.cancelFn()    // 取消 context，停止所有基于此 context 的 goroutine
        a.cancelFn = nil
    }
}

func (a *App) Resume() {
    var ctx context.Context
    ctx, a.cancelFn = context.WithCancel(context.Background())
    go a.clusterUpdater(ctx)     // 重启集群更新循环
    // ... 重启配置/皮肤/自定义视图 watcher
}
```

**`Halt` 取消 context 的作用**：所有使用该 context 的后台 goroutine（如集群健康检查、资源刷新）会收到取消信号而退出，确保切换期间不会有旧集群的请求在飞行。

### 5.2 `initFactory` — Informer Factory 重建

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L527-L530)

```go
func (a *App) initFactory(ns string) {
    a.factory.Terminate()   // 关闭旧 stopChan，清空所有 informer 工厂
    a.factory.Start(ns)     // 创建新 stopChan，启动新 informer 工厂
}
```

`Factory.Terminate()` 的实现：

文件：[factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/watch/factory.go#L60-L72)

```go
func (f *Factory) Terminate() {
    f.mx.Lock()
    defer f.mx.Unlock()

    if f.stopChan != nil {
        close(f.stopChan)   // 关闭 channel → 所有 informer 停止
        f.stopChan = nil
    }
    for k := range f.factories {
        delete(f.factories, k)   // 清空所有 namespace 的 informer 工厂
    }
    f.forwarders.DeleteAll()     // 关闭所有端口转发
}
```

`Factory` 内部维护了一个 `map[string]di.DynamicSharedInformerFactory`，每个 namespace 对应一个 informer 工厂。`Terminate` 关闭 stop channel 让所有 informer 停止 watches，然后清空 map。下次 `Start` 时用新客户端重建。

---

## 六、默认命名空间处理

命名空间的解析有多个层级，优先级从高到低：

### 6.1 优先级链

```
命令行 -n 参数
  → kubeconfig context.namespace 字段
    → k9s 配置文件中的 namespace.active
      → "default" 硬编码兜底
```

### 6.2 `Config.CurrentNamespaceName` — 命名空间解析

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L332-L344)

```go
func (c *Config) CurrentNamespaceName() (string, error) {
    // 优先级1：CLI -n 参数覆盖
    ns, overridden, err := c.clientConfig().Namespace()
    if overridden {
        return ns, nil
    }
    // 优先级2：kubeconfig context 中的 namespace
    return c.CurrentContextNamespace()
}
```

### 6.3 上下文激活时的命名空间处理

文件：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L256-L301)

```go
func (k *K9s) ActivateContext(contextName string) (*data.Context, error) {
    k.setActiveContextName(contextName)

    // 从 kubeconfig 获取上下文信息
    ct, err := k.ks.GetContext(contextName)

    // 从磁盘加载/生成 k9s 上下文配置
    cfg, err := k.dir.Load(contextName, ct)
    k.setActiveConfig(cfg)

    // 处理代理设置
    if cfg.Context.Proxy != nil {
        k.ks.SetProxy(...)
        k.conn.Config().SetProxy(...)
    }

    k.Validate(k.conn, contextName, ct.Cluster)

    // 命名空间决策：
    // 1. 如果 kubeconfig context 指定了 namespace → 使用它
    // 2. 如果 k9s 配置中 active namespace 为空 → 使用 "default"
    if ns := ct.Namespace; ns != client.BlankNamespace {
        k.getActiveConfig().Context.Namespace.Active = ns
    } else if k.getActiveConfig().Context.Namespace.Active == "" {
        k.getActiveConfig().Context.Namespace.Active = client.DefaultNamespace
    }

    return k.getActiveConfig().Context, nil
}
```

### 6.4 `Namespace.Validate` — 命名空间合法性校验

文件：[ns.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/data/ns.go#L62-L80)

```go
func (n *Namespace) Validate(conn client.Connection) {
    n.mx.RLock()
    defer n.mx.RUnlock()

    // 验证 active namespace 是否在集群中存在
    if conn == nil || !conn.IsValidNamespace(n.Active) {
        return   // 连接不可用时跳过校验，保留用户设置
    }
    // 清理不存在的收藏命名空间
    for _, ns := range n.Favorites {
        if !conn.IsValidNamespace(ns) {
            n.rmFavNS(ns)
        }
    }
    n.trimFavNs()
}
```

### 6.5 命名空间切换流程

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L448-L460)

```go
func (a *App) switchNS(ns string) error {
    if a.Config.ActiveNamespace() == ns {
        return nil   // 幂等：相同命名空间不重复切换
    }
    if ns == client.ClusterScope {
        ns = client.BlankNamespace
    }
    if err := a.Config.SetActiveNamespace(ns); err != nil {
        return err
    }
    return a.factory.SetActiveNS(ns)   // 确保 informer 工厂存在
}
```

---

## 七、并发命令处理

### 7.1 `Command.mx` — 命令执行互斥锁

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L35-L39)

```go
type Command struct {
    app   *App
    alias *dao.Alias
    mx    sync.Mutex    // 命令执行互斥锁
}
```

`Command.Reset` 方法使用此锁：

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L71-L87)

```go
func (c *Command) Reset(path string, nuke bool) error {
    c.mx.Lock()
    defer c.mx.Unlock()
    // ...
}
```

### 7.2 `APIClient.mx` — 客户端字段读写锁

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L44-L55)

```go
type APIClient struct {
    client, logClient kubernetes.Interface
    dClient           dynamic.Interface
    nsClient          dynamic.NamespaceableResourceInterface
    mxsClient         *versioned.Clientset
    cachedClient      *disk.CachedDiscoveryClient
    config            *Config
    mx                sync.RWMutex      // 保护所有客户端字段
    cache             *cache.LRUExpireCache
    connOK            bool
}
```

所有客户端字段的读写都通过 `mx` 保护的 getter/setter：

```go
func (a *APIClient) setClient(k kubernetes.Interface) {
    a.mx.Lock()
    defer a.mx.Unlock()
    a.client = k
}

func (a *APIClient) getClient() kubernetes.Interface {
    a.mx.RLock()
    defer a.mx.RUnlock()
    return a.client
}
```

### 7.3 `Config.mx` — Kubeconfig 配置读写锁

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L33-L37)

```go
type Config struct {
    flags *genericclioptions.ConfigFlags
    mx    sync.RWMutex
    proxy func(*http.Request) (*url.URL, error)
}
```

### 7.4 `Factory.mx` — Informer 工厂读写锁

文件：[factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/watch/factory.go#L29-L35)

```go
type Factory struct {
    factories  map[string]di.DynamicSharedInformerFactory
    client     client.Connection
    stopChan   chan struct{}
    forwarders Forwarders
    mx         sync.RWMutex
}
```

### 7.5 Halt/Resume 模式 — 上下文切换的并发安全

上下文切换期间，`App` 通过 `Halt()` 取消全局 context 来停止所有后台 goroutine。这确保了：

1. **不会有旧集群的 API 请求在飞行**：所有基于旧 context 的 HTTP 请求会被取消
2. **Informer watches 被关闭**：`Terminate()` 关闭 stopChan，所有 informer 停止
3. **配置文件不会被覆盖**：`ToggleContextSwitch(true)` 阻止 `Reload()` 在切换期间写入

```go
a.Halt()           // 取消 context → 后台 goroutine 退出
defer a.Resume()   // 重建 context → 重启后台 goroutine
```

---

## 八、完整切换时序图

```
用户选择上下文 "prod"
│
├── view.Context.useCtx("prod")
│   └── view.useContext(app, "prod")
│       ├── app.Content.Top().Stop()           // 停止当前视图
│       ├── dao.Context.Switch("prod")
│       │   └── APIClient.SwitchContext("prod")
│       │       ├── Config.SwitchContext("prod")   // 新建 ConfigFlags
│       │       ├── APIClient.reset()              // 清空所有客户端+缓存
│       │       ├── ResetMetrics()                 // 重置全局 Metrics 单例
│       │       ├── NewConfig(flags)               // 用新 flags 重建 Config
│       │       ├── CheckConnectivity()            // 连通性检查 + 预热 Dial 客户端
│       │       ├── DynDial()                      // 预热 Dynamic 客户端
│       │       └── invalidateCache()              // Discovery 缓存失效
│       │
│       └── App.switchContext(ci, force=true)
│           ├── Halt()                             // 取消 context → 停止后台 goroutine
│           ├── Config.Reset()                     // 清空活跃上下文名
│           ├── Config.ActivateContext("prod")
│           │   ├── K9s.ActivateContext("prod")
│           │   │   ├── ks.GetContext("prod")       // 获取 kubeconfig 上下文
│           │   │   ├── dir.Load("prod", ct)        // 加载/生成 k9s 上下文配置
│           │   │   ├── 设置 Proxy（如有）
│           │   │   └── 设置 Active Namespace
│           │   │       ├── kubeconfig ctx.namespace → 优先
│           │   │       └── "default" → 兜底
│           │   └── 验证命名空间合法性
│           ├── Config.Save(true)                  // 持久化配置
│           ├── Factory.Terminate()                // 关闭旧 informer + stopChan
│           ├── Factory.Start(ns)                  // 重建 informer
│           ├── Command.Reset(aliasesPath, nuke=true) // 重置别名
│           ├── ReloadStyles()                     // 重载皮肤
│           ├── gotoResource(activeView)           // 导航到活跃视图
│           ├── clusterModel.Reset(factory)         // 异步重置集群模型
│           └── Resume()                           // 重启后台 goroutine
│
└── 刷新上下文列表视图
```

---

## 九、缓存层次总结

| 缓存层 | 位置 | 重建方式 | 生命周期 |
|--------|------|----------|----------|
| LRU Auth 缓存 | `APIClient.cache` | `reset()` 重建新实例 | 每次 SwitchContext |
| Discovery 磁盘缓存 | `~/.kube/cache/discovery/<host>/` | `Invalidate()` 清除 | 每次 SwitchContext |
| HTTP 缓存 | `~/.kube/cache/http/` | 随 Discovery 重建 | 每次 SwitchContext |
| Metrics 缓存 | `MetricsServer.cache` | `ResetMetrics()` → nil | 每次 SwitchContext |
| Informer 本地缓存 | `Factory.factories[ns]` | `Terminate()` 清空 map | 每次 SwitchContext |
| K9s 上下文配置 | `K9s.activeConfig` | `Reset()` → `ActivateContext()` | 每次 SwitchContext |
| 命令别名 | `Command.alias` | `Reset(nuke=true)` 清空重建 | 每次 SwitchContext |

---

## 十、关键设计洞察

1. **懒初始化**：`reset()` 只清空不重建，客户端在首次 `Dial()` 时按需创建。`SwitchContext` 主动预热了 `Dial`（通过 `CheckConnectivity`）和 `DynDial`，因为这两个是后续操作最常用的。

2. **ConfigFlags 重建而非修改**：`SwitchContext` 创建全新的 `genericclioptions.ConfigFlags` 实例，避免了修改共享状态可能引发的并发问题。`UsePersistentConfig=true` 启用了客户端传输层缓存。

3. **Halt/Resume 模式**：通过 `context.WithCancel` 实现优雅的启停，而不是用锁阻塞。这确保切换期间不会有旧集群的请求或回调干扰新集群的状态。

4. **ToggleContextSwitch 标志**：防止配置文件 watcher 在上下文切换中间状态触发 `Reload()`，导致配置被覆盖。

5. **Discovery 缓存天然隔离**：不同集群的缓存路径基于 API Server 地址，但 `Invalidate()` 仍然必要——如果同一 API Server 有不同认证上下文，旧的缓存可能导致权限错误。
