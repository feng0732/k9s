# K9s 启动配置加载、合并与容错机制全流程分析

## 一、启动入口总览

程序入口位于 [main.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/main.go#L44-L46)，调用 `cmd.Execute()`。核心启动逻辑在 [cmd/root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L76-L131) 的 `run()` 函数中，配置加载集中在 [loadConfiguration()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L133-L178)。

### 启动时序

```
main() → cmd.Execute() → run()
  ├─ config.InitLogLoc()            [阶段0: 日志路径初始化]
  ├─ config.InitLocs()              [阶段1: 目录路径初始化]
  ├─ loadConfiguration()            [阶段2: 配置加载核心]
  │   ├─ client.NewConfig(k8sFlags)
  │   ├─ config.NewConfig(k8sCfg)
  │   ├─ client.InitConnection()
  │   ├─ k9sCfg.Load()              [从文件加载]
  │   ├─ k9sCfg.K9s.Override()      [CLI参数覆盖]
  │   ├─ k9sCfg.Refine()            [精炼/激活上下文]
  │   └─ k9sCfg.Save()
  └─ view.NewApp(cfg) → app.Init() → app.Run()
```

---

## 二、三层配置来源

### 2.1 默认值（代码内硬编码）

默认值是最底层的配置来源，当配置文件和 CLI 参数都没有提供时生效。

| 配置项 | 默认值 | 定义位置 |
|--------|--------|----------|
| RefreshRate | `2.0` | [flags.go#L8](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/flags.go#L8) 和 [types.go#L7](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/types.go#L7) |
| LogLevel | `"info"` | [flags.go#L11](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/flags.go#L11) |
| Command | `""` | [flags.go#L14](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/flags.go#L14) |
| MaxConnRetry | `5` | [types.go#L8](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/types.go#L8) |
| APIServerTimeout | `"2m0s"` (120s) | [client/config.go#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/client/config.go#L23) |
| PortForwardAddress | `"localhost"` | [helpers.go#L17-L18](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/helpers.go#L17-L18) |
| ShellPod.Image | `"busybox:1.37.0"` | [shell_pod.go#L10](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/shell_pod.go#L10) |
| Logger.TailCount | `100` | [logger.go#L8](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/logger.go#L8) |
| Logger.BufferSize | `5000` | [logger.go#L11](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/logger.go#L11) |
| Logger.LogBufferSize | `50` | [logger.go#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/logger.go#L17) |
| DefaultView | `"po"` | [data/view.go#L6](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/view.go#L6) |
| Namespace.Active | `"default"` | [data/ns.go#L30](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/ns.go#L30) |
| MaxFavoritesNS | `9` | [data/ns.go#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/ns.go#L17) |
| Threshold CPU Warn/Critical | `70`/`90` | [threshold.go#L28-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/threshold.go#L28-L31) |

默认值通过 [NewK9s()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L69-L85) 构造函数写入 `K9s` 结构体，这是配置的初始状态。

### 2.2 配置文件

K9s 有两层配置文件：

1. **全局配置文件** (`$XDG_CONFIG_HOME/k9s/config.yaml` 或 `$K9S_CONFIG_DIR/config.yaml`)
   - 由 [config.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L271-L293) 加载
   - 对应 `AppConfigFile` 变量，在 [files.go#L67](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L67) 中声明

2. **上下文配置文件** (`$XDG_DATA_HOME/k9s/clusters/<cluster>/<context>/config.yaml`)
   - 由 [Dir.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L35-L55) 加载
   - 当上下文被激活时，在 [K9s.ActivateContext()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L256-L302) 中加载
   - 如果文件不存在或为空，会通过 [Dir.genConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L57-L63) 自动生成默认配置

### 2.3 启动参数（CLI Flags）

分为两组：

- **K9s 自有标志**：由 [Flags](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/flags.go#L18-L32) 结构体定义，在 [initK9sFlags()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L193-L274) 注册
- **K8s 标志**：使用 `genericclioptions.ConfigFlags`，在 [initK8sFlags()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L276-L325) 注册

K9s 标志详情：

| 标志 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--refresh` / `-r` | float32 | 2.0 | 刷新间隔（秒） |
| `--logLevel` / `-l` | string | "info" | 日志级别 |
| `--logFile` | string | AppLogFile | 日志文件路径 |
| `--headless` | bool | false | 隐藏头部 |
| `--logoless` | bool | false | 隐藏 Logo |
| `--crumbsless` | bool | false | 隐藏面包屑 |
| `--splashless` | bool | false | 隐藏启动画面 |
| `--invert` | bool | false | 反转皮肤颜色 |
| `--all-namespaces` / `-A` | bool | false | 启动时查看所有命名空间 |
| `--command` / `-c` | string | "" | 覆盖启动时的默认视图 |
| `--readonly` | bool | false | 只读模式 |
| `--write` | bool | false | 写模式（覆盖 readonly） |
| `--screen-dump-dir` | string | AppDumpsDir | 屏幕截图目录 |

K8s 标志详情：

| 标志 | 说明 |
|------|------|
| `--kubeconfig` | kubeconfig 文件路径 |
| `--context` | kubeconfig 上下文名称 |
| `--cluster` | 集群名称 |
| `--user` | 用户名称 |
| `--namespace` / `-n` | 命名空间 |
| `--request-timeout` | 请求超时 |
| `--as` / `--as-group` | 模拟用户/组 |
| `--insecure-skip-tls-verify` | 跳过 TLS 验证 |
| `--token` | Bearer Token |
| 证书相关标志 | CA/Client Key/Cert 文件 |

---

## 三、配置加载与合并的完整流程

### 3.1 阶段一：路径初始化

[InitLogLoc()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L89-L113) 和 [InitLocs()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L116-L122) 决定所有配置路径。

路径选择优先级：

```
K9S_LOGS_DIR 环境变量 → 日志目录
K9S_CONFIG_DIR 环境变量 → 统一配置目录（initK9sEnvLocs）
XDG 标准路径 → 分散配置目录（initXDGLocs）
```

- 设置 `K9S_CONFIG_DIR` 时，所有数据（配置、上下文、皮肤、截图等）都放在同一目录下
- 使用 XDG 标准时，配置在 `~/.config/k9s/`，状态数据在 `~/.local/state/k9s/`，上下文数据在 `~/.local/share/k9s/clusters/`

### 3.2 阶段二：构造初始配置对象

```go
// cmd/root.go#L136-L138
k8sCfg := client.NewConfig(k8sFlags)    // K8s 连接配置
k9sCfg := config.NewConfig(k8sCfg)       // K9s 配置（内含 NewK9s() 默认值）
```

[NewConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L31-L36) 内部调用 [NewK9s()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L69-L85)，此时所有字段都是代码默认值。

### 3.3 阶段三：从全局配置文件加载并合并

```go
// cmd/root.go#L146-L148
k9sCfg.Load(config.AppConfigFile, false)
```

[Config.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L271-L293) 的逻辑：

1. 文件不存在 → 调用 `Save()` 创建默认配置文件
2. 读取文件内容
3. JSON Schema 校验（使用 [JSONValidator.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/json/validator.go#L142-L170)）
4. YAML 反序列化到临时 `Config` 结构体
5. 调用 `c.Merge(&cfg)` 将文件内容**覆盖**到当前配置

**关键**：[K9s.Merge()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L129-L156) 是**全量覆盖**而非增量合并——文件中出现的字段会完全替换默认值，文件中未出现的字段保持默认值（因为 YAML 反序列化时零值字段不会覆盖非零默认值）。

### 3.4 阶段四：CLI 参数覆盖

```go
// cmd/root.go#L149
k9sCfg.K9s.Override(k9sFlags)
```

[K9s.Override()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L330-L349) 的逻辑：

- **RefreshRate**：只有当 CLI 值 != `DefaultRefreshRate`(2.0) 时才覆盖 → 存入 `manualRefreshRate`
- **UI 布尔类**（headless/logoless/crumbsless/splashless/invert）：直接存入 `UI.manualXxx` 指针字段
- **ReadOnly**：`--readonly` 为 true 时覆盖 → 存入 `manualReadOnly`
- **Write**：`--write` 为 true 时，将 `manualReadOnly` 设为 `false`（**write 优先级高于 readonly**）
- **Command**：直接存入 `manualCommand`
- **ScreenDumpDir**：直接存入 `manualScreenDumpDir`

**设计要点**：CLI 覆盖不是直接修改配置字段，而是存入独立的 `manual*` 字段。运行时读取时通过 getter 方法判断——如果 `manual*` 有值则用它，否则用配置文件/默认值。

### 3.5 阶段五：Refine 精炼配置

```go
// cmd/root.go#L150-L153
k9sCfg.Refine(k8sFlags, k9sFlags, k8sCfg)
```

[Config.Refine()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L89-L147) 负责：

1. **API 超时**：如果 CLI 未设 `--request-timeout`，使用配置文件中的 `APIServerTimeout`，解析失败则回退到 `DefaultCallTimeoutDuration`(120s)
2. **激活上下文**：
   - `--context` 有值 → 用它激活上下文
   - 否则 → 从 kubeconfig 获取当前上下文名称并激活
3. **命名空间决定**（三级优先级）：
   - `--all-namespaces` (-A) → 使用 `""`（所有命名空间）
   - `--namespace` (-n) 有值 → 使用它
   - 以上都无 → 使用上下文中保存的 active namespace
4. 确保 screenDumpDir 目录存在

上下文激活 [K9s.ActivateContext()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L256-L302) 会：
- 从 kubeconfig 获取上下文信息
- 加载上下文专属配置文件（[Dir.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L35-L55)）
- 设置代理（如果上下文配置中有 Proxy）
- 调用 [K9s.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L436-L463) 校验配置

### 3.6 阶段六：保存配置

```go
// cmd/root.go#L172-L175
k9sCfg.Save(false)
```

[Config.Save()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L296-L316) 的逻辑：
- 无活跃上下文 → 跳过保存
- 保存上下文配置到 `clusters/<cluster>/<context>/config.yaml`（仅在文件不存在或 force=true 时）
- 如果全局配置文件不存在，也创建一份

---

## 四、配置覆盖优先级总表

从低到高（后者覆盖前者）：

```
代码默认值 (NewK9s)
  ↓ 被覆盖
全局配置文件 ($XDG_CONFIG_HOME/k9s/config.yaml)
  ↓ 被覆盖
上下文配置文件 ($XDG_DATA_HOME/k9s/clusters/<cluster>/<context>/config.yaml)
  ↓ 被覆盖
CLI 启动参数 (manual* 字段)
  ↓ 被覆盖
环境变量 (仅少数几项)
```

### 环境变量覆盖项

| 环境变量 | 覆盖的配置 | 位置 |
|----------|-----------|------|
| `K9S_CONFIG_DIR` | 所有配置/数据目录 | [files.go#L21](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L21) |
| `K9S_LOGS_DIR` | 日志目录 | [files.go#L24](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L24) |
| `K9S_DEFAULT_PF_ADDRESS` | PortForwardAddress | [helpers.go#L16](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/helpers.go#L16) |
| `K9S_FEATURE_GATE_NODE_SHELL` | NodeShell FeatureGate | [data/helpers.go#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/helpers.go#L17) |

### 各配置项的运行时读取优先级

| 配置项 | getter 方法 | 优先级（高→低） |
|--------|-------------|-----------------|
| RefreshRate | [GetRefreshRate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L397-L415) | `manualRefreshRate` → `RefreshRate`（配置文件）→ `DefaultRefreshRate`(下限保护) |
| Headless | [IsHeadless()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L352-L358) | `UI.manualHeadless` → `UI.Headless` |
| Logoless | [IsLogoless()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L361-L367) | `UI.manualLogoless` → `UI.Logoless` |
| Crumbsless | [IsCrumbsless()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L370-L376) | `UI.manualCrumbsless` → `UI.Crumbsless` |
| Splashless | [IsSplashless()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L379-L385) | `UI.manualSplashless` → `UI.Splashless` |
| Invert | [IsInvert()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L388-L394) | `UI.manualInvert` → `UI.Invert` |
| ReadOnly | [IsReadOnly()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L423-L433) | `manualReadOnly`(CLI) → `Context.ReadOnly`(上下文) → `ReadOnly`(全局) |
| ActiveView | [ActiveView()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L211-L226) | `manualCommand`(CLI,仅一次) → `Context.View.Active` → `DefaultView`("po") |
| ScreenDumpDir | [AppScreenDumpDir()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L159-L170) | `manualScreenDumpDir`(CLI) → `ScreenDumpDir`(配置文件) → `AppDumpsDir`(默认) |
| Namespace | [Refine()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L124-L144) | `--all-namespaces`(CLI) → `--namespace`(CLI) → 上下文 active namespace → `"default"` |
| APITimeout | [Refine()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L94-L100) | `--request-timeout`(CLI) → `APIServerTimeout`(配置文件) → `DefaultCallTimeoutDuration` |
| Context | [Refine()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L101-L119) | `--context`(CLI) → kubeconfig current-context |

---

## 五、非法参数容错机制

### 5.1 配置文件校验（JSON Schema）

K9s 使用内嵌的 JSON Schema 对配置文件做严格校验。

**全局配置文件校验**——在 [Config.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L282-L284) 中：
```go
if err := data.JSONValidator.Validate(json.K9sSchema, bb); err != nil {
    errs = errors.Join(errs, fmt.Errorf("k9s config file %q load failed:\n%w", path, err))
}
```
- 校验失败时，错误被收集但**不中断加载**
- YAML 反序列化继续执行，未通过校验的字段被忽略（零值）
- 最终返回聚合错误

**上下文配置文件校验**——在 [Dir.loadConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L89-L94) 中：
```go
if err := JSONValidator.Validate(json.ContextSchema, bb); err != nil {
    slog.Warn("Validation failed. Please update your config and restart!", ...)
}
```
- 校验失败仅输出 Warn 日志，**不返回错误**，不影响程序运行

常见校验错误类型（来自测试用例 [config_test.go#L307-L333](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config_test.go#L307-L333)）：
- `Additional property xxx is not allowed` — 多余字段
- `Invalid type. Expected: boolean, given: string` — 类型不匹配

### 5.2 Validate() 中的值域保护

[K9s.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L436-L463) 对关键字段做合法性检查：

```go
if k.RefreshRate <= 0 {
    k.RefreshRate = defaultRefreshRate   // 无效值回退默认
}
if k.MaxConnRetry <= 0 {
    k.MaxConnRetry = defaultMaxConnRetry // 无效值回退默认
}
if k.PortForwardAddress == "" {
    k.PortForwardAddress = defaultPFAddress() // 空值回退
}
```

[Logger.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/logger.go#L43-L59) 做边界保护：

```go
if l.TailCount <= 0 { l.TailCount = DefaultLoggerTailCount }     // 太小回退
if l.TailCount > MaxLogThreshold { l.TailCount = MaxLogThreshold } // 太大截断
if l.BufferSize <= 0 || l.BufferSize > MaxLogThreshold { l.BufferSize = MaxLogThreshold }
if l.SinceSeconds == 0 { l.SinceSeconds = DefaultSinceSeconds }
if l.LogBufferSize <= 0 { l.LogBufferSize = DefaultLogBufferSize }
```

[Threshold.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/threshold.go#L64-L75) 和 [Severity.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/threshold.go#L35-L43)：

```go
// 值必须在 1-100 范围内，否则回退默认
func validateRange(v int) bool {
    return v > 0 && v <= 100
}
```

[ShellPod.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/shell_pod.go#L46-L53)：

```go
if s.Image == "" { s.Image = defaultDockerShellImage }
if len(s.Limits) == 0 { s.Limits = defaultLimits() }
```

[View.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/view.go#L19-L22)：

```go
if v.Active == "" { v.Active = DefaultView }  // 空视图回退到 "po"
```

### 5.3 RefreshRate 的下限保护

[GetRefreshRate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L397-L415) 专门处理了过低的刷新率：

```go
func (k *K9s) GetRefreshRate() float32 {
    rate := k.RefreshRate
    if k.manualRefreshRate != 0 {
        rate = k.manualRefreshRate
    }
    if rate < DefaultRefreshRate {
        if !k.refreshRateWarned {
            slog.Warn("Refresh rate is below minimum, capping to minimum value", ...)
            k.refreshRateWarned = true  // 只警告一次
        }
        return DefaultRefreshRate       // 强制不低于 2 秒
    }
    return rate
}
```

- 低于 `DefaultRefreshRate`(2.0) 的值会被强制提升到 2.0
- 仅在首次触发时打印警告（`refreshRateWarned` 标志）

### 5.4 ReadOnly 与 Write 的互斥逻辑

[Override()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L340-L348) 中的处理：

```go
if k9sFlags.ReadOnly != nil && *k9sFlags.ReadOnly {
    k.manualReadOnly = k9sFlags.ReadOnly      // --readonly → true
}
if k9sFlags.Write != nil && *k9sFlags.Write {
    var falseVal bool
    k.manualReadOnly = &falseVal              // --write → false
}
```

- `--write` 覆盖 `--readonly`（代码顺序在后，写操作覆盖读操作）
- 同时传 `--readonly --write` 时，write 生效

### 5.5 CLI Command 仅生效一次

[ActiveView()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L211-L226) 中的设计：

```go
if c.K9s.manualCommand != nil && *c.K9s.manualCommand != "" {
    v = *c.K9s.manualCommand
    *c.K9s.manualCommand = ""   // 用完清空，只在启动时生效一次
}
```

`--command` / `-c` 参数只在启动时决定初始视图，之后不再影响视图切换。

### 5.6 Namespace 空值保护

[Refine()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L139-L144) 中：

```go
if ns == "" {
    ns = client.DefaultNamespace  // 空命名空间回退到 "default"
}
```

### 5.7 上下文不存在的容错

- 激活不存在的上下文会返回错误，但不会 panic
- [loadConfiguration()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L108-L114) 中，即使配置加载出错，只要 `cfg != nil` 仍会继续创建 App
- 无上下文配置时跳过连接检查和配置保存

### 5.8 连接失败的容错

[loadConfiguration()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L133-L178) 收集所有错误但**不中断**：

```go
var errs error
// 连接失败
if err != nil { errs = errors.Join(errs, err) }
// 配置加载失败
if err := k9sCfg.Load(...); err != nil { errs = errors.Join(errs, err) }
// Refine 失败
if err := k9sCfg.Refine(...); err != nil { errs = errors.Join(errs, err) }
// 连接检查失败
if !conn.CheckConnectivity() { errs = errors.Join(errs, ...) }
```

所有错误通过 `errors.Join` 聚合后一起返回，由上层 `run()` 函数决定是否退出。

---

## 六、配置文件缺失时的自动生成行为

### 6.1 全局配置文件

[Config.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L272-L276)：

```go
if _, err := os.Stat(path); errors.Is(err, fs.ErrNotExist) {
    if err := c.Save(force); err != nil { return err }
}
```

文件不存在时，将当前内存中的配置（默认值 + 已加载的内容）写入磁盘。

### 6.2 上下文配置文件

[Dir.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L42-L54)：

```go
if errors.Is(err, fs.ErrNotExist) || (f != nil && f.Size() == 0) {
    return d.genConfig(path, ct)  // 生成默认上下文配置
}
```

文件不存在**或为空**时，都会生成新的默认配置。生成时调用 [NewConfig(ct)](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/config.go#L23-L27)，从 kubeconfig 的 api.Context 中提取 cluster 和 namespace 信息。

### 6.3 上下文配置保存时的合并

[Dir.Save()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L66-L79)：

```go
func (d *Dir) Save(path string, c *Config) error {
    if cfg, err := d.loadConfig(path); err == nil {
        c.Merge(cfg)   // 保存前先合并已有配置（保留 favorites 等）
    }
    // ... 写入磁盘
}
```

保存前会尝试加载已有配置并合并，避免覆盖用户已有的 favorites 等个性化设置。

---

## 七、配置体系架构图

```
┌─────────────────────────────────────────────────────────┐
│                     Config (config.go)                   │
│  ┌─────────────────────────────────────────────────────┐│
│  │                  K9s (k9s.go)                       ││
│  │  ┌─────────┐  ┌──────────┐  ┌───────────────────┐  ││
│  │  │  UI     │  │ Logger   │  │ ShellPod          │  ││
│  │  │ headless│  │ tail     │  │ image             │  ││
│  │  │ logoless│  │ buffer   │  │ namespace         │  ││
│  │  │ ...     │  │ ...      │  │ limits            │  ││
│  │  │ +manual*│  └──────────┘  └───────────────────┘  ││
│  │  └─────────┘                                        ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  ││
│  │  │Threshold │  │ImageScans│  │ manualRefreshRate│  ││
│  │  │ cpu/mem  │  │ enable   │  │ manualReadOnly   │  ││
│  │  │ warn/crit│  │ exclusns │  │ manualCommand    │  ││
│  │  └──────────┘  └──────────┘  │ manualScreenDump │  ││
│  │                              └──────────────────┘  ││
│  │  activeConfig → data.Config                        ││
│  │  ┌──────────────────────────────────────────────┐  ││
│  │  │         data.Config (data/config.go)         │  ││
│  │  │  ┌────────────────────────────────────────┐  │  ││
│  │  │  │        data.Context (data/context.go)  │  │  ││
│  │  │  │  ClusterName                           │  │  ││
│  │  │  │  ReadOnly *bool                        │  │  ││
│  │  │  │  Skin                                  │  │  ││
│  │  │  │  ┌────────────┐  ┌──────────────┐     │  │  ││
│  │  │  │  │ Namespace  │  │ View         │     │  │  ││
│  │  │  │  │ active     │  │ active ("po")│     │  │  ││
│  │  │  │  │ favorites  │  └──────────────┘     │  │  ││
│  │  │  │  └────────────┘                       │  │  ││
│  │  │  │  ┌────────────┐  ┌──────────────┐     │  │  ││
│  │  │  │  │FeatureGates│  │ Proxy        │     │  │  ││
│  │  │  │  └────────────┘  └──────────────┘     │  │  ││
│  │  │  └────────────────────────────────────────┘  │  ││
│  │  └──────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘

外部配置来源：
  config.yaml (全局)  ──Merge()──→  K9s 字段
  clusters/<c>/<ctx>/config.yaml  ──Dir.Load()──→  data.Config (activeConfig)
  CLI Flags  ──Override()──→  manual* 字段
  Env Vars  ──各处读取──→  PortForwardAddress / 路径 / FeatureGates
```

---

## 八、关键代码调用链

```
run()
 ├─ config.InitLogLoc()               [files.go#L89]
 ├─ config.InitLocs()                  [files.go#L116]
 └─ loadConfiguration()                [root.go#L133]
     ├─ client.NewConfig(k8sFlags)     [client/config.go#L40]
     ├─ config.NewConfig(k8sCfg)       [config.go#L31]
     │   └─ NewK9s(nil, ks)           [k9s.go#L69]  ← 默认值注入
     ├─ client.InitConnection()        [client/client.go#L67]
     ├─ k9sCfg.Load()                  [config.go#L271]
     │   ├─ JSONValidator.Validate()   [json/validator.go#L142]  ← Schema校验
     │   ├─ yaml.Unmarshal()           ← 文件反序列化
     │   └─ c.Merge(&cfg)             [config.go#L266] → [k9s.go#L129]  ← 文件覆盖默认
     ├─ k9sCfg.K9s.Override(k9sFlags)  [k9s.go#L330]  ← CLI参数写入manual*字段
     ├─ k9sCfg.Refine()                [config.go#L89]
     │   ├─ setK8sTimeout()            ← API超时覆盖
     │   ├─ k9sCfg.K9s.ActivateContext() [k9s.go#L256]
     │   │   ├─ k.dir.Load()           [data/dir.go#L35]  ← 加载上下文配置
     │   │   ├─ k.Validate()           [k9s.go#L436]  ← 值域保护
     │   │   └─ cfg.Context.Validate() ← 上下文级校验
     │   └─ c.SetActiveNamespace()     ← 命名空间设置
     ├─ conn.CheckConnectivity()       ← 连接检查
     └─ k9sCfg.Save()                  [config.go#L296]  ← 持久化
```
