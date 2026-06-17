# Helm Release 浏览、历史与回滚流程代码分析

## 一、整体架构概览

k9s 中 Helm 功能采用标准的 **DAO（数据访问层）→ Model（数据模型层）→ View（视图层）→ Render（渲染层）** 四层架构，通过 GVR（Group/Version/Resource）作为资源标识符串联各层。

### 1.1 GVR 定义

Helm 相关的两个核心 GVR 在 [gvrs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/client/gvrs.go#L70-L72) 中定义：

```go
// Helm...
HmGVR  = NewGVR("helm")           // Helm Release 列表
HmhGVR = NewGVR("helm-history")   // Helm Release 历史版本
```

这两个 GVR 属于 k9s 自定义的非 K8s 标准资源，被标记为 `helmCat` 类别。

### 1.2 各层注册关系

| 层级 | 注册位置 | Helm Release (HmGVR) | Helm History (HmhGVR) |
|------|----------|---------------------|----------------------|
| DAO 层 | [accessor.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/accessor.go#L35-L36) | `dao.HelmChart` | `dao.HelmHistory` |
| Model 层 | [registry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/model/registry.go#L33-L40) | `dao.HelmChart` + `helm.Chart` | `dao.HelmHistory` + `helm.History` |
| View 层 | [registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/registrar.go#L23-L27) | `NewHelmChart` | - (由 HelmChart 内部跳转) |
| DAO Meta | [registry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/registry.go#L277-L292) | 注册 API 元信息 | 注册 API 元信息 |

---

## 二、Helm Release 列表查询流程

### 2.1 视图初始化

用户进入 Helm Release 列表页面时，通过 [NewHelmChart](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_chart.go#L21-L34) 创建视图：

```go
func NewHelmChart(gvr *client.GVR) ResourceViewer {
    c := HelmChart{
        ResourceViewer: NewValueExtender(NewBrowser(gvr)),
    }
    // 设置颜色主题
    c.GetTable().SetBorderFocusColor(tcell.ColorMediumSpringGreen)
    // 绑定按键
    c.AddBindKeysFn(c.bindKeys)
    // 设置回车键行为：跳转到历史版本
    c.GetTable().SetEnterFn(c.viewReleases)
    // 设置上下文函数
    c.SetContextFn(c.chartContext)
    return &c
}
```

关键绑定：
- **回车键** (`SetEnterFn`)：调用 `viewReleases` 跳转到历史版本页面
- **R 键**：通过 `bindKeys` 绑定 `historyCmd`，与回车键行为一致

### 2.2 DAO 层数据查询

列表数据由 [HelmChart.List](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_chart.go#L35-L55) 方法获取：

```go
func (h *HelmChart) List(_ context.Context, ns string) ([]runtime.Object, error) {
    // 1. 初始化 Helm 配置（指定 namespace）
    cfg, err := ensureHelmConfig(h.Client().Config().Flags(), ns)
    if err != nil {
        return nil, err
    }

    // 2. 使用 Helm SDK 的 action.NewList 获取所有 Release
    list := action.NewList(cfg)
    list.All = true          // 包含所有状态的 Release
    list.SetStateMask()      // 设置状态掩码
    rr, err := list.Run()    // 执行查询
    if err != nil {
        return nil, err
    }

    // 3. 包装为 k9s 内部资源对象
    oo := make([]runtime.Object, 0, len(rr))
    for _, r := range rr {
        oo = append(oo, helm.ReleaseRes{Release: r})
    }
    return oo, nil
}
```

### 2.3 Helm 配置初始化

[ensureHelmConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_chart.go#L147-L165) 是所有 Helm 操作的基础：

```go
func ensureHelmConfig(flags *genericclioptions.ConfigFlags, ns string) (*action.Configuration, error) {
    // 复制 kubeconfig 配置，并锁定 namespace
    settings := &genericclioptions.ConfigFlags{
        Namespace:        &ns,
        Context:          flags.Context,
        BearerToken:      flags.BearerToken,
        APIServer:        flags.APIServer,
        CAFile:           flags.CAFile,
        KubeConfig:       flags.KubeConfig,
        Impersonate:      flags.Impersonate,
        Insecure:         flags.Insecure,
        TLSServerName:    flags.TLSServerName,
        ImpersonateGroup: flags.ImpersonateGroup,
        WrapConfigFn:     flags.WrapConfigFn,
    }
    cfg := new(action.Configuration)
    // 初始化 Helm 配置，使用环境变量 HELM_DRIVER 指定存储驱动
    err := cfg.Init(settings, ns, os.Getenv("HELM_DRIVER"), helmLogger)
    return cfg, err
}
```

### 2.4 渲染层：列表表格展示

[helm.Chart.Render](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/render/helm/chart.go#L53-L72) 将 `ReleaseRes` 转换为表格行：

```go
func (c Chart) Render(o any, _ string, r *model1.Row) error {
    h := o.(ReleaseRes)
    // 行ID格式: namespace/name（不含版本号）
    r.ID = client.FQN(h.Release.Namespace, h.Release.Name)
    r.Fields = model1.Fields{
        h.Release.Namespace,                       // NAMESPACE
        h.Release.Name,                            // NAME
        strconv.Itoa(h.Release.Version),           // REVISION
        h.Release.Info.Status.String(),            // STATUS
        h.Release.Chart.Metadata.Name + "-" + h.Release.Chart.Metadata.Version, // CHART
        h.Release.Chart.Metadata.AppVersion,       // APP VERSION
        render.AsStatus(c.diagnose(...)),          // VALID
        render.ToAge(metav1.Time{Time: h.Release.Info.LastDeployed.Time}), // AGE
    }
    return nil
}
```

健康检查逻辑：只有 `deployed` 状态被视为有效，否则 VALID 列显示错误。

---

## 三、从列表到历史版本的导航

### 3.1 触发方式

有两种方式进入历史版本页面：
1. 在 Release 行按下 **回车键**
2. 在 Release 行按下 **R 键**

两种方式最终都调用 [HelmChart.viewReleases](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_chart.go#L47-L53)：

```go
func (c *HelmChart) viewReleases(app *App, _ ui.Tabular, _ *client.GVR, _ string) {
    // 创建 History 视图，GVR 为 helm-history
    v := NewHistory(client.HmhGVR)
    // 设置上下文函数：将当前选中的 Release path 注入 context
    v.SetContextFn(c.helmContext)
    // 注入并显示新视图
    if err := app.inject(v, false); err != nil {
        app.Flash().Err(err)
    }
}
```

### 3.2 Context 传递机制

[HelmChart.helmContext](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_chart.go#L65-L73) 是关键的数据传递桥梁：

```go
func (c *HelmChart) helmContext(ctx context.Context) context.Context {
    path := c.GetTable().GetSelectedItem() // 获取选中行的 FQN: namespace/name
    if path == "" {
        return ctx
    }
    // 将 FQN 存入 context，供 DAO 层读取
    ctx = context.WithValue(ctx, internal.KeyFQN, path)
    return context.WithValue(ctx, internal.KeyPath, path)
}
```

### 3.3 历史视图初始化

[NewHistory](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L28-L40) 创建历史版本视图：

```go
func NewHistory(gvr *client.GVR) ResourceViewer {
    h := History{
        ResourceViewer: NewValueExtender(NewBrowser(gvr)),
    }
    h.GetTable().SetColorerFn(helm.History{}.ColorerFunc())
    h.GetTable().SetBorderFocusColor(tcell.ColorMediumSpringGreen)
    h.GetTable().SetSelectedStyle(...)
    h.AddBindKeysFn(h.bindKeys)
    h.SetContextFn(h.HistoryContext)
    // 回车键行为：查看该版本的 Values
    h.GetTable().SetEnterFn(h.getValsCmd)
    return &h
}
```

### 3.4 DAO 层：历史版本查询

[HelmHistory.List](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L34-L57) 通过 context 中的 FQN 来查询指定 Release 的历史：

```go
func (h *HelmHistory) List(ctx context.Context, _ string) ([]runtime.Object, error) {
    // 1. 从 context 中读取上一层传入的 FQN
    path, ok := ctx.Value(internal.KeyFQN).(string)
    if !ok {
        return nil, fmt.Errorf("expecting FQN in context")
    }
    ns, n := client.Namespaced(path) // 拆分 namespace 和 name

    // 2. 初始化 Helm 配置
    cfg, err := ensureHelmConfig(h.Client().Config().Flags(), ns)
    if err != nil {
        return nil, err
    }

    // 3. 使用 Helm SDK action.NewHistory 获取该 Release 的所有版本
    hh, err := action.NewHistory(cfg).Run(n)
    if err != nil {
        return nil, err
    }

    // 4. 包装为 ReleaseRes
    oo := make([]runtime.Object, 0, len(hh))
    for _, r := range hh {
        oo = append(oo, helm.ReleaseRes{Release: r})
    }
    return oo, nil
}
```

### 3.5 渲染层：历史表格展示

[helm.History.Render](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/render/helm/history.go#L45-L63) 的关键区别在于行 ID：

```go
func (c History) Render(o any, _ string, r *model1.Row) error {
    h := o.(ReleaseRes)
    // 行ID格式: namespace/name:revision（含版本号，用于定位具体版本）
    r.ID = client.FQN(h.Release.Namespace, h.Release.Name)
    r.ID += ":" + strconv.Itoa(h.Release.Version)
    r.Fields = model1.Fields{
        strconv.Itoa(h.Release.Version),    // REVISION
        h.Release.Info.Status.String(),     // STATUS
        h.Release.Chart.Metadata.Name + "-" + h.Release.Chart.Metadata.Version, // CHART
        h.Release.Chart.Metadata.AppVersion, // APP VERSION
        h.Release.Info.Description,         // DESCRIPTION
        render.AsStatus(c.diagnose(...)),   // VALID
    }
    return nil
}
```

**关键区别**：历史列表的行 ID 附加了 `:revision` 后缀（如 `default/my-release:3`），这是回滚操作定位版本的关键。

### 3.6 查看指定版本的 Values

在历史列表中按回车键，调用 [History.getValsCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L68-L81)：

```go
func (h *History) getValsCmd(app *App, _ ui.Tabular, _ *client.GVR, path string) {
    ns, n := client.Namespaced(path)
    tt := strings.Split(n, ":") // 拆分 name:revision
    if len(tt) < 2 {
        app.Flash().Err(fmt.Errorf("unable to parse version in %q", path))
        return
    }
    name, rev := tt[0], tt[1]
    // 创建 RevValues 模型并打开 LiveView
    h.Values = model.NewRevValues(h.GVR(), client.FQN(ns, name), rev)
    v := NewLiveView(h.App(), "Values", h.Values)
    if err := v.app.inject(v, false); err != nil {
        v.app.Flash().Err(err)
    }
}
```

[RevValues](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/model/rev_values.go#L46-L56) 通过 DAO 获取 Values：

```go
func getRevValues(path, _ string) []string {
    vals, err := getHelmHistDao().GetValues(path, true)
    if err != nil {
        slog.Error("Failed to get Helm values", slogs.Error, err)
    }
    return strings.Split(string(vals), "\n")
}
```

---

## 四、回滚操作流程

### 4.1 触发：R 键绑定

在历史版本视图中，按下 **R 键** 触发回滚。该键位仅在非只读模式下通过 [bindDangerousKeys](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L83-L90) 注册：

```go
func (h *History) bindDangerousKeys(aa *ui.KeyActions) {
    aa.Add(ui.KeyR, ui.NewKeyActionWithOpts("RollBackTo...", h.rollbackCmd,
        ui.ActionOpts{
            Visible:   true,
            Dangerous: true, // 标记为危险操作
        },
    ))
}
```

### 4.2 确认对话框

[History.rollbackCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L92-L119) 首先展示确认对话框：

```go
func (h *History) rollbackCmd(evt *tcell.EventKey) *tcell.EventKey {
    path := h.GetTable().GetSelectedItem() // 获取 namespace/name:revision
    if path == "" {
        return evt
    }

    ns, nrev := client.Namespaced(path)
    tt := strings.Split(nrev, ":")
    n, rev := nrev, ""
    if len(tt) == 2 {
        n, rev = tt[0], tt[1] // 解析出 release name 和目标版本号
    }

    h.Stop() // 暂停视图刷新
    defer h.Start()

    // 构建确认消息：RollingBack chart my-release to release <3>?
    msg := fmt.Sprintf("RollingBack chart [yellow::b]%s[-::-] to release <[orangered::b]%s[-::-]>?", n, rev)
    dialog.ShowConfirmAck(h.App().App, h.App().Content.Pages, n, false,
        "Confirm Rollback", msg,
        func() { // 用户确认后的回调
            ctx, cancel := context.WithTimeout(
                context.Background(),
                h.App().Conn().Config().CallTimeout(), // 使用配置的调用超时
            )
            defer cancel()
            if err := h.rollback(ctx, client.FQN(ns, n), rev); err != nil {
                h.App().Flash().Err(err) // 失败：显示错误
            } else {
                h.App().Flash().Infof("Rollout restart in progress for char `%s...", n) // 成功：提示
            }
        },
        func() {}, // 用户取消的回调：无操作
    )
    return nil
}
```

### 4.3 DAO 层：执行回滚

[HelmHistory.Rollback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L138-L153) 使用 Helm SDK 执行回滚：

```go
func (h *HelmHistory) Rollback(_ context.Context, path, rev string) error {
    ns, n := client.Namespaced(path)
    // 1. 初始化 Helm 配置
    cfg, err := ensureHelmConfig(h.Client().Config().Flags(), ns)
    if err != nil {
        return err
    }

    // 2. 将版本号字符串转为整数
    ver, err := strconv.Atoi(rev)
    if err != nil {
        return fmt.Errorf("could not convert revision to a number: %w", err)
    }

    // 3. 使用 Helm SDK action.NewRollback 执行回滚
    clt := action.NewRollback(cfg)
    clt.Version = ver    // 指定目标版本
    return clt.Run(n)    // 对指定 Release 执行回滚
}
```

回滚成功后，视图层调用 `h.Refresh()` 刷新历史列表。

---

## 五、失败处理机制

### 5.1 错误分类与处理方式

| 阶段 | 错误来源 | 处理方式 | 代码位置 |
|------|---------|---------|---------|
| **列表查询** | Helm 配置初始化失败 | 返回 error，由上层 Flash 显示 | [helm_chart.go:36-39](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_chart.go#L36-L39) |
| **列表查询** | Helm SDK List 调用失败 | 返回 error，由上层 Flash 显示 | [helm_chart.go:44-47](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_chart.go#L44-L47) |
| **历史查询** | Context 中缺少 FQN | 返回 error "expecting FQN in context" | [helm_history.go:35-38](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L35-L38) |
| **历史查询** | Helm SDK History 调用失败 | 返回 error，由上层 Flash 显示 | [helm_history.go:46-49](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L46-L49) |
| **单版本查询** | Path 格式错误（缺少 revision） | 返回 error "invalid path" | [helm_history.go:61-64](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L61-L64) |
| **回滚操作** | 版本号解析失败 | 返回 error "could not convert revision to a number" | [helm_history.go:145-148](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L145-L148) |
| **回滚操作** | Helm SDK Rollback 调用失败 | 返回 error，Flash 显示 | [helm_history.go:152](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L152) |
| **Values 解析** | Path 缺少 revision 后缀 | Flash.Err 提示 "unable to parse version" | [helm_history.go:70-74](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L70-L74) |

### 5.2 错误展示：Flash 通知机制

View 层统一通过 `app.Flash()` 展示用户可见的错误信息，主要有两种：

```go
// 错误提示（红色）
h.App().Flash().Err(err)

// 信息提示（正常颜色）
h.App().Flash().Infof("Rollout restart in progress for char `%s...", n)
```

### 5.3 超时控制

回滚操作使用 context 超时控制，超时时间来自 k9s 连接配置：

```go
ctx, cancel := context.WithTimeout(
    context.Background(),
    h.App().Conn().Config().CallTimeout(),
)
defer cancel()
```

### 5.4 Values 轮询失败重试

在 [RevValues.updater](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/model/rev_values.go#L137-L159) 中，Values 自动刷新采用指数退避重试：

```go
func (v *RevValues) updater(ctx context.Context) {
    backOff := NewExpBackOff(ctx, defaultReaderRefreshRate, maxReaderRetryInterval)
    delay := defaultReaderRefreshRate
    for {
        select {
        case <-ctx.Done():
            return
        case <-time.After(delay):
            if err := v.refresh(ctx); err != nil {
                v.fireResourceFailed(err)           // 通知视图层失败
                if delay = backOff.NextBackOff(); delay == backoff.Stop {
                    slog.Error("Giving up retrieving chart values", slogs.Error, err)
                    return  // 达到最大重试次数后放弃
                }
            } else {
                backOff.Reset()                     // 成功后重置退避
                delay = defaultReaderRefreshRate
            }
        }
    }
}
```

### 5.5 Helm 内部日志

Helm SDK 的内部日志通过 [helmLogger](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_chart.go#L167-L172) 转发到 k9s 的 slog 系统（Debug 级别）：

```go
func helmLogger(fmat string, args ...any) {
    slog.Debug("Log",
        slogs.Log, fmt.Sprintf(fmat, args...),
        slogs.Subsys, "helm",
    )
}
```

---

## 六、完整数据流转图

```
用户操作
   │
   ▼
[View: HelmChart (HmGVR)]
   │  回车键/R键
   │  helmContext() 注入 FQN 到 context
   ▼
[View: History (HmhGVR)]
   │  R键（危险操作）
   │  解析 path → namespace/name:revision
   │  ShowConfirmAck 二次确认
   ▼
[DAO: HelmHistory]
   │  ensureHelmConfig(ns) 初始化 Helm
   │  action.NewRollback(cfg).Run(name)
   ▼
[Helm SDK → Kubernetes]
   │
   ├─ 成功 → Flash.Infof + Refresh()
   └─ 失败 → Flash.Err(err)
```

## 七、核心数据结构

### 7.1 ReleaseRes：统一资源包装

在 [chart.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/render/helm/chart.go#L96-L108) 中定义，包装 Helm SDK 的 `*release.Release` 以适配 k8s runtime.Object 接口：

```go
type ReleaseRes struct {
    Release *release.Release // Helm SDK 的原生 Release 对象
}

func (ReleaseRes) GetObjectKind() schema.ObjectKind { return nil } // 非标准 K8s 资源
func (h ReleaseRes) DeepCopyObject() runtime.Object { return h }
```

### 7.2 路径格式约定

| 场景 | 格式 | 示例 |
|------|------|------|
| Release 列表行 ID | `namespace/name` | `default/nginx-ingress` |
| History 列表行 ID | `namespace/name:revision` | `default/nginx-ingress:3` |
| context KeyFQN | `namespace/name` | `default/nginx-ingress` |

