# Port-Forward 生命周期深度解析

本文基于 K9s 源码，深入解析 port-forward 从启动到回收的完整生命周期，包括本地端口分配、连接保持和失败清理机制。

---

## 一、核心数据结构

### 1.1 PortForwarder 结构体

定义于 [port_forwarder.go:31-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L31-L40)：

```go
type PortForwarder struct {
    Factory
    genericclioptions.IOStreams

    stopChan, readyChan chan struct{}
    active              bool
    path                string
    tunnel              port.PortTunnel
    age                 time.Time
}
```

- `stopChan`：停止信号通道，用于通知端口转发协程退出
- `readyChan`：就绪信号通道，端口转发成功建立后关闭
- `active`：标记转发是否处于活跃状态
- `tunnel`：端口隧道配置，包含地址、容器名、本地端口和容器端口
- `age`：转发创建时间，用于失效检测

### 1.2 PortTunnel 结构体

定义于 [tunnel.go:30-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/tunnel.go#L30-L32)：

```go
type PortTunnel struct {
    Address, Container, LocalPort, ContainerPort string
}
```

### 1.3 ContainerPortSpec 结构体

定义于 [co_portspec.go:103-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/co_portspec.go#L103-L107)：

```go
type ContainerPortSpec struct {
    Container string
    PortName  string
    PortNum   string
}
```

用于描述 Pod 中容器暴露的 TCP 端口，来自 Pod Spec 解析。

### 1.4 Annotations 类型

定义于 [ann.go:10](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/ann.go#L10-L10)：

```go
type Annotations map[string]string
```

---

## 二、启动流程详解

### 2.1 触发入口

用户在 UI 中按 `Shift-F` 触发端口转发，调用链如下：

1. [pf_extender.go:47-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_extender.go#L47-L66) - `portFwdCmd()` 处理按键事件
2. 检查 Pod 是否处于 Running 状态
3. 进入 `showFwdDialog()`，检查自动端口转发注解

### 2.2 两种注解的区别与进入端口选择的流程

K9s 支持两种端口转发注解，它们的处理流程**完全不同**：

| 注解 Key | 行为 | 处理阶段 |
|----------|------|----------|
| `k9scli.io/auto-port-forwards` | **自动启动**，无需用户交互 | `showFwdDialog()` 早期检测，直接跳过对话框 |
| `k9scli.io/port-forwards` | **预配置端口**，作为对话框默认值 | 弹出对话框后，通过 `PreferredPorts()` 预填 |

#### 2.2.1 自动转发注解（K9sAutoPortForwardsKey）

定义于 [pf_extender.go:179-L209](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_extender.go#L179-L209)：

```go
func showFwdDialog(v ResourceViewer, path string, cb PortForwardCB) error {
    mm, anns, err := fetchPodPorts(v.App().factory, path)
    // ... 构造 ContainerPortSpecs ...

    // 关键分支：检测到自动转发注解时，不显示对话框，直接启动
    if spec, ok := anns[port.K9sAutoPortForwardsKey]; ok {
        pfs, err := port.ParsePFs(spec)           // 1. 解析注解字符串
        if err != nil { return err }
        pts, err := pfs.ToTunnels(                 // 2. 转为 PortTunnel 并检查端口可用性
            v.App().Config.K9s.PortForwardAddress,
            ports,                                 // 注意：这里 ports 参数未被使用
            port.IsPortFree)                       // 3. 用 IsPortFree 检查本地端口
        if err != nil { return err }
        return startFwdCB(v, path, pts)            // 4. 直接启动，不弹对话框
    }

    ShowPortForwards(v, path, ports, anns, cb)     // 否则显示对话框
    return nil
}
```

**流程要点**：
- `ParsePFs()` 将逗号分隔的注解字符串解析为 `PFAnns` 切片
- `ToTunnels()` 遍历每个注解，调用 `IsPortFree` 做端口可用性检查
- 如果端口被占用，直接返回错误，不启动任何转发

#### 2.2.2 预设端口注解（K9sPortForwardsKey）→ PreferredPorts 匹配流程

当没有自动转发注解时，弹出对话框 `ShowPortForwards()`：

定义于 [pf_dialog.go:37-L50](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_dialog.go#L37-L50)：

```go
func ShowPortForwards(...) {
    // ... 创建表单 ...

    // 调用 PreferredPorts 获取推荐的端口配置
    pf, err := aa.PreferredPorts(ports)

    // ToPortSpec 生成对话框的默认填充值
    p1, p2 := pf.ToPortSpec(ports)
    f.AddInputField("Container Port:", p1, ...)   // 容器端口默认值
    f.AddInputField("Local Port:", p2, ...)       // 本地端口默认值
}
```

**PreferredPorts 详细流程**，定义于 [ann.go:12-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/ann.go#L12-L23)：

```go
func (a Annotations) PreferredPorts(specs ContainerPortSpecs) (PFAnns, error) {
    if len(specs) == 0 {
        return nil, errors.New("no exposed ports")
    }

    // 分支1：没有预设注解 → 取第一个暴露端口作为默认值
    value, ok := a[K9sPortForwardsKey]
    if !ok {
        return PFAnns{specs[0].ToPFAnn()}, nil    // specs[0] 作为兜底默认
    }

    // 分支2：有预设注解 → 用 MatchAnnotations 匹配容器实际端口
    return specs.MatchAnnotations(value), nil
}
```

**MatchAnnotations 匹配流程**，定义于 [co_portspec.go:73-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/co_portspec.go#L73-L87)：

```go
func (c ContainerPortSpecs) MatchAnnotations(s string) PFAnns {
    pfs, err := ParsePFs(s)                        // 1. 解析注解字符串为 PFAnns
    if err != nil { return nil }

    mm := make(PFAnns, 0, len(c))
    for _, pf := range pfs {
        if pf.Match(c) {                           // 2. 逐个与 ContainerPortSpecs 匹配
            mm = append(mm, pf)                    // 3. 匹配成功才加入结果
        }
    }
    return mm
}
```

**PFAnn.Match 单条匹配流程**，定义于 [pf.go:87-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/pf.go#L87-L96) 与 [co_portspec.go:152-L165](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/co_portspec.go#L152-L165)：

```go
// PFAnn.Match 遍历 ContainerPortSpecs
func (p *PFAnn) Match(ss ContainerPortSpecs) bool {
    for _, s := range ss {
        if s.Match(p) {                            // 调用 ContainerPortSpec.Match
            p.containerPortNum = s.PortNum          // 匹配成功后回填实际端口号
            return true
        }
    }
    return false
}

// ContainerPortSpec.Match 具体匹配规则
func (c ContainerPortSpec) Match(ann *PFAnn) bool {
    if c.Container != ann.Container {              // 规则1：容器名必须相等
        return false
    }
    switch ann.ContainerPort.Type {
    case intstr.String:
        return c.PortName == ann.ContainerPort.String()   // 规则2a：按端口名匹配
    case intstr.Int:
        return c.PortNum == ann.ContainerPort.String()    // 规则2b：按端口号匹配
    default:
        return false
    }
}
```

**匹配示例**：
| 注解值 | 容器实际端口 | 匹配结果 | 说明 |
|--------|-------------|----------|------|
| `c1::http` | c1: PortName="http", PortNum="8080" | ✅ 匹配 | 按端口名 String 匹配 |
| `c1::8080` | c1: PortName="http", PortNum="8080" | ✅ 匹配 | 按端口号 Int 匹配 |
| `c1::4321:8080` | c1: PortNum="8080" | ✅ 匹配 | 本地 4321 → 容器 8080 |
| `c2::8080` | c1: PortNum="8080" | ❌ 不匹配 | 容器名不同 |

**匹配后的回填机制**：
匹配成功后 `p.containerPortNum = s.PortNum`，确保即使注解只给了端口名（如 `http`），最终也能获得实际的数字端口号用于转发。

#### 2.2.3 ToPortSpec 生成对话框默认值

定义于 [pfs.go:19-L34](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/pfs.go#L19-L34)：

```go
func (aa PFAnns) ToPortSpec(pp ContainerPortSpecs) (ports, localPorts string) {
    specs, lps := make([]string, 0, len(aa)), make([]string, 0, len(aa))
    for _, a := range aa {
        specs = append(specs, a.AsSpec())          // "container::portNum" 格式
        if a.LocalPort == "" {
            if spec, ok := pp.Find(a); ok {        // LocalPort 为空时从匹配结果找
                a.LocalPort = spec.PortNum         // 默认与容器端口相同
            }
        }
        if a.LocalPort != "" {
            lps = append(lps, a.LocalPort)
        }
    }
    return strings.Join(specs, ","), strings.Join(lps, ",")
}
```

### 2.3 交互式端口转发

对话框中用户可以修改预填的默认值：

定义于 [pf_dialog.go:73-L86](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_dialog.go#L73-L86)：

```go
f.AddButton("OK", func() {
    // 用户点击 OK 后，从输入框取值并转为 PortTunnels
    tt, err := port.ToTunnels(address, coField.GetText(), loField.GetText())
    if err != nil {
        v.App().Flash().Err(err)
        return
    }
    if err := okFn(v, path, tt); err != nil {      // okFn = startFwdCB
        v.App().Flash().Err(err)
    }
})
```

### 2.4 权限检查

启动前进行两次 RBAC 权限检查，定义于 [port_forwarder.go:124-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L124-L150)：

1. **Pod 读取权限**：`GET pods` - 确认 Pod 存在且处于 Running 状态
2. **端口转发权限**：`CREATE pods/portforward` - 确认有权限创建端口转发

---

## 三、本地端口分配与可用性检查

### 3.1 端口可用性检查

定义于 [tunnel.go:59-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/tunnel.go#L59-L68)：

```go
func IsPortFree(ctx context.Context, t PortTunnel) bool {
    var ncfg net.ListenConfig
    s, err := ncfg.Listen(ctx, "tcp", fmt.Sprintf("%s:%s", t.Address, t.LocalPort))
    if err != nil {
        slog.Warn("Port is not available", slogs.Port, t.LocalPort, slogs.Address, t.Address)
        return false
    }
    return s.Close() == nil
}
```

**关键机制**：
- 尝试在指定地址和端口上进行 TCP 监听（非 `0.0.0.0` 通配，而是精确绑定到配置地址）
- 如果监听成功，立即关闭并返回 `true`（端口可用）
- 如果监听失败，返回 `false`（端口被占用）
- 这是一种"尝试绑定"的检测方式，比单纯扫描更可靠

### 3.2 批量端口检查

定义于 [tunnel.go:19-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/tunnel.go#L19-L27)：

```go
func (t PortTunnels) CheckAvailable(ctx context.Context) error {
    for _, pt := range t {
        if !IsPortFree(ctx, pt) {
            return fmt.Errorf("port %s is not available on host", pt.LocalPort)
        }
    }
    return nil
}
```

**注意**：`startFwdCB` 中先调用 `CheckAvailable` 全部通过后才继续，避免部分成功部分失败的状态。

### 3.3 重复转发检测

在启动前检查是否已有相同的转发存在，定义于 [pf_extender.go:155-L157](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_extender.go#L155-L157)：

```go
if _, ok := v.App().factory.ForwarderFor(dao.PortForwardID(path, pt.Container, pt.PortMap())); ok {
    return fmt.Errorf("port-forward is already active on pod %s", path)
}
```

**转发 ID 生成规则**，定义于 [port_forwarder.go:203-L209](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L203-L209)：
- 格式：`namespace/podName|containerName|localPort:containerPort`
- 注意：`PortForwardID()` 中有分支判断，如果 path 已包含 `|`，则不加容器名

---

## 四、连接建立与保持

### 4.1 转发启动流程

定义于 [port_forwarder.go:121-L171](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L121-L171)：

```go
func (p *PortForwarder) Start(path string, tt port.PortTunnel) (*portforward.PortForwarder, error) {
    p.path, p.tunnel, p.age = path, tt, time.Now()   // 记录 age 用于失效检测
    // ... 权限检查 ...
    req := clt.Post().
        Resource("pods").
        Namespace(ns).
        Name(podName).
        SubResource("portforward")
    return p.forwardPorts("POST", req.URL(), tt.Address, tt.PortMap())
}
```

### 4.2 协议降级机制

定义于 [port_forwarder.go:173-L197](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L173-L197)：

```go
func (p *PortForwarder) forwardPorts(method string, u *url.URL, addr, portMap string) (*portforward.PortForwarder, error) {
    transport, upgrader, err := spdy.RoundTripperFor(cfg)
    dialer := spdy.NewDialer(upgrader, &http.Client{Transport: transport, Timeout: defaultTimeout}, method, u)

    if !cmdutil.PortForwardWebsockets.IsDisabled() {
        tunnelingDialer, err := portforward.NewSPDYOverWebsocketDialer(u, cfg)
        // 优先使用 WebSocket，失败时降级到 SPDY
        dialer = portforward.NewFallbackDialer(tunnelingDialer, dialer, func(err error) bool {
            return httpstream.IsUpgradeFailure(err) || httpstream.IsHTTPSProxyError(err)
        })
    }

    return portforward.NewOnAddresses(dialer, []string{addr}, []string{portMap},
        p.stopChan, p.readyChan, p.Out, p.ErrOut)
}
```

**连接策略**：
1. **首选 WebSocket**：通过 `SPDYOverWebsocketDialer` 建立连接，更易穿透代理
2. **降级到 SPDY**：当 WebSocket 连接失败（升级失败或代理错误）时，自动回退到 SPDY 协议
3. **回退条件**：`httpstream.IsUpgradeFailure` 或 `httpstream.IsHTTPSProxyError`

### 4.3 连接保持协程（失败退出的完整流程）

**这是理解转发表生命周期的关键代码**，定义于 [pf_extender.go:131-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_extender.go#L131-L146)：

```go
func runForward(v ResourceViewer, pf watch.Forwarder, f *portforward.PortForwarder) {
    // ── 阶段1：注册到转发表 ──
    v.App().factory.AddForwarder(pf)              // 加入 forwarders map

    v.App().QueueUpdateDraw(func() {
        DismissPortForwards(v, v.App().Content.Pages)  // 关闭对话框
    })

    pf.SetActive(true)                            // 标记活跃状态

    // ── 阶段2：阻塞执行端口转发 ──
    // ForwardPorts() 是阻塞调用：
    //   - 正常情况下一直阻塞，直到 stopChan 被关闭或连接中断
    //   - 出错时立即返回 error（如端口被占用、连接失败等）
    if err := f.ForwardPorts(); err != nil {
        v.App().Flash().Warnf("PortForward failed for %s: %s. Deleting!", pf.ID(), err)
    }

    // ── 阶段3：从转发表移除（无论 ForwardPorts() 成功还是失败退出，都会执行到这里） ──
    v.App().QueueUpdateDraw(func() {
        v.App().factory.DeleteForwarder(pf.ID())  // ⭐ 关键：从 map 中移除
        pf.SetActive(false)                       // 标记非活跃
    })
}
```

**失败退出后从转发表移除的调用链详解**：

```
ForwardPorts() 返回（无论正常/异常）
    ↓
QueueUpdateDraw(...)          // 将清理操作序列化到 UI 主线程执行
    ↓
factory.DeleteForwarder(pf.ID())   // [factory.go:304-L311]
    ↓
forwarders.Kill(path)              // [forwarders.go:95-L113]
    ↓
遍历 forwarders map：
  匹配 prefix = path + "|" 或完全相等
    ├─ f.Stop()                   // 关闭 stopChan（但这里 ForwardPorts 已经返回了）
    └─ delete(ff, k)              // 从 map 中删除
```

**DeleteForwarder → Kill 具体代码**，定义于 [factory.go:304-L311](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/factory.go#L304-L311) 和 [forwarders.go:95-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/forwarders.go#L95-L113)：

```go
// factory.go
func (f *Factory) DeleteForwarder(path string) {
    count := f.forwarders.Kill(path)    // 调用 Forwarders.Kill
    slog.Warn("Deleted portforward", slogs.Count, count, slogs.GVR, path)
}

// forwarders.go
func (ff Forwarders) Kill(path string) int {
    var stats int
    prefix := path + "|"                 // ⭐ 加 '|' 防止前缀误匹配
    for k, f := range ff {
        if k == path || strings.HasPrefix(k, prefix) {  // 完全相等或前缀匹配
            stats++
            slog.Debug("Stop and delete port-forward", slogs.Name, k)
            f.Stop()                     // 调用 Forwarder.Stop()
            delete(ff, k)                // 从 map 删除
        }
    }
    return stats
}
```

**关键点说明**：
1. `runForward` 中的清理逻辑**位于 goroutine 尾部**，是 Go 的结构化编程保证——只要函数返回就一定执行
2. `ForwardPorts()` 返回的情况包括：端口绑定失败、连接断开、APIServer 拒绝、`stopChan` 被关闭等
3. 即使 `ForwardPorts()` 启动时就失败（如端口已占用），流程仍会走到清理逻辑，确保不会留下僵尸条目
4. `QueueUpdateDraw` 将操作提交到 UI 主线程，避免 map 并发读写冲突

### 4.4 Kubernetes 端口转发底层原理

`portforward.NewOnAddresses()` 创建的转发器会：
1. 在本地监听指定地址和端口
2. 建立到 APIServer 的 SPDY/WebSocket 连接
3. 为每个本地连接创建两条 SPDY 流（`data` 和 `error`）
4. 在本地端口和容器端口之间双向复制数据

---

## 五、定时校验：ValidatePortForwards 的触发节奏

### 5.1 触发时机与调用链

**定时器启动入口**，定义于 [app.go:367-L394](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/app.go#L367-L394)：

```go
// 常量定义 [app.go:39]
const clusterRefresh = 15 * time.Second      // 正常检查间隔：15秒

func (a *App) startClusterInfoUpdater(ctx context.Context) {
    if a == nil || a.factory == nil || ctx.Err() != nil {
        return
    }
    // 指数退避：初始 15s，失败后倍增，最大 2min
    bf := model.NewExpBackOff(ctx, clusterRefresh, 2*time.Minute)
    delay := clusterRefresh
    for {
        select {
        case <-ctx.Done():                    // 程序退出，context 取消
            slog.Debug("ClusterInfo updater canceled!")
            return
        case <-time.After(delay):             // ⭐ 每 delay 秒触发一次
            if err := a.refreshCluster(ctx); err != nil {
                // 连接失败，指数退避
                if delay = bf.NextBackOff(); delay == backoff.Stop {
                    a.BailOut(1)               // 退避到顶，退出程序
                    return
                }
            } else {
                bf.Reset()                     // 连接成功，重置退避
                delay = clusterRefresh         // 恢复 15s 间隔
            }
        }
    }
}
```

**ValidatePortForwards 被调用位置**，定义于 [app.go:397-L446](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/app.go#L397-L446)：

```go
func (a *App) refreshCluster(context.Context) error {
    // ...
    c := a.Content.Top()
    if ok := a.Conn().CheckConnectivity(); ok {   // 先检查 K8s 连通性
        // ... 连接恢复处理 ...
        a.factory.ValidatePortForwards()           // ⭐ 连通性 OK 后才校验转发
    } else if c != nil {
        atomic.AddInt32(&a.conRetry, 1)
        c.Stop()
    }
    // ... 重试次数超限处理 ...
    return nil
}
```

### 5.2 触发节奏总结

| 场景 | 校验间隔 |
|------|----------|
| 连接正常 | **15 秒** 固定间隔 |
| 连接失败后首次重试 | 15 秒 |
| 连接失败后第 N 次重试 | 15s × 2^(N-1)，最多 2 分钟 |
| 连接恢复后 | 立即重置为 15 秒 |
| 程序退出（ctx.Done） | 停止校验 |

**额外说明**：
- 校验**不是**单独的定时器，而是依附于 `clusterInfoUpdater` 的集群健康检查循环
- 只有当 `CheckConnectivity()` 成功时才会执行 `ValidatePortForwards()`——网络不通时不做无用功
- `startClusterInfoUpdater` 在 App 初始化连接成功后启动（`go a.startClusterInfoUpdater(ctx)`）

### 5.3 ValidatePortForwards 内部逻辑（与转发表交互）

定义于 [factory.go:333-L359](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/factory.go#L333-L359)：

```go
func (f *Factory) ValidatePortForwards() {
    for k, fwd := range f.forwarders {            // 遍历所有活跃转发
        tokens := strings.Split(k, ":")
        if len(tokens) != 2 {
            slog.Error("Invalid port-forward key", slogs.Key, k)
            return                                 // ⚠️ 注意：这里是 return 不是 continue
        }
        paths := strings.Split(tokens[0], "|")
        if len(paths) < 1 {
            slog.Error("Invalid port-forward path", slogs.Path, tokens[0])
        }

        // 检查1：Pod 是否还存在
        o, err := f.Get(client.PodGVR, paths[0], false, labels.Everything())
        if err != nil {
            fwd.Stop()                             // Stop: 关闭 stopChan
            delete(f.forwarders, k)                // ⭐ 直接从 map 删除（不走 Kill）
            continue
        }

        // 检查2：Pod 是否被重建（创建时间新于转发启动时间）
        var pod v1.Pod
        if err := runtime.DefaultUnstructuredConverter.FromUnstructured(...&pod); err != nil {
            continue
        }
        if pod.GetCreationTimestamp().Unix() > fwd.Age().Unix() {
            fwd.Stop()                             // Pod 已重启，旧转发失效
            delete(f.forwarders, k)                // ⭐ 直接从 map 删除
        }
    }
}
```

**与 runForward 清理路径的区别**：

| 清理路径 | 触发者 | 删除方式 | 是否调 Stop | 并发安全 |
|----------|--------|----------|-------------|----------|
| `runForward` 尾部 | goroutine 结束时 | `DeleteForwarder` → `Kill`（前缀匹配） | 是 | QueueUpdateDraw 主线程 |
| `ValidatePortForwards` | 15s 定时循环 | 直接 `delete(map, key)` | 是 | Factory mx.RLock/RUnlock? ⚠️ **注意无锁** |
| 用户 Ctrl-D 删除 | UI 事件 | `Delete` → `DeleteForwarder` → `Kill` | 是 | UI 主线程 |

**⚠️ 代码缺陷提示**：`ValidatePortForwards` 遍历和修改 `f.forwarders` map 时，**没有**使用 `f.mx` 锁保护，与 `AddForwarder/DeleteForwarder` 中的加锁操作不一致，存在潜在的数据竞争风险。

### 5.4 主动停止

#### 5.4.1 用户手动删除

用户按 `Ctrl-D` 删除选中的转发，调用链：
- [pf.go:152-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf.go#L152-L189) - `deleteCmd()` 处理删除
- [port_forward.go:33-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forward.go#L33-L37) - `Delete()` 调用 `factory.DeleteForwarder()`

#### 5.4.2 停止实现

定义于 [port_forwarder.go:102-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go#L102-L108)：

```go
func (p *PortForwarder) Stop() {
    p.active = false
    if p.stopChan != nil {
        close(p.stopChan)                          // 关闭通道通知 ForwardPorts() 退出
        p.stopChan = nil
    }
}
```

**原理**：关闭 `stopChan` 通道，通知 `ForwardPorts()` 阻塞调用退出。这是 Go 中常见的"优雅停止"模式。

**⚠️ 潜在问题**：`Stop()` 中 `stopChan` 检查与关闭不是原子的，并发调用可能触发 "close of closed channel" panic。

#### 5.4.3 批量删除

- **按 Pod 删除**：[forwarders.go:95-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/forwarders.go#L95-L113) - `Kill(path)` 删除指定 Pod 的所有转发
- **全部删除**：[forwarders.go:86-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/forwarders.go#L86-L92) - `DeleteAll()` 停止所有转发

### 5.5 程序退出时清理

定义于 [factory.go:60-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/factory.go#L60-L72)：

```go
func (f *Factory) Terminate() {
    f.mx.Lock()
    defer f.mx.Unlock()

    if f.stopChan != nil {
        close(f.stopChan)
        f.stopChan = nil
    }
    for k := range f.factories { delete(f.factories, k) }
    f.forwarders.DeleteAll()                      // 遍历 Stop + delete
}
```

---

## 六、从转发表移除的三种路径汇总

```
┌──────────────────────────────────────────────────────────────────┐
│                    转发条目从转发表移除的三条路径                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  路径1: ForwardPorts() 返回（正常退出/连接断开/启动失败）           │
│  └─ runForward() 尾部代码                                         │
│     └─ QueueUpdateDraw() → DeleteForwarder(ID) → Kill()          │
│        └─ 完全匹配 ID，f.Stop() + delete()                       │
│                                                                  │
│  路径2: 15秒定时校验 ValidatePortForwards()                       │
│  ├─ 场景A: Pod 已不存在 → fwd.Stop() + delete(map, key)          │
│  └─ 场景B: Pod 创建时间新于转发年龄 → fwd.Stop() + delete(map, key)│
│                                                                  │
│  路径3: 用户主动操作                                               │
│  ├─ UI Ctrl-D: PortForward.Delete → DeleteForwarder(path)        │
│  ├─ 删除 Pod/Workload: 资源删除回调 DeleteForwarder(path)         │
│  └─ 程序退出: Terminate → DeleteAll()                             │
│     └─ Kill() 前缀匹配（path + "|"），批量清理 Pod 所有转发        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 七、完整生命周期时序图

```
用户操作 (Shift-F)
    ↓
[pf_extender.go:47] portFwdCmd()
    ├─ ensurePodPortFwdAllowed() 检查 Pod Running
    ↓
[pf_extender.go:179] showFwdDialog()
    ├─ fetchPodPorts() 获取容器端口 + 注解
    ├─ 检测 K9sAutoPortForwardsKey?
    │   ├─ YES → ParsePFs() → ToTunnels(IsPortFree) → startFwdCB()
    │   └─ NO  → ShowPortForwards 对话框
    │            └─ PreferredPorts():
    │                ├─ 有 K9sPortForwardsKey → MatchAnnotations() 匹配实际端口
    │                │   └─ 容器名+端口名/号双重校验 + 回填 PortNum
    │                └─ 无注解 → specs[0] 作为默认值
    │            └─ 用户点击 OK → ToTunnels() → startFwdCB()
    ↓
[pf_extender.go:148] startFwdCB()
    ├─ PortTunnels.CheckAvailable() 全部端口可用性检查
    ├─ 检查转发是否已存在 (ForwarderFor ID)
    ├─ 每个 PortTunnel:
    │   ├─ NewPortForwarder()
    │   └─ pf.Start(path, pt)
    │       ├─ age = time.Now() [用于后续失效检测]
    │       ├─ RBAC: GET pods + CREATE pods/portforward
    │       └─ forwardPorts(): WebSocket优先 → SPDY降级
    └─ go runForward(v, pf, fwd)  // 启动独立goroutine
        ↓
    [goroutine] runForward()
        ├─ factory.AddForwarder(pf)  // 注册到转发表
        ├─ DismissPortForwards 对话框
        ├─ pf.SetActive(true)
        ├─ f.ForwardPorts()  // ⚡ 阻塞调用:监听端口+隧道数据复制
        │    │               //   直到 stopChan 关闭或连接中断
        │    ↓ (任何原因返回)
        └─ QueueUpdateDraw():
            ├─ factory.DeleteForwarder(pf.ID)
            │   └─ Kill(ID): Stop() + delete(map, key)
            └─ pf.SetActive(false)

─────────────────────────────────────────────────────────
后台线程 (15秒周期, app.startClusterInfoUpdater)
    ├─ time.After(15s)
    ├─ CheckConnectivity() OK?
    │   └─ YES → factory.ValidatePortForwards()
    │        ├─ 遍历 forwarders
    │        ├─ Pod 不存在? → Stop() + delete(map, key)
    │        └─ Pod 创建时间 > age? → Stop() + delete(map, key)
    └─ 连接失败 → 指数退避(15s→30s→...→2min)

─────────────────────────────────────────────────────────
用户 Ctrl-D / 删除资源 / 程序退出
    └─ DeleteForwarder(path) / Terminate()
        └─ Kill(path) / DeleteAll()
            └─ Stop() + delete(map, key)
```

---

## 八、关键设计要点总结

### 8.1 端口分配
- 没有自动分配随机端口的机制，端口由用户指定或注解配置
- 通过实际绑定测试来检查端口可用性，确保准确性
- 启动前双重检查（可用性+重复转发检测）
- `PreferredPorts` 支持注解中用端口名匹配，内部回填实际端口号

### 8.2 连接保持
- 使用独立 goroutine 运行阻塞的 `ForwardPorts()` 调用
- 支持 WebSocket → SPDY 协议降级，提高网络兼容性
- 连接中断后自动执行清理，避免僵尸转发（goroutine 尾部保证）

### 8.3 失败清理
- **主动清理**：用户删除、程序退出时的 `Terminate()`
- **被动清理**：每 15 秒 `ValidatePortForwards()` 检测失效转发（附于集群健康检查）
- **异常清理**：`ForwardPorts()` 退出时的函数尾部保证——无论正常失败都会走清理
- **通道关闭模式**：通过关闭 `stopChan` 实现优雅停止
- **三种从转发表移除的路径**：goroutine 退出、定时校验、用户操作

### 8.4 定时校验节奏
- 默认 15 秒固定间隔，与集群健康检查共享循环
- 连接失败时启用指数退避，最大 2 分钟
- 连接成功后立即重置回 15 秒
- 只有 K8s 连通性 OK 时才执行转发校验，避免误判

### 8.5 并发安全
- `Factory` 使用 `sync.RWMutex` 保护 `forwarders` map（Add/Delete 加锁，ValidatePortForwards 未加锁⚠️）
- 转发器注册和删除都在锁保护下进行（除 ValidatePortForwards）
- UI 更新通过 `QueueUpdateDraw()` 序列化到主线程
- `Kill(path)` 中使用 `path + "|"` 前缀匹配防止误删同名 Pod（如 web-0 vs web-0-bla）

---

## 九、潜在问题与优化建议

### 9.1 现有代码的潜在问题

1. **端口竞态条件**：`IsPortFree()` 检查通过后，在实际绑定前端口可能被其他程序占用
   - 但 `portforward.PortForwarder` 内部会再次绑定，失败会报错，属于"最终失败"而非"静默失败"

2. **ValidatePortForwards 未加锁**：遍历和删除 `f.forwarders` 时未持有 `f.mx` 锁，与 AddForwarder/DeleteForwarder 的加锁操作不一致，存在 data race 风险

3. **ValidatePortForwards 提前 return**：当遇到格式错误的 key 时使用 `return` 而非 `continue`，导致后续合法的转发也不被校验
   - 代码位置：[factory.go:336](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/factory.go#L336-L339)

4. **`stopChan` 重复关闭风险**：`Stop()` 方法中只检查 `stopChan != nil`，但并发调用仍可能导致 panic
   - 路径：`ValidatePortForwards.Stop()` + `runForward` 退出时 `Kill().Stop()` 可能同时执行

5. **失效检测检测周期**：连接失败进入退避期间（最长 2 分钟），`ValidatePortForwards` 不会执行，可能存在僵尸转发窗口

### 9.2 代码优化建议

**优化1：Stop() 用 sync.Once 防止重复关闭**

```go
type PortForwarder struct {
    // ... 现有字段 ...
    stopOnce sync.Once
}

func (p *PortForwarder) Stop() {
    p.active = false
    p.stopOnce.Do(func() {
        if p.stopChan != nil {
            close(p.stopChan)
            p.stopChan = nil
        }
    })
}
```

**优化2：ValidatePortForwards 加锁 + 改为 continue**

```go
func (f *Factory) ValidatePortForwards() {
    f.mx.Lock()                    // 加写锁
    defer f.mx.Unlock()

    for k, fwd := range f.forwarders {
        tokens := strings.Split(k, ":")
        if len(tokens) != 2 {
            slog.Error("Invalid port-forward key", slogs.Key, k)
            continue               // continue 而不是 return
        }
        // ... 后续逻辑不变 ...
    }
}
```

---

## 十、核心文件速查表

| 文件 | 主要职责 | 关键函数/类型 |
|------|----------|---------------|
| [port_forwarder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/dao/port_forwarder.go) | 转发器核心实现，启动/停止/连接建立 | `PortForwarder`, `Start()`, `Stop()`, `forwardPorts()` |
| [tunnel.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/tunnel.go) | 端口隧道结构，端口可用性检查 | `PortTunnel`, `IsPortFree()`, `CheckAvailable()` |
| [forwarders.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/forwarders.go) | 转发器集合管理，批量删除 | `Forwarders`, `Kill()`, `DeleteAll()` |
| [factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/watch/factory.go) | 转发器注册/删除/失效检测 | `AddForwarder()`, `DeleteForwarder()`, `ValidatePortForwards()` |
| [pf_extender.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_extender.go) | UI 交互，注解分流，启动协程 | `portFwdCmd()`, `showFwdDialog()`, `startFwdCB()`, `runForward()` |
| [pf_dialog.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/pf_dialog.go) | 端口转发配置对话框 | `ShowPortForwards()`, `DismissPortForwards()` |
| [pf.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/pf.go) | 端口转发注解解析 | `PFAnn`, `ParsePF()`, `ParsePFs()`, `Match()` |
| [ann.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/ann.go) | 预设端口选择 | `Annotations.PreferredPorts()` |
| [co_portspec.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/co_portspec.go) | 容器端口规格与匹配 | `ContainerPortSpec`, `MatchAnnotations()`, `Match()` |
| [pfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/port/pfs.go) | 注解集合转隧道 | `PFAnns.ToTunnels()`, `PFAnns.ToPortSpec()` |
| [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-k9s/internal/view/app.go) | 定时校验触发器（15s） | `startClusterInfoUpdater()`, `refreshCluster()` |
