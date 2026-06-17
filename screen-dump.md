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

#### 2.2.1 表格保存 (CSV) - `saveTable`

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L45-L84)

```go
func saveTable(dir, title, path string, mdata *model1.TableData) (string, error)
```

- 输入：目录路径、资源标题、资源路径、表格数据
- 输出：保存的文件路径
- 流程：
  1. 通过 `computeFilename` 计算文件名
  2. 以 `0600` 权限创建文件
  3. 使用 `csv.Writer` 写入列名和所有行数据
  4. 刷新缓冲区并返回文件路径

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

### 3.1 命名计算函数 - `computeFilename`

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go#L22-L43)

核心逻辑：

```go
name := title + "-" + data.SanitizeFileName(path)
if path == "" {
    name = title
}

// 集群范围资源（无命名空间）
if ns == client.ClusterScope {
    fName = fmt.Sprintf(ui.NoNSFmat, name, now)  // "%s-%d.csv"
} else {
    fName = fmt.Sprintf(ui.FullFmat, name, ns, now)  // "%s-%s-%d.csv"
}
```

### 3.2 文件名格式常量

位置：[table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/ui/table_helper.go#L35-L39)

| 常量名 | 格式 | 适用场景 |
|--------|------|---------|
| `FullFmat` | `%s-%s-%d.csv` | 有命名空间的资源 |
| `NoNSFmat` | `%s-%d.csv` | 集群范围资源 |

### 3.3 时间戳

使用 `time.Now().UnixNano()` 纳秒级时间戳，确保文件名唯一性。

### 3.4 文件名清洗 - `SanitizeFileName`

位置：[helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/data/helpers.go#L27-L29)

```go
var invalidPathCharsRX = regexp.MustCompile(`[:/]+`)

func SanitizeFileName(name string) string {
    return invalidPathCharsRX.ReplaceAllString(name, "-")
}
```

将 `:` 和 `/` 替换为 `-`，确保文件名合法。

### 3.5 不同类型的文件命名示例

| 视图类型 | 命名模式 | 示例 |
|---------|---------|------|
| 表格 (有命名空间) | `{title}-{path}-{ns}-{timestamp}.csv` | `pods-nginx-ns1-default-1620000000000000000.csv` |
| 表格 (集群范围) | `{title}-{timestamp}.csv` | `nodes-1620000000000000000.csv` |
| 日志 | `{fqn}-{timestamp}.log` | `default-nginx-7f-1620000000000000000.log` |
| YAML 详情 | `{name}--{timestamp}.yaml` | `pod-nginx--1620000000.yaml` |

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
    ↓
table_helper.go: computeFilename(dumpPath, ns, title, path)
    ↓
创建 CSV 文件，写入数据
    ↓
Flash 提示保存成功
```

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
| [internal/view/table_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/table_helper.go) | 表格保存和文件名计算 |
| [internal/view/yaml.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/yaml.go) | YAML 保存函数 |
| [internal/dao/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/dao/screen_dump.go) | Screen Dump DAO 层 |
| [internal/render/screen_dump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/render/screen_dump.go) | Screen Dump 渲染器 |
| [internal/config/k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/k9s.go) | 目录配置相关 |
| [internal/config/files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/files.go) | 目录初始化 |
| [internal/config/data/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/data/helpers.go) | 文件名清洗 |
| [internal/client/gvrs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/client/gvrs.go) | SdGVR 定义 |
| [internal/view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/view/registrar.go) | 视图注册 |
| [internal/config/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-k9s/internal/config/alias.go) | 命令别名 |
