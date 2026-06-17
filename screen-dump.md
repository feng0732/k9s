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

## 四、目录结构与生命周期

### 4.1 目录层级

```
screen-dumps/                    # 基础目录 (AppDumpsDir)
└── {cluster-name}/              # 集群名（已清洗）
    └── {context-name}/          # 上下文名（已清洗）
        ├── pods-xxx.csv         # 快照文件
        ├── nodes-xxx.csv
        └── xxx.log
```

### 4.2 基础目录 - `AppDumpsDir`

位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L60-L61)

基础目录的确定有两种方式：

1. **K9S_CONFIG_DIR 环境变量方式**：
   - `AppDumpsDir = $K9S_CONFIG_DIR/screen-dumps`
   - 位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L124-L133)

2. **XDG 标准方式**：
   - `AppDumpsDir = $XDG_STATE_HOME/k9s/screen-dumps`
   - 位置：[files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L166-L193)

### 4.3 上下文目录 - `ContextScreenDumpDir`

位置：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L172-L175)

```go
func (k *K9s) ContextScreenDumpDir() string {
    return filepath.Join(k.AppScreenDumpDir(), k.contextPath())
}
```

上下文路径由 `contextPath()` 计算：

位置：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L177-L186)

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

即：`{cluster}/{context}`，两级目录都经过文件名清洗。

### 4.4 目录生命周期

| 阶段 | 时机 | 操作 | 代码位置 |
|-----|------|------|---------|
| 初始化 | 应用启动时 | 创建 `AppDumpsDir` 基础目录 | [files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go#L130-L133) |
| 视图切换 | ScreenDump 视图初始化 | 确保上下文目录存在 | [screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/screen_dump.go#L38-L46) |
| 保存时 | 每次保存快照 | 确保目录存在（`ensureDir`） | [table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L26-L28) |
| 上下文切换 | 切换 k8s 上下文 | 目录路径随之变化 | [k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L173-L175) |

### 4.5 自定义目录

可通过以下方式自定义 screen dump 目录：

1. **配置文件**：`k9s.screenDumpDir`
   - 位置：[k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go#L39)

2. **命令行标志**：`--screen-dump-dir`
   - 位置：[flags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/flags.go)
   - 优先级高于配置文件

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
| [internal/config/k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go) | 目录配置相关 |
| [internal/config/files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go) | 目录初始化 |
| [internal/config/data/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/data/helpers.go) | 文件名清洗 |
| [internal/client/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/types.go#L16-L55) | 命名空间相关常量定义 |
| [internal/client/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/helpers.go#L20-L23) | IsClusterWide 函数 |
| [internal/client/gvrs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/gvrs.go) | SdGVR 定义 |
| [internal/view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/registrar.go) | 视图注册 |
| [internal/config/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/alias.go) | 命令别名 |
| [internal/ui/table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/ui/table_helper.go#L35-L39) | 文件名格式常量定义 |
