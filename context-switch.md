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

### 4.1 `Config.SwitchContext` — 重建 ConfigFlags（**无锁**）

文件：[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L91-L111)

```go
func (c *Config) SwitchContext(name string) error {
    ct, err := c.GetContext(name)
    if err != nil {
        return fmt.Errorf("context %q does not exist", name)
    }
    // !!BOZO!! Do you need to reset the flags?
    flags := genericclioptions.NewConfigFlags(UsePersistentConfig)
    flags.Context, flags.ClusterName = &name, &ct.Cluster
    flags.Namespace = c.flags.Namespace
    flags.Timeout = c.flags.Timeout
    flags.KubeConfig = c.flags.KubeConfig
    flags.Impersonate = c.flags.Impersonate
    flags.ImpersonateGroup = c.flags.ImpersonateGroup
    flags.ImpersonateUID = c.flags.ImpersonateUID
    flags.Insecure = c.flags.Insecure
    flags.BearerToken = c.flags.BearerToken

    c.flags = flags   // ❌ 直接赋值指针，无 Config.mx 保护

    return nil
}
```

**锁边界事实**：整个 `SwitchContext` 方法 **完全没有持有 `Config.mx`**。`c.flags = flags` 是裸指针替换。同样：
- 读取 `c.flags.Namespace` / `c.flags.Timeout` 等字段也是**裸读**
- 唯一使用 `Config.mx` 的方法是 `ConfigAccess()`（RLock）和...没有其他方法了

`Config.mx` 这个 RWMutex 几乎是虚设的：在整个文件中，只有 `ConfigAccess()` 方法加了 `RLock`（[config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L348-L349)），其他所有读写 `c.flags` 的地方都没有锁。

**为什么当前安全**：`Config.SwitchContext` 只在 `APIClient.SwitchContext` 中调用，而后者在 `useContext → App.switchContext` 流程中被调用。因为 `Halt()` 先停止了所有后台 goroutine，所以此时没有其他线程在并发读取 `flags`。但这是**时序上的安全**，不是**锁的保护**。如果未来有任何 goroutine 在切换期间读取 `c.flags`（例如通过 `config.CurrentContextName()`），会有数据竞态。

### 4.2 `APIClient.reset` — 逐字段锁边界分析

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L588-L599)

```go
func (a *APIClient) reset() {
    a.config.reset()                         // 空函数
    a.cache = cache.NewLRUExpireCache(cacheSize)  // ❌ 直接替换指针，无 APIClient.mx
    a.nsClient = nil                         // ❌ 直接赋值，无 APIClient.mx

    a.setDClient(nil)                        // ✅ setDClient 内有 APIClient.mx.Lock
    a.setMxsClient(nil)                      // ✅ setMxsClient 内有 APIClient.mx.Lock
    a.setCachedClient(nil)                   // ✅ setCachedClient 内有 APIClient.mx.Lock
    a.setClient(nil)                         // ✅ setClient 内有 APIClient.mx.Lock
    a.setLogClient(nil)                      // ✅ setLogClient 内有 APIClient.mx.Lock
    a.setConnOK(true)                        // ✅ setConnOK 内有 APIClient.mx.Lock
}
```

**逐字段对比表（reset 中）**：

| 字段 | 处理方式 | 经过 APIClient.mx？ | 说明 |
|------|----------|---------------------|------|
| `config` | `a.config.reset()` — 空 | — | 不在 reset 中替换，在 SwitchContext 第 577 行单独替换 |
| `cache` | `a.cache = New...` | ❌ **直接替换，无锁** | 新指针替换旧指针 |
| `nsClient` | `a.nsClient = nil` | ❌ **直接赋值，无锁** | 死代码：整个项目从未读取过此字段（[client.go:591](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L591) 是唯一引用） |
| `dClient` | `setDClient(nil)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |
| `mxsClient` | `setMxsClient(nil)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |
| `cachedClient` | `setCachedClient(nil)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |
| `client` | `setClient(nil)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |
| `logClient` | `setLogClient(nil)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |
| `connOK` | `setConnOK(true)` | ✅ `mx.Lock` → 赋值 → `Unlock` |  |

### 4.3 `APIClient.SwitchContext` 逐行锁边界

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L570-L586)

```go
func (a *APIClient) SwitchContext(name string) error {
    slog.Debug("Switching context", slogs.Context, name)
    if err := a.config.SwitchContext(name); err != nil {
        return err                    // 内部 ❌ 无 Config.mx
    }
    a.reset()                         // 内部：cache/nsClient ❌ 无锁；其他 ✅ mx.Lock
    ResetMetrics()                    // 全局变量 MetricsDial = nil ❌ 无任何锁
    a.config = NewConfig(a.config.flags) // ❌ 直接替换 a.config 指针，无 APIClient.mx
    if !a.CheckConnectivity() {       // 内部：setClient/setConnOK ✅ mx.Lock
        slog.Warn("SwitchContext: connectivity check failed", slogs.Context, name)
    }

    if _, err := a.DynDial(); err != nil {  // DynDial: getDClient ✅ RLock；setDClient ✅ Lock
        slog.Warn("SwitchContext: DynDial pre-warm failed", slogs.Error, err)
    }
    return a.invalidateCache()              // CachedDiscovery: getCachedClient ✅ RLock
}
```

**SwitchContext 全流程逐字段锁追踪**：

| 步骤 | 操作 | 涉及字段 | 锁保护 |
|------|------|----------|--------|
| 1 | `config.SwitchContext(name)` | `Config.flags` 指针 | ❌ 无 Config.mx |
| 2.1 | `reset() → cache = New` | `APIClient.cache` 指针 | ❌ 无 APIClient.mx |
| 2.2 | `reset() → nsClient = nil` | `APIClient.nsClient` | ❌ 无 APIClient.mx（死代码） |
| 2.3 | `reset() → setDClient(nil)` | `APIClient.dClient` | ✅ `APIClient.mx.Lock` |
| 2.4 | `reset() → setMxsClient(nil)` | `APIClient.mxsClient` | ✅ `APIClient.mx.Lock` |
| 2.5 | `reset() → setCachedClient(nil)` | `APIClient.cachedClient` | ✅ `APIClient.mx.Lock` |
| 2.6 | `reset() → setClient(nil)` | `APIClient.client` | ✅ `APIClient.mx.Lock` |
| 2.7 | `reset() → setLogClient(nil)` | `APIClient.logClient` | ✅ `APIClient.mx.Lock` |
| 2.8 | `reset() → setConnOK(true)` | `APIClient.connOK` | ✅ `APIClient.mx.Lock` |
| 3 | `ResetMetrics()` | 全局 `MetricsDial` 指针 | ❌ 全局变量无锁 |
| 4 | `a.config = NewConfig(...)` | `APIClient.config` 指针 | ❌ 无 APIClient.mx |
| 5 | `CheckConnectivity()` | `APIClient.client`, `connOK`, `cache` | client/connOK ✅；cache ❌ |
| 6 | `DynDial()` | `APIClient.dClient` | ✅ `getDClient(RLock)` → 创建 → `setDClient(Lock)` |
| 7 | `invalidateCache()` | `APIClient.cachedClient` | ✅ `getCachedClient(RLock)` |

### 4.4 `CheckConnectivity` 逐行锁边界

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L307-L342)

```go
func (a *APIClient) CheckConnectivity() bool {
    defer func() {
        if err := recover(); err != nil {
            a.setConnOK(false)          // ✅ mx.Lock
        }
        if !a.getConnOK() {
            a.clearCache()              // ❌ clearCache 内无 mx（逐 Remove + cache 内部锁）
        }
    }()

    cfg, err := a.config.RESTConfig()    // ❌ a.config 指针裸读；flags 裸读
    if err != nil {
        a.connOK = false                 // ❌ 直接赋值，无 mx！与 setConnOK 的锁策略不一致
        return a.connOK
    }
    cfg.Timeout = a.config.CallTimeout() // ❌ a.config 裸读
    client, err := kubernetes.NewForConfig(cfg)
    if err != nil {
        a.setConnOK(false)               // ✅ mx.Lock
        return a.getConnOK()
    }

    if _, err := client.ServerVersion(); err == nil {
        a.setClient(client)              // ✅ mx.Lock
        if !a.getConnOK() {
            a.reset()                    // cache/nsClient ❌；其他 ✅
        }
    } else {
        a.setConnOK(false)               // ✅ mx.Lock
    }

    return a.getConnOK()                 // ✅ mx.RLock
}
```

**`CheckConnectivity` 的关键不一致**：
- `a.connOK = false`（[client.go:320](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L320)）是**裸赋值**，无 `APIClient.mx.Lock`
- 其他所有 connOK 写入（setConnOK(true/false)）都经 `mx.Lock`
- 读取：`getConnOK()` 经 `mx.RLock`，但 `ConnectionOK()`（[client.go:87-89](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L87-L89)）是**裸读** `a.connOK`

### 4.5 `Config()` getter 的锁边界

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L344-L347)

```go
func (a *APIClient) Config() *Config {
    return a.config   // ❌ 裸返回 Config 指针，无 APIClient.mx.RLock
}
```

这是 `APIClient` 中**所有客户端/配置 getter 里唯一不加锁的一个**。对照：

| getter 方法 | 是否加锁 |
|-------------|----------|
| `getClient()` | ✅ `mx.RLock` |
| `getLogClient()` | ✅ `mx.RLock` |
| `getDClient()` | ✅ `mx.RLock` |
| `getMxsClient()` | ✅ `mx.RLock` |
| `getCachedClient()` | ✅ `mx.RLock` |
| `getConnOK()` | ✅ `mx.RLock` |
| **`Config()`** | **❌ 无锁** |

### 4.6 `ResetMetrics` — 全局单例无锁替换

文件：[metrics.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/metrics.go#L26-L41)

```go
var MetricsDial *MetricsServer   // 全局变量

func DialMetrics(c Connection) *MetricsServer {
    if MetricsDial == nil {      // ❌ 全局变量裸读
        MetricsDial = NewMetricsServer(c)  // ❌ 全局变量裸写
    }
    return MetricsDial           // ❌ 全局变量裸读
}

func ResetMetrics() {
    MetricsDial = nil            // ❌ 全局变量裸写，无任何锁
}
```

`MetricsDial` 是包级全局指针，**完全没有互斥保护**。`DialMetrics` 中的 if-check+赋值也不是原子操作。但 `ResetMetrics` 只在 `SwitchContext` 中调用（Halt 已生效），所以当前时序下安全。

### 4.7 `invalidateCache` — Discovery 缓存失效

文件：[client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L559-L567)

```go
func (a *APIClient) invalidateCache() error {
    dial, err := a.CachedDiscovery()   // getCachedClient ✅ RLock
    if err != nil {
        return err
    }
    dial.Invalidate()                  // disk cache 自带内部锁处理
    return nil
}
```

`Invalidate()` 由 `disk.CachedDiscoveryClient` 内部实现。

---

## 附录 A：字段级锁归属完整矩阵

汇总 `APIClient` 和 `Config` 所有字段的**读路径锁**和**写路径锁**：

### A.1 `APIClient` 字段

| 字段 | 初始化/写入位置 | 写锁 | 读取位置 | 读锁 | 风险评估 |
|------|----------------|------|----------|------|----------|
| `client` | `setClient()` 在 CheckConnectivity/Dial 中 | ✅ `mx.Lock` | `getClient()` 在 Dial/ConnectionOK 路径 | ✅ `mx.RLock` | 安全 |
| `logClient` | `setLogClient()` 在 DialLogs 中 | ✅ `mx.Lock` | `getLogClient()` 在 DialLogs 中 | ✅ `mx.RLock` | 安全 |
| `dClient` | `setDClient()` 在 DynDial/reset 中 | ✅ `mx.Lock` | `getDClient()` 在 DynDial 中 | ✅ `mx.RLock` | 安全 |
| `mxsClient` | `setMxsClient()` 在 MXDial/reset 中 | ✅ `mx.Lock` | `getMxsClient()` 在 MXDial 中 | ✅ `mx.RLock` | 安全 |
| `cachedClient` | `setCachedClient()` 在 CachedDiscovery/reset 中 | ✅ `mx.Lock` | `getCachedClient()` 在 CachedDiscovery/invalidateCache 中 | ✅ `mx.RLock` | 安全 |
| `config` | 结构体初始化；`SwitchContext:L577` `a.config = NewConfig(...)` | ❌ 直接赋值 | `Config()` getter；`Dial/CallTimeout/RESTConfig` 等 | ❌ 裸读 | **潜在竞态**（Halt 时序保护） |
| `cache` | 结构体初始化；`reset()` `a.cache = New...`；`CheckConnectivity` defer 中 `clearCache()`（逐 Remove，内部有锁） | ❌ 直接替换 / `clearCache` 无 mx | `CanI/ServerVersion/ValidNamespaceNames/checkCacheBool/supportsMetricsResources` 中的 `cache.Get/Add` | ❌ 裸访问（但 LRUExpireCache 内部自带 mutex） | 指针替换为原子操作；内部 map 操作有 cache 自有锁。**当前安全** |
| `nsClient` | `reset()` 中 `= nil` | ❌ 直接赋值 | **整个代码库无读取者** | — | 死代码，无风险 |
| `connOK` | `InitConnection` 初始化(true)；`setConnOK()`；`CheckConnectivity:L320` 裸赋值 | ✅ / ❌ 有不一致 | `getConnOK()`；`ConnectionOK()` 裸读 | ✅ / ❌ 有不一致 | `L320 裸写 + ConnectionOK() 裸读` 构成竞态窗口（极低概率） |
| `log` | 结构体初始化，之后只读 | — | 各处 `slog.With` / `a.log.Debug/Warn` | — | 只读，安全 |
| `mx` | sync.RWMutex 零值 | — | — | — | — |

### A.2 `Config` 字段

| 字段 | 初始化/写入位置 | 写锁 | 读取位置 | 读锁 | 风险评估 |
|------|----------------|------|----------|------|----------|
| `flags` | `NewConfig()`；`SwitchContext:L108` `c.flags = flags` | ❌ 直接赋值 | 几乎所有 Config 方法（CurrentContext/Namespace/RESTConfig 等）；`ConfigAccess()` | ❌ 裸读 / ✅ `ConfigAccess` 有 RLock | **潜在竞态**（Halt 时序保护） |
| `proxy` | `SetProxy()` 中直接赋值 | ❌ 直接赋值 | `RESTConfig()` 中裸读 | ❌ 裸读 | 只读一次写入，极低频，安全 |
| `mx` | sync.RWMutex 零值 | — | — | — | 几乎不使用，仅 ConfigAccess 加 RLock |

### A.3 全局变量

| 变量 | 写入 | 写锁 | 读取 | 读锁 | 风险评估 |
|------|------|------|------|------|----------|
| `MetricsDial` | `DialMetrics`（懒创建）；`ResetMetrics()`（置 nil） | ❌ 无锁 | `DialMetrics`（每次调用） | ❌ 无锁 | 但 `ResetMetrics` 在 Halt 保护下，**当前安全** |
| `customViewers` | `Command.Init` 中赋值 `loadCustomViewers()` | ❌ 无锁 | 各处 view 创建时查询 | ❌ 裸读 | 启动时一次写入，之后只读，安全 |

---

## 附录 B：锁边界总结（避免笼统归类）

之前笼统地说"客户端和缓存重建都受 `APIClient.mx` 保护"是不准确的。事实是：

1. **6 个客户端字段**（client/logClient/dClient/mxsClient/cachedClient/connOK）经 `APIClient.mx` 读写锁保护——**正确**
2. **`cache` 字段指针替换**（`reset()` 中 `= New...`）**无** `APIClient.mx`，依赖指针原子性；内部 LRU map 操作由 cache 自带锁保护
3. **`nsClient`** 是死代码，**无**任何锁保护，但无人读取
4. **`config` 字段指针**（SwitchContext:L577）和 **`Config.flags` 指针**（SwitchContext:L108）**两层都无锁**，完全依赖 Halt 时序保障
5. **`Config.mx`** 声明了 RWMutex，但只在 `ConfigAccess()` 一个方法中使用了读锁——其余所有 flags 读写都是裸操作
6. **`connOK`**：`CheckConnectivity:320` 是裸写，与 `ConnectionOK()` 裸读形成**不一致的锁策略**
7. **全局 `MetricsDial`**：无任何锁，依赖调用时序

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

K9s 的并发模型基于**多层锁 + context 取消 + 原子标志**的混合策略。本节从代码事实出发，详细拆解四组并发要素：普通命令、别名重置、后台刷新、客户端锁之间的协同关系，并逐一指出未受保护的路径。

### 7.1 并发要素矩阵

| 锁/机制 | 类型 | 保护对象 | 所在文件 |
|---------|------|----------|----------|
| `Command.mx` | `sync.Mutex` | **仅** `Command.Reset` 中的 alias Clear/Ensure | [command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L38) |
| `Aliases.mx` | `sync.RWMutex` | `Aliases.Alias` map 的并发读写（Get/Define/Clear/AliasesFor/ShortNames/LoadFile/loadDefaultAliases） | [alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/alias.go#L33) |
| `APIClient.mx` | `sync.RWMutex` | 各客户端字段（client/dClient/mxsClient 等） | [client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/client.go#L44-L55) |
| `Config.mx` | `sync.RWMutex` | `Config.flags`（ConfigFlags 指针） | [config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/client/config.go#L33-L37) |
| `Factory.mx` | `sync.RWMutex` | `factories` map、`stopChan`、`forwarders` | [factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/watch/factory.go#L29-L35) |
| `K9s.mx` | `sync.RWMutex` | `activeConfig`、`activeContextName`、`contextSwitch`、`conn` | [k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/config/k9s.go#L87-L99) |
| `cancelFn` | `context.CancelFunc` | 全局后台 goroutine 的生命周期 | [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L334-L362) |
| `Table.inUpdate` | `atomic.Int32` | 防止同一张表并发刷新 | [table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/model/table.go#L229-L247) |

### 7.2 `Command.mx` 的保护范围 vs 盲区（代码事实纠正）

#### 事实：`Command.Init` **没有**使用 `Command.mx`

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L57-L68)

```go
func (c *Command) Init(path string) error {
    if c.app.factory != nil {
        c.alias = dao.NewAlias(c.app.factory)       // ❌ 无 mx.Lock
        if _, err := c.alias.Ensure(path); err != nil {
            slog.Error("Ensure aliases failed", slogs.Error, err)
            return err
        }
    }
    customViewers = loadCustomViewers()
    return nil
}
```

`Init` 直接赋值 `c.alias`，不持有 `Command.mx`。这在启动阶段没有问题（单线程），但如果存在并发调用 `Init` 的场景（目前代码中不存在），则 `c.alias` 指针的赋值没有互斥保护。

#### 事实：`Command.Reset` 是**唯一**使用 `Command.mx` 的方法

文件：[command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L71-L87)

```go
func (c *Command) Reset(path string, nuke bool) error {
    c.mx.Lock()                     // ✅ 这是 Command 中唯一加 mx.Lock 的方法
    defer c.mx.Unlock()

    if c.alias == nil {
        return nil                  // ❌ 注意：alias 为 nil 直接返回，不创建
    }

    if nuke {
        c.alias.Clear()             // Clear 内部有 Aliases.mx.Lock
    }
    if _, err := c.alias.Ensure(path); err != nil {
        return err
    }

    return nil
}
```

**关键区别**：`Init` 在 `c.alias == nil` 时创建新 alias；`Reset` 在 `c.alias == nil` 时直接返回。这意味着 `Reset` 依赖 `Init` 已经完成初始化。

#### 不受 `Command.mx` 保护的方法

| 方法 | 作用 | 是否读 alias | 锁保护情况 |
|------|------|-------------|------------|
| `Init()` | 首次加载别名 | ✅ 写 `c.alias` 指针 | ❌ 无 Command.mx |
| `run()` | 执行普通命令 | ✅ `viewMetaFor → alias.Resolve` | ❌ 无 Command.mx |
| `exec()` | 创建视图并注入 | ❌ | 不涉及 |
| `defaultCmd()` | 默认视图 | ✅ 间接调 `run()` | ❌ 同 run |
| `specialCmd()` | 特殊命令分发 | ✅ `xrayCmd → alias.Resolve` | ❌ 无 Command.mx |
| `contextCmd()` | ctx 命令 | ✅ `viewMetaFor → alias.Resolve` | ❌ 无 Command.mx |
| `aliasCmd()` | 别名列表视图 | ❌ 创建新 Alias DAO | 不涉及 |
| `xrayCmd()` | Xray 视图 | ✅ `c.alias.Resolve` | ❌ 无 Command.mx |
| `viewMetaFor()` | 解析命令→GVR | ✅ `c.alias.Resolve` | ❌ 无 Command.mx，内层 Aliases.Get 有 Aliases.mx |
| `AliasesFor()` | 命令补全/提示 | ✅ `alias.AliasesFor` | ❌ 无 Command.mx，内层 Aliases.AliasesFor 有 Aliases.mx |

**核心风险链路**：`Command.run()` → `viewMetaFor()` → `c.alias.Resolve(p)` → `Aliases.Get()`

```go
// command.go:315-334
func (c *Command) viewMetaFor(p *cmd.Interpreter) (*client.GVR, *MetaViewer, *cmd.Interpreter, error) {
    if c.alias == nil {                         // ❌ 读 c.alias 指针，无锁
        return client.NoGVR, nil, nil, fmt.Errorf("no connection available")
    }
    gvr, ok := c.alias.Resolve(p)              // ❌ 通过指针调用，无锁
    // ...
}
```

### 7.3 `Aliases.mx` 的精确保护范围

`Aliases.mx` 是真正保护 alias map 数据一致性的锁。逐方法核对：

| 方法 | 是否持有 Aliases.mx | 锁类型 |
|------|---------------------|--------|
| `Get(alias)` | ✅ | RLock |
| `Define(gvr, aliases...)` | ✅ | Lock |
| `Clear()` | ✅ | Lock |
| `AliasesFor(gvr)` | ✅ | RLock |
| `ShortNames()` | ✅ | RLock |
| `Resolve(p)` | **间接** — 自身不加锁，调用的 `Get` 加 RLock | — |
| `Load(path)` | **间接** — 调用 `loadDefaultAliases`（Lock）和 `LoadFile`（Lock） | — |
| `LoadFile(path)` | ✅ | Lock |
| `loadDefaultAliases()` | ✅ | Lock（内调无锁 `declare`） |
| `declare(gvr, aliases...)` | ❌ **无锁** — 但仅被 `loadDefaultAliases` 在持锁状态下调用 | — |
| `Save()` | ✅ | RLock |

**特别注意**：`Aliases.Resolve` 方法本身不加锁，但它只调用 `Get`（加 RLock）做查找，所以是安全的。然而 `Resolve` 可能在多次 `Get` 调用之间被 `Clear` 打断——即第一次 `Get` 找到了别名，第二次 `Get` 时 map 已被清空。这是语义层面的竞态，不会 panic（因为每次 `Get` 独立加锁），但可能导致命令解析到不完整的别名链。

### 7.4 后台刷新 goroutine 与客户端锁的协同

有四类后台 goroutine 持续运行（受 `cancelFn` context 控制）：

#### 7.4.1 集群连通性检查循环 `clusterUpdater`

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

#### 7.4.2 资源视图刷新循环 `Table.updater`

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

#### 7.4.3 配置/皮肤/自定义视图 文件 watcher

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

#### 7.4.4 `ClusterInfo.Reset` 异步 goroutine

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

### 7.5 别名重置与命令执行的并发细节

#### 7.5.1 别名重置流程（代码事实）

上下文切换时调用 `Command.Reset(aliasesPath, nuke=true)`，**只有外层 `Command.mx` 保护**，内部每个操作依赖 `Aliases.mx`：

```
Command.Reset(path, true)
  [Command.mx.Lock]                    ← 唯一持有 Command.mx 的位置
    → c.alias.Clear()
        [Aliases.mx.Lock]              ← 内层锁
          → delete 所有 alias 条目
        [Aliases.mx.Unlock]
    → c.alias.Ensure(path)
        → return a.Alias, a.load(path)  ← Ensure 返回 (config.Alias, error)
        → Alias.load(path)
            → Aliases.Load(path)
                → loadDefaultAliases()
                    [Aliases.mx.Lock]
                      → declare() 设置 h/q/ctx/dir 等默认别名  ← declare 自身无锁，但在持锁状态调用
                    [Aliases.mx.Unlock]
                → LoadFile(AppAliasesFile)
                    [Aliases.mx.Lock]
                      → yaml.Unmarshal 到 a.Alias map
                    [Aliases.mx.Unlock]
                → LoadFile(contextPath)
            → 遍历 MetaAccess.AllGVRs()
              → Define(gvr, ...)
                  [Aliases.mx.Lock]
                    → a.Alias[alias] = gvr
                  [Aliases.mx.Unlock]
            → 遍历 CRD GVRs → Define()
    ← Reset 中丢弃了 Ensure 返回的 aliasMap
  [Command.mx.Unlock]
```

**重要纠正**：之前文档称 `Reset` 中有 `c.alias.Alias = aliasMap` 赋值。实际代码中 `Reset` 完全没有使用 `Ensure` 的返回值 `aliasMap`。`Ensure` 返回 `a.Alias`（当前 map 的快照），但 `Reset` 丢弃了这个返回值——别名数据已经通过 `load → Define` 直接写入了 `c.alias.Aliases.Alias` map 中，不需要额外赋值。

`Command.Init` 的实现不同——它也不使用 `Ensure` 返回的 `aliasMap`，而是直接使用 `c.alias` 指针：

```go
func (c *Command) Init(path string) error {
    if c.app.factory != nil {
        c.alias = dao.NewAlias(c.app.factory)   // 创建新 Alias 实例
        if _, err := c.alias.Ensure(path); err != nil {
            // Ensure 内部已经填充了 alias.Alias map
            return err
        }
    }
    // ...
}
```

#### 7.5.2 命令执行读取别名的路径

用户在命令行输入 `po <Enter>` 触发：

```
App.gotoCmd(evt)
  → a.gotoResource("po", "", true, true)
      → Command.run(NewInterpreter("po"), "", true, true)
          [❌ 无 Command.mx 保护]
          → Command.viewMetaFor(p)
              → c.alias 指针读取                 [❌ 无 Command.mx]
              → c.alias.Resolve(p)
                  → Aliases.Get("po")             [✅ Aliases.mx.RLock]
                    → 查 a.Alias["po"] → GVR(v1/pods)
                  → Aliases.mx.RUnlock
          → Command.exec() → 创建 Browser 视图
```

#### 7.5.3 补全建议读取别名的路径

用户输入字符触发补全：

```
FishBuff.Add(rune) → FishBuff.Notify()
  → FishBuff.suggestionFn(text)
      → App.suggestCommand() 返回的闭包
          → maps.Keys(a.command.alias.Alias)    [❌ 直接遍历 map，无 Aliases.mx]
          → a.factory.Client().ValidNamespaceNames()
```

**这是最高风险点**。`suggestCommand` 闭包中 `maps.Keys(a.command.alias.Alias)` 直接遍历 Go map，而 `Aliases.mx` 完全没有介入。如果同时 `Reset` 在执行 `Clear/Define`（通过 `Aliases.mx.Lock`），则 Go runtime 会检测到并发读写 map 并直接 panic。

#### 7.5.4 视图标题/菜单中别名提示的路径

```
Browser.Aliases()
  → Command.AliasesFor(gvr)
      → c.alias.AliasesFor(gvr)
          → Aliases.AliasesFor(gvr)
              [Aliases.mx.RLock]           ✅ 安全
              → 遍历 a.Alias
              [Aliases.mx.RUnlock]

Xray.Aliases()
  → Command.AliasesFor(gvr)               ✅ 同上

Table.Start()
  → t.command.Aliases()                    ← 这是 cmd.Interpreter.Aliases()
  ← 注意：Table.command 是 *cmd.Interpreter，不是 *Command
  ← Interpreter.Aliases() 只返回字符串切片，不涉及 alias map
```

**纠正**：之前文档将 `Table.command.Aliases()` 误认为与 `Command.alias` 有关。实际上 `Table.command` 的类型是 `*cmd.Interpreter`（不是 `*Command`），其 `Aliases()` 方法只返回解析时保存的字符串切片，完全不涉及 `Aliases.Alias` map。**此处安全**。

同样，`Alias.aliasContext` 中 `a.App().command.alias` 传递的是 `*dao.Alias` 指针给 context value，后续的 `List` 方法使用 `ShortNames()`（有 `Aliases.mx.RLock`），**安全**。

### 7.6 未受命令互斥锁保护的代码路径清单（修正版）

| # | 路径 | 操作 | Command.mx | Aliases.mx | 风险 |
|---|------|------|------------|------------|------|
| 1 | `Command.Init` | 写 `c.alias` 指针 | ❌ 无 | ✅ Ensure 内部有 | 启动阶段单线程，实际安全 |
| 2 | `Command.run/viewMetaFor` | 读 `c.alias` + `Resolve` | ❌ 无 | ✅ Get 有 RLock | Reset 期间可能读到部分填充的 map，不 panic |
| 3 | `Command.xrayCmd` | 读 `c.alias.Resolve` | ❌ 无 | ✅ Get 有 RLock | 同上 |
| 4 | `App.suggestCommand` | `maps.Keys(a.command.alias.Alias)` | ❌ 无 | ❌ **无** | **并发读写 map → panic** |
| 5 | `Browser/Xray.Aliases` | `Command.AliasesFor` | ❌ 无 | ✅ AliasesFor 有 RLock | 安全 |
| 6 | `Alias.aliasContext` | 传 `*dao.Alias` 给 ctx | ❌ 无 | ✅ List 用 ShortNames(RLock) | 安全 |
| 7 | `Table.Start → t.command.Aliases()` | `cmd.Interpreter.Aliases()` | N/A | N/A | 不涉及 alias map，安全 |

### 7.7 并发协同工作的完整时序

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
     │   └── APIClient.SwitchContext("prod")  [⚠️ 非全部有锁]
     │       ├── Config.SwitchContext()        [❌ 无 Config.mx — 裸替换 flags 指针]
     │       ├── reset()                        [部分有锁：6 个 setter 走 APIClient.mx.Lock；cache/nsClient 直接替换]
     │       ├── ResetMetrics()                 [❌ 全局变量 MetricsDial = nil，无任何锁]
     │       ├── a.config = NewConfig(...)      [❌ 无 APIClient.mx — 裸替换 config 指针]
     │       ├── CheckConnectivity()            [client/connOK 经 mx；connOK L320 裸写不一致]
     │       ├── DynDial()                      [getDClient(RLock) → 创建 → setDClient(Lock)]
     │       └── invalidateCache()              [getCachedClient(RLock)]
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

### 7.7 发现的并发缺陷汇总（修正版）

| # | 位置 | 问题 | 触发条件 | 后果 | 严重程度 |
|---|------|------|----------|------|----------|
| 1 | [suggestCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L195-L244) L210 | `maps.Keys(a.command.alias.Alias)` 绕过 `Aliases.mx` 直接遍历 map | 上下文切换进行中时用户输入命令触发补全 | Go runtime panic: concurrent map read and map write | **高** |
| 2 | [Command.Init](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/command.go#L57-L68) | 写 `c.alias` 指针无 `Command.mx` 保护 | 目前不存在并发场景 | 潜在数据竞态 | **低**（仅启动时调用） |
| 3 | [switchContext](file:///d:/fz/0601-2/solo-dogfeeding/code/6-k9s/internal/view/app.go#L519-L521) | `go clusterModel.Reset` 与 `Resume` 后 `clusterUpdater` 重叠 | 每次上下文切换 | UI 可能收到两次集群信息更新回调（闪烁） | 低（UX 问题） |

#### 缺陷 1 的修复建议

`suggestCommand` 应通过 `ShortNames()` 或 `AliasesFor` 来遍历，让 `Aliases.mx` 介入：

```go
// 修复前（有问题）：
for alias := range maps.Keys(a.command.alias.Alias) {
    if suggest, ok := cmd.ShouldAddSuggest(ls, alias); ok {
        entries = append(entries, suggest)
    }
}

// 修复后方案 A — 使用 ShortNames()（内部有 Aliases.mx.RLock）：
if a.command.alias != nil {
    for _, aliasList := range a.command.alias.ShortNames() {
        for _, alias := range aliasList {
            if suggest, ok := cmd.ShouldAddSuggest(ls, alias); ok {
                entries = append(entries, suggest)
            }
        }
    }
}

// 修复后方案 B — 使用 AliasesFor 遍历所有已知 GVR：
// （需要获取所有 GVR 列表，可从 MetaAccess 获取）
```

#### 缺陷 2 的分析

`Command.Init` 仅在 `App.Init` 中调用（启动阶段），且 `Command` 在 `Init` 之前不会被其他 goroutine 使用，所以当前安全。但如果未来引入并发初始化场景，需要加 `Command.mx`。

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
│       │       ├── Config.SwitchContext("prod")   // ❌ 无 Config.mx，裸替换 flags 指针
│       │       ├── APIClient.reset()              // 部分锁：6 setter 经 APIClient.mx；cache/nsClient ❌ 直接替换
│       │       ├── ResetMetrics()                 // ❌ 全局变量 MetricsDial = nil，无锁
│       │       ├── a.config = NewConfig(flags)    // ❌ 无 APIClient.mx，裸替换 config 指针
│       │       ├── CheckConnectivity()            // client/connOK ✅ mx；connOK L320 裸写不一致；cache ❌
│       │       ├── DynDial()                      // getDClient(RLock) → 新建 → setDClient(Lock) ✅
│       │       └── invalidateCache()              // getCachedClient(RLock) ✅；磁盘 Invalidate 内部锁
│       │
│       └── App.switchContext(ci, force=true)
│           ├── Halt()                             // 取消 context → 停止后台 goroutine
│           ├── Config.Reset()                     // K9s.mx 各 setter ✅
│           ├── Config.ActivateContext("prod")
│           │   ├── K9s.ActivateContext("prod")
│           │   │   ├── ks.GetContext("prod")       // 获取 kubeconfig 上下文
│           │   │   ├── dir.Load("prod", ct)        // 加载/生成 k9s 上下文配置
│           │   │   ├── 设置 Proxy（如有）            // Config.proxy 直接赋值 ❌
│           │   │   └── 设置 Active Namespace
│           │   │       ├── kubeconfig ctx.namespace → 优先
│           │   │       └── "default" → 兜底
│           │   └── 验证命名空间合法性（IsValidNamespace → cache）
│           ├── Config.Save(true)                  // 持久化配置
│           ├── Factory.Terminate()                // Factory.mx.Lock → close(stopChan) + clear map
│           ├── Factory.Start(ns)                  // Factory.mx → 新建 stopChan
│           ├── Command.Reset(aliasesPath, nuke=true) // Command.mx.Lock → Aliases.mx 各操作
│           ├── ReloadStyles()                     // 重载皮肤
│           ├── gotoResource(activeView)           // 导航到活跃视图（主 goroutine 串行）
│           ├── go clusterModel.Reset(factory)     // 异步重置集群模型，与 Resume 后 clusterUpdater 重叠
│           └── Resume()                           // 新建 ctx → 重启 clusterUpdater / ConfigWatcher 等
│
└── 刷新上下文列表视图
```

---

## 九、缓存层次总结（按真实锁保护方式）

| 缓存层 | 位置 | 重建方式 | 真实锁保护方式 | 生命周期 |
|--------|------|----------|----------------|----------|
| LRU Auth 缓存 | `APIClient.cache` | `reset()` 中 `a.cache = New...` **直接替换指针** | ❌ **无 `APIClient.mx`**；指针替换靠原子性；内部 LRU map 操作由 `LRUExpireCache` 自带 mutex 保护 | 每次 SwitchContext |
| Discovery 磁盘缓存 | `~/.kube/cache/discovery/<host>/` | `CachedDiscovery().Invalidate()` | ✅ 路径：`getCachedClient(RLock)` → `Invalidate()`（disk cache 自带内部互斥） | 每次 SwitchContext |
| HTTP 缓存 | `~/.kube/cache/http/` | 随 Discovery 重建 | — | 每次 SwitchContext |
| Metrics 缓存 | `MetricsServer.cache`（独立 LRU） | `ResetMetrics()` → 全局 `MetricsDial = nil` **直接替换** | ❌ **全局变量无任何锁**；`MetricsServer.cache` 内部 LRU map 操作靠自带 mutex | 每次 SwitchContext |
| Informer 本地缓存 | `Factory.factories[ns]` | `Terminate()` → `close(stopChan)` + `delete(map, k)` | ✅ `Factory.mx.Lock` 全程保护 | 每次 SwitchContext |
| K9s 上下文配置 | `K9s.activeConfig` | `Reset()` → `ActivateContext()` | ✅ `K9s.mx` getter/setter（`setActiveConfig`/`getActiveConfig`） | 每次 SwitchContext |
| 命令别名 map | `Aliases.Alias`（在 `Command.alias.Aliases.Alias` 内） | `Reset(nuke=true)` → `Clear()` 删全部 + `Ensure()` 重建 | ✅ `Command.mx.Lock`（外层）→ `Aliases.mx`（内层 Get/Define/Clear） | 每次 SwitchContext |

---

## 十、关键设计洞察（按代码事实修正）

1. **懒初始化 + 主动预热**：`reset()` 只清空不重建，客户端在首次 `Dial()` 时按需创建。`SwitchContext` 主动预热了 `Dial`（通过 `CheckConnectivity`）和 `DynDial`，因为这两个是后续操作最常用的。

2. **ConfigFlags 重建而非修改，但** **❌ 完全无锁保护**：`Config.SwitchContext` 创建全新的 `genericclioptions.ConfigFlags` 实例（`NewConfigFlags(UsePersistentConfig=true)`），避免修改共享状态——但这不是锁的功劳，而是**时序保障**：`Halt()` 先停止所有后台 goroutine，之后才执行 `SwitchContext`。`Config.mx` 几乎是虚设的，仅在 `ConfigAccess()` 一处使用了读锁。`UsePersistentConfig=true` 启用了客户端传输层缓存（HTTP/Disk）。

3. **Halt/Resume 模式是并发安全的基石**：通过 `context.WithCancel` 实现优雅的启停，而不是用锁阻塞。这确保切换期间不会有旧集群的请求或回调干扰新集群的状态。如果没有 Halt/Resume，多个无锁指针替换（config/flags/cache/MetricsDial）都会立即变成竞态。

4. **ToggleContextSwitch 标志**：防止配置文件 watcher 在上下文切换中间状态触发 `Reload()`，导致配置被覆盖。这是 `K9s.mx` 保护的布尔标志。

5. **Discovery 缓存天然隔离**：不同集群的缓存路径基于 API Server 地址，但 `Invalidate()` 仍然必要——如果同一 API Server 有不同认证上下文，旧的缓存可能导致权限错误。

6. **锁策略不一致是遗留问题**：
   - `connOK`：`CheckConnectivity:L320` 裸写 vs `setConnOK()` 用 mx.Lock
   - `ConnectionOK()` 裸读 vs `getConnOK()` 用 mx.RLock
   - 6 个客户端字段都经 setter/getter 加锁，但 `cache` 和 `nsClient` 直接赋值
   - `Config.mx` 声明了但几乎不用
   这些不一致在当前时序下（Halt 保证切换时无并发）没有实际问题，但未来若引入并行初始化/切换场景，会是隐患。

7. **`nsClient` 是死代码**：在 `reset()` 中被置 nil，整个代码库从未被读取。如果被清理，不会影响功能。
