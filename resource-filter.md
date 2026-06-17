# K9s 资源筛选机制深度分析

## 概述

K9s 的资源筛选系统采用**两层筛选架构**，包含服务器端筛选和客户端筛选。这两种筛选共享同一个输入框，但工作机制完全不同，容易造成混淆。

---

## 一、核心架构：两层筛选模型

```
用户输入 (cmdBuff)
      │
      ├─► 判断筛选类型 ──┐
      │                  │
      ▼                  ▼
┌─────────────┐   ┌─────────────┐
│ 标签选择器  │   │ 正则匹配   │
│ (服务器端)  │   │ (客户端)    │
└──────┬──────┘   └──────┬──────┘
       │                  │
       ▼                  ▼
  K8s API 过滤      TableData 过滤
       │                  │
       └──────────┬───────┘
                  ▼
            最终展示数据
```

### 1.1 服务器端筛选：标签选择器 (Label Selector)

**位置**：`internal/dao/table.go:59-84`、`internal/dao/resource.go:27-34` 等 DAO 层

**工作原理**：
- 将标签选择器通过 `context` 的 `internal.KeyLabels` 传递给 DAO 层
- DAO 层将其作为 `ListOptions.LabelSelector` 参数发送给 Kubernetes API
- Kubernetes API 服务器直接返回过滤后的结果
- 优势：数据量小，性能好
- 触发时机：需要刷新模型数据（从服务器重新拉取）

**状态存储**：
```go
// internal/model/table.go
type Table struct {
    labelSelector labels.Selector  // 存储标签选择器状态
    ...
}
```

### 1.2 客户端筛选：正则/Fuzzy

**位置**：`internal/model1/table_data.go:143-198`

**工作原理**：
- 在已获取的表格数据上进行内存过滤
- 不触发服务器请求，响应速度快
- 直接作用于 `RowEvents` 数据集

**状态存储**：
```go
// internal/ui/table.go
type Table struct {
    cmdBuff *model.FishBuff  // 存储用户输入的过滤文本
    toast   bool             // 是否只显示问题资源
    ...
}
```

---

## 二、筛选类型判断机制

所有筛选输入都先经过类型判断，决定走哪条筛选路径。

### 2.1 判断入口函数

**位置**：`internal/helpers.go`

| 函数 | 作用 | 返回值 |
|------|------|--------|
| `IsLabelSelector(s)` | 判断是否为标签选择器 | `bool` |
| `IsFuzzySelector(s)` | 判断是否为 Fuzzy 过滤 | `(string, bool)` - 返回提取的关键词 |
| `IsInverseSelector(s)` | 判断是否为反向过滤 | `bool` |

### 2.2 标签选择器的判断逻辑

```go
// internal/helpers.go
func IsLabelSelector(s string) bool {
    if labelRx.MatchString(s) {  // 匹配 ^-l
        return true
    }
    return !strings.Contains(s, " ") && cmd.ToLabels(s) != nil
}
```

**两种形式都会被识别为标签选择器**：

1. **显式标记**：以 `-l` 开头
   - `-l app=nginx,env=prod`
   - `-lapp=nginx`（没有空格也可以）

2. **隐式格式**：不包含空格且能解析为键值对
   - `app=nginx`
   - `app!=nginx`
   - `app=nginx,env=prod`

> **⚠️ 容易混淆的点**：
> 输入 `app=nginx` 会被当作**标签选择器**处理，而不是正则匹配！
> 它会触发服务器端数据刷新，而不是在客户端过滤。

### 2.3 各筛选类型一览

| 类型 | 前缀/格式 | 示例 | 过滤层面 |
|------|-----------|------|----------|
| 标签选择器 | `-l` 或 `key=value` 格式 | `-l app=nginx`、`app=nginx` | 服务器端 |
| Fuzzy 过滤 | `-f` 开头 | `-f nginx` | 客户端 |
| 反向过滤 | `!` 开头 | `!nginx` | 客户端（正则） |
| Toast 过滤 | 切换标志 | （快捷键触发） | 客户端 |
| 正则匹配 | 默认 | `nginx`、`nginx.*` | 客户端 |

---

## 三、筛选状态与表格数据的配合

### 3.1 状态存储分布

筛选状态分散在三个层级：

| 层级 | 状态 | 位置 | 说明 |
|------|------|------|------|
| **Model 层** | `labelSelector` | `internal/model/table.go:48` | 标签选择器，影响服务器数据获取 |
| **UI 层** | `cmdBuff` | `internal/ui/table.go:48` | 命令缓冲区，存储用户输入的原始字符串 |
| **UI 层** | `toast` | `internal/ui/table.go:54` | 是否只显示问题资源 |

### 3.2 完整筛选流程

```
用户输入
    │
    ▼
cmdBuff 接收输入 ──► 触发 FilterInput()
    │
    ▼
filtered() 调用 TableData.Filter()
    │
    ├─► Toast 过滤（如果开启）
    │
    ├─► 判断是否为标签选择器？
    │    ├─ 是：直接返回（服务器端已过滤）
    │    └─ 否：继续客户端过滤
    │
    ├─► 判断是否为 Fuzzy 过滤？
    │    └─ 是：fuzzyFilter()
    │
    └─► 默认：正则匹配（rxFilter）
         ├─ 支持反向匹配（! 开头）
         └─ 大小写不敏感
```

### 3.3 标签选择器的特殊处理流程

标签选择器的处理比正则匹配多了一步"模型刷新"：

```
BufferCompleted 事件
    │
    ▼
判断是否为标签选择器？
    ├─ 是：
    │    ├─ 设置 model.labelSelector
    │    └─ 触发模型 Refresh() → 从服务器重新拉取数据
    │
    └─ 否：
         └─ 重置 labelSelector 为 labels.Everything()
```

**关键代码**：`internal/view/browser.go:216-251`

```go
func (b *Browser) BufferCompleted(text, _ string) {
    if internal.IsLabelSelector(text) {
        if sel, err := ui.ExtractLabelSelector(text); err == nil {
            b.GetModel().SetLabelSelector(sel)
        }
    } else {
        b.GetModel().SetLabelSelector(labels.Everything())
    }
}
```

### 3.4 TableData.Filter 方法详解

**位置**：`internal/model1/table_data.go:143-164`

```go
func (t *TableData) Filter(f FilterOpts) *TableData {
    td := NewTableDataFromTable(t)

    // 1. Toast 过滤（只显示有问题的资源）
    if f.Toast {
        td.rowEvents = t.filterToast()
    }
    
    // 2. 如果是空或标签选择器，直接返回
    //    （标签选择器已在服务器端过滤）
    if f.Filter == "" || internal.IsLabelSelector(f.Filter) {
        return td
    }
    
    // 3. Fuzzy 过滤
    if f, ok := internal.IsFuzzySelector(f.Filter); ok {
        td.rowEvents = t.fuzzyFilter(f)
        return td
    }
    
    // 4. 正则匹配（默认）
    rr, err := t.rxFilter(f.Filter, internal.IsInverseSelector(f.Filter))
    if err == nil {
        td.rowEvents = rr
    }
    
    return td
}
```

**过滤顺序（优先级从高到低）**：
1. Toast 过滤
2. 标签选择器（跳过客户端过滤）
3. Fuzzy 过滤
4. 正则匹配

---

## 四、容易混淆的关键点

### 4.1 标签选择器 vs 正则匹配：互斥关系

**重要：标签选择器和正则匹配不能同时生效，它们是互斥的！**

- 当输入被识别为标签选择器时 → 重置正则匹配，使用服务器端筛选
- 当输入不是标签选择器时 → 重置标签选择器为 `Everything()`，使用客户端筛选

**原因**：它们共享同一个输入源 `cmdBuff`，输入内容只能是其中一种类型。

### 4.2 输入示例与行为对照表

| 输入内容 | 识别为 | 过滤层面 | 行为说明 |
|----------|--------|----------|----------|
| ``（空） | 无 | - | 不过滤 |
| `nginx` | 正则匹配 | 客户端 | 在所有列中进行正则匹配 nginx |
| `nginx pod` | 无（含空格，正则短路） | 客户端（实际不生效） | 含空格，rxFilter 短路，**无过滤效果** |
| `app=nginx` | 标签选择器 | 服务器端 | 从 K8s API 获取带 app=nginx 标签的资源 |
| `-l app=nginx` | 标签选择器 | 服务器端 | 同上，显式标记 |
| `-f nginx` | Fuzzy 匹配 | 客户端 | 按名称（ID 列）进行 Fuzzy 匹配 |
| `-f nginx pod` | Fuzzy 匹配 | 客户端 | **只捕获 `nginx` 进行 Fuzzy，`pod` 被静默丢弃** |
| `!nginx` | 反向正则匹配 | 客户端 | 显示不包含 nginx 的行 |

### 4.3 Fuzzy 匹配 vs 正则匹配的区别

| 特性 | Fuzzy 匹配 (`-f`) | 正则匹配 |
|------|-------------------|---------|
| 匹配范围 | 只匹配名称列 (ID) | 匹配所有可见列 |
| 算法 | Fuzzy 匹配 (fuzzy.Find) | 正则表达式匹配 |
| 性能 | 较快 | 稍慢（多列匹配） |
| 大小写 | 取决于库 | 不敏感 `(?i)` |

**Fuzzy 过滤**：`internal/model1/table_data.go:200-219`
```go
func (t *TableData) fuzzyFilter(q string) *RowEvents {
    // 只对 Row.ID 进行 fuzzy 匹配
    ss := make([]string, 0, t.RowCount()/2)
    t.rowEvents.Range(func(_ int, re RowEvent) bool {
        ss = append(ss, re.Row.ID)
        return true
    })
    mm := fuzzy.Find(q, ss)
    // ...
}
```

### 4.4 Toast 过滤：可与其他筛选叠加

Toast 过滤是**可以**和其他筛选叠加的，因为它在 Filter 方法中最先执行。

例如：开启 Toast + 输入 `nginx` → 先过滤出问题资源，再在其中匹配 nginx。

---

## 五、标题显示逻辑中的筛选状态

标题栏会显示当前的筛选状态，逻辑如下：

**位置**：`internal/ui/table.go:708-723`

```
优先级从高到低：
1. cmdBuff 中的内容是标签选择器 → 显示解析后的 selector
2. model 中存在非空 labelSelector → 显示该 selector
3. cmdBuff 中有文本 → 显示该文本
4. 都没有 → 不显示筛选信息
```

这也解释了为什么会让人困惑：标签选择器有两个可能的来源（cmdBuff 和 model.labelSelector），它们可能不一致。

---

## 六、关键数据结构

### 6.1 FilterOpts

**位置**：`internal/model1/table_data.go:41-45`

```go
type FilterOpts struct {
    Toast  bool   // 是否只显示有问题的资源
    Filter string // 过滤文本
    Invert bool   // 是否反向匹配（实际通过 ! 前缀判断）
}
```

### 6.2 FishBuff / CmdBuff

**位置**：`internal/model/cmd_buff.go`

命令缓冲区，存储用户输入的原始字符串，支持：
- 字符输入/删除
- 输入延迟防抖（100ms）
- 监听者模式（BufferChanged/BufferCompleted/BufferActive）

---

## 七、边界情况深度分析

### 7.1 空格查询处理

空格在 K9s 筛选系统中扮演着极其关键的**分流边界**角色，它同时影响两个层面：类型判断和正则匹配。

#### 7.1.1 空格对标签选择器判断的影响

`IsLabelSelector` 内部的隐式判断路径直接依赖空格作为分隔条件：

```go
// internal/helpers.go
func IsLabelSelector(s string) bool {
    if labelRx.MatchString(s) {  // 以 -l 开头 → 直接判定为标签
        return true
    }
    // 关键：含空格则跳过隐式标签判断，不含空格才尝试解析为 key=value
    return !strings.Contains(s, " ") && cmd.ToLabels(s) != nil
}
```

**空格决定了同一种输入格式走完全不同的路径**：

| 输入 | 含空格？ | `IsLabelSelector` 结果 | 实际过滤类型 |
|------|----------|------------------------|-------------|
| `app=nginx` | 否 | `true` | 标签选择器（服务器端） |
| `app=nginx, env=prod` | **是**（逗号后有空格） | `false` | 正则匹配 → 空格短路 → **无过滤** |
| `app=nginx,env=prod` | 否 | `true` | 标签选择器（服务器端） |
| `-l app=nginx, env=prod` | 是 | `true`（匹配 `-l` 前缀优先，跳过空格检查） | 标签选择器（服务器端，**两个标签完整解析**） |
| `-l  app=nginx` | 是（两空格） | `true`（匹配 `-l` 前缀优先） | 标签选择器（服务器端） |
| `-l environment in (production, staging)` | 是（多个空格） | `true`（匹配 `-l` 前缀优先） | 标签选择器（服务器端，**in 操作符完整解析**） |

> **⚠️ 陷阱**：`app=nginx, env=prod`（逗号后多了一个空格）不会被识别为标签选择器，而会被当作正则表达式处理！但正则路径中含空格的查询也会被短路（见 7.1.2），所以实际效果等于**完全没有过滤**。

#### 7.1.2 空格对正则匹配的影响

`rxFilter` 中，含空格的查询直接短路返回未过滤数据：

```go
// internal/model1/table_data.go
func (t *TableData) rxFilter(q string, inverse bool) (*RowEvents, error) {
    if strings.Contains(q, " ") {
        return t.rowEvents, nil  // 含空格 → 直接返回全部行，不做任何过滤
    }
    // ... 编译正则并匹配
}
```

**这意味着**：如果用户输入 `nginx pod`，不会执行任何过滤，表格显示全部数据——等价于没有筛选。且没有任何 UI 提示告知用户输入无效。

#### 7.1.3 空格在不同场景的完整行为对照

| 输入 | `IsLabelSelector` | `IsFuzzySelector` | `rxFilter` 空格检查 | 最终行为 |
|------|-------------------|-------------------|--------------------|---------|
| `nginx` | `false` | `("", false)` | 通过 | 正则匹配所有可见列 |
| `nginx pod` | `false`（含空格） | `("", false)` | **短路返回全量** | **无过滤，显示全部** |
| `app=nginx` | `true` | - | - | 服务器端标签过滤 |
| `app=nginx, env=prod` | `false`（含空格） | `("", false)` | **短路返回全量** | **无过滤，显示全部** |
| `-f nginx pod` | `false`（含空格，但 fuzzy 优先级更高） | `("nginx", true)`（`fuzzyRx` 捕获第一个词 `nginx`，空格后的 `pod` 不参与匹配） | 不进入 rxFilter 路径 | **Fuzzy 按 nginx 匹配名称列，pod 被丢弃** |
| `-f nginx` | `false` | `("nginx", true)` | 不进入 rxFilter 路径 | Fuzzy 匹配名称列 |

> **核心结论**：空格对不同类型的查询影响完全不同——正则匹配查询因空格短路等于没过滤，Fuzzy 前缀查询虽然含空格但正则只捕获首词仍能进入 fuzzy 路径，只有带 `-l` 前缀的标签选择器能完整利用空格后的所有内容。

---

### 7.2 非法正则表达式处理

#### 7.2.1 TableData 中的处理（表格资源视图）

```go
// internal/model1/table_data.go
func (t *TableData) rxFilter(q string, inverse bool) (*RowEvents, error) {
    // 空格检查已在上方分析
    if inverse {
        q = q[1:]
    }
    rx, err := regexp.Compile(`(?i)(` + q + `)`)
    if err != nil {
        return nil, fmt.Errorf("invalid rx filter %q: %w", q, err)
    }
    // ... 匹配逻辑
}
```

回到 `Filter` 方法的调用处：

```go
// internal/model1/table_data.go
rr, err := t.rxFilter(f.Filter, internal.IsInverseSelector(f.Filter))
if err == nil {
    td.rowEvents = rr
} else {
    slog.Error("RX filter failed", slogs.Error, err)
}
```

**处理流程**：
1. `rxFilter` 返回 `(nil, error)`
2. `Filter` 方法中 `err != nil`，走 `else` 分支
3. 仅记录一条 `slog.Error` 日志
4. `td.rowEvents` 保持为 `NewTableDataFromTable(t)` 复制来的原始全量数据
5. **返回全量未过滤数据，UI 不显示任何错误提示**

**常见非法正则输入示例**：

| 输入 | 编译结果 | 用户看到的效果 |
|------|---------|--------------|
| `nginx` | 合法 | 正常过滤 |
| `nginx.*` | 合法 | 正则过滤 |
| `nginx(` | **非法**（未闭合括号） | **显示全量，无提示** |
| `nginx[` | **非法**（未闭合字符类） | **显示全量，无提示** |
| `*nginx` | **非法**（`*` 无前置） | **显示全量，无提示** |
| `!nginx(` | **非法**（去掉 `!` 后 `nginx(` 非法） | **显示全量，无提示** |

#### 7.2.2 Text 模型中的处理（日志/描述等文本视图）

日志和描述视图使用的是 `model/text.go` 中的 `rxFilter`：

```go
// internal/model/helpers.go
func rxFilter(q string, lines []string) fuzzy.Matches {
    rx, err := regexp.Compile(`(?i)` + q)
    if err != nil {
        return nil  // 非法正则 → 返回空匹配集
    }
    // ... 匹配逻辑
}
```

**与 TableData 的差异**：
- TableData：非法正则 → 返回**全量数据**（因为 `td` 已复制了原始行）
- Text：非法正则 → 返回**空匹配集**（`nil`），意味着**没有行被高亮**

#### 7.2.3 非法正则处理的完整流程图

```
用户输入非法正则（如 "nginx("）
    │
    ▼
cmdBuff.GetText() 返回 "nginx("
    │
    ▼
TableData.Filter() 判断不是标签/Fuzzy → 调用 rxFilter()
    │
    ▼
rxFilter: regexp.Compile("(?i)(nginx()") → 编译失败
    │
    ▼
返回 (nil, error)
    │
    ▼
Filter: slog.Error(...) → 仅日志记录
    │
    ▼
td.rowEvents 保持原始全量数据
    │
    ▼
UI 展示全部数据，无错误提示，用户可能以为筛选条件没生效
```

---

### 7.3 标签选择器与客户端正则互斥的完整示例链

#### 7.3.1 互斥机制的两处代码

**第一处**：`Browser.BufferCompleted`（`internal/view/browser.go:217-225`）— 清除对立面状态

```go
func (b *Browser) BufferCompleted(text, _ string) {
    if internal.IsLabelSelector(text) {
        // 是标签选择器 → 设置 model.labelSelector
        if sel, err := ui.ExtractLabelSelector(text); err == nil {
            b.GetModel().SetLabelSelector(sel)
        }
    } else {
        // 不是标签选择器 → 重置 labelSelector 为 Everything()
        b.GetModel().SetLabelSelector(labels.Everything())
    }
}
```

**第二处**：`TableData.Filter`（`internal/model1/table_data.go:149-151`）— 跳过客户端过滤

```go
if f.Filter == "" || internal.IsLabelSelector(f.Filter) {
    return td  // 标签选择器不走客户端 rxFilter
}
```

#### 7.3.2 场景一：从正则切换到标签选择器

```
1. 用户输入 "nginx"
   → IsLabelSelector("nginx") = false
   → model.labelSelector 重置为 Everything()
   → rxFilter("nginx") 在客户端匹配 → 显示匹配行

2. 用户清除后输入 "app=nginx"
   → IsLabelSelector("app=nginx") = true
   → model.labelSelector 设置为 app=nginx
   → 触发 Refresh() 从服务器重新拉取数据
   → Filter() 中检测到标签选择器 → 跳过 rxFilter
   → 显示服务器返回的匹配资源

结果：之前的客户端正则过滤被完全替换，不可能两者叠加
```

#### 7.3.3 场景二：从标签选择器切换到正则

```
1. 用户输入 "app=nginx"
   → IsLabelSelector = true
   → model.labelSelector = app=nginx
   → 服务器端过滤

2. 用户清除后输入 "prod"
   → IsLabelSelector("prod") = false（不含 =，ToLabels 返回 nil）
   → model.labelSelector 重置为 Everything()
   → 触发 Refresh() 从服务器重新拉取全量数据
   → rxFilter("prod") 在客户端匹配 → 显示匹配行

结果：之前的服务器端标签过滤被清除，重新拉取了全量数据
```

#### 7.3.4 场景三：标签选择器解析失败的边界

```go
// internal/view/browser.go:217-225
func (b *Browser) BufferCompleted(text, _ string) {
    if internal.IsLabelSelector(text) {
        if sel, err := ui.ExtractLabelSelector(text); err == nil {
            b.GetModel().SetLabelSelector(sel)
        }
        // ⚠️ 如果 ExtractLabelSelector 失败，什么都不做！
        // model.labelSelector 保持上一次的值
    } else {
        b.GetModel().SetLabelSelector(labels.Everything())
    }
}
```

**问题**：当 `IsLabelSelector` 返回 `true` 但 `ExtractLabelSelector` 解析失败时，`model.labelSelector` 不会被更新，也不会被重置。它保持上一次的值。

**实际触发条件**：理论上 `-l` 前缀会绕过 `ToLabels` 检查。如果输入 `-l` 开头的内容但后面不是合法的标签表达式（如 `-l !!!`），`IsLabelSelector` 返回 `true`，但 `ExtractLabelSelector` 调用 `labels.Parse("!!!")` 会失败，此时 `model.labelSelector` 保持上次值不变。

```
1. 用户输入 "app=nginx"
   → IsLabelSelector = true, ExtractLabelSelector 成功
   → model.labelSelector = app=nginx

2. 用户输入 "-l !!!"
   → IsLabelSelector = true（-l 前缀匹配成功）
   → ExtractLabelSelector("-l !!!") → labels.Parse("!!!") → 解析失败
   → model.labelSelector 保持 app=nginx 不变！

结果：用户看到标题栏显示 "!!! " 的筛选标记，
      但实际数据仍按 app=nginx 在服务器端过滤，产生不一致
```

#### 7.3.5 场景四：`-l` 前缀绕过空格检查

`-l` 前缀的标签选择器不受空格限制，因为 `labelRx`（`\A\-l`）匹配在空格检查之前：

```go
// internal/helpers.go
func IsLabelSelector(s string) bool {
    if labelRx.MatchString(s) {  // 先检查 -l 前缀
        return true              // 匹配到就直接返回 true，不管后面有没有空格
    }
    return !strings.Contains(s, " ") && cmd.ToLabels(s) != nil
}
```

| 输入 | labelRx 匹配 | 结果 | 路径 |
|------|-------------|------|------|
| `-l app=nginx` | ✅ | 标签选择器 | 服务器端 |
| `-l environment in (production, staging)` | ✅ | 标签选择器 | 服务器端（支持 `in`/`notin` 操作符） |
| `environment in (production)` | ❌ | **不是标签选择器**（含空格） | 客户端正则（但因空格短路，实际无过滤） |

> **重要区别**：同样的标签条件，不带 `-l` 前缀时因为含空格而被拒绝；带 `-l` 前缀则可以正常使用 `in`/`notin` 等包含空格的集合操作符。

#### 7.3.6 互斥机制的完整状态转换图

```
                  ┌──────────────────────┐
                  │    初始状态           │
                  │ labelSelector=Everything │
                  │ cmdBuff=""            │
                  └──────────┬───────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                 ▼
    输入 "nginx"      输入 "app=nginx"   输入 "-f ng"
            │                │                 │
            ▼                ▼                 ▼
    IsLabel=false     IsLabel=true       IsLabel=false
            │                │                 │
            ▼                ▼                 ▼
    label=Everything   label=app=nginx   label=Everything
    cmdBuff="nginx"    cmdBuff="app=nginx"  cmdBuff="-f ng"
            │                │                 │
            ▼                ▼                 ▼
    客户端 rxFilter     服务器端过滤      客户端 fuzzyFilter
    匹配所有可见列      API 重新拉取       匹配名称列(ID)
```

---

### 7.4 Fuzzy 前缀多词处理与三种空格写法对比

#### 7.4.1 Fuzzy 前缀接多个词的代码分析

Fuzzy 过滤的类型判断由 `IsFuzzySelector` 函数完成，其核心是一个正则表达式：

```go
// internal/helpers.go:14
var fuzzyRx = regexp.MustCompile(`\A-f\s?([\w-]+)\b`)

// internal/helpers.go:38-45
func IsFuzzySelector(s string) (string, bool) {
    mm := fuzzyRx.FindStringSubmatch(s)
    if len(mm) != 2 {
        return "", false
    }
    return mm[1], true
}
```

**正则表达式拆解**：
- `\A` — 字符串开头锚点
- `-f` — 字面量 `-f` 前缀
- `\s?` — 可选的**一个**空白字符（只能有 0 或 1 个空格）
- `([\w-]+)` — 捕获组：**一个或多个**单词字符（字母、数字、下划线）或连字符 `-`
- `\b` — 单词边界

**关键特性**：
1. **只捕获第一个词**：`[\w-]+` 不包含空格，遇到空格就停止匹配
2. **忽略后续内容**：`FindStringSubmatch` 只找第一个匹配，后面的词不会被捕获，也不会报错
3. **静默丢弃**：多出来的词没有任何处理，也没有任何提示

#### 7.4.2 Fuzzy 前缀 + 空格 + 多词的完整执行路径

以输入 `-f nginx pod` 为例，完整调用链如下：

```
用户输入 "-f nginx pod"
    │
    ▼
cmdBuff 接收完整字符串
    │
    ▼
TableData.Filter() 被调用
    │
    ├─ 1. Toast 过滤（可选）
    │
    ├─ 2. IsLabelSelector("-f nginx pod") ?
    │    ├─ labelRx (\A\-l) 不匹配（-f 不是 -l）
    │    ├─ 含空格 → !ContainsSpace = false
    │    └─ 返回 false → 继续
    │
    ├─ 3. IsFuzzySelector("-f nginx pod") ?
    │    ├─ fuzzyRx 匹配：FindStringSubmatch 找到 "-f nginx"
    │    ├─ 捕获组 mm[1] = "nginx"（只取第一个词）
    │    └─ 返回 ("nginx", true)
    │
    └─ 4. 执行 fuzzyFilter("nginx")
         ├─ 只对 Row.ID 列进行 fuzzy 匹配
         ├─ "pod" 这个词被完全忽略，没有任何警告
         └─ 返回匹配结果
```

**多词输入的行为对照表**：

| 输入 | 捕获到的 fuzzy 关键词 | 实际过滤行为 | 其余内容的命运 |
|------|---------------------|-------------|--------------|
| `-f nginx` | `nginx` | 正常 fuzzy 匹配名称列 | - |
| `-f nginx pod` | `nginx` | 只按 `nginx` fuzzy 匹配 | **pod 被静默丢弃** |
| `-f nginx pod svc` | `nginx` | 只按 `nginx` fuzzy 匹配 | **pod、svc 全部静默丢弃** |
| `-f nginx-pod` | `nginx-pod` | 按 `nginx-pod` fuzzy 匹配（连字符 `-` 属于 `[\w-]` 字符集） | - |
| `-f  nginx`（两空格） | -（匹配失败） | 不进入 fuzzy 路径 | 整体落入 rxFilter → 空格短路 → 全量显示 |

**关于 `\s?` 边界的推导**：

正则 `\A-f\s?([\w-]+)\b` 中，`\s?` 只匹配 0 或 1 个空白字符。当 `-f` 后有两个空格时：
1. `\A-f` 匹配 `-f`
2. `\s?` 匹配第一个空格（贪心匹配 1 个）
3. 此时游标指向第二个空格，`[\w-]+` 无法匹配空格字符
4. 又因为 `[\w-]+` 是**至少一个**字符的贪婪匹配，不能匹配空串
5. 整个正则匹配失败，`FindStringSubmatch` 返回空切片

**结论**：**`-f` 后接两个或更多空格时，`IsFuzzySelector` 返回 `false`**，输入会走到 `rxFilter` 路径，然后因为含空格被短路返回全量数据。用户输入 `-f  nginx`（多打了一个空格）时，实际效果等于没有筛选。

#### 7.4.3 三种空格写法的横向对比

将三类筛选遇到空格时的行为放在一起对比：

| 维度 | 普通带空格正则查询 | Fuzzy 前缀 + 空格 + 多词 | 标签选择器 + 空格 |
|------|-----------------|----------------------|-----------------|
| **示例** | `nginx pod` | `-f nginx pod` | `-l app=nginx, env=prod` |
| **类型判断** | 非标签非 fuzzy | Fuzzy 匹配（提取第一个词） | 标签选择器（`-l` 前缀优先） |
| **空格处理位置** | `rxFilter` 开头短路 | `fuzzyRx` 正则中只捕获第一个词 | `ExtractLabelSelector` 去掉 `-l` 后交 K8s 解析 |
| **代码位置** | `internal/model1/table_data.go:167-169` | `internal/helpers.go:14` | `internal/ui/table_helper.go:62-69` |
| **过滤层面** | 客户端 | 客户端 | 服务器端 |
| **空格后的内容** | 整个查询失效，返回全量 | 后面的词被静默丢弃 | 空格后的内容参与标签解析 |
| **用户感知** | 无提示，表格显示全部 | 无提示，只按第一个词筛选 | 正常工作，支持 in/notin |
| **最终效果** | **等于没有筛选** | **部分筛选（只按首词）** | **完整筛选** |

#### 7.4.4 四种空格相关输入的完整行为矩阵

| 输入 | `IsLabelSelector` | `IsFuzzySelector` | 最终路径 | 实际效果 |
|------|-------------------|-------------------|---------|---------|
| `nginx` | `false` | `("", false)` | rxFilter（正则匹配） | 所有列正则匹配 nginx |
| `nginx pod` | `false`（含空格） | `("", false)` | rxFilter → 空格短路 | **全量显示（无过滤）** |
| `-f nginx` | `false` | `("nginx", true)` | fuzzyFilter | 名称列 Fuzzy 匹配 nginx |
| `-f nginx pod` | `false`（含空格） | `("nginx", true)` | fuzzyFilter("nginx") | **只按 nginx 做 Fuzzy 匹配，pod 被丢弃** |
| `-f  nginx`（两空格） | `false` | `("", false)`（匹配失败） | rxFilter → 空格短路 | **全量显示（无过滤）** |
| `app=nginx` | `true` | - | 服务器端标签过滤 | 按 app=nginx 拉取 |
| `app=nginx, env=prod` | `false`（含空格） | - | rxFilter → 空格短路 | **全量显示（无过滤）** |
| `-l app=nginx, env=prod` | `true`（`-l` 优先） | - | 服务器端标签过滤 | 按两个标签组合过滤 |
| `-l app=nginx,  env=prod`（两空格） | `true`（`-l` 优先） | - | 服务器端标签过滤（K8s 解析） | 取决于 labels.Parse 是否容忍多余空格 |

#### 7.4.5 代码层面的三种空格处理策略总结

| 策略 | 实现方式 | 所在位置 | 适用场景 |
|------|---------|---------|---------|
| **空格即短路** | `strings.Contains(q, " ")` → 直接返回全量 | `rxFilter` in `internal/model1/table_data.go` | 正则匹配 |
| **空格分词取首词** | `[\w-]+\b` 正则只捕获第一个词 | `fuzzyRx` in `internal/helpers.go` | Fuzzy 匹配 |
| **空格参与解析** | 去掉前缀后交给专业解析器（labels.Parse） | `ExtractLabelSelector` in `internal/ui/table_helper.go` | 标签选择器（仅 `-l` 前缀路径） |

> **为什么三种策略不一样？**
> - 正则匹配：多列匹配，空格会导致正则语义模糊，干脆短路
> - Fuzzy 匹配：本来就只匹配名称列，设计上就假设是单个关键词
> - 标签选择器（-l）：利用 K8s 官方库的完整解析能力，支持复杂表达式

---

## 八、总结

### 8.1 为什么容易混淆？

1. **同一入口，不同路径**：所有筛选都通过 `/` 命令输入，但背后是完全不同的两套机制
2. **隐式类型推断**：`app=nginx` 看起来像正则匹配，实际是标签选择器
3. **状态分散**：筛选状态分布在 cmdBuff、labelSelector、toast 三个地方
4. **互斥性不直观**：标签选择器和正则匹配不能同时生效，但用户可能期望它们叠加
5. **空格是隐形杀手**：含空格的输入在正则路径被静默忽略，在标签路径则改变类型判断
6. **非法正则无反馈**：编译失败时静默回退到全量数据，用户无从得知输入有误
7. **Fuzzy 多词静默丢弃**：`-f nginx pod` 中第二个词被悄悄忽略，用户可能以为多词更精确
8. **三种空格策略不统一**：正则短路、Fuzzy 取首词、标签全解析，行为不一致

### 8.2 设计思路

- **标签选择器**：利用 Kubernetes 原生能力，适合大数据量筛选
- **客户端过滤**：快速响应，适合在已获取数据中进一步筛选
- **分层设计**：服务器端粗筛 + 客户端精筛，但当前实现是互斥而非叠加

### 8.3 改进想象空间

如果需要支持叠加筛选，可以考虑：
- 标签选择器和正则匹配同时生效（服务器端 + 客户端双层过滤）
- 更明确的筛选类型标记
- 可视化的筛选状态展示
- 空格和非法正则的 UI 反馈提示
- 标签选择器解析失败时的状态重置策略
- Fuzzy 多词支持或多词时的提示
- 统一三种筛选的空格处理策略，减少用户心智负担
