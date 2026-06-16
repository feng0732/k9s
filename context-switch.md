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

## 七、并发边界全景分析

K9s 的并发模型基于**多层锁 + context 取消 + 原子标志**的混合策略。本节详细拆解四组并发要素：普通命令、别名重置、后台刷新、客户端锁之间的协同关系，并逐一指出未受保护的路径。

### 7.1 并发要素矩阵

以下是关键锁、它们的保护对象、持有者，以及协作关系：

| 锁/机制 | 类型 | 保护对象 | 所在文件 |
|---------|------|----------|----------|
| `Command.mx` | `sync.Mutex` | `Command.alias` 读写一致性（仅限 `Reset`/`Init`） | [command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L35-L39) |
| `Aliases.mx` | `sync.RWMutex` | `Aliases.Alias` map 的并发读写 | [alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/alias.go#L31-L34) |
| `APIClient.mx` | `sync.RWMutex` | 各客户端字段（client/dClient/mxsClient 等） | [client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L44-L55) |
| `Config.mx` | `sync.RWMutex` | `Config.flags`（ConfigFlags 指针） | [config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L33-L37) |
| `Factory.mx` | `sync.RWMutex` | `factories` map、`stopChan`、`forwarders` | [factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/watch/factory.go#L29-L35) |
| `K9s.mx` | `sync.RWMutex` | `activeConfig`、`activeContextName`、`contextSwitch`、`conn` | [k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L87-L99) |
| `cancelFn` | `context.CancelFunc` | 全局后台 goroutine 的生命周期 | [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L334-L362) |
| `Table.inUpdate` | `atomic.Int32` | 防止同一张表并发刷新 | [table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/model/table.go#L229-L247) |

### 7.2 `Command.mx` 的保护范围 vs 盲区

`Command.mx` 是最容易被误解的锁。让我们逐一检查 `Command` 的所有方法：

#### 受 `Command.mx` 保护的方法

只有两个方法持有该锁：

1. **`Init`**（初始化时加载别名）
2. **`Reset`**（上下文/命名空间切换时重建别名）

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L57-L101)

```go
func (c *Command) Init(path string) error {
    c.mx.Lock()                    // ✅ 加锁
    defer c.mx.Unlock()
    alias := NewAlias(c.app.factory)
    aliasMap, err := alias.Ensure(path)
    c.alias = alias
    c.alias.Alias = aliasMap
    return nil
}

func (c *Command) Reset(path string, nuke bool) error {
    c.mx.Lock()                    // ✅ 加锁
    defer c.mx.Unlock()
    if c.alias == nil {
        c.alias = NewAlias(c.app.factory)
    }
    if nuke {
        c.alias.Clear()            // Clear 有内部 RWMutex，双重保护
    }
    aliasMap, err := c.alias.Ensure(path)
    c.alias.Alias = aliasMap
    return nil
}
```

#### **不受 `Command.mx` 保护的方法（盲区）**

| 方法 | 作用 | 是否操作 alias | 风险 |
|------|------|----------------|------|
| `run()` | 执行普通命令（pod/svc/ctx...） | ✅ 通过 `viewMetaFor → alias.Resolve` 读取 | ⚠️ 并发 Reset 可能读到部分填充的 alias map |
| `exec()` | 创建视图并注入 | ❌ 不操作 alias | 安全 |
| `defaultCmd()` | 默认视图（ctx 或 pod） | ✅ 间接调用 `run()` | 同 `run` |
| `specialCmd()` | 特殊命令（ctx/xray/alias） | ✅ 通过 `aliasCmd` 读取 | 同 `run` |
| `contextCmd()` | `ctx <name>` 命令 | ❌ 不操作 alias | 安全 |
| `aliasCmd()` | 别名列表视图 | ✅ 创建 `Alias` DAO | 安全（只读） |
| `xrayCmd()` | Xray 视图 | ❌ 不操作 alias | 安全 |
| `viewMetaFor()` | 解析命令→GVR | ✅ `alias.Resolve` 读 alias map | ⚠️ 核心风险点 |
| `AliasesFor()` | 命令补全/提示 | ✅ 遍历 alias map | ⚠️ Reset 期间遍历旧 map |

最关键的风险链路：`Command.run()` → `viewMetaFor()` → `alias.Resolve()` → `Aliases.Get()`

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L176-L243) + [command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L315-L351)

```go
func (c *Command) run(p *cmd.Interpreter, fqn string, clearStack, pushCmd bool) error {
    // ...
    // ❌ 此处无 Command.mx 保护
    gvr, meta, interp, err := c.viewMetaFor(p)  // → 调用 alias.Resolve
    // ...
}

func (c *Command) viewMetaFor(p *cmd.Interpreter) (*client.GVR, *MetaViewer, *cmd.Interpreter, error) {
    // ...
    // ❌ 直接访问 c.alias 指针，无 Command.mx 读保护
    if c.alias != nil {
        gvr, ok = c.alias.Resolve(p)   // 内部有 Aliases.mx RLock，但指针本身无保护
    }
    // ...
}
```

**风险场景**：线程 A 在 `viewMetaFor` 里读 `c.alias` 指针（刚拿到，还没调用 Resolve），此时线程 B（上下文切换）在 `Reset` 里把 `c.alias.Clear()` + 重新填充。如果刚好 B 的 `Ensure` 正在执行 `Define`，A 的 `Resolve` 可能看到部分别名（即只有 k9s 默认别名，没有 context 特定别名和 CRD 别名）。

**实际危害等级：低**。因为 `Aliases` 内部有自己的 `mx RWMutex` 保护 map 本身（`Get/Define/Clear` 都加锁），且上下文切换期间 `Halt()` 会停止后台 goroutine 和 UI 事件循环，用户输入的命令在切换期间不会被 tview 的主循环处理。但如果在 `Resume()` 之后、`Reset()` 完成之前有异步 goroutine 触发了命令（例如配置文件 watcher 触发的 Reload），仍有竞态窗口。

### 7.3 后台刷新 goroutine 与客户端锁的协同

有四类后台 goroutine 持续运行（受 `cancelFn` context 控制）：

#### 7.3.1 集群连通性检查循环 `clusterUpdater`

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L364-L395)

```go
func (a *App) clusterUpdater(ctx context.Context) {
    // ...
    delay := clusterRefresh
    for {
        select {
        case <-ctx.Done():            // ✅ Halt() 时收到取消信号，退出
            return
        case <-time.After(delay):
            if err := a.refreshCluster(ctx); err != nil {
                // 指数退避
            }
        }
    }
}
```

`refreshCluster` 调用链：

```
refreshCluster(ctx)
  → a.Conn().CheckConnectivity()    // 调用 APIClient.CheckConnectivity()
  → a.factory.ValidatePortForwards() // 遍历端口转发列表
```

`CheckConnectivity` 内部通过 `APIClient.mx` 的 getter/setter 保护客户端字段的并发安全。

#### 7.3.2 资源视图刷新循环 `Table.updater`

文件：[table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/model/table.go#L203-L227)

```go
func (t *Table) updater(ctx context.Context) {
    rate := initRefreshRate
    for {
        select {
        case <-ctx.Done():           // ✅ 视图 Stop() 时退出
            return
        case <-time.After(rate):
            backoff.Retry(func() error {
                return t.refresh(ctx)  // → reconcile → list → DAO.List()
            }, backoff.WithContext(bf, ctx))
        }
    }
}
```

`refresh` 用 `atomic.CompareAndSwapInt32(&t.inUpdate, 0, 1)` 防止重入。list 操作最终会走 `factory.Client().Dial()`/`DynDial()`，这些方法内部有 `APIClient.mx` 保护。

#### 7.3.3 配置/皮肤/自定义视图 文件 watcher

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/ui/config.go#L191-L231)

```go
func (c *Configurator) ConfigWatcher(ctx context.Context, s synchronizer) error {
    w, _ := fsnotify.NewWatcher()
    go func() {
        for {
            select {
            case evt := <-w.Events:
                if evt.Name == AppConfigFile {
                    c.Config.Load(evt.Name, false)     // ⚠️ 无切换标志保护
                } else {
                    c.Config.K9s.Reload()             // ✅ 有 getContextSwitch() 检查
                }
                s.QueueUpdateDraw(func() { c.RefreshStyles(s) })
            case <-ctx.Done():                        // ✅ Halt() 时退出
                w.Close()
                return
            }
        }
    }()
}
```

**关键点**：`K9s.Reload` 会检查 `getContextSwitch()`，如果正在切换上下文则直接返回，避免竞态。但 `Config.Load`（加载全局 k9s.yaml）没有这个检查。

#### 7.3.4 `ClusterInfo.Reset` 异步 goroutine

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L519-L521)

```go
// 11. 异步重置集群模型
if a.clusterModel != nil {
    go a.clusterModel.Reset(a.factory)   // ⚠️ 在 defer Resume() 之前启动
}
```

这是一个**特殊的并发边界**：`go a.clusterModel.Reset(a.factory)` 启动时，`Halt()` 已经执行但 `Resume()` 还没执行（在 defer 里）。

`ClusterInfo.Reset` 的实现：

文件：[cluster_info.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/model/cluster_info.go#L118-L129)

```go
func (c *ClusterInfo) Reset(f dao.Factory) {
    c.mx.Lock()
    c.cluster, c.data = NewCluster(f), NewClusterMeta()   // 替换内部 Cluster 引用
    c.mx.Unlock()

    c.Refresh()   // → 调用 c.cluster.Metrics() → factory.Client() → APIClient
}
```

此处的 `factory` 参数是 `switchContext` 调用方传入的 `a.factory`，其内部的 `client.Connection` 已经完成了 `SwitchContext`，所以**数据是一致的**。但这个 goroutine 与 `Resume()` 之后启动的 `clusterUpdater` 之间存在重叠窗口：

```
时序：
  T1: switchContext 中 → Halt() 完成
  T2: go clusterModel.Reset(factory)  启动 goroutine G1
  T3: Resume() → go clusterUpdater(ctx) 启动 goroutine G2
  T4: G1 正在执行 c.Refresh() → 访问 factory.Client().Dial()
  T5: G2 正在执行 refreshCluster() → 访问 factory.Client().CheckConnectivity()
```

G1 和 G2 都调用 `APIClient` 的方法，但这些方法内部有 `APIClient.mx` 保护，所以客户端层面是安全的。但 `ClusterInfo.Refresh` 写 `c.data` 用了 `ClusterInfo.mx` 保护，`fireMetaChanged` 回调没有锁，两个 goroutine 可能先后触发 UI 更新，导致闪烁（功能正确但 UX 欠佳）。

### 7.4 别名重置与命令执行的并发细节

#### 7.4.1 别名重置流程

别名有**两层保护**：外层 `Command.mx` 和内层 `Aliases.mx`。

上下文切换时调用 `Command.Reset(aliasesPath, nuke=true)`，内部流程：

```
Command.Reset(path, true)
  [Command.mx.Lock]
    → alias.Clear()
        [Aliases.mx.Lock]         // 内层锁
          → delete 所有 alias 条目
        [Aliases.mx.Unlock]
    → alias.Ensure(path)
        → MetaAccess.LoadResources(factory)   // 加载 Discovery API（APIClient.mx 保护）
        → Alias.load(path)
            → Aliases.loadDefaultAliases()
                [Aliases.mx.Lock]
                  → declare() 设置 h/q/ctx/dir 等默认别名
                [Aliases.mx.Unlock]
            → EnsureAliasesCfgFile()
            → LoadFile(AppAliasesFile)
                [Aliases.mx.Lock]
                  → yaml.Unmarshal 到 a.Alias map
                  → 将所有值重写为 NewGVR 指针
                [Aliases.mx.Unlock]
            → LoadFile(contextPath)   // 同上，context 特定别名
            → 遍历 MetaAccess.AllGVRs()
              → Define(gvr, ...) 为每个标准资源设别名
                [Aliases.mx.Lock]
                  → a.Alias[alias] = gvr
                [Aliases.mx.Unlock]
            → 遍历 CRD GVRs → Define()（同上）
    → c.alias.Alias = aliasMap  // 替换指针
  [Command.mx.Unlock]
```

#### 7.4.2 命令执行读取别名的路径

用户在命令行输入 `po <Enter>` 触发：

```
App.gotoCmd(evt)
  → a.gotoResource("po", "", true, true)
      → command.run(NewInterpreter("po"), "", true, true)
          [❌ 无 Command.mx 保护]
          → viewMetaFor(p)
              → c.alias.Resolve(p)     // 指针直接访问
                  → Aliases.Get("po")    // [Aliases.mx.RLock]
                    → 查 a.Alias["po"] → 返回 GVR(v1/pods)
                  → Aliases.mx.RUnlock
          → exec() → 创建 Browser 视图
```

**竞态分析**：

| 场景 | 结果 | 原因 |
|------|------|------|
| 切换期间 `run` 与 `Reset` 同时调用 `Aliases.Get` 和 `Clear` | ✅ 安全 | `Aliases.mx` 保护 map 本身 |
| `Reset` 的 `Ensure` 中途，`run` 调用 `Resolve` | ⚠️ 读到部分别名 | 部分别名已 Define，部分还没，可能命令找不到但不会崩溃 |
| `Reset` 执行 `c.alias.Alias = aliasMap` 指针替换瞬间 | ✅ 安全 | Go 中指针赋值是原子操作（64 位平台） |
| `run` 拿到旧 `c.alias` 指针后，`Reset` 重建了新指针 | ⚠️ 使用旧别名 | 旧指针内容完整，但没有新 context 的 CRD 别名 |

**实际风险可控**的关键原因：tview 是单线程事件循环。用户按下 Enter 触发 `gotoCmd` 和切换上下文的 `useContext` 都在 tview 主 goroutine 中顺序执行，不会并行。只有以下异步路径可能触发并发：

1. `ClusterInfo.Reset` 的 goroutine（不读 alias，安全）
2. ConfigWatcher 的 `Reload()`（会跳过切换中状态）
3. 命令补全 suggestionFn（在用户输入时触发，同样在主循环）

### 7.5 未受命令互斥锁保护的代码路径清单

以下是**所有**绕过 `Command.mx` 直接或间接访问 alias/client 的代码路径：

#### 路径 1：命令补全建议 `suggestCommand`

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L195-L244)

```go
func (a *App) suggestCommand() model.SuggestionFunc {
    return func(s string) (entries sort.StringSlice) {
        // ❌ 无 Command.mx 保护，直接遍历 alias map
        for alias := range maps.Keys(a.command.alias.Alias) {
            if suggest, ok := cmd.ShouldAddSuggest(ls, alias); ok {
                entries = append(entries, suggest)
            }
        }
        // ❌ 无 Factory.mx 保护
        namespaceNames, err := a.factory.Client().ValidNamespaceNames()
    }
}
```

**风险**：
- 直接访问 `a.command.alias.Alias`（没有 `Aliases.mx.RLock`），与 `Reset` 的 `Clear/Define` 并发时可能读到不一致的 map（Go 1.6+ 中并发读写 map 会直接 panic）
- **这是本分析中发现的最高风险点**。`Reset` 的 `Clear()` 会遍历并删除所有 key，此时 `maps.Keys` 正在遍历同一个 map → **并发读写 panic**。

#### 路径 2：视图标题/菜单中的别名提示

文件：[browser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/browser.go#L291) + [xray.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/xray.go#L299) + [table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/table.go#L171-L177)

```go
// browser.go:291
return aliases(b.meta, b.app.command.AliasesFor(client.NewGVRFromMeta(b.meta)))
// table.go:175
for _, a := range t.command.Aliases() {
    cmds = append(cmds, a)
}
```

`AliasesFor` 的调用链：

```
Command.AliasesFor(gvr)
  → Alias.AliasesFor(gvr)
      → Aliases.AliasesFor(gvr)
          [Aliases.mx.RLock]    // ✅ 内层有读锁
            → 遍历 a.Alias
          [Aliases.mx.RUnlock]
```

**风险等级：低**。内层 `Aliases.mx.RLock` 保护了遍历。但指针本身 `Command.alias` 可能在 Reset 中被替换（`Command.Reset` 中 `c.alias.Alias = aliasMap` 是赋值给 map 字段，不是替换指针）。

更正：仔细看代码，`Command.Reset` 没有替换 `c.alias` 指针本身，只是操作内部的 `Alias` map：

```go
// Command.Reset:
c.alias.Clear()            // 清空 map
aliasMap, err := c.alias.Ensure(path)  // 重新填充 map
c.alias.Alias = aliasMap   // 直接替换 map 字段
```

指针 `c.alias` 不变（除非第一次 `Init`），变的是 `c.alias.Alias` 字段。而 `AliasesFor` 调用 `a.Aliases.AliasesFor`，有 `Aliases.mx.RLock` 保护，所以安全。

但 `suggestCommand` 的 `maps.Keys(a.command.alias.Alias)` 直接访问 map 字段，**绕过了 `Aliases.mx`**，与 `Clear/Define` 并发 → panic 风险。

#### 路径 3：Table/Browser 刷新循环中的 DAO 调用

`Table.updater` → `refresh` → `list` → `a.List(ctx, ns)` → 最终到 `APIClient.Dial()`/`DynDial()`。

这些路径都有 `APIClient.mx` 保护（每个 Dial 方法内部加锁），**安全**。

但 `CheckConnectivity` 中有一个小窗口：

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L307-L342)

```go
func (a *APIClient) CheckConnectivity() bool {
    defer func() {
        if err := recover(); err != nil {
            a.setConnOK(false)
        }
        if !a.getConnOK() {
            a.clearCache()   // 重建 LRU cache，无锁（不是 mx，是 cache 自身的锁）
        }
    }()

    cfg, err := a.config.RESTConfig()     // Config.mx 保护
    cfg.Timeout = a.config.CallTimeout()  // Config.mx 保护
    client, err := kubernetes.NewForConfig(cfg)

    if _, err := client.ServerVersion(); err == nil {
        a.setClient(client)               // APIClient.mx 保护
        if !a.getConnOK() {
            a.reset()                     // APIClient.mx 保护所有字段，但 cache 单独重建
        }
    }
    // ...
}
```

`a.clearCache()` 在 defer 中执行：

```go
func (a *APIClient) clearCache() {
    a.cache = cache.NewLRUExpireCache(cacheSize)   // ❌ 无 APIClient.mx 保护
}
```

与 `Dial()` → `CanI()` → `a.cache.Get()` 并发时，如果正在替换 cache 指针，理论上有风险。但 `cache.NewLRUExpireCache` 返回的是新指针，Go 的指针赋值是原子的（64 位对齐），`cache.Get/Add` 由 `cache` 内部锁保护，所以实际安全。

#### 路径 4：配置文件 watcher 触发的 `K9s.Reload`

文件：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L304-L330)

```go
func (k *K9s) Reload() error {
    if k.getContextSwitch() {     // ✅ 切换中跳过
        return nil
    }
    ctxName := k.getActiveContextName()
    ct, _ := k.ks.GetContext(ctxName)
    cfg, _ := k.dir.Load(k.getActiveContextName(), ct)
    k.setActiveConfig(cfg)        // K9s.mx 保护
    if cfg.Context.Proxy != nil {
        k.conn.Config().SetProxy(...)   // Config.mx 保护
    }
    k.Validate(k.conn, ctxName, ct.Cluster)
    return nil
}
```

**安全**：`getContextSwitch()` 检查 + 所有字段读写都有对应锁。

#### 路径 5：`ToggleContextSwitch` 本身的锁范围

文件：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L87-L99)

```go
func (k *K9s) ToggleContextSwitch(b bool) {
    k.mx.Lock()
    defer k.mx.Unlock()
    k.contextSwitch = b
}

func (k *K9s) getContextSwitch() bool {
    k.mx.RLock()             // ✅ 对应 RLock
    defer k.mx.RUnlock()
    return k.contextSwitch
}
```

**安全**：读写都加锁。

### 7.6 并发协同工作的完整时序

上下文切换期间，各并发机制如何协同：

```
 T1: 用户按 Enter → useContext("prod") 开始（tview 主 goroutine）
     │
     ├── Content.Top().Stop()
     │   └── Browser.Stop()
     │       ├── cancelFn() → Table.updater 等 goroutine 的 ctx 收到 Done ✅
     │       └── 移除 cmdBuff listener
     │
     ├── ToggleContextSwitch(true)   [K9s.mx.Lock]
     │   └── ConfigWatcher 后续触发的 Reload 将被跳过 ✅
     │
     ├── Config.Save(true)           （保存旧配置快照）
     │
     ├── dao.Context.Switch("prod")
     │   └── APIClient.SwitchContext("prod")  [APIClient.mx 内部保护]
     │       ├── Config.SwitchContext()        [Config.mx.Lock]
     │       ├── reset()                        [APIClient.mx 各 setter]
     │       ├── ResetMetrics()                 (全局单例指针替换，原子)
     │       ├── CheckConnectivity()            [APIClient.mx getter/setter]
     │       ├── DynDial()                      [APIClient.mx.Lock]
     │       └── invalidateCache()
     │
     ├── App.switchContext(ci, force=true)
     │   │
     │   ├── Halt()
     │   │   └── cancelFn() → 以下 goroutine 全部退出 ✅
     │   │       ├── clusterUpdater (连通性检查)
     │   │       ├── ConfigWatcher / SkinsWatcher / CustomViewsWatcher
     │   │       └── CustomJumpsWatcher
     │   │
     │   ├── Config.Reset() + ActivateContext()  [K9s.mx 各 setter]
     │   ├── Config.Save(true)
     │   ├── Factory.Terminate()  [Factory.mx.Lock]
     │   │   └── close(stopChan) → 所有 Informer 停止 ✅
     │   ├── Factory.Start(ns)    [Factory.mx 保护]
     │   │
     │   ├── Command.Reset(path, nuke=true)  [Command.mx.Lock]
     │   │   └── Aliases.Clear()/Ensure()    [Aliases.mx 保护]
     │   │
     │   ├── gotoResource(activeView)
     │   │   └── Command.run()  (主 goroutine 串行，无并发)
     │   │
     │   ├── go clusterModel.Reset(factory)  ← 启动 goroutine G1
     │   │   └── (异步执行，此时 Halt 已生效，后台 watcher 都已停)
     │   │
     │   └── [defer] Resume()
     │       ├── 新 ctx + cancelFn
     │       ├── go clusterUpdater(ctx)  ← 启动 goroutine G2
     │       ├── go ConfigWatcher(ctx)   ← 启动 goroutine G3
     │       └── ...重启其他 watcher
     │
     └── ToggleContextSwitch(false)  [defer, K9s.mx.Lock]
         └── ConfigWatcher 的 Reload 恢复正常
```

### 7.7 发现的并发缺陷汇总

| # | 位置 | 问题 | 触发条件 | 后果 | 严重程度 |
|---|------|------|----------|------|----------|
| 1 | [suggestCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L195-L244) L210 | `maps.Keys(a.command.alias.Alias)` 绕过 `Aliases.mx` 直接遍历 map | 用户在上下文切换进行中输入命令触发补全（极罕见，但可能） | Go runtime panic: concurrent map read and map write | **高** |
| 2 | [clearCache](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L588-L599) | `a.cache = ...` 无 `APIClient.mx` 锁 | `CheckConnectivity` 失败重建 cache 时，另一个 goroutine 在用旧 cache | 理论上的竞态，实际因指针赋值原子性+cache 内部锁，概率极低 | 低 |
| 3 | [switchContext](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L519-L521) L519-521 | `go clusterModel.Reset(a.factory)` 与 `Resume` 后 `clusterUpdater` 重叠 | 每次上下文切换 | UI 可能收到两次集群信息更新回调（闪烁） | 低（UX 问题） |

#### 缺陷 1 的修复建议

`suggestCommand` 应通过 `AliasesFor` 或加读锁来遍历：

```go
// 修复前（有问题）：
for alias := range maps.Keys(a.command.alias.Alias) {
    if suggest, ok := cmd.ShouldAddSuggest(ls, alias); ok {
        entries = append(entries, suggest)
    }
}

// 修复后：
if a.command.alias != nil {
    // Aliases.ShortNames() 内部有 Aliases.mx.RLock
    for gvr, aliasList := range a.command.alias.ShortNames() {
        for _, alias := range aliasList {
            if suggest, ok := cmd.ShouldAddSuggest(ls, alias); ok {
                entries = append(entries, suggest)
            }
        }
        _ = gvr // 忽略
    }
}
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
