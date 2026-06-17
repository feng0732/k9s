# Screen Dump 保存与读取实现脉络

## 一、概述

Screen Dump（屏幕快照）是 k9s 提供的一项功能，允许用户将当前视图的内容保存到本地文件系统，并可以后续查看和管理这些快照文件。保存的内容包括表格数据、日志、YAML 详情等多种格式。

---

## 二、保存快照（Save）

### 2.1 触发方式

**统一快捷键：`Ctrl+S`**

在支持的视图中按下 `Ctrl+S` 即可触发保存操作。以下视图均支持此功能：

| 视图类型 | 触发方法 | 保存格式 | 核心代码位置 |
|---------|---------|---------|-------------|
| 表格视图 (Table) | `saveCmd` | CSV | [table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table.go#L215-L220) |
| 日志视图 (Log) | `SaveCmd` | LOG | [log.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/log.go#L422-L431) |
| 详情视图 (Details) | `saveCmd` | YAML | [details.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/details.go#L309-L314) |
| 实时视图 (LiveView) | `saveCmd` | YAML | [live_view.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/live_view.go#L379-L384) |
| Logger 视图 | `saveCmd` | YAML | [logger.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/logger.go#L157-L162) |

### 2.2 保存实现

保存操作主要有三个核心函数，分别处理不同格式的文件：

##### 2.2.1 表格保存 (CSV) - `saveTable`

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L45-L84)

```go
func saveTable(dir, title, path string, mdata *model1.TableData) (string, error)
```

- 输入：目录路径、资源标题、资源路径、表格数据
- 输出：保存的文件路径

**调用入口**（[table.go#L215-L223](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table.go#L215-L223)）：

```go
saveTable(
    t.app.Config.K9s.ContextScreenDumpDir(),  // 目录
    t.GVR().R(),                               // title: 资源名，如 "pods", "nodes"
    t.Path,                                    // path: 来自 KeyPath，通常是选中资源的 FQN
    t.GetFilteredData(),                       // 表格数据
)
```

**`t.Path` 参数说明**：
- 来自 context 中的 `internal.KeyPath`（[browser.go#L266-L267](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/browser.go#L266-L267)）
- 通常是选中资源的 FQN（完全限定名），如 `"default/nginx-pod"`
- 在顶层列表视图（无具体资源选中）下为空字符串 `""`

**命名空间预处理（关键逻辑）**：

```go
ns := mdata.GetNamespace()
if client.IsClusterWide(ns) {
    ns = client.NamespaceAll  // 统一转换为 "all"
}
```

这里是整个命名逻辑的关键转换点，详细说明见第三章。

- 完整流程：
  1. 获取表格的命名空间 `ns`
  2. 如果 `ns` 是集群范围相关值，统一转换为 `"all"`
  3. 通过 `computeFilename` 计算文件名（传入转换后的 ns）
  4. 以 `0600` 权限创建文件
  5. 使用 `csv.Writer` 写入列名和所有行数据
  6. 刷新缓冲区并返回文件路径

#### 2.2.2 日志保存 (LOG) - `saveData`

位置：[log.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/log.go#L437-L461)

```go
func saveData(dir, fqn, logs string) (string, error)
```

- 输入：目录路径、资源全限定名、日志文本
- 输出：保存的文件路径
- 格式：`{fqn}-{timestamp}.log`

#### 2.2.3 YAML 保存 - `saveYAML`

位置：[yaml.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/yaml.go#L72-L97)

```go
func saveYAML(dir, name, raw string) (string, error)
```

- 输入：目录路径、名称、原始文本
- 输出：保存的文件路径
- 格式：`{name}--{timestamp}.yaml`

---

## 三、文件命名规则

### 3.1 关键常量定义

位置：[types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/types.go#L16-L55)

| 常量名 | 值 | 含义 |
|--------|----|------|
| `NamespaceAll` | `"all"` | 表示"所有命名空间"视图 |
| `ClusterScope` | `"-"` | 表示资源本身是集群范围的（无命名空间） |
| `BlankNamespace` | `""` | 空命名空间 |

### 3.2 `IsClusterWide` 判断函数

位置：[helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/helpers.go#L20-L23)

```go
func IsClusterWide(ns string) bool {
    return ns == NamespaceAll || ns == BlankNamespace || ns == ClusterScope
}
```

该函数在 `ns` 为以下任意值时返回 `true`：
- `"all"` - 用户切换到所有命名空间视图
- `""` - 空命名空间
- `"-"` - 资源本身是集群范围的（如 nodes、namespaces）

### 3.3 从保存入口到文件名生成的完整路径

**核心流程**：

```
saveTable()
    ↓ 步骤1
ns = mdata.GetNamespace()  // 获取当前命名空间
    ↓ 步骤2
if IsClusterWide(ns) {     // 判断是否为集群范围相关
    ns = NamespaceAll      // 统一转换为 "all" ← 关键转换！
}
    ↓ 步骤3
computeFilename(dir, ns, title, path)  // 传入转换后的 ns
    ↓ 步骤4
if ns == ClusterScope {    // 判断 ns == "-"
    // 注意：此分支实际上永远不会执行！
    // 因为步骤2已经把所有符合条件的 ns 都转换成了 "all"
} else {
    fName = fmt.Sprintf(FullFmat, name, ns, now)  // 始终走此分支
}
```

**关键发现**：
- `computeFilename` 中的 `if ns == client.ClusterScope` 判断实际上**永远不会成立**
- 因为在 `saveTable` 中，所有 `IsClusterWide` 的值（包括 `"-"`）都已被提前替换为 `"all"`
- 因此，**所有表格快照的文件名都会包含命名空间部分**，使用 `FullFmat` 格式

### 3.4 命名计算函数 - `computeFilename`

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L22-L43)

核心逻辑（注意 `path` 对文件名的影响）：

```go
func computeFilename(dumpPath, ns, title, path string) (string, error) {
    now := time.Now().UnixNano()
    
    // 构建名称主体：分两种情况
    // 情况1：path 为空（顶层列表视图）→ name = title
    // 情况2：path 非空（选中了具体资源）→ name = title + "-" + SanitizeFileName(path)
    name := title + "-" + data.SanitizeFileName(path)
    if path == "" {
        name = title  // ← 此时不会出现多余的 "-"
    }

    // 由于 saveTable 中的转换，ns 只能是具体命名空间或 "all"
    // 所以这里始终使用 FullFmat 格式
    if ns == client.ClusterScope {
        fName = fmt.Sprintf(ui.NoNSFmat, name, now)  // 实际上永远不会执行
    } else {
        fName = fmt.Sprintf(ui.FullFmat, name, ns, now)  // "%s-%s-%d.csv"
    }

    return strings.ToLower(filepath.Join(dir, fName)), nil
}
```

**path 参数对文件名的影响详解**：

| path 取值 | 来源场景 | `name` 构建方式 | `name` 结果（以 pods 为例） |
|----------|---------|----------------|---------------------------|
| `""` | 顶层列表视图、未选中具体资源 | `name = "pods"` | `"pods"` |
| `"default/nginx-pod"` | 选中了 default 命名空间下的 nginx-pod | `name = "pods-" + "default-nginx-pod"` | `"pods-default-nginx-pod"` |
| `"workload-abc"` | 选中了某个非 namespaced 路径（如 workload） | `name = "pods-" + "workload-abc"` | `"pods-workload-abc"` |

`SanitizeFileName` 会把 path 中的 `/` 替换为 `-`。

### 3.5 文件名格式常量

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/ui/table_helper.go#L35-L39)

| 常量名 | 格式 | 实际使用场景 |
|--------|------|-------------|
| `FullFmat` | `%s-%s-%d.csv` | **所有表格快照**（包括集群范围资源） |
| `NoNSFmat` | `%s-%d.csv` | 定义但未使用（死代码） |

最终文件名格式统一为：`{name}-{ns}-{timestamp}.csv`，其中 `{name}` 取决于 path 是否为空。

### 3.6 三种命名空间场景 × 两种 path 情况的实际命名

共产生 **6 种组合**的文件名：

| 场景 | 原始 ns | 转换后 ns | path 取值 | `name` | 最终文件名格式 | 示例 |
|-----|---------|----------|----------|--------|--------------|------|
| 具体命名空间 | `"default"` | `"default"` | `""`（空） | `"pods"` | `{title}-{ns}-{timestamp}.csv` | `pods-default-1620000000000000000.csv` |
| 具体命名空间 | `"default"` | `"default"` | `"default/nginx-pod"` | `"pods-default-nginx-pod"` | `{name}-{ns}-{timestamp}.csv` | `pods-default-nginx-pod-default-1620000000000000000.csv` |
| all 命名空间视图 | `"all"` | `"all"` | `""`（空） | `"pods"` | `{title}-all-{timestamp}.csv` | `pods-all-1620000000000000000.csv` |
| all 命名空间视图 | `"all"` | `"all"` | `"default/nginx-pod"` | `"pods-default-nginx-pod"` | `{name}-all-{timestamp}.csv` | `pods-default-nginx-pod-all-1620000000000000000.csv` |
| 集群范围资源 | `"-"` | `"all"` | `""`（空） | `"nodes"` | `{title}-all-{timestamp}.csv` | `nodes-all-1620000000000000000.csv` |
| 集群范围资源 | `"-"` | `"all"` | `"node-1"` | `"nodes-node-1"` | `{name}-all-{timestamp}.csv` | `nodes-node-1-all-1620000000000000000.csv` |

**关键说明**：
- **all 命名空间 + path 为空**：文件名格式为 `{title}-all-{timestamp}.csv`，注意**没有双横杠**，因为 `path == ""` 时 `name = title`，不会拼接多余的 `-`
- **all 命名空间 + path 非空**：path 会被清洗后拼接到 title 后，再追加 `-all-{timestamp}.csv`
- **集群范围资源**：ns 被转换为 `"all"`，所以文件名中也会包含 `-all-`

### 3.7 时间戳

使用 `time.Now().UnixNano()` 纳秒级时间戳，确保文件名唯一性。

### 3.8 文件名清洗 - `SanitizeFileName`

位置：[helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/data/helpers.go#L27-L29)

```go
var invalidPathCharsRX = regexp.MustCompile(`[:/]+`)

func SanitizeFileName(name string) string {
    return invalidPathCharsRX.ReplaceAllString(name, "-")
}
```

将 `:` 和 `/` 替换为 `-`，确保文件名合法。这也会影响资源 path 的拼接结果。

### 3.9 不同类型的文件命名总结

| 类型 | 说明 | 命名模式 | 示例 |
|-----|------|---------|------|
| 表格（具体命名空间，path 空） | 列表视图、选中 default ns | `{title}-{ns}-{timestamp}.csv` | `pods-default-1620000000000000000.csv` |
| 表格（具体命名空间，path 非空） | 选中某资源后保存 | `{title}-{sanitized_path}-{ns}-{timestamp}.csv` | `pods-default-nginx-pod-default-1620000000000000000.csv` |
| 表格（all 命名空间，path 空） | 列表视图、选中 all ns | `{title}-all-{timestamp}.csv` | `pods-all-1620000000000000000.csv` |
| 表格（all 命名空间，path 非空） | all ns 下选中某资源 | `{title}-{sanitized_path}-all-{timestamp}.csv` | `pods-default-nginx-pod-all-1620000000000000000.csv` |
| 表格（集群范围资源，path 空） | nodes 列表视图 | `{title}-all-{timestamp}.csv` | `nodes-all-1620000000000000000.csv` |
| 表格（集群范围资源，path 非空） | 选中某 node 后保存 | `{title}-{sanitized_path}-all-{timestamp}.csv` | `nodes-node-1-all-1620000000000000000.csv` |
| 日志 | 保存容器日志 | `{fqn}-{timestamp}.log` | `default-nginx-7f-1620000000000000000.log` |
| YAML 详情 | 保存资源 YAML | `{name}--{timestamp}.yaml` | `pod-nginx--1620000000.yaml` |

---

## 四、目录配置来源与覆盖关系

### 4.1 三个配置来源

Screen dump 目录有三个配置来源，按优先级从低到高：

| 优先级 | 来源 | 存储/传入方式 | 代码位置 |
|--------|------|-------------|---------|
| 1（最低） | 系统默认 `AppDumpsDir` | 全局变量 | [files.go#L60-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L60-L61) |
| 2 | 配置文件 `k9s.screenDumpDir` | YAML 字段 | [k9s.go#L39](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L39) |
| 3（最高） | 命令行 `--screen-dump-dir` | CLI 标志 | [root.go#L267-L272](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L267-L272) |

### 4.2 启动时的初始化流程

在 [root.go#L133-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L133-L178) 的 `loadConfiguration()` 中，按以下顺序执行：

```
步骤0: init() 函数执行                ← root.go#L55-L67
    ↓  NewFlags() → ScreenDumpDir = &AppDumpsDir
    ↓  initK9sFlags() 注册 StringVar，pflag 默认值为 ""
    ↓  cobra.Execute() 解析命令行参数
    ├─  用户未指定 --screen-dump-dir → *ScreenDumpDir = ""（覆盖！）
    └─  用户指定了 → *ScreenDumpDir = 用户值

步骤1: config.InitLocs()                     ← root.go#L77
    ↓  初始化 AppDumpsDir 全局变量（系统默认值）
步骤2: NewK9s()                               ← config.go#L34
    ↓  K9s.ScreenDumpDir = AppDumpsDir        ← k9s.go#L75
步骤3: k9sCfg.Load(AppConfigFile, false)      ← root.go#L146
    ↓  从 YAML 文件读取 screenDumpDir 字段
    ↓  通过 Merge() 覆盖 K9s.ScreenDumpDir    ← k9s.go#L140
步骤4: k9sCfg.K9s.Override(k9sFlags)          ← root.go#L149
    ↓  k.manualScreenDumpDir = k9sFlags.ScreenDumpDir  ← 无条件赋值指针
    ├─  用户未指定 → 指向的值为 ""
    └─  用户指定了 → 指向的值为用户路径
步骤5: k9sCfg.Refine()                        ← root.go#L150
    ↓  调用 AppScreenDumpDir() 确定最终目录
    ├─  isStringSet(manualScreenDumpDir) 判断
    ├─  命令行非空则覆盖，否则用配置文件值
    └─  确保最终目录存在                         ← config.go#L146
```

### 4.3 来源一：系统默认 `AppDumpsDir`

位置：[files.go#L115-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L115-L122)

`AppDumpsDir` 是一个包级全局变量，在 `InitLocs()` 中初始化，有两种路径：

**情况 A：设置了 `K9S_CONFIG_DIR` 环境变量**

```
AppDumpsDir = $K9S_CONFIG_DIR/screen-dumps
```

- 位置：[files.go#L124-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L124-L133)
- 初始化时立即创建目录
- `K9S_CONFIG_DIR` 环境变量通过 `isEnvSet()` 检测（值不为空即视为设置）

**情况 B：未设置 `K9S_CONFIG_DIR`（默认）**

```
AppDumpsDir = $XDG_STATE_HOME/k9s/screen-dumps
```

- 位置：[files.go#L166-L193](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L190-L193)
- 使用 `xdg.StateFile()` 计算路径
- 在 Linux 上默认为 `~/.local/state/k9s/screen-dumps`
- 在 macOS 上默认为 `~/Library/Application Support/k9s/screen-dumps`

**`AppDumpsDir` 的角色**：它是最低优先级的兜底值，只在配置文件和命令行都未指定时生效。

### 4.4 来源二：配置文件 `k9s.screenDumpDir`

位置：[k9s.go#L39](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L39)

```yaml
k9s:
  screenDumpDir: /tmp/screen-dumps
```

**读取流程**：

1. `k9sCfg.Load(AppConfigFile)` 从 `config.yaml` 读取（[config.go#L271-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/config.go#L271-L293)）
2. 反序列化 YAML 到 `Config.K9s.ScreenDumpDir` 字段
3. 通过 `Config.Merge()` → `K9s.Merge()` 将值写入当前配置（[k9s.go#L140](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L140)）

```go
func (k *K9s) Merge(k1 *K9s) {
    // ...
    k.ScreenDumpDir = k1.ScreenDumpDir  // 直接覆盖
    // ...
}
```

**注意**：`Merge()` 是无条件覆盖，即使 YAML 中的值为空字符串也会覆盖掉 `NewK9s()` 中设置的默认值。但 YAML 标签有 `omitempty`，所以保存时如果值为空不会写入文件。

### 4.5 来源三：命令行 `--screen-dump-dir`

**标志注册**：[root.go#L267-L272](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L267-L272)

```go
rootCmd.Flags().StringVar(
    k9sFlags.ScreenDumpDir,
    "screen-dump-dir",
    "",                          // pflag 默认值为空字符串 ← 关键！
    "Sets a path to a dir for a screen dumps",
)
```

**Flags 初始化**：[flags.go#L49](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/flags.go#L49)

```go
func NewFlags() *Flags {
    return &Flags{
        // ...
        ScreenDumpDir: strPtr(AppDumpsDir),  // NewFlags 默认值是 AppDumpsDir
    }
}
```

**关键细节 - pflag 的值覆盖行为**：

pflag 的 `StringVar` 工作方式是：
- 它接收一个指针 `p *string` 和默认值 `value`
- 在命令行解析阶段，**如果用户没有指定该标志**，它会将 `*p` 设置为 `value`（即 `""`）
- 如果用户指定了，它会将 `*p` 设置为用户输入的值

**执行时序**：

```
1. NewFlags() 执行                    → *ScreenDumpDir = AppDumpsDir（非空）
2. initK9sFlags() 注册 StringVar     → pflag 记住这个指针和默认值 ""
3. 命令行解析阶段（cobra.Execute）    → 如果用户未指定，*ScreenDumpDir = ""（覆盖！）
                                          如果用户指定了，*ScreenDumpDir = 用户值
```

所以 `NewFlags()` 中设置的 `AppDumpsDir` 实际上会被 pflag **覆盖**，它只在注册标志时作为指针的初始值存在。

**注入到 K9s 配置**：[k9s.go#L348](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L348)

```go
func (k *K9s) Override(k9sFlags *Flags) {
    // ...
    k.manualScreenDumpDir = k9sFlags.ScreenDumpDir  // 无条件赋值指针
}
```

`Override()` 是无条件赋值，但此时 `*k9sFlags.ScreenDumpDir` 的值已经是 pflag 解析后的结果：
- 如果用户**未指定** `--screen-dump-dir`：值为 `""`（空字符串）
- 如果用户**指定了** `--screen-dump-dir /my/path`：值为 `"/my/path"`

### 4.6 `AppScreenDumpDir()` 的回退逻辑

位置：[k9s.go#L158-L170](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L158-L170)

```go
func (k *K9s) AppScreenDumpDir() string {
    d := k.ScreenDumpDir                    // 来源：配置文件或默认值
    if isStringSet(k.manualScreenDumpDir) { // 来源：命令行（仅当用户实际指定时）
        d = *k.manualScreenDumpDir
        k.ScreenDumpDir = d                 // 回写到 ScreenDumpDir 确保一致性
    }
    if d == "" {                            // 兜底：系统默认
        d = AppDumpsDir
    }

    return d
}
```

`isStringSet` 定义（[helpers.go#L25-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/helpers.go#L25-L27)）：

```go
func isStringSet(s *string) bool {
    return s != nil && *s != ""  // 指针非空 且 指向的值非空
}
```

**关键判断逻辑**：

当用户**未指定** `--screen-dump-dir` 时：
- `*k.manualScreenDumpDir = ""`（pflag 默认值）
- `isStringSet("")` → `s != nil` 为 `true`，`*s != ""` 为 `false` → 返回 `false`
- 不覆盖，使用配置文件值（如果有）

当用户**指定了** `--screen-dump-dir /my/path` 时：
- `*k.manualScreenDumpDir = "/my/path"`（用户输入）
- `isStringSet("/my/path")` → `true` → 返回 `true`
- 覆盖为命令行值

**完整决策流程图**：

```
AppScreenDumpDir()
    ↓
d = k.ScreenDumpDir          ← 配置文件值（或 NewK9s 的默认值 AppDumpsDir）
    ↓
isStringSet(k.manualScreenDumpDir)?
    ├─ YES（用户指定了命令行）→ d = *k.manualScreenDumpDir  ← 命令行值覆盖
    │                           k.ScreenDumpDir = d          ← 回写确保一致性
    └─ NO（用户未指定命令行） → d 不变（使用配置文件值）
    ↓
d == "" ?
    ├─ YES → d = AppDumpsDir                ← 系统默认兜底
    └─ NO  → d 不变
    ↓
return d
```

### 4.7 实际场景下的目录确定

| 场景 | 配置文件 | 命令行 | `k.ScreenDumpDir` | `*k.manualScreenDumpDir` | `isStringSet` | 最终目录 | 说明 |
|-----|---------|--------|--------------------|-------------------------|--------------|---------|------|
| 全部默认 | 未设置 | 未指定 | `AppDumpsDir` | `""` | `false` | `AppDumpsDir` | 配置文件值就是 AppDumpsDir |
| 仅配置文件 | `/my/dumps` | 未指定 | `/my/dumps` | `""` | `false` | `/my/dumps` | ✅ 配置文件生效 |
| 仅命令行 | 未设置 | `/my/dumps` | `AppDumpsDir` | `/my/dumps` | `true` | `/my/dumps` | 命令行覆盖默认值 |
| 两者都设置 | `/cfg/dumps` | `/cli/dumps` | `/cfg/dumps` | `/cli/dumps` | `true` | `/cli/dumps` | 命令行优先级更高 |
| 命令行空字符串 | `/my/dumps` | `--screen-dump-dir ""` | `/my/dumps` | `""` | `false` | `/my/dumps` | 空字符串视为未设置，配置文件生效 |
| 配置文件空字符串 | `""` | 未指定 | `""` | `""` | `false` | `AppDumpsDir` | 兜底到系统默认 |
| 全空 | `""` | `--screen-dump-dir ""` | `""` | `""` | `false` | `AppDumpsDir` | 兜底到系统默认 |

**正确的优先级关系**：

1. **命令行 `--screen-dump-dir`（非空值）** - 最高优先级，仅当用户实际传入非空值时生效
2. **配置文件 `k9s.screenDumpDir`** - 中优先级，用户未指定命令行时生效
3. **系统默认 `AppDumpsDir`** - 最低优先级，兜底使用

**与其他标志的行为差异**：

值得注意的是 `--log-file` 标志的行为不同。在 [flags.go#L39](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/flags.go#L39) 中：
- `LogFile: strPtr(AppLogFile)` - NewFlags 默认值
- 但在 `initK9sFlags()` 中，`LogFile` **没有**注册 `StringVar`（看 [root.go#L267-L272](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L267-L272)）
- 所以 `LogFile` 的值不会被 pflag 覆盖，始终是 `AppLogFile`
- 在 `run()` 中直接使用 `*k9sFlags.LogFile`（[root.go#L81](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L81)）

而 `--screen-dump-dir` 注册了 `StringVar`，所以会被 pflag 管理和覆盖。这是两种不同的设计模式。

### 4.8 `ContextScreenDumpDir()` - 上下文子目录

位置：[k9s.go#L172-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L172-L175)

```go
func (k *K9s) ContextScreenDumpDir() string {
    return filepath.Join(k.AppScreenDumpDir(), k.contextPath())
}
```

`contextPath()` 计算逻辑（[k9s.go#L177-L186](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L177-L186)）：

```go
func (k *K9s) contextPath() string {
    if k.getActiveConfig() == nil {
        return "na"
    }
    return data.SanitizeContextSubpath(
        k.getActiveConfig().Context.GetClusterName(),
        k.ActiveContextName(),
    )
}
```

最终目录结构：

```
{AppScreenDumpDir()}/{cluster-name}/{context-name}/
```

- 两级子目录都经过 `SanitizeFileName` 清洗
- 如果无活跃配置，使用 `"na"` 作为子目录名

### 4.9 目录生命周期

| 阶段 | 时机 | 操作 | 代码位置 |
|-----|------|------|---------|
| 系统默认初始化 | `InitLocs()` 应用启动 | 创建 `AppDumpsDir` 基础目录 | [files.go#L115-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L115-L122) |
| 配置确定后 | `Refine()` 中 | 确保 `AppScreenDumpDir()` 目录存在 | [config.go#L146](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/config.go#L146) |
| 视图切换 | ScreenDump 视图初始化 | 确保上下文子目录存在 | [screen_dump.go#L38-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/screen_dump.go#L38-L46) |
| 保存时 | 每次保存快照 | `ensureDir` 确保目录存在 | [table_helper.go#L26-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L26-L28) |
| 上下文切换 | 切换 k8s 上下文 | `contextPath()` 变化，目录路径随之变化 | [k9s.go#L172-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L172-L175) |

---

## 五、读取/查看快照

### 5.1 访问方式

**命令别名**：`screendump` 或 `sd`

在 k9s 命令栏输入 `:sd` 或 `:screendump` 即可进入快照列表视图。

- 别名定义：[alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/alias.go#L195)
- GVR 定义：[gvrs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/gvrs.go#L63)

### 5.2 架构层次

Screen Dump 功能采用经典的三层架构：

```
View (视图层)     screen_dump.go      展示、交互
    │
Model (模型层)    registry.go         注册 DAO 和 Renderer
    │
DAO (数据层)      screen_dump.go      文件系统操作
    │
Render (渲染层)   screen_dump.go      行数据渲染
```

#### 5.2.1 视图层 - ScreenDump

位置：[screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/screen_dump.go)

- 基于 `ResourceViewer`（Browser）实现
- 按时间倒序排序（最新的在前）
- 按 Enter 调用编辑器打开文件
- 支持删除操作

关键方法：
- `dirContext`：设置目录上下文，将目录路径注入 context
- `edit`：选中文件后用编辑器打开

#### 5.2.2 DAO 层 - ScreenDump

位置：[dao/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/dao/screen_dump.go)

实现两个接口：
- `Accessor`：通用访问接口
- `Nuker`：删除接口

核心方法：
- `List`：读取目录下所有文件，封装为 `FileRes` 对象列表
- `Delete`：删除指定路径的文件

```go
func (*ScreenDump) List(ctx context.Context, _ string) ([]runtime.Object, error) {
    dir, ok := ctx.Value(internal.KeyDir).(string)
    // 读取目录、封装为 FileRes 列表
}
```

#### 5.2.3 渲染层 - ScreenDump

位置：[render/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/render/screen_dump.go)

渲染列：
| 列名 | 说明 |
|-----|------|
| NAME | 文件名 |
| DIR | 目录路径 |
| VALID | 预留列 |
| AGE | 文件修改时间距今 |

数据结构 `FileRes`：
- 包含 `os.FileInfo` 和目录路径
- 实现了 `runtime.Object` 接口（`GetObjectKind`、`DeepCopyObject`）

### 5.3 查看流程

1. **进入视图**：输入 `:sd` 命令
2. **注册查找**：通过 `SdGVR` 在 registrar 中找到 `NewScreenDump`
   - 位置：[registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/registrar.go#L78-L80)
3. **上下文设置**：调用 `dirContext` 设置目录路径
4. **数据加载**：DAO 的 `List` 方法读取目录文件
5. **渲染展示**：Renderer 将文件信息渲染为表格行
6. **文件打开**：选中文件按 Enter，调用系统编辑器打开
7. **文件删除**：选中文件按 `Ctrl+D` 删除

---

## 六、完整调用链路

### 6.1 保存链路（以 Table 为例）

```
用户按 Ctrl+S
    ↓
table.go: saveCmd()
    ↓
table_helper.go: saveTable(dir, title, path, mdata)
    ├─ title = GVR().R()  (如 "pods", "nodes")
    ├─ path = t.Path       (来自 KeyPath，空或资源 FQN)
    ├─ ns = mdata.GetNamespace()
    └─ if IsClusterWide(ns) { ns = "all" }   ← 关键转换
    ↓
table_helper.go: computeFilename(dumpPath, ns, title, path)
    ├─ 步骤A: 构建 name
    │   ├─ path == "" → name = title
    │   └─ path != "" → name = title + "-" + SanitizeFileName(path)
    ├─ 步骤B: ns 只能是具体命名空间或 "all"（永远不会是 "-"）
    └─ 步骤C: 始终使用 FullFmat: name + "-" + ns + "-" + timestamp + ".csv"
    ↓
strings.ToLower(完整路径) → 最终文件路径全部小写
    ↓
创建 CSV 文件（权限 0600），写入数据
    ↓
Flash 提示保存成功
```

**路径为空 vs 非空的关键差异**：
- path 为空（顶层列表保存）：`{title}-{ns}-{ts}.csv`（无多余横杠）
- path 非空（选中资源后保存）：`{title}-{sanitized_path}-{ns}-{ts}.csv`

### 6.2 读取链路

```
用户输入 :sd
    ↓
registrar.go: 查找 SdGVR -> NewScreenDump
    ↓
screen_dump.go: NewScreenDump() 创建视图
    ↓
screen_dump.go: dirContext() 设置目录上下文
    ↓
dao/screen_dump.go: List() 读取目录文件
    ↓
render/screen_dump.go: Render() 渲染文件列表
    ↓
展示文件列表表格
```

---

## 七、关键代码文件索引

| 文件 | 作用 |
|-----|------|
| [internal/view/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/screen_dump.go) | Screen Dump 视图实现 |
| [internal/view/table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go) | 表格保存和文件名计算核心逻辑 |
| [internal/view/yaml.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/yaml.go) | YAML 保存函数 |
| [internal/view/log.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/log.go) | 日志保存函数 |
| [internal/dao/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/dao/screen_dump.go) | Screen Dump DAO 层 |
| [internal/render/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/render/screen_dump.go) | Screen Dump 渲染器 |
| [cmd/root.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L133-L178) | 启动流程和配置加载 |
| [cmd/root.go#L267-L272](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/cmd/root.go#L267-L272) | --screen-dump-dir 标志注册 |
| [internal/config/k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go) | 目录配置、覆盖和回退逻辑 |
| [internal/config/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/config.go) | 配置加载和 Refine |
| [internal/config/files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go) | 系统默认目录初始化 |
| [internal/config/flags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/flags.go) | 命令行标志定义和默认值 |
| [internal/config/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/helpers.go) | isStringSet、isEnvSet 辅助函数 |
| [internal/config/data/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/data/helpers.go) | 文件名清洗 |
| [internal/client/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/types.go#L16-L55) | 命名空间相关常量定义 |
| [internal/client/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/helpers.go#L20-L23) | IsClusterWide 函数 |
| [internal/client/gvrs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/gvrs.go) | SdGVR 定义 |
| [internal/view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/registrar.go) | 视图注册 |
| [internal/config/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/alias.go) | 命令别名 |
| [internal/ui/table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/ui/table_helper.go#L35-L39) | 文件名格式常量定义 |
