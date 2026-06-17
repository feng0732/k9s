# K9s 启动配置加载、合并与容错机制全流程分析

> **核心发现**：全局配置文件合并不是"增量合并"，而是**破坏性全量赋值**。
> YAML 中缺少某个字段时，反序列化后的 Go **零值会覆盖代码默认值**，然后由 Validate / Refine / Getter 三道防线逐一修复。
> 这不是一个"设计优雅"的模式，而是一个"先破坏、再抢救"的过程。

---

## 一、启动入口总览

程序入口位于 [main.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/main.go#L44-L46)，调用 `cmd.Execute()`。核心启动逻辑在 [cmd/root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L76-L131) 的 `run()` 函数中，配置加载集中在 [loadConfiguration()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/cmd/root.go#L133-L178)。

### 启动时序

```
main() → cmd.Execute() → run()
  ├─ config.InitLogLoc()            [阶段0: 日志路径初始化]
  ├─ config.InitLocs()              [阶段1: 目录路径初始化]
  └─ loadConfiguration()            [阶段2: 配置加载核心]
       ├─ client.NewConfig(k8sFlags)       ← K8s 连接标志包装
       ├─ config.NewConfig(k8sCfg)         ← 默认值注入 (NewK9s)
       ├─ client.InitConnection()          ← K8s 连接
       ├─ k9sCfg.Load()                    ← 【关键】YAML反序列化 + Merge(全量赋值覆盖默认值)
       ├─ k9sCfg.K9s.Override(k9sFlags)    ← CLI 参数写入 manual* 字段
       ├─ k9sCfg.Refine()                  ← 【关键】激活上下文 + 部分字段回退 + Validate()
       └─ k9sCfg.Save()                    ← 持久化上下文配置
```

---

## 二、核心问题：Merge 的破坏性全量赋值

### 2.1 问题根源

[K9s.Merge()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L129-L156) 是理解整个优先级机制的关键。它不是"存在才覆盖"的增量合并，而是除 `Thresholds` 外**无条件 `=` 赋值**：

```go
func (k *K9s) Merge(k1 *K9s) {
    // 除了 GPUVendors 是追加、Thresholds 有 nil 保护外：
    k.LiveViewAutoRefresh = k1.LiveViewAutoRefresh  // 全量 =
    k.DefaultView = k1.DefaultView                  // 全量 =
    k.ScreenDumpDir = k1.ScreenDumpDir              // 全量 =
    k.RefreshRate = k1.RefreshRate                  // 全量 =
    k.APIServerTimeout = k1.APIServerTimeout        // 全量 =
    k.MaxConnRetry = k1.MaxConnRetry                // 全量 =
    k.ReadOnly = k1.ReadOnly                        // 全量 =
    // ... 其余所有字段都是直接 =
    k.ShellPod = k1.ShellPod                        // 指针直接 =（会变成 nil！）
    k.Logger = k1.Logger                            // struct 整体 =（所有字段变零值）
    k.ImageScans = k1.ImageScans                    // struct 整体 =
    k.UI = k1.UI                                    // struct 整体 =

    if k1.Thresholds != nil {                       // 唯一有 nil 保护的 map
        k.Thresholds = k1.Thresholds
    }
    for k, v := range k1.GPUVendors {               // 追加不覆盖
        KnownGPUVendors[k] = v
    }
}
```

### 2.2 场景复现

假设配置文件只写了 1 个字段：
```yaml
k9s:
  refreshRate: 10
```

此时 YAML 反序列化得到的 `k1` 结构体中：
- `k1.RefreshRate = 10`（用户显式设置）
- `k1.MaxConnRetry = 0`（缺省，Go int32 零值）
- `k1.APIServerTimeout = ""`（缺省，Go string 零值）
- `k1.ShellPod = nil`（缺省，Go 指针零值）
- `k1.Logger.TailCount = 0`（缺省，Go struct 内字段零值）
- `k1.Thresholds = nil`（缺省，Go map 零值）

**Merge 执行后**：所有缺省字段全部被零值覆盖！原本 `NewK9s()` 注入的默认值被洗成零。

### 2.3 与上下文配置 Merge 的根本差异

[上下文配置的 Merge](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/config.go#L29-L37) 完全相反——是温和的**增量合并且只合并 favorites**：

```go
func (c *Config) Merge(c1 *Config) {
    if c1 == nil { return }
    if c.Context != nil && c1.Context != nil {
        c.Context.merge(c1.Context)  // 只调用 Context.merge
    }
}
// Context.merge 只合并 Namespace.Favorites，其余字段完全不动
func (c *Context) merge(old *Context) {
    if old == nil || old.Namespace == nil { return }
    if c.Namespace == nil { c.Namespace = NewNamespace() }
    c.Namespace.merge(old.Namespace)  // 追加去重的 favorites，不覆盖其他
}
```

| 维度 | 全局 K9s.Merge | 上下文 data.Config.Merge |
|------|---------------|-------------------------|
| 策略 | 全量 `=` 赋值（破坏式） | 增量合并且仅 favorites（温和式） |
| 缺省字段 | 零值覆盖默认值 | 完全不影响原有值 |
| 触发场景 | 全局配置文件加载 | 上下文配置保存前（Dir.Save 中） |
| 目的 | 把文件内容替换进来 | 保留已有 favorites 不丢失 |

**重要：上下文配置文件加载（Dir.Load）根本不经过 Merge**——直接反序列化后整体 `setActiveConfig(cfg)` 替换，所以是"要么全有、要么全新"的替换策略，之后由 Validate 修复。

---

## 三、三道防线：Validate → Refine → Getter

Merge 把默认值洗成零值后，三道防线逐一抢救。以下是**每个字段的完整生命周期追踪**。

### 3.1 逐字段生命周期表

| 字段 | ① 默认值 (NewK9s) | ② Merge零值后 | ③ Validate 修复 | ④ Refine 修复 | ⑤ Getter 最终修复 | 风险评估 |
|------|------------------|---------------|----------------|---------------|-----------------|----------|
| RefreshRate | `2.0` | `0` | `if <=0 → 2.0` ✅ | - | `if <2.0 → 2.0` 且警告一次 | ✅ 完全修复 |
| MaxConnRetry | `5` | `0` | `if <=0 → 5` ✅ | - | 无 | ✅ 完全修复 |
| APIServerTimeout | `"2m0s"` | `""` | 不修复 ❌ | `ParseDuration("")`失败→`120s` ✅ | 无 | ✅ Refine 兜底 |
| PortForwardAddress | `"localhost"` | `""` | `if "" → env → "localhost"` ✅ | - | 无 | ✅ 完全修复 |
| ScreenDumpDir | `AppDumpsDir`(路径) | `""` | 不修复 ❌ | 确保目录存在（但不回退值） | `if "" → AppDumpsDir` ✅ | ✅ Getter 兜底 |
| ShellPod | `*ShellPod{busybox:1.37.0}` | `nil` | `if !=nil才调Validate`（nil 跳过不修复）❌ | - | 无（exec.go 直接访问） | ⚠️ 真实风险：可能 panic |
| Logger.TailCount | `100` | `0` | `if <=0 → 100` ✅ | - | 无 | ✅ 完全修复 |
| Logger.BufferSize | `5000` | `0` | `if 非法 → 5000` ✅ | - | 无 | ✅ 完全修复 |
| Logger.SinceSeconds | `-1` | `0` | `if ==0 → -1` ✅ | - | 无 | ✅ 完全修复 |
| Logger.LogBufferSize | `50` | `0` | `if <=0 → 50` ✅ | - | 无 | ✅ 完全修复 |
| Thresholds | `map{CPU,MEM}` | `nil` | **Merge 时 nil 不覆盖**（根本没破坏） | - | `Validate()`补全缺失的 key | ✅ nil 保护 |
| ImageScans | `ImageScans{}` | `零值struct` | 不修复（零值语义正确） | - | 无 | ✅ 零值无害 |
| DefaultView | `""` | `""` | 不修复 | - | `ActiveView()` 回退上下文 → `"po"` | ✅ 间接兜底 |
| ReadOnly | `false` | `false` | 零值恰好等于默认值 ✅ | - | `IsReadOnly()`三级判断 | 🟡 隐性安全 |
| UI.* 所有 bool | 都是 `false` | 都是 `false` | 零值恰好等于默认值 ✅ | - | `IsXxx()` 返回 manual* 或 UI字段 | 🟡 隐性安全 |
| LiveViewAutoRefresh | `false` | `false` | 零值恰好等于默认值 ✅ | - | 无 | 🟡 隐性安全 |
| NoExitOnCtrlC | `false` | `false` | 零值恰好等于默认值 ✅ | - | 无 | 🟡 隐性安全 |
| SkipLatestRevCheck | `false` | `false` | 零值恰好等于默认值 ✅ | - | 无 | 🟡 隐性安全 |
| DisablePodCounting | `false` | `false` | 零值恰好等于默认值 ✅ | - | 无 | 🟡 隐性安全 |
| GPUVendors | 包级 KnownGPUVendors | 零值不覆盖 | 循环追加模式（而非赋值） | - | 读取 KnownGPUVendors | ✅ 追加模式 |

**结论**：bool 类字段的"零值恰好等于默认值"是一种**隐性安全**——不是设计使然，而是恰好匹配。如果未来某字段默认值变成 true，Merge 后会被静默地破坏成 false。

### 3.2 风险点：ShellPod = nil 可能 panic

在 [view/exec.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/view/exec.go#L342-L349) 中：

```go
// 第 342 行：直接访问 .Namespace，无 nil 检查
ns := a.Config.K9s.ShellPod.Namespace

// 第 349 行：直接赋值后访问 .Command，无 nil 检查
cfg := a.Config.K9s.ShellPod
if len(cfg.Command) > 0 { ... }
```

只有在 [nukeK9sShell()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/view/exec.go#L381-L387) 中做了 nil 判断：
```go
if !ct.FeatureGates.NodeShell || a.Config.K9s.ShellPod == nil {
    return nil  // 有保护
}
```

**触发路径**：当用户全局配置文件完全不写 `shellPod:` 字段（缺省）→ Merge 后 ShellPod = nil → 执行 kubectl exec 进入节点 shell（NodeShell FeatureGate 开启）时，`sshIn()` 函数第 342/349 行会 nil pointer panic。

**为什么目前没大量发生**：
1. 大部分用户的全局配置文件是 k9s 自动生成的完整字段，不是手写缺省的
2. NodeShell 功能本身是 FeatureGate 控制的，默认可能未启用

---

## 四、真实的配置覆盖优先级（逐字段级）

### 4.1 优先级总链（修正版）

```
代码默认值 (NewK9s)
  ↓ 被【全量零值覆盖】     ← K9s.Merge()
全局配置文件 ($XDG_CONFIG_HOME/k9s/config.yaml)
  ↓ 被【Validate 部分修复】 ← K9s.Validate()
  ↓ 被【Refine 部分修复】   ← Config.Refine() → setK8sTimeout
上下文配置文件 (clusters/<c>/<ctx>/config.yaml)
  ↓ 整体替换 activeConfig   ← setActiveConfig()
  ↓ 被【Context.Validate修复】← data.Context.Validate()
CLI 启动参数 (写入 manual* 字段)
  ↓ 被【Getter 最终裁决】   ← IsXxx() / GetRefreshRate() / IsReadOnly()
环境变量 (仅少数几项)       ← Validate 中读 env 优先级最高
```

### 4.2 各字段的真实运行时读取路径

#### RefreshRate

```go
// 调用链：GetRefreshRate()
1. 有 manualRefreshRate（CLI --refresh != 2.0） → 用它
2. 否则用 RefreshRate 字段值（已被 Merge + Validate 修复过）
3. 如果 < 2.0 → 强制返回 2.0，且只警告一次
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L397-L415)

#### UI 类布尔 (Headless/Logoless/Crumbsless/Splashless/Invert)

```go
// 调用链：IsHeadless() 等
1. 有 manualHeadless（CLI 传了 --headless）→ 用它
2. 否则用 UI.Headless（Merge 后零值恰好为 false = 默认值）
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L352-L394)

#### ReadOnly —— 三级判断最复杂

```go
// 调用链：IsReadOnly()
1. 有 manualReadOnly（CLI --readonly 或 --write）→ 用它
   - 注意 --write 优先级高于 --readonly：两个都传时 write 把 manualReadOnly 设为 false
2. 否则看上下文配置：Context.ReadOnly != nil → 用 Context.ReadOnly
3. 否则用全局 K9s.ReadOnly（Merge 后零值恰好为 false = 默认值）
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L423-L433)

#### ActiveView —— 四级回退

```go
// 调用链：ActiveView()
1. 有 manualCommand 且非空（CLI --command / -c）→ 用它，用完清空（仅启动时一次生效）
2. 上下文配置 Context.View.Active（Validate 中空值回退 "po"）
3. K9s.DefaultView（可能为 ""）
4. data.DefaultView（硬编码 "po"）
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L211-L226)

#### ScreenDumpDir

```go
// 调用链：AppScreenDumpDir()
1. 有 manualScreenDumpDir（CLI --screen-dump-dir）→ 用它（还会写入字段）
2. ScreenDumpDir 字段（Merge 后可能为 ""）
3. 字段为 "" → 回退到全局常量 AppDumpsDir
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L159-L170)

#### APIServerTimeout

```go
// 不经过 getter，在 Refine() 中设置
1. CLI --request-timeout 有值 → 直接用（不经过配置文件）
2. 否则解析 K9s.APIServerTimeout 字段
3. 解析失败（如 Merge 后空字符串）→ 回退 DefaultCallTimeoutDuration(120s)
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L89-L100)

#### PortForwardAddress

```go
// Validate 中优先级最高：
1. K9S_DEFAULT_PF_ADDRESS 环境变量非空 → 用 env（最高优先级！）
2. PortForwardAddress == "" → 回退 defaultPFAddress() = "localhost"
3. 否则用字段值
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L444-L449)

#### Namespace

```go
// 在 Refine() 中决定
1. CLI --all-namespaces / -A → ns = ""（表示所有命名空间）
2. CLI --namespace / -n 有值 → 用它
3. 以上都无 → 从上下文配置 activeNamespace 取
4. activeNamespace 还是空 → 回退 client.DefaultNamespace("default")
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L124-L144)

#### Context

```go
// 在 Refine() 中决定
1. CLI --context 有值 → 用它激活
2. 否则从 kubeconfig 取 CurrentContextName() 激活
```
[源码位置](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L101-L119)

---

## 五、非法参数容错机制全量梳理

### 5.1 配置文件校验（JSON Schema）

**全局配置文件校验**——[Config.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/config.go#L282-L284)：
- 校验失败，错误被收集但不中断加载
- YAML 反序列化继续，未通过校验的字段视为缺省（产生零值 → 进入 Merge → 被 Validate 修复）

**上下文配置文件校验**——[Dir.loadConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L89-L94)：
- 校验失败仅输出 Warn 日志，不返回错误

常见校验错误类型：
- `Additional property xxx is not allowed` — 多余字段（typo 会被忽略且零值化）
- `Invalid type. Expected: boolean, given: string` — 类型不匹配

### 5.2 Validate() 值域保护修复矩阵

[K9s.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/k9s.go#L436-L463)：
| 修复项 | 条件 | 回退值 |
|--------|------|--------|
| RefreshRate | `<= 0` | `2.0` |
| MaxConnRetry | `<= 0` | `5` |
| PortForwardAddress | `== ""` | 先读 `K9S_DEFAULT_PF_ADDRESS` env，再 `"localhost"` |
| Logger | 整体 | `Logger.Validate()` 返回值（赋值回去） |
| Thresholds | 整体 | `Thresholds.Validate()` 返回值 |
| ShellPod | `!= nil` 才调 Validate | nil 时不修复，潜在风险 |
| activeConfig | `== nil` | 尝试重新激活上下文 |

[Logger.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/logger.go#L43-L59)：
| 修复项 | 条件 | 回退值 |
|--------|------|--------|
| TailCount | `<=0` → `<=100` → `>5000` 截断 | `100` / `5000` |
| BufferSize | `<=0` 或 `>5000` | `5000` |
| SinceSeconds | `== 0` | `-1`(all logs) |
| LogBufferSize | `<=0` | `50` |

[Thresholds.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/threshold.go#L64-L75)：
- 补全缺失的 CPU / MEM key
- 子 Severity.Warn / Crit 值不在 1-100 范围 → 回退默认 70/90

[ShellPod.Validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/shell_pod.go#L46-L53)：
- Image 为空 → `busybox:1.37.0`
- Limits 为空 → `{cpu: 100m, memory: 100Mi}`
- **但整个 ShellPod 为 nil 时，父级 Validate 不调用本方法**

### 5.3 其他容错点

**RefreshRate 下限双重保护**：
- Validate 时 `<=0 → 2.0`
- Getter 时 `<2.0 → 2.0` 且只警告一次（`refreshRateWarned` 单例标志）

**ReadOnly / Write 互斥**：
- `--write` 代码分支在 `--readonly` 之后执行，直接覆盖 `manualReadOnly = false`
- 两参数同传时 write 生效（代码顺序决定优先级）

**Command 一次性消费**：
- `manualCommand` 使用完置空，只在启动时决定初始视图
- 后续运行时切换视图不被 CLI 参数持续干扰

**Save 前 favorites 合并保护**：
- [Dir.Save()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L66-L79) 保存前先 `loadConfig()` 把旧 favorites 合并进来，避免覆盖用户收藏

**上下文文件缺失/空文件自动生成**：
- [Dir.Load()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/data/dir.go#L42-L54) 中 `NotExist` 或 `Size()==0` 都自动生成默认配置

**错误聚合不中断**：
- loadConfiguration 中 `errors.Join` 收集所有错误（连接失败/配置加载失败/连接检查失败），聚合后统一返回，不中途 panic

---

## 六、配置文件来源与路径

### 6.1 路径初始化

[InitLogLoc()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L89-L113) + [InitLocs()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-k9s/internal/config/files.go#L116-L122)：

| 路径 | 环境变量优先 | XDG 标准路径 | K9S_CONFIG_DIR 合并 |
|------|--------------|-------------|-------------------|
| AppConfigFile | - | `$XDG_CONFIG_HOME/k9s/config.yaml` | `$K9S_CONFIG_DIR/config.yaml` |
| AppContextsDir | - | `$XDG_DATA_HOME/k9s/clusters` | `$K9S_CONFIG_DIR/clusters` |
| AppDumpsDir | - | `$XDG_STATE_HOME/k9s/screen-dumps` | `$K9S_CONFIG_DIR/screen-dumps` |
| AppLogFile | `K9S_LOGS_DIR` 优先 | `$XDG_STATE_HOME/k9s/k9s.log` | `$K9S_CONFIG_DIR/k9s.log` |

### 6.2 三层配置来源

1. **全局配置文件** (`config.yaml`)
   - 管理：RefreshRate / ReadOnly / UI / ShellPod / Logger / Thresholds 等 K9s 全局行为
   - 加载：`Config.Load()` → YAML → Merge(全量赋值破坏默认值)
   
2. **上下文配置文件** (`clusters/<cluster>/<context>/config.yaml`)
   - 管理：Namespace favorites / View.Active / Skin / Context.ReadOnly / Proxy / FeatureGates
   - 加载：`Dir.Load()` → YAML → 整体替换 activeConfig（不 Merge）

3. **启动参数 + 环境变量**
   - 通过 `manual*` 字段或 getter 内直接读 env，优先级最高

---

## 七、关键代码调用链

```
run()
 ├─ config.InitLogLoc()               [files.go#L89]
 ├─ config.InitLocs()                 [files.go#L116]
 └─ loadConfiguration()               [root.go#L133]
     ├─ client.NewConfig(k8sFlags)    [client/config.go#L40]
     ├─ config.NewConfig(k8sCfg)      [config.go#L31]
     │   └─ NewK9s(nil, ks)          [k9s.go#L69]       ← ① 代码默认值注入
     ├─ client.InitConnection()       [client/client.go#L67]
     ├─ k9sCfg.Load()                 [config.go#L271]
     │   ├─ JSONValidator.Validate()  [json/validator.go]  ← Schema 校验（失败不中断）
     │   ├─ yaml.Unmarshal()          ← 缺省字段 → Go零值
     │   └─ c.Merge(&cfg)            [config.go#L266] → [k9s.go#L129]  ← ② 零值覆盖默认值！
     ├─ k9sCfg.K9s.Override(k9sFlags) [k9s.go#L330]      ← ③ CLI 写入 manual* 字段
     ├─ k9sCfg.Refine()               [config.go#L89]
     │   ├─ setK8sTimeout()           [config.go#L60]   ← ④ API 超时字段回退
     │   ├─ k9sCfg.K9s.ActivateContext() [k9s.go#L256]
     │   │   ├─ k.dir.Load()          [data/dir.go#L35]  ← 上下文配置整体替换
     │   │   ├─ k.Validate()          [k9s.go#L436]      ← ⑤ Validate 批量修复
     │   │   └─ cfg.Context.Validate() [data/context.go] ← 上下文内部修复
     │   └─ c.SetActiveNamespace()    [config.go#L150]   ← 命名空间最终回退
     ├─ conn.CheckConnectivity()      ← 错误收集不中断
     └─ k9sCfg.Save()                 [config.go#L296]   ← 上下文保存前 Merge favorites
```

---

## 八、设计总结与潜在问题

### 设计优点

1. **CLI 参数不污染持久化**：`manual*` 字段模式使得命令行参数只影响当前运行，不会被 Save() 写入配置文件
2. **多道容错防线**：即使 Merge 破坏了默认值，也有 3-4 层回退机制
3. **上下文配置隔离**：每个集群/上下文独立配置，切换无干扰
4. **Schema 校验**：非法字段（拼写错误、类型错误）至少能给出 Warn/Error，不会静默吞掉用户配置意图

### 设计隐患

1. **ShellPod nil 风险**：全局配置缺省 shellPod 字段 + 运行节点 shell → 潜在 panic
2. **bool 隐性安全**：所有 bool 默认值恰好为 false，掩盖了 Merge 破坏式赋值的问题；未来如果某 bool 默认值改成 true，会被静默破坏
3. **Validate 不完整**：APIServerTimeout、ScreenDumpDir、DefaultView、ShellPod nil 等情况 Validate 不修复，依赖后续的 Refine/Getter 或调用链上的间接兜底
4. **Merge 语义不一致**：全局 K9s.Merge 是破坏式，上下文 data.Config.Merge 是温和式，两种策略增加理解成本
5. **用户配置文件字段的"隐形强制全写"**：如果用户手写最小配置文件，表面功能正常但内部字段都被洗过一轮零值再修复，有潜在的调试困难
