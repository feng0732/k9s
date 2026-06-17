# K9s Skin 颜色系统分析

## 一、Skin 加载机制

### 1.1 加载入口

Skin 的加载主要由 [Configurator](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L31-L38) 结构体管理，核心方法包括：

- `activeSkin()` - 确定当前使用的 skin 名称
- `loadSkinFile()` - 加载 skin 文件
- `RefreshStyles()` - 刷新样式
- `SkinsDirWatcher()` - 监听 skin 目录变化（热重载）

### 1.2 Skin 文件查找路径

Skin 文件位于 `AppSkinsDir` 目录下，通过 [SkinFileFromName()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/files.go#L297-L303) 函数生成路径：

```go
func SkinFileFromName(n string) string {
    if n == "" {
        n = "stock"
    }
    return filepath.Join(AppSkinsDir, n+".yaml")
}
```

`AppSkinsDir` 的位置由以下规则确定（参见 [files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/files.go)）：

- 如果设置了 `K9S_CONFIG_DIR` 环境变量，则为 `$K9S_CONFIG_DIR/skins`
- 否则使用 XDG 标准配置目录下的 `k9s/skins`

### 1.3 加载流程

Skin 加载的完整流程在 [loadSkinFile()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L324-L354) 中实现：

1. 首先调用 `Styles.Reset(invert)` 重置为 stock 默认样式
2. 然后调用 `Styles.Load(skinFile, invert)` 加载自定义 skin 文件
3. 如果加载失败（文件不存在或解析错误），回退到 stock skin
4. 最后调用 `updateStyles()` 应用样式

### 1.4 热重载

当 `UI.Reactive` 为 true 时，通过 fsnotify 监听 skin 目录变化，皮肤文件修改后会自动重新加载（参见 [SkinsDirWatcher()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L154-L188)）。

---

## 二、样式数据结构

### 2.1 整体层级结构

样式定义在 [styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go) 中，层级结构如下：

```
Styles
  └── K9s (Style)
        ├── Body          - 主体样式
        ├── Prompt        - 命令提示样式
        ├── Help          - 帮助面板样式
        ├── Frame         - 框架样式
        │     ├── Title     - 标题
        │     ├── Border    - 边框
        │     ├── Menu      - 菜单
        │     ├── Crumb     - 面包屑导航
        │     └── Status    - 资源状态颜色
        ├── Info          - 集群信息面板样式
        ├── Views         - 各视图样式
        │     ├── Table     - 表格视图
        │     ├── Xray      - Xray 视图
        │     ├── Charts    - 图表视图
        │     ├── Yaml      - YAML 视图
        │     ├── Picker    - 选择器
        │     └── Log       - 日志视图
        └── Dialog        - 对话框样式
```

### 2.2 各模块颜色字段

#### Body（主体）
| 字段 | 说明 |
|------|------|
| `fgColor` | 前景色 |
| `bgColor` | 背景色 |
| `logoColor` | Logo 颜色 |
| `logoColorMsg` | Logo 消息色 |
| `logoColorInfo` | Logo 信息色 |
| `logoColorWarn` | Logo 警告色 |
| `logoColorError` | Logo 错误色 |

#### Frame.Status（资源状态）
| 字段 | 说明 | 对应 model1 变量 |
|------|------|-----------------|
| `newColor` | 新增资源颜色 | `StdColor` |
| `modifyColor` | 修改资源颜色 | `ModColor` |
| `addColor` | 添加资源颜色 | `AddColor` |
| `pendingColor` | 待处理资源颜色 | `PendingColor` |
| `errorColor` | 错误资源颜色 | `ErrColor` |
| `highlightColor` | 高亮颜色 | `HighlightColor` |
| `killColor` | 删除资源颜色 | `KillColor` |
| `completedColor` | 已完成颜色 | `CompletedColor` |

#### Views.Table（表格）
| 字段 | 说明 |
|------|------|
| `fgColor` | 表格前景色 |
| `bgColor` | 表格背景色 |
| `cursorFgColor` | 光标行前景色 |
| `cursorBgColor` | 光标行背景色 |
| `markColor` | 标记行颜色 |
| `header.fgColor` | 表头前景色 |
| `header.bgColor` | 表头背景色 |
| `header.sorterColor` | 排序列颜色 |
| `header.selectedSortColumnColor` | 当前排序列颜色 |

---

## 三、颜色类型与转换

### 3.1 Color 类型

颜色定义在 [color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/color.go) 中，`Color` 本质是 `string` 类型：

```go
type Color string
```

支持的颜色格式：

1. **命名颜色**：如 `"cadetblue"`、`"black"`、`"orange"` 等（由 tcell 库解析）
2. **十六进制颜色**：如 `"#ff0000"`、`"#282a36"`（7 位格式，带 # 前缀）
3. **特殊值**：
   - `"default"` - 使用终端默认颜色
   - `"-"` - 透明色（使用终端背景色）

### 3.2 转换为终端颜色

通过 `Color()` 方法将配置颜色转换为 `tcell.Color`：

```go
func (c Color) Color() tcell.Color {
    if c == DefaultColor {
        return tcell.ColorDefault
    }
    return tcell.GetColor(string(c)).TrueColor()
}
```

转换流程：
1. 如果是 `default`，直接返回 `tcell.ColorDefault`
2. 否则调用 `tcell.GetColor()` 解析颜色名称或 hex 值
3. 调用 `.TrueColor()` 确保转换为真彩色格式

### 3.3 颜色反转（Invert）

支持通过 `Invert` 选项反转所有颜色，使用 Oklch 色彩空间的亮度反转算法，同时尽量保持饱和度。

反转逻辑在 [InvertColor()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/color.go#L145-L188) 中实现：

- 对于无色相的灰度颜色，直接反转亮度 `L = 1.0 - L`
- 对于彩色颜色，在保持一定饱和度的前提下反转亮度
- 特殊颜色（default、transparent、空字符串）保持不变
- 由 `chromaPreserveFactor = 0.5` 控制饱和度保留比例

每个样式结构体都有 `Invert()` 方法递归反转所有颜色字段。

### 3.4 反转算法的幂等性（双反转还原验证）

根据单元测试 [TestInvertGrayRoundTrip](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/color_test.go#L257-L280) 和 [TestInvertColorOutOfGamut](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/color_test.go#L300-L357) 的结果：

| 颜色类型 | 双反转效果 | 原因 |
|---------|-----------|------|
| **灰度颜色**（C < 0.01） | ✅ **完全还原**，`Invert(Invert(c)) == c` | 灰度色没有色相，直接 `L = 1.0 - (1.0 - L) = L`，数学恒等 |
| **彩色颜色** | ⚠️ **基本还原，可能有微小偏差** | 由于色域限制（sRGB gamut），在极端亮度值处需调整 L 以保留饱和度，Clamped() 后可能产生 1~2 位十六进制的量化误差 |
| **特殊颜色**（default、"-"、空串） | ✅ 保持不变 | 代码开头直接 `return c` 短路 |

这意味着：**连续调用两次 `Invert()` 对于绝大多数颜色基本等价于恒等变换**。这个特性是理解后面"部分填充 + invert"场景的关键。

---

## 四、Skin 到终端样式的映射

### 4.1 全局 tview 样式映射

`Styles.Update()` 方法（参见 [styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go#L793-L809)）将 skin 样式映射到 tview 全局样式：

| tview.Styles 字段 | 映射来源 |
|-------------------|----------|
| `PrimitiveBackgroundColor` | `Body.BgColor` |
| `ContrastBackgroundColor` | `Body.BgColor` |
| `MoreContrastBackgroundColor` | `Body.BgColor` |
| `PrimaryTextColor` | `Body.FgColor` |
| `BorderColor` | `Frame.Border.FgColor` |
| `FocusColor` | `Frame.Border.FocusColor` |
| `TitleColor` | `Body.FgColor` |
| `GraphicsColor` | `Body.FgColor` |
| `SecondaryTextColor` | `Body.FgColor` |
| `TertiaryTextColor` | `Body.FgColor` |
| `InverseTextColor` | `Body.FgColor` |
| `ContrastSecondaryTextColor` | `Body.FgColor` |

### 4.2 表格行状态颜色映射

在 [updateStyles()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L356-L371) 中，将 Frame.Status 中的颜色映射到 `model1` 包的全局变量：

| model1 变量 | 映射来源 |
|-------------|----------|
| `ModColor` | `Frame.Status.ModifyColor` |
| `AddColor` | `Frame.Status.AddColor` |
| `ErrColor` | `Frame.Status.ErrorColor` |
| `StdColor` | `Frame.Status.NewColor` |
| `PendingColor` | `Frame.Status.PendingColor` |
| `HighlightColor` | `Frame.Status.HighlightColor` |
| `KillColor` | `Frame.Status.KillColor` |
| `CompletedColor` | `Frame.Status.CompletedColor` |

这些全局变量被表格渲染时用于设置不同状态行的文字颜色（参见 [model1/color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/model1/color.go)）。

### 4.3 UI 组件的样式应用

各 UI 组件通过 `Styles` 对象获取具体样式并应用：

- **Table 组件**（[ui/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/table.go)）：
  - 边框颜色使用 `Frame.Border.FgColor` / `FocusColor`
  - 选中行样式使用 `Table.CursorFgColor` / `CursorBgColor`
  - 表头颜色使用 `Table.Header.FgColor` / `BgColor`
  - 标记行颜色使用 `Table.MarkColor`

- **其他组件**：Menu、Logo、Prompt、Crumbs 等组件都在初始化时传入 Styles 对象，并在 `StylesChanged()` 回调中更新自身样式。

### 4.4 样式变更通知

`Styles` 结构体实现了观察者模式（参见 [styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go#L17-L21)）：

- `AddListener()` - 注册样式监听器
- `RemoveListener()` - 移除监听器
- `fireStylesChanged()` - 触发样式变更通知

当样式更新时，所有注册的监听器都会收到 `StylesChanged` 通知，从而更新各自的显示。

---

## 五、冲突与优先级规则

### 5.1 Skin 选择优先级

Skin 的选择遵循以下优先级（从高到低），在 [activeSkin()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L248-L282) 中实现：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1（最高） | 环境变量 `K9S_SKIN` | 通过环境变量指定，优先于所有配置 |
| 2 | Context 级配置 | 当前活跃 context 的 `skin` 字段 |
| 3 | 全局 UI 配置 | `k9s.ui.skin` 字段 |
| 4（最低） | Stock 默认 | 内置的 stock skin |

**注意**：每一级都需要验证对应的 skin 文件是否存在，如果不存在则降级到下一级。

### 5.2 样式覆盖规则

Skin 文件加载时采用「重置 + 覆盖」的策略：

1. 首先调用 `Styles.Reset(invert)` 将所有样式重置为 stock 默认值
2. 然后通过 `yaml.Unmarshal` 将自定义 skin 文件内容反序列化到 Styles 对象
3. YAML 反序列化会覆盖已设置的字段，未指定的字段保持默认值

这意味着：
- 自定义 skin 可以只定义需要修改的部分，其余使用默认值
- 不存在「部分合并」或「深层合并」的逻辑，完全是基于结构体字段的覆盖

### 5.3 「部分填充 + Invert」场景的反转次数深度分析

这是整个颜色系统中**最容易混淆、最容易出 bug**的核心逻辑。让我们深入 `loadSkinFile()` 的完整执行流程来拆解。

#### 5.3.1 关键代码路径

在 [ui/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go#L324-L354) 中，核心执行顺序是：

```go
func (c *Configurator) loadSkinFile(synchronizer) {
    invert := c.Config.K9s.IsInvert()
    // ...
    c.Styles.Reset(invert)           // 第335行：第一次 Invert
    if err := c.Styles.Load(skinFile, invert); err != nil {  // 第336行：第二次 Invert
```

**两个函数都接收了同一个 `invert` 参数**，并且各自内部独立判断是否调用 `Invert()`：

- [Styles.Reset(invert)](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go#L481-L489)：
  ```go
  func (s *Styles) Reset(invert bool) {
      yaml.Unmarshal(stockSkinTpl, s)   // 恢复默认颜色
      if invert {
          s.K9s.Invert()                 // ← 反转所有颜色（第1次）
      }
  }
  ```

- [Styles.Load(path, invert)](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go#L773-L791)：
  ```go
  func (s *Styles) Load(path string, invert bool) error {
      yaml.Unmarshal(bb, s)              // 覆盖用户自定义颜色
      if invert {
          s.K9s.Invert()                 // ← 再次反转所有颜色（第2次）
      }
      return nil
  }
  ```

#### 5.3.2 分步状态追踪（以 `invert=true` 为例）

假设自定义 skin 只定义了部分颜色，例如：
```yaml
k9s:
  body:
    fgColor: red      # 只定义前景色
    # bgColor 未定义，使用默认值
  frame:
    status:
      errorColor: "#ff0000"  # 只定义错误色
      # 其他 status 颜色未定义
```

让我们追踪 `bgColor`（默认值，用户未定义）和 `fgColor`（用户自定义 `red`）的完整生命周期：

| 阶段 | 操作 | `bgColor`（默认，用户未定义） | `fgColor`（用户自定义 red） |
|------|------|-----------------------------|---------------------------|
| 初始状态 | `NewStyles()` 或之前状态 | `black`（默认） | `cadetblue`（默认） |
| ↓ | | | |
| **第1步** | `Reset(true)` → Unmarshal(stock) | 恢复默认 `black` | 恢复默认 `cadetblue` |
| **第2步** | `Reset(true)` → `Invert()` 第1次 | **反转**：`black` → `white` | **反转**：`cadetblue` → 反色 |
| ↓ | | | |
| **第3步** | `Load()` → Unmarshal(自定义 skin) | 保持 `white`（用户未覆盖） | 被覆盖为 `red`（用户定义） |
| **第4步** | `Load()` → `Invert()` 第2次 | **再次反转**：`white` → `black` | **反转**：`red` → 反色（青色调） |
| ↓ | | | |
| **最终结果** | | **`black`**（等于默认值） | **反色后的 red** |

**结论**：
- ✅ **用户自定义的颜色**：反转 **1 次**（只有 Load 时的第 2 次反转）
- ❌ **未定义的默认颜色**：反转 **2 次**（Reset 第 1 次 + Load 第 2 次）≈ **等于原默认值**

#### 5.3.3 双反转的实际效果

根据 3.4 节的分析，双反转对于不同颜色类型的效果不同：

| 颜色类型 | 双反转效果 | 未定义默认色的最终结果 |
|---------|-----------|----------------------|
| 灰度颜色（如 `black`、`white`、`gray`） | 精确还原 | 等于默认值，完全没被反转 |
| 彩色颜色（如 `cadetblue`、`orange`） | 基本还原，可能有微小偏差 | 约等于默认值，基本没被反转 |
| 特殊颜色（`default`、`"-"`） | 保持不变 | 等于默认值 |

**实际表现**：当开启 invert 但只自定义部分颜色时，**只有用户自定义的颜色会被反转，未自定义的默认颜色几乎保持原色**。

#### 5.3.4 设计意图与潜在问题

这个行为可能是**有意的设计**，也可能是**无意中的 bug**：

**支持"有意设计"的理由**：
- 用户提供的自定义 skin 可能是"亮色主题"的颜色定义
- 开启 invert 后，应该把这些亮色主题颜色反转为暗色主题
- 而未定义的部分使用默认暗色主题颜色，不需要反转
- 这样可以实现"部分自定义 + 自动适配明暗"

**支持"bug"的理由**：
- 用户直觉上会认为"开启 invert 就是反转所有颜色"
- 未定义的默认色没有被反转，会导致明暗不一致
- 例如：默认 bgColor=black（暗色），反转后应该是 white（亮色），但实际仍是 black
- 这会导致部分自定义的皮肤在 invert 模式下出现"混搭"效果

#### 5.3.5 各错误路径的反转次数汇总

`loadSkinFile()` 有多个分支，不同分支的反转次数也不同：

| 场景 | 执行路径 | 反转次数（默认色/自定义色） | 最终效果 |
|------|---------|---------------------------|---------|
| **无自定义 skin** | `!ok` → `updateStyles("", invert)` | Reset 1 次 / 无 | 所有颜色反转 1 次 |
| **skin 文件不存在** | Load 失败 → `updateStyles("", invert)` | Reset 1 次 + updateStyles 中的 Reset 1 次 = 2 次 / 无 | 所有颜色还原（≈没反转） |
| **skin 解析错误** | Load 失败 → `updateStyles(skinFile, invert)` | Reset 1 次 / 无 | 所有颜色反转 1 次（Reset 后没有再反转） |
| **正常加载成功** | Load 成功 → `updateStyles(skinFile, invert)` | Reset 1 次 + Load 1 次 = 2 次 / Load 1 次 | 默认色不反转，自定义色反转 1 次 |

**注意**：`updateStyles(f, invert)` 只有当 `f == ""` 时才会再次调用 `Reset(invert)`。

### 5.4 Invert 选项优先级

颜色反转（Invert）也有类似的优先级：

| 优先级 | 来源 |
|--------|------|
| 1（最高） | CLI 参数 `--invert` |
| 2 | 全局 UI 配置 `ui.invert` |
| 3（最低） | 默认 false |

参见 [k9s.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/k9s.go#L387-L394) 中的 `IsInvert()` 方法。

> ⚠️ **注意**：Invert 并非在"样式加载完成后统一应用一次"，而是在 `Reset()` 和 `Load()` 两个函数中**各自独立地被调用一次**。这个细节是理解部分填充皮肤行为的关键，详见 5.3 节。

### 5.5 JSON Schema 验证

加载 skin 文件时会先通过 JSON Schema 验证格式合法性（参见 [styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go#L779-L781)）：

```go
if err := data.JSONValidator.Validate(json.SkinSchema, bb); err != nil {
    return err
}
```

Schema 定义在 [skin.json](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/json/schemas/skin.json)，验证失败会导致加载失败并回退到默认样式。

### 5.6 上下文切换时的样式重载

当切换 context 时（参见 [view/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/view/app.go)），会调用 `ReloadStyles()` 重新加载 skin，因为不同 context 可能配置了不同的 skin。

---

## 六、最终终端配色对应表

### 6.1 完整加载流程总览

下面是从 skin 配置到终端显示的完整数据流：

```
Skin 配置文件（YAML）
       │
       ▼  选择优先级：环境变量 > Context > 全局 > stock
  activeSkin()  [ui/config.go#L248-L282]
       │
       ▼  invert 优先级：CLI > 配置 > false
  IsInvert()  [k9s.go#L387-L394]
       │
       ▼  加载流程（invert=true 时）
  ┌─────────────────────────────────────────┐
  │ 1. Reset(invert=true)                   │
  │   - Unmarshal stock → 默认颜色          │
  │   - Invert() → 全部反转 ①               │
  │                                         │
  │ 2. Load(skinFile, invert=true)          │
  │   - Unmarshal 自定义 → 覆盖部分字段     │
  │   - Invert() → 全部再反转 ②             │
  │                                         │
  │ 3. updateStyles()                       │
  │   - Styles.Update() → 映射到 tview      │
  │   - 映射到 model1 全局颜色变量          │
  └─────────────────────────────────────────┘
       │
       ▼
  tview.Styles.* 全局变量
  model1.ModColor / AddColor / ... 全局变量
       │
       ▼
  UI 组件渲染（Table、Menu、Prompt 等）
```

### 6.2 正常加载成功场景（invert=true）

假设自定义 skin 内容：
```yaml
k9s:
  body:
    fgColor: red      # 用户自定义
    # bgColor 未定义
  frame:
    status:
      errorColor: "#ff0000"  # 用户自定义
      # newColor 未定义
```

| Skin 字段 | 用户是否定义 | 反转次数 | 最终终端颜色 | 说明 |
|----------|------------|---------|-------------|------|
| `body.fgColor` | ✅ 是（`red`） | 1 次（Load 时） | `red` 的反色（青色调） | 用户定义的亮色主题色被反转为暗色 |
| `body.bgColor` | ❌ 否 | 2 次（Reset + Load） | `black`（默认值） | 双反转还原，保持默认暗色 |
| `frame.status.errorColor` | ✅ 是（`#ff0000`） | 1 次（Load 时） | `#ff0000` 的反色（青色调） | 自定义颜色被反转 |
| `frame.status.newColor` | ❌ 否 | 2 次（Reset + Load） | `lightskyblue`（默认值） | 双反转还原，保持默认 |
| `views.table.cursorBgColor` | ❌ 否 | 2 次（Reset + Load） | `aqua`（默认值） | 双反转还原 |
| `views.table.fgColor` | ❌ 否 | 2 次（Reset + Load） | `aqua`（默认值） | 双反转还原 |

**结论**：当 `invert=true` 且自定义 skin 只定义部分颜色时，**只有自定义的颜色被反转，未定义的默认色保持原暗色主题**。

### 6.3 无自定义 skin 场景（invert=true）

当没有自定义 skin 时（`!ok` 分支）：

| Skin 字段 | 反转次数 | 最终终端颜色 | 说明 |
|----------|---------|-------------|------|
| 所有字段 | 1 次（updateStyles → Reset） | 全部默认色被反转 | 所有默认暗色被反转为亮色 |

**结论**：无自定义 skin 时，`invert=true` 会将整个默认暗色主题反转为亮色主题。

### 6.4 invert=false 场景

当 `invert=false` 时，`Reset()` 和 `Load()` 都不会调用 `Invert()`：

| Skin 字段 | 用户是否定义 | 反转次数 | 最终终端颜色 | 说明 |
|----------|------------|---------|-------------|------|
| 用户定义的字段 | ✅ 是 | 0 次 | 用户定义的值 | 直接使用用户颜色 |
| 未定义的字段 | ❌ 否 | 0 次 | stock 默认值 | 直接使用默认颜色 |

**结论**：`invert=false` 时行为符合直觉，没有歧义。

### 6.5 错误场景对比

| 错误场景 | 处理路径 | 反转次数（默认色） | 最终效果 |
|---------|---------|------------------|---------|
| **无自定义 skin** | `updateStyles("", invert)` | 1 次（Reset） | 全部颜色被反转 |
| **skin 文件不存在** | `updateStyles("", invert)` | 2 次（Reset + updateStyles 中的 Reset） | 全部颜色还原（≈没反转） |
| **skin 解析错误** | `updateStyles(skinFile, invert)` | 1 次（Reset） | 全部颜色被反转（Reset 后没有再反转） |
| **正常加载成功** | `updateStyles(skinFile, invert)` | 2 次（Reset + Load） | 默认色不反转，自定义色反转 1 次 |

**注意**：`skin 文件不存在` 和 `无自定义 skin` 虽然都调用了 `updateStyles("", invert)`，但前者在调用之前已经执行过一次 `Reset(invert)`，所以总共反转 2 次。

### 6.6 实际使用建议

根据上述分析，给出以下使用建议：

1. **完整自定义 skin + invert=true**：
   - 预期效果：用户提供的亮色主题被整体反转为暗色主题
   - 建议：skin 文件中明确定义所有需要的颜色字段，避免依赖默认值

2. **部分自定义 skin + invert=true**：
   - 预期效果：自定义部分反转，默认部分不反转
   - 注意：这可能导致明暗不一致，使用前请确认效果

3. **完整自定义 skin + invert=false**：
   - 预期效果：直接使用用户定义的颜色
   - 建议：最直观的使用方式，推荐

4. **无自定义 skin + invert=true**：
   - 预期效果：默认暗色主题反转为亮色主题
   - 建议：通过 CLI 参数 `--invert` 快速切换亮色模式

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| [internal/config/styles.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/styles.go) | 样式结构体定义、默认值、加载、更新 |
| [internal/config/color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/color.go) | Color 类型、颜色转换、反转算法 |
| [internal/config/files.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/files.go) | 配置路径、skin 文件路径生成 |
| [internal/config/data/context.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/data/context.go) | Context 级 skin 配置 |
| [internal/config/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/types.go) | UI 配置结构体（含 Skin 字段） |
| [internal/ui/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/ui/config.go) | Configurator、skin 选择与加载逻辑、热重载 |
| [internal/model1/color.go](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/model1/color.go) | 表格行状态颜色全局变量 |
| [internal/config/templates/stock-skin.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/9-k9s/internal/config/templates/stock-skin.yaml) | Stock 默认皮肤模板 |
