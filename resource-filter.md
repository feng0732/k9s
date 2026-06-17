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
│ 标签选择器  │   │ 文本/正则   │
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

**位置**：[dao/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/dao/table.go#L59-L84)、[dao/resource.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/dao/resource.go#L27-L34) 等 DAO 层

**工作原理**：
- 将标签选择器通过 `context` 的 `internal.KeyLabels` 传递给 DAO 层
- DAO 层将其作为 `ListOptions.LabelSelector` 参数发送给 Kubernetes API
- Kubernetes API 服务器直接返回过滤后的结果
- 优势：数据量小，性能好
- 触发时机：需要刷新模型数据（从服务器重新拉取）

**状态存储**：
```go
// model/table.go
type Table struct {
    labelSelector labels.Selector  // 存储标签选择器状态
    ...
}
```

### 1.2 客户端筛选：文本/正则/Fuzzy

**位置**：[model1/table_data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model1/table_data.go#L143-L198)

**工作原理**：
- 在已获取的表格数据上进行内存过滤
- 不触发服务器请求，响应速度快
- 直接作用于 `RowEvents` 数据集

**状态存储**：
```go
// ui/table.go
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

**位置**：[internal/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/helpers.go)

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
> 输入 `app=nginx` 会被当作**标签选择器**处理，而不是文本过滤！
> 它会触发服务器端数据刷新，而不是在客户端过滤。

### 2.3 各筛选类型一览

| 类型 | 前缀/格式 | 示例 | 过滤层面 |
|------|-----------|------|----------|
| 标签选择器 | `-l` 或 `key=value` 格式 | `-l app=nginx`、`app=nginx` | 服务器端 |
| Fuzzy 过滤 | `-f` 开头 | `-f nginx` | 客户端 |
| 反向过滤 | `!` 开头 | `!nginx` | 客户端（正则） |
| Toast 过滤 | 切换标志 | （快捷键触发） | 客户端 |
| 文本/正则 | 默认 | `nginx`、`nginx.*` | 客户端 |

---

## 三、筛选状态与表格数据的配合

### 3.1 状态存储分布

筛选状态分散在三个层级：

| 层级 | 状态 | 说明 |
|------|------|------|
| **Model 层** | `labelSelector` | [model/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model/table.go#L48) 标签选择器，影响服务器数据获取 |
| **UI 层** | `cmdBuff` | [ui/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/ui/table.go#L48) 命令缓冲区，存储用户输入的原始字符串 |
| **UI 层** | `toast` | [ui/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/ui/table.go#L54) 是否只显示问题资源 |

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
    └─► 默认：正则/文本过滤（rxFilter）
         ├─ 支持反向匹配（! 开头）
         └─ 大小写不敏感
```

### 3.3 标签选择器的特殊处理流程

标签选择器的处理比文本过滤多了一步"模型刷新"：

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

**关键代码**：[view/browser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/view/browser.go#L216-L251)

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

**位置**：[model1/table_data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model1/table_data.go#L143-L164)

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
    
    // 4. 正则/文本过滤（默认）
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
4. 正则/文本过滤

---

## 四、容易混淆的关键点

### 4.1 标签选择器 vs 文本过滤：互斥关系

**重要：标签选择器和文本过滤不能同时生效，它们是互斥的！**

- 当输入被识别为标签选择器时 → 重置文本过滤，使用服务器端筛选
- 当输入不是标签选择器时 → 重置标签选择器为 `Everything()`，使用客户端筛选

**原因**：它们共享同一个输入源 `cmdBuff`，输入内容只能是其中一种类型。

### 4.2 输入示例与行为对照表

| 输入内容 | 识别为 | 过滤层面 | 行为说明 |
|----------|--------|----------|----------|
| ``（空） | 无 | - | 不过滤 |
| `nginx` | 文本/正则 | 客户端 | 在所有列中模糊匹配 nginx |
| `app=nginx` | 标签选择器 | 服务器端 | 从 K8s API 获取带 app=nginx 标签的资源 |
| `-l app=nginx` | 标签选择器 | 服务器端 | 同上，显式标记 |
| `-f nginx` | Fuzzy 过滤 | 客户端 | 按名称进行模糊匹配 |
| `!nginx` | 反向正则 | 客户端 | 显示不包含 nginx 的行 |

### 4.3 Fuzzy 过滤 vs 正则过滤的区别

| 特性 | Fuzzy 过滤 (`-f`) | 正则/文本过滤 |
|------|-------------------|---------------|
| 匹配范围 | 只匹配名称列 (ID) | 匹配所有可见列 |
| 算法 | 模糊匹配 (fuzzy.Find) | 正则表达式匹配 |
| 性能 | 较快 | 稍慢（多列匹配） |
| 大小写 | 取决于库 | 不敏感 `(?i)` |

**Fuzzy 过滤**：[model1/table_data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model1/table_data.go#L200-L219)
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

**位置**：[ui/table.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/ui/table.go#L708-L723)

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

**位置**：[model1/table_data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model1/table_data.go#L41-L45)

```go
type FilterOpts struct {
    Toast  bool   // 是否只显示有问题的资源
    Filter string // 过滤文本
    Invert bool   // 是否反向匹配（实际通过 ! 前缀判断）
}
```

### 6.2 FishBuff / CmdBuff

**位置**：[model/cmd_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/12-k9s/internal/model/cmd_buff.go)

命令缓冲区，存储用户输入的原始字符串，支持：
- 字符输入/删除
- 输入延迟防抖（100ms）
- 监听者模式（BufferChanged/BufferCompleted/BufferActive）

---

## 七、总结

### 7.1 为什么容易混淆？

1. **同一入口，不同路径**：所有筛选都通过 `/` 命令输入，但背后是完全不同的两套机制
2. **隐式类型推断**：`app=nginx` 看起来像文本过滤，实际是标签选择器
3. **状态分散**：筛选状态分布在 cmdBuff、labelSelector、toast 三个地方
4. **互斥性不直观**：标签选择器和文本过滤不能同时生效，但用户可能期望它们叠加

### 7.2 设计思路

- **标签选择器**：利用 Kubernetes 原生能力，适合大数据量筛选
- **客户端过滤**：快速响应，适合在已获取数据中进一步筛选
- **分层设计**：服务器端粗筛 + 客户端精筛，但当前实现是互斥而非叠加

### 7.3 改进想象空间

如果需要支持叠加筛选，可以考虑：
- 标签选择器和文本过滤同时生效（服务器端 + 客户端双层过滤）
- 更明确的筛选类型标记
- 可视化的筛选状态展示
