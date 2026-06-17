# K9s 自定义视图列裁剪与默认列回退逻辑深度分析

## 一、整体架构概览

自定义视图功能涉及四个核心层次的协作：

```
┌──────────────────────────────────────────────────────┐
│  配置层 (config/views.go)                             │
│  - ViewSetting: 存储 columns 列表和 sortColumn        │
│  - CustomView: 按 GVR@Namespace 匹配加载视图设置      │
└──────────────────────┬───────────────────────────────┘
                       │ SetViewSetting()
                       ▼
┌──────────────────────────────────────────────────────┐
│  列定义解析层 (render/cust_col.go)                    │
│  - parse(): 正则解析 NAME[:spec]|FLAGS 格式           │
│  - colDef → HeaderColumn 转换                         │
└──────────────────────┬───────────────────────────────┘
                       │ parseSpecs()
                       ▼
┌──────────────────────────────────────────────────────┐
│  渲染映射层 (render/cust_cols.go)                     │
│  - ColumnSpecs.Header(): 表头合并(自定义列+默认列)    │
│  - ColumnSpecs.realize(): 逐行数据渲染(核心逻辑)      │
│  - hydrate(): JSONPath/JQ 求值 + 默认列回退           │
└──────────────────────┬───────────────────────────────┘
                       │ realize() → hydrateRow()
                       ▼
┌──────────────────────────────────────────────────────┐
│  数据模型层 (model1/header.go, model1/table_data.go)  │
│  - Header.Customize(): 列索引裁剪                     │
│  - Header.FilterColIndices(): 可见列过滤              │
│  - Header.MapIndices(): 列名→索引映射                 │
└──────────────────────────────────────────────────────┘
```

---

## 二、配置层：视图定义与匹配

### 2.1 核心数据结构

定义于 [views.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/config/views.go#L35-L46)：

```go
type ViewSetting struct {
    Columns    []string `yaml:"columns"`     // 列定义字符串数组
    SortColumn string   `yaml:"sortColumn"`  // 排序列: "NAME:asc|desc"
}
```

### 2.2 视图配置匹配规则

[getVS()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/config/views.go#L177-L223) 函数的匹配优先级（从高到低）：

1. **精确匹配带命名空间正则**：键包含 `@` 时，将 `gvr@ns` 与正则匹配
2. **GVR 前缀匹配**：`strings.HasPrefix(key, gvr)` 或反向
3. **精确相等匹配**：`key == gvr`
4. **双段拆分匹配**：将 `gvr ns` 拆分为 `gvr@ns` 精确查找

> 匹配时 keys 按字典序降序排列，长 key 优先匹配，避免短前缀误命中。

---

## 三、列定义解析层：字符串 → 结构化规格

### 3.1 列定义语法

解析器正则定义于 [cust_col.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_col.go#L17)：

```regex
^([\w\s%/-]+):?([\w\W]*?)\|?([NTWSLRH]{0,3})$
  └────┬─────┘  └───┬───┘  └────┬─────┘
       │            │           └─ FLAGS: 属性标志
       │            └─ spec: JSONPath 或 JQ 表达式
       └─ NAME: 列标题名
```

### 3.2 FLAGS 属性标志

| 标志 | 含义 | 对应 Attrs 字段 |
|------|------|-----------------|
| `N`  | 数字/容量类型，右对齐 | `Capacity=true, Align=Right` |
| `T`  | 时间类型，转人类可读时长 | `Time=true` |
| `W`  | 宽屏列，默认视图隐藏 | `Wide=true` |
| `S`  | 强制显示，取消 Wide | `Show=true, Wide=false` |
| `L`  | 左对齐 | `Align=Left` |
| `R`  | 右对齐 | `Align=Right` |
| `H`  | 隐藏列 | `Hide=true` |

> **注意**：`W` 和 `S` 互斥，后设置的覆盖前者；`N` 隐含 `R`（右对齐）。

### 3.3 解析流程

[parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_col.go#L80-L96) → [toHeaderCol()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_col.go#L98-L113)：

```
"CPU/RL:.spec.containers[].resources.limits.cpu|NW"
        │
        ▼ parse()
colDef{
    name: "CPU/RL",
    spec: "{.spec.containers[].resources.limits.cpu}",  // RelaxedJSONPathExpression 包装
    colAttrs: {wide:true, capacity:true, align:Right}
}
        │
        ▼ toHeaderCol()
model1.HeaderColumn{
    Name: "CPU/RL",
    Attrs: {Wide:true, Capacity:true, Align:Right, ...}
}
```

### 3.4 JSONPath vs JQ 判定

[isJQSpec()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L267-L269)：

```go
func isJQSpec(spec string) bool {
    return len(strings.Split(spec, "|")) > 2  // 管道符≥3段 → JQ 语法
}
```

- **JSONPath**：`{.metadata.name}`、`{.spec.replicas}` 等，由 `k8s.io/client-go/util/jsonpath` 求值
- **JQ**：`{.status.containerStatuses[]|select(.ready==false)|.name}` 等，由 `itchyny/gojq` 求值

---

## 四、表头合并：自定义列与默认列的超集构建

### 4.1 合并入口

渲染器通过 [Base.SetViewSetting()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/base.go#L45-L57) 注入配置：

```go
func (b *Base) SetViewSetting(vs *config.ViewSetting) {
    cols := vs.Columns
    specs, err := NewColsSpecs(cols...).parseSpecs()  // → ColumnSpecs
    b.specs = specs
}
```

各资源渲染器的 `Header()` 方法（如 Pod）调用：

```go
func (p *Pod) Header(string) model1.Header {
    return p.doHeader(defaultPodHeader)  // 见 base.go L36-L42
}
```

### 4.2 ColumnSpecs.Header() 合并算法

[ColumnSpecs.Header()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L94-L110)：

```go
func (cc ColumnSpecs) Header(rh model1.Header) model1.Header {
    // Step1: 以自定义列顺序为基底
    hh := make(model1.Header, 0, len(cc))
    for _, h := range cc {
        hh = append(hh, h.Header)
    }

    // Step2: 遍历默认表头，同名列合并属性，新列追加
    for _, h := range rh {
        if idx, ok := hh.IndexOf(h.Name, true); ok {
            hh[idx].Attrs = hh[idx].Merge(h.Attrs)  // 属性合并
            continue
        }
        hh = append(hh, h)  // 默认列不在自定义列表中，追加到末尾
    }

    return hh
}
```

### 4.3 Attrs.Merge() 优先级规则

[Attrs.Merge()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go#L31-L57) —— **自定义列配置优先，默认列属性兜底**：

| 字段 | 合并规则 | 说明 |
|------|---------|------|
| `MX/MXC/MXM/Decorator/VS` | `a = b` | 默认值覆盖（自定义一般不设置） |
| `Align` | `a==0 → a=b` | 自定义未显式对齐时取默认 |
| `Hide` | `!a.Hide → a=b` | 自定义未隐藏才接受默认隐藏 |
| `Wide` | `!a.Show && !a.Wide → a=b` | 自定义未显式 Show/Wide 才取默认 |
| `Time/Capacity` | `!a.Xxx → a=b` | 自定义未设才取默认 |

### 4.4 合并示例

假设 Pod 默认头含 `RESTARTS(Align=Right)`、`AGE(Time=true)`，用户配置：
```yaml
columns:
  - "NAME"
  - "RESTARTS|TR"  # 自定义加 Time 标志
  - "MY_COLUMN:.spec.foo"  # 全新自定义列
```

合并结果：
```
[0] NAME           (自定义，无特殊属性)
[1] RESTARTS       (Time=true 来自自定义，Align=Right 从默认Merge过来)
[2] MY_COLUMN      (自定义，无默认属性)
[3] NAMESPACE      (默认追加，Wide=false)
[4] STATUS         (默认追加)
[5] AGE            (默认追加，Time=true)
...其余默认列按序追加
```

---

## 五、核心：列裁剪与数据渲染

### 5.1 渲染调用链（以 Pod 为例）

[Pod.Render()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/pod.go#L132-L149)：

```go
func (p *Pod) Render(o any, _ string, row *model1.Row) error {
    pwm := o.(*PodWithMetrics)
    
    // Step1: 先渲染默认行（所有默认列的值）
    if err := p.defaultRow(pwm, row); err != nil {
        return err
    }
    
    // 无自定义配置 → 直接返回默认行
    if p.specs.isEmpty() {
        return nil
    }
    
    // Step2: 自定义列重渲染 + 默认列回退
    cols, err := p.specs.realize(pwm.Raw.DeepCopy(), defaultPodHeader, row)
    
    // Step3: 用渲染结果替换 row.Fields
    cols.hydrateRow(row)  // 将 RenderedCols.Value 提取为 Fields 数组
    return nil
}
```

### 5.2 realize()：列裁剪 + 默认列回退主逻辑

[ColumnSpecs.realize()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L112-L146) 分为两大阶段：

#### 阶段一：自定义列求值（hydrate）

```go
vv, err := hydrate(o, cc, parsers, rh, row)
```

对每个自定义列规格独立求值，结果顺序与自定义列配置顺序**严格一致**。

#### 阶段二：默认列回退补全

```go
for _, hc := range rh {                 // 遍历默认表头的每一列
    if vv.HasHeader(hc.Name) {          // 如果已在 hydrate 结果中 → 跳过
        continue
    }
    if idx, ok := rh.IndexOf(hc.Name, true); ok {
        rc := RenderedCol{Header: hc, Value: row.Fields[idx]}  // 取默认行数据
        rc.Header.Wide = true            // 关键：回退列强制标记 Wide=true
        vv = append(vv, rc)              // 追加到结果末尾
    }
}
```

> **列裁剪的本质**：自定义列配置就是「白名单」，不在白名单中的默认列虽然也追加回来，但被强制标记 `Wide=true`，在非宽屏模式下被隐藏。

### 5.3 hydrate()：四种列值获取路径

[hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L148-L265) 对每列按 `parser` 是否为 `nil` 分支处理：

```
列定义有无 Spec?
    │
    ├─ 无 Spec (parser=nil) → 默认列引用模式
    │       │
    │       ├─ 在默认表头中找到同名?
    │       │     ├─ 是 → 取 row.Fields[idx]，同时合并默认表头的属性
    │       │     └─ 否 → Value = NAValue("n/a")，Warn 日志
    │       │
    │       └─ row.Fields 索引越界? → Value = NAValue
    │
    └─ 有 Spec (parser≠nil) → 表达式求值模式
            │
            ├─ 优先 JQ 模式 (管道>2段)?
            │     ├─ 成功 → JQ 结果转字符串
            │     └─ 失败/非JQ → JSONPath 求值
            │
            ├─ runtime.Object 类型分支
            │     ├─ Unstructured → UnstructuredContent() 传 map
            │     └─ 结构化类型 → reflect.ValueOf().Elem().Interface()
            │
            ├─ 结果集处理
            │     ├─ vals[0] 为空 → MissingValue("<none>")
            │     └─ 多值 → 用 "," 连接
            │
            └─ 类型转换 (MXC/MXM/Time)
                  ├─ MXC: resource.Quantity → milli-core 值
                  ├─ MXM: resource.Quantity → MiB 值
                  └─ Time: RFC3339 字符串 → ToAge() 人类时长
```

### 5.4 缺失值层级定义

定义于 [types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/types.go#L37-L52)，按语义从重到轻：

| 常量 | 值 | 触发场景 |
|------|-----|---------|
| `NAValue` | `"n/a"` | 无 Spec 且默认表头找不到列名；字段索引越界；对象为 nil |
| `MissingValue` | `"<none>"` | JSONPath/JQ 求值成功但结果为空数组 |
| `UnknownValue` | `"<unknown>"` | 时间解析失败 (toAgeHuman)；时间零值 (ToAge) |
| `ZeroValue` | `"0"` | MXC/MXM 转换时值为 0（区别于缺失） |
| `UnsetValue` | `"<unset>"` | 暂未见使用，保留位 |

---

## 六、数据模型层：可见性裁剪与索引映射

### 6.1 Header.Customize()：宽屏模式列裁剪

[Header.Customize()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go#L133-L166) 用于 View 层的列选择：

```go
func (h Header) Customize(cols []string, wide bool) Header {
    // Step1: 按 cols 顺序构建核心列
    for _, c := range cols {
        idx, ok := h.IndexOf(c, true)
        if !ok {
            cc = append(cc, HeaderColumn{Name: c})  // 找不到就占位空列
            continue
        }
        col := h[idx].Clone()
        col.Wide = false    // 白名单列强制非 Wide
        cc = append(cc, col)
    }

    // Step2: wide=true 时追加其余列（强制 Wide=true）
    if wide {
        for i, c := range h {
            if _, ok := xx[i]; ok { continue }
            col := c.Clone()
            col.Wide = true
            cc = append(cc, col)
        }
    }
    return cc
}
```

> 此函数与 `ColumnSpecs.Header()` 逻辑对偶：前者作用于渲染器的 Header 方法返回合并头，后者作用于 UI 层按需裁剪显示列。

### 6.2 Header.FilterColIndices()：最终可见列判定

[Header.FilterColIndices()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go#L177-L192)：

```go
func (h Header) FilterColIndices(ns string, wide bool) sets.Set[int] {
    for i, c := range h {
        if c.Name == "AGE" || !wide && c.Wide || c.Hide || (nsed && c.Name == "NAMESPACE") {
            continue  // 排除条件：AGE列单独处理、非宽屏的Wide列、Hide列、命名空间下的NAMESPACE列
        }
        cc.Insert(i)
    }
}
```

**「不在自定义白名单的默认列如何被隐藏」的完整链路**：

```
用户 columns: [NAME, STATUS]
        │
        ▼ realize() 阶段二
默认列 RESTARTS 被追加时 rc.Header.Wide = true
        │
        ▼ 切换到非宽屏 (wide=false)
FilterColIndices() 跳过 !wide && c.Wide 的列
        │
        ▼
UI 表格只显示 NAME, STATUS，RESTARTS 等不在可见集合内
```

### 6.3 Header.MapIndices()：列名→索引的重排

[Header.MapIndices()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go#L110-L131)：用于表格导出/排序时的列索引转换，`wide=true` 时同样追加剩余列索引。

---

## 七、完整端到端流程示例

### 场景：Pod 自定义视图

**配置** (`views.yaml`)：
```yaml
views:
  v1/pods:
    columns:
      - "NAMESPACE"
      - "NAME"
      - "STATUS"
      - "NODE_IP:.status.hostIP"
      - "AGE|TR"
    sortColumn: "AGE:asc"
```

**执行流程**：

```
① config/CustomView.Load()
   └─ getVS("v1/pods", "default") 匹配成功
      └─ 返回 ViewSetting{Columns:[5], SortColumn:"AGE:asc"}

② view/Table 监听回调，调用 renderer.SetViewSetting(vs)
   └─ ColsSpecs.parseSpecs()
      ├─ "NAMESPACE"         → Spec="" , Attrs{}
      ├─ "NAME"              → Spec="" , Attrs{}
      ├─ "STATUS"            → Spec="" , Attrs{}
      ├─ "NODE_IP:.status.hostIP" → Spec="{.status.hostIP}", Attrs{}
      └─ "AGE|TR"            → Spec="" , Attrs{Time:true, Align:Right}
   └─ 存入 b.specs (Base 结构体字段)

③ Pod.Header() → doHeader(defaultPodHeader)
   └─ ColumnSpecs.Header()
      ├─ 基底顺序: NAMESPACE, NAME, STATUS, NODE_IP, AGE
      ├─ Merge默认属性: AGE 合并到用户自定 Time+Right
      └─ 追加默认其余列: READY, RESTARTS, CPU, MEM, IP, NODE, ...
                      (追加时保持默认，后续 Wide 控制显示)

④ Pod.Render(podWithMetrics, ns, row)
   ├─ p.defaultRow()  → row.Fields 按 defaultPodHeader 填 26 个默认值
   │
   └─ p.specs.realize(rawPod, defaultPodHeader, row)
      ├─ parsers[3] = JSONPath("{.status.hostIP}")  (仅 NODE_IP 有 Spec)
      │
      ├─ hydrate() 逐个处理 5 个自定义列:
      │   ├─ [0] NAMESPACE (Spec="")
      │   │    └─ rh.IndexOf("NAMESPACE")=0 → row.Fields[0] = "default"
      │   ├─ [1] NAME (Spec="")
      │   │    └─ rh.IndexOf("NAME")=1 → row.Fields[1] = "nginx-abc"
      │   ├─ [2] STATUS (Spec="")
      │   │    └─ rh.IndexOf("STATUS")=5 → row.Fields[5] = "Running"
      │   ├─ [3] NODE_IP (Spec≠nil)
      │   │    └─ JSONPath 求值 → "10.0.1.5"
      │   └─ [4] AGE (Spec="", Attrs.Time=true)
      │        └─ rh.IndexOf("AGE")=25 → row.Fields[25] = "3h25m"
      │
      └─ 默认列回补: 遍历 defaultPodHeader 26 列
           ├─ 跳过已含的 NAMESPACE, NAME, STATUS, AGE
           └─ 追加: READY(Wide), RESTARTS(Wide), LAST_RESTART(Wide),
                  CPU(Wide), CPU/RL(Wide), MEM(Wide), ..., VALID(Wide)
              共追加 26-4 = 22 列，全部 Wide=true

⑤ cols.hydrateRow(row) → row.Fields = [
     "default", "nginx-abc", "Running", "10.0.1.5", "3h25m",
     "1/1",       # READY (Wide)
     "0",         # RESTARTS (Wide)
     "<none>",    # LAST RESTART (Wide)
     ... 共 27 个字段
   ]

⑥ UI 层渲染时 FilterColIndices(ns="default", wide=false)
   ├─ 跳过: c.Name=="AGE"? 否
   ├─ 跳过: !wide && c.Wide? → 字段索引 5~26 全部 Wide=true → 跳过
   ├─ 跳过: c.Hide? 否
   └─ 跳过: nsed && c.Name=="NAMESPACE"? 是 → 索引 0 跳过
   
   最终可见列索引: {1, 2, 3, 4}
   → 屏幕上显示 4 列: NAME | STATUS | NODE_IP | AGE
```

---

## 八、设计要点与易错点

### 8.1 自定义列的「白名单」语义

配置 `columns` 数组是**显示顺序+显示选择**双重语义：
- 列表内的列 → 非 Wide（除非手动加 `\|W`），非宽屏必显示
- 列表外的默认列 → 强制 Wide，仅宽屏模式可见
- 全新自定义列（`.status.hostIP` 这类）→ 本身不在默认头内，不影响回退

### 8.2 无 Spec 列的「引用复用」机制

```
"STATUS"（无 Spec）≠  重新计算 STATUS 值
                 ＝  从默认渲染好的 row.Fields 中按列名索引复用
```

这意味着：自定义配置中写 `"STATUS"` 和写 `"STATUS:.status.phase"` **结果可能不同**。前者复用渲染器的复杂状态判定逻辑（Running/Pending/CrashLoopBackOff 等），后者直接取原始字段，丢失了 Pod 的 Ready 条件、容器状态等综合判断。

### 8.3 默认列回退的「Wide 强制」

回退逻辑在 [cust_cols.go:140](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L138-L142) 中硬编码 `Wide=true`，**覆盖默认列原本的 Wide 属性**。例如原本 `RESTARTS` 默认是普通列（Wide=false），只要不在自定义白名单里，就被降级为宽屏列。

### 8.4 JQ vs JSONPath 的选择边界

- JSONPath 失败只告警，不中断渲染（slog.Warn 后 parser 仍为非 nil，后续走空数组 → MissingValue）
- JQ 解析失败降级为 JSONPath（isJQSpec 返回 false），可能产生意外结果
- JQ 判定阈值 `Split("|") > 2`（即至少 3 段 / 2 个管道），单/双管道仍走 JSONPath

### 8.5 对象深拷贝保护

`Pod.Render()` 传 `pwm.Raw.DeepCopy()` 给 `realize()`，避免 JSONPath/JQ 求值过程中（通过反射）意外修改原始对象。其他资源渲染器（如 Service, Deployment 等）也统一采用 `raw.DeepCopy()` 模式。

---

## 九、关键文件导航

| 文件 | 核心职责 | 关键函数/结构 |
|------|---------|--------------|
| [config/views.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/config/views.go) | 视图配置加载与匹配 | `ViewSetting`, `CustomView.getVS()` |
| [render/cust_col.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_col.go) | 单列字符串解析 | `parse()`, `colDef`, `newColFlags()` |
| [render/cust_cols.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go) | 列集合渲染与回退 | `ColumnSpecs.Header()`, `realize()`, `hydrate()` |
| [render/base.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/base.go) | 渲染器基类，配置注入 | `Base.SetViewSetting()`, `doHeader()` |
| [render/pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/pod.go) | Pod 渲染完整示例 | `Pod.Header()`, `Pod.Render()` |
| [render/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/types.go) | 缺失值常量定义 | `MissingValue`, `NAValue` 等 |
| [model1/header.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go) | 表头结构与裁剪 | `Header.Customize()`, `FilterColIndices()`, `Attrs.Merge()` |
| [model1/table_data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/table_data.go) | 表格数据模型 | `TableData.Render()`, `ComputeSortCol()` |

---

## 十、从测试用例看边界场景

### 测试参考：[cust_cols_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols_test.go)

1. **纯列名（无 Spec）**：从默认行按列名取数，属性 Merge 默认值
2. **列名 + Spec + Flags**：JSONPath 求值，Flags 设置 Wide/Capacity/Time 等
3. **nil 对象安全**：`hydrate(nil, ...)` 返回 `NAValue`，不 panic
4. **Spec 解析失败**：`parseSpecs()` 直接返回错误，不部分生效（非降级容错）
5. **列名引用不存在**：hydrate 返回 `NAValue` + slog.Warn，不中断渲染

---

## 十一、补充：无表达式列复用默认值时的属性设置细节

### 11.1 问题的核心矛盾

当用户在自定义列中写 `"STATUS"` （无 Spec，无表达式），[hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L155-L175) 的 `parser == nil` 分支**用默认表头的 HeaderColumn 覆盖了用户自定义的 HeaderColumn 属性**：

```go
// hydrate() L155-L175, parser == nil 分支
if parser == nil {
    ix, ok := rh.IndexOf(cc[idx].Header.Name, true)
    if !ok {
        cols[idx] = RenderedCol{
            Header: cc[idx].Header,   // ← 用自定义的 Header（找不到默认列时）
            Value:  NAValue,
        }
        continue
    }
    var v string
    if ix >= len(row.Fields) {
        v = NAValue
    } else {
        v = row.Fields[ix]
    }
    cols[idx] = RenderedCol{
        Header: rh[ix],              // ← 关键：用默认表头的 Header 替换！
        Value:  v,
    }
    continue
}
```

**属性被替换，而非合并**：此处 `Header: rh[ix]` 直接用默认表头的 `HeaderColumn` 覆盖了 `cc[idx].Header`（用户自定义的属性）。这意味着用户在无 Spec 列上设置的 FLAGS（如 `|TR`、`|W`、`|H` 等）**在 hydrate 阶段被完全丢弃**。

### 11.2 两阶段属性博弈

属性不是完全丢失，而是经历了两阶段的"博弈"：

| 阶段 | 位置 | 操作 | 属性来源 |
|------|------|------|---------|
| ① 表头合并 | [ColumnSpecs.Header()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L94-L110) | 自定义列属性与默认列属性 Merge | 自定义优先 + 默认兜底 |
| ② 行数据填充 | [hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L155-L175) parser=nil | **Header 直接替换为 rh[ix]** | 默认列属性覆盖自定义 |

**最终效果**：对无 Spec 列，hydrate 输出的 `RenderedCol.Header` 来自默认表头 `rh[ix]`，**而非**阶段 ① 合并后的结果。也就是说，阶段 ① 的 Merge 产出被阶段 ② 覆盖了。

### 11.3 实际影响与具体示例

以 Pod 视图为例，用户配置 `"RESTARTS|TR"`（加 Time 标志和右对齐）：

```
阶段① ColumnSpecs.Header() 合并结果:
  RESTARTS → Attrs{Time:true, Align:Right, Wide:false}
  (Time 来自用户 FLAGS "TR"；Align=Right 从默认 RESTARTS Merge 过来)

阶段② hydrate() parser=nil:
  rh.IndexOf("RESTARTS") → 找到默认 defaultPodHeader[6]
  cols[idx] = RenderedCol{
      Header: rh[6],    // → Attrs{Align:Right, Wide:false} ← Time=true 丢失！
      Value:  row.Fields[6],
  }

最终 RenderedCol:
  RESTARTS → Attrs{Align:Right, Wide:false}  ← Time 标志丢失
```

**结论**：对无 Spec 列，用户在 FLAGS 中设置的 `T`(Time)、`W`(Wide)、`S`(Show)、`H`(Hide) 等属性在 hydrate 阶段被默认表头属性覆盖。但 `N`(Number/Capacity) 和 `R`(Right Align) 如果恰好与默认属性一致则无明显差异。

### 11.4 有 Spec 列的属性保留对比

有 Spec 的列（如 `"RESTARTS:.status.restartCount|TR"`）走 `parser != nil` 分支：

```go
cols[idx] = RenderedCol{
    Header: cc[idx].Header,   // ← 保留自定义的 Header，属性不丢失
    Value:  strings.Join(values, ","),
}
```

有 Spec 列**完整保留用户自定义属性**，因为 `Header` 直接取 `cc[idx].Header`。

### 11.5 对比总结表

| 列类型 | hydrate 中 Header 来源 | 用户 FLAGS 是否生效 |
|--------|----------------------|-------------------|
| 无 Spec + 默认头中找到 | `rh[ix]`（默认表头） | **否**，被默认属性覆盖 |
| 无 Spec + 默认头中找不到 | `cc[idx].Header`（自定义） | 是 |
| 有 Spec + Unstructured 对象 | `cc[idx].Header`（自定义） | 是 |
| 有 Spec + 结构化对象 | `cc[idx].Header`（自定义） | 是 |

---

## 十二、补充：默认列回补的字段索引对应机制

### 12.1 前提：row.Fields 的顺序保证

[realize()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L112-L146) 接收的 `row` 参数由各渲染器的 `defaultRow()` 方法填充。**`row.Fields` 的索引严格对应 `defaultXxxHeader`（即参数 `rh`）的列序号**。

以 [Pod.defaultRow()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/pod.go#L152-L211) 为例：

```go
row.Fields = model1.Fields{
    ns,                    // [0]  → NAMESPACE
    n,                     // [1]  → NAME
    computeVulScore(...),  // [2]  → VS
    "●",                   // [3]  → PF
    ...,                   // [5]  → STATUS
    ...,                   // [6]  → RESTARTS
    ...,                   // [25] → AGE
}
```

而 `rh`（defaultPodHeader）的列定义顺序与上述完全一致。

### 12.2 阶段二回补的索引查找逻辑

```go
for _, hc := range rh {                              // 遍历默认表头每个 HeaderColumn
    if vv.HasHeader(hc.Name) {                       // 是否已在 hydrate 结果中
        continue
    }
    if idx, ok := rh.IndexOf(hc.Name, true); ok {    // 在默认头中查找列名→索引
        rc := RenderedCol{
            Header: hc,                              // 默认头自身（含完整属性）
            Value:  row.Fields[idx],                  // 用同一个 idx 取 row.Fields
        }
        rc.Header.Wide = true                         // 强制标记 Wide
        vv = append(vv, rc)
    }
}
```

**关键保证**：`rh.IndexOf(hc.Name, true)` 返回的 `idx` 与 `row.Fields` 数组索引**一一对应**，因为 `row.Fields` 本身就是按 `rh` 的顺序由 `defaultRow()` 生成的。

### 12.3 Header.IndexOf() 的 Wide 参数影响

[Header.IndexOf()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/model1/header.go#L244-L255)：

```go
func (h Header) IndexOf(colName string, includeWide bool) (int, bool) {
    for i, c := range h {
        if c.Wide && !includeWide {
            continue       // Wide 列在 includeWide=false 时被跳过
        }
        if c.Name == colName {
            return i, true
        }
    }
    return -1, false
}
```

在 realize() 和 hydrate() 中，所有 `IndexOf` 调用均传 `includeWide=true`，因此：
- **不会因为 Wide 属性跳过任何列**
- 默认头中标记 `Wide=true` 的列（如 Pod 的 `LAST RESTART`、`SERVICE-ACCOUNT`）同样可以通过列名找到正确的 `row.Fields` 索引

### 12.4 字段索引越界保护

在 hydrate() 的 parser=nil 分支中：

```go
var v string
if ix >= len(row.Fields) {   // 索引越界保护
    v = NAValue
} else {
    v = row.Fields[ix]
}
```

**何时可能越界**：当 `rh`（默认表头）中的列数多于 `row.Fields` 实际长度时。理论上两者应一致，但防御性编码处理了：
- 渲染器 defaultRow() 未填充全部字段的情况
- CRD/自定义资源的 ServerSideTable 列定义与实际数据行不匹配的情况

### 12.5 realize() 阶段二跳过条件 HasHeader() 的精确语义

```go
func (rr RenderedCols) HasHeader(n string) bool {
    for _, r := range rr {
        if r.has(n) {          // r.Header.Name == n
            return true
        }
    }
    return false
}
```

**注意**：`HasHeader` 是按列名精确匹配，不区分列的属性差异。也就是说：

- 如果用户自定义了 `"STATUS"` （无 Spec），hydrate 阶段产出的 `RenderedCol.Header.Name = "STATUS"`，**虽然 Header 属性来自默认表头，但 Name 匹配成功**，阶段二跳过该列
- 如果用户自定义了 `"STATUS:.status.phase"`（有 Spec），hydrate 产出的 `RenderedCol.Header.Name = "STATUS"`，同样匹配，阶段二跳过
- 只有默认头中的列在 hydrate 结果里**完全不存在**时，阶段二才会追加

---

## 十三、补充：JQ 解析失败降级 JSONPath 的边界情况

### 13.1 三层降级机制全路径

以 Unstructured 对象为例，有 Spec 列的完整求值路径：

```
cc[idx].Spec 非空
    │
    ├─ realize() 预处理: parser = jsonpath.New().Parse(cc[idx].Spec)
    │     │
    │     ├─ Parse 成功 && !isJQSpec → parser 正常，后续走 JSONPath
    │     ├─ Parse 成功 && isJQSpec  → parser 正常但后续优先走 JQ
    │     └─ Parse 失败 && !isJQSpec → slog.Warn，parser 仍非 nil，后续走 JSONPath (FindResults 会再失败)
    │     └─ Parse 失败 && isJQSpec  → 不告警！parser 仍非 nil，后续优先走 JQ
    │
    └─ hydrate() 运行时:
          │
          ├─ Unstructured 对象? → 是
          │     │
          │     ├─ jqParse(cc[idx].Spec, unstructured.UnstructuredContent())
          │     │     │
          │     │     ├─ !isJQSpec(spec) → return "", false → 跳过 JQ
          │     │     │     (非 JQ 格式，直接走 JSONPath)
          │     │     │
          │     │     ├─ gojq.Parse(exp) 失败 → slog.Warn, return "", false
          │     │     │     → 降级走 JSONPath
          │     │     │
          │     │     ├─ gojq.Run() 迭代中遇 error → slog.Error, continue
          │     │     │     → 非中断，继续迭代其余结果
          │     │     │
          │     │     ├─ 迭代完但 rr 为空 → return "", false
          │     │     │     → 降级走 JSONPath
          │     │     │
          │     │     └─ 迭代有结果 → return strings.Join(rr,","), true
          │     │           → JQ 成功，直接返回
          │     │
          │     └─ JQ 返回 false → 降级走 JSONPath
          │           parser.FindResults(unstructured.UnstructuredContent())
          │             │
          │             ├─ FindResults 成功 → 正常处理结果
          │             └─ FindResults 失败 → return nil, err → hydrate 整体报错返回
          │
          └─ 非 Unstructured → 直接走 JSONPath，无 JQ 尝试
```

### 13.2 边界情况 1：JQ 表达式含管道符但不是 JQ 语法

`isJQSpec()` 的判定仅基于管道段数：`len(strings.Split(spec, "|")) > 2`。

但这与 `parse()` 阶段的 `RelaxedJSONPathExpression` 包装存在交互：

```
用户输入: "IP:.status.addresses|W"
    │
    ▼ parse() 正则匹配
mm[1]="IP", mm[2]=".status.addresses", mm[3]="W"
    │
    ▼ RelaxedJSONPathExpression(".status.addresses")
spec = "{.status.addresses}"
    │
    ▼ isJQSpec("{.status.addresses}")
Split("{.status.addresses}", "|") → 1 段 → 不是 JQ → 走 JSONPath ✓
```

此时管道符 `|W` 在正则阶段已被截断为 FLAGS，不进入 spec 字段，所以不影响 JQ 判定。

### 13.3 边界情况 2：spec 内部含管道符的"伪 JQ"

```
用户输入: "INFO:.metadata.annotations|W"
    │
    ▼ parse() 正则匹配
mm[1]="INFO", mm[2]=".metadata.annotations", mm[3]="W"
    │
    ▼ RelaxedJSONPathExpression(".metadata.annotations")
spec = "{.metadata.annotations}"
    │
    ▼ isJQSpec("{.metadata.annotations}") → 1段 → 不是 JQ → 走 JSONPath
```

但如果 spec 部分本身含多个 `|`（这是合法的 JQ 语法）：

```
用户输入: "BAD_PODS:.status.containerStatuses[]|select(.ready==false)|name|W"
    │
    ▼ parse() 正则匹配
正则 ^([\w\s%/-]+):?([\w\W]*?)\|?([NTWSLRH]{0,3})$
  → mm[1]="BAD_PODS"
  → mm[2]=".status.containerStatusees[]|select(.ready==false)|name"
  → mm[3]="W"   ← 最后一个 | 后面被截为 FLAGS
    │
    ▼ RelaxedJSONPathExpression(mm[2])
spec = "{.status.containerStatuses[]|select(.ready==false)|name}"
    │
    ▼ isJQSpec(spec) → Split by "|" → 3段 > 2 → 是 JQ!
    │
    ▼ realize() 预处理: parser.Parse(spec)
  → JSONPath 解析此 spec 很可能失败
  → isJQSpec=true → 不告警，parser 仍非 nil
    │
    ▼ hydrate() 运行时:
  jqParse(spec, unstructuredContent)
    exp = spec[1:len-1] = ".status.containerStatuses[]|select(.ready==false)|name"
    gojq.Parse(exp) → 成功
    gojq.Run(o) → 正确求值 → 返回 JQ 结果
```

**但如果用户只写了 1 个管道符 + 非法尾段**：

```
用户输入: "MY_COL:.spec.foo|bar"
    │
    ▼ parse() 正则匹配（尾锚定约束导致回退）
  → mm[1]="MY_COL", mm[2]=".spec.foo|bar", mm[3]=""
  → 'b' ∉ [NTWSLRH]，组3只能匹配 0 次（空串），整段 |bar 回溯进表达式段
  → newColFlags("")：无标志字符，不打 Warn
  → spec = "{.spec.foo|bar}"
    │
    ▼ isJQSpec("{.spec.foo|bar}")
    Split("{.spec.foo|bar}", "|") = ["{.spec.foo", "bar}"] → 2段
    → 2 > 2? false → 不是 JQ
    │
    ▼ 走 JSONPath 路径（但 parser.Parse("{.spec.foo|bar}") 大概率语法错误）
```

这种情况下管道符回退进入表达式段，spec 内含 1 个管道符，但由于 isJQSpec 要求 `Split 段数 > 2`（即至少 2 个管道符），所以 JQ 判定仍为 false，最终走 JSONPath。

### 13.3.1 单管道的最终统一结论

对任何单管道列定义（管道后无论是否合法 FLAGS）：

- 管道后跟合法 FLAGS(≤3)：管道归分隔符，spec 内无管道 → Split=1段 → 不触发 JQ
- 管道后跟非法尾段：管道回退入表达式段，spec 内含 1 管道 → Split=2段 → 仍不触发 JQ
- 管道后跟合法 FLAGS>3：多余部分+管道回退入表达式段，spec 内含 1 管道 → Split=2段 → 仍不触发 JQ

**JQ 触发的唯一条件：包装后的 spec 内至少包含 2 个管道符（即表达式段含 ≥2 个 `|`），使 Split 结果 ≥ 3 段。**

### 13.4 边界情况 3：JQ 运行时错误不中断

[jqParse()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L271-L300) 中的迭代错误处理：

```go
iter := jq.Run(o)
for v, ok := iter.Next(); ok; v, ok = iter.Next() {
    if e, cool := v.(error); cool && e != nil {
        if errors.Is(e, new(gojq.HaltError)) {
            break               // HaltError → 停止迭代
        }
        slog.Error("JQ expression evaluation failed. Check your query", slogs.Error, e)
        continue                // 其他错误 → 记录日志，继续迭代
    }
    rr = append(rr, fmt.Sprintf("%v", v))
}
```

- **gojq.HaltError**：JQ 的 `halt`/`halt_error` 语句，直接 break
- **其他运行时错误**（如类型不匹配、字段不存在）：slog.Error + continue，继续尝试后续迭代值
- **结果为空**（`len(rr) == 0`）：返回 `("", false)` → 降级走 JSONPath

**这意味着**：JQ 表达式部分结果报错、部分成功时，成功的部分仍然被收集。例如 `.items[] | select(.status == "running") | .name` 中某些 items 缺少 status 字段时，报错的 items 被跳过，其余正常返回。

### 13.5 边界情况 4：JSONPath 预解析失败但 isJQSpec=true 时的静默降级

```go
// realize() L119-L128
parsers[ix] = jsonpath.New(fmt.Sprintf("column%d", ix)).AllowMissingKeys(true)
if err := parsers[ix].Parse(cc[ix].Spec); err != nil && !isJQSpec(cc[ix].Spec) {
    slog.Warn("Unable to parse custom column", ...)
}
```

当 `isJQSpec(spec) == true` 时，即使 JSONPath Parse 失败也**不告警**。此时期望运行时走 JQ 路径，但如果 JQ 也失败（`jqParse` 返回 false），则降级到 JSONPath 的 `FindResults`——此时 **parser 内部状态是 Parse 失败后的残留状态**。

`jsonpath.JSONPath.FindResults()` 在未成功 Parse 的情况下被调用，会返回错误，导致 [hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L210-L212) **整体报错返回**：

```go
if err != nil {
    return nil, err   // ← 整行渲染失败
}
```

**这是唯一会导致 hydrate 整体报错的路径**：JQ 格式判定为 true → JSONPath Parse 静默失败 → JQ 运行时也失败 → 降级 JSONPath FindResults 报错 → hydrate 返回 error → 渲染中断。

### 13.6 边界情况 5：JQ 空结果降级后 JSONPath 有结果

```go
// hydrate() Unstructured 分支
if vals, ok := jqParse(cc[idx].Spec, unstructured.UnstructuredContent()); ok {
    cols[idx] = RenderedCol{Header: cc[idx].Header, Value: vals}
    continue                   // JQ 成功，直接返回
}
vals, err = parser.FindResults(unstructured.UnstructuredContent())  // JQ 失败，降级
```

**场景**：JQ 表达式语法合法但匹配结果为空（如 `select(.nonexistent == "foo")`），`jqParse` 返回 `("", false)`。此时降级到 JSONPath，而 JSONPath 可能因为不同的语义匹配到数据。

**示例**：
```
spec = "{.items[] | .name}"
JQ:  .items[] | .name  → 如果 .items 不存在 → 空结果 → return "", false
JSONPath: {.items[] | .name} → 可能解析失败或返回不同结果
```

这种降级不是等价替换，两种语法的语义差异可能导致**结果不一致**。

### 13.7 结构化对象不经过 JQ 路径

[hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L198-L209) 中，非 Unstructured 对象直接走 JSONPath 反射求值：

```go
} else {
    rv := reflect.ValueOf(o)
    if !rv.IsValid() || (rv.Kind() == reflect.Ptr && rv.IsNil()) {
        cols[idx] = RenderedCol{Header: cc[idx].Header, Value: NAValue}
        continue
    }
    vals, err = parser.FindResults(rv.Elem().Interface())
}
```

**JQ 只对 Unstructured 对象生效**。对于 Pod、Deployment 等传 `DeepCopy()` 后的类型化对象，`o.(runtime.Unstructured)` 断言失败，直接走 JSONPath 反射。但由于 [Pod.Render()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/pod.go#L143) 传的是 `pwm.Raw.DeepCopy()`（`*unstructured.Unstructured`），实际仍走 JQ 路径。

而 [Service.Render()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/svc.go#L55) 传的是 `raw`（`*unstructured.Unstructured`），同样走 JQ 路径。

**Table 渲染器**的 [realize() 调用](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/table.go#L99-L103) 传入 `row.Object.Object`，类型为 `runtime.Object`，可能是 Unstructured 也可能不是：
```go
obj := row.Object.Object
if obj != nil {
    obj = obj.DeepCopyObject()
}
cols, err := t.specs.realize(obj, t.defaultHeader(), r)
```

当 `obj == nil` 时，hydrate 中会走 `o == nil` 分支，返回 `NAValue`。

### 13.8 JQ 与 JSONPath 降级全景决策树

```
有 Spec 的列 (parser ≠ nil)
    │
    ├─ o == nil → NAValue
    │
    ├─ o 是 runtime.Unstructured?
    │     │
    │     ├─ jqParse(spec, UnstructuredContent())
    │     │     │
    │     │     ├─ isJQSpec=false → return "", false → 降级 JSONPath
    │     │     │
    │     │     ├─ gojq.Parse 失败 → Warn + return "", false → 降级 JSONPath
    │     │     │
    │     │     ├─ gojq.Run 有结果 → return (result, true) → 完成 ✓
    │     │     │
    │     │     └─ gojq.Run 无结果 → return "", false → 降级 JSONPath
    │     │
    │     └─ JQ 降级 → parser.FindResults(map)
    │           │
    │           ├─ 成功 → 处理结果（可能 MissingValue）
    │           └─ 失败 → hydrate 返回 error ✗
    │
    └─ o 非 Unstructured → parser.FindResults(reflect)
          │
          ├─ rv 无效/nil → NAValue
          ├─ 成功 → 处理结果
          └─ 失败 → hydrate 返回 error ✗
```

---

## 十四、校准：默认列回补取字段值时的越界保护缺失

### 14.1 代码事实

[realize()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L134-L143) 阶段二（默认列回补）中取字段值的代码：

```go
for _, hc := range rh {
    if vv.HasHeader(hc.Name) {
        continue
    }
    if idx, ok := rh.IndexOf(hc.Name, true); ok {
        rc := RenderedCol{Header: hc, Value: row.Fields[idx]}  // ← 直接取，无越界检查
        rc.Header.Wide = true
        vv = append(vv, rc)
    }
}
```

对比 [hydrate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L165-L170) 中 `parser == nil` 分支取字段值的代码：

```go
var v string
if ix >= len(row.Fields) {   // ← 有越界保护
    v = NAValue
} else {
    v = row.Fields[ix]
}
```

**校准结论**：hydrate 有越界保护，realize 阶段二**没有**。

### 14.2 实际风险分析

阶段二的 `idx` 来自 `rh.IndexOf(hc.Name, true)`，而 `hc` 本身就是 `rh` 的遍历元素，所以 `idx` 一定在 `[0, len(rh))` 范围内。

理论上 `row.Fields` 的长度应等于 `len(rh)`（因为 `defaultRow()` 按 `rh` 顺序填充），但以下场景可能导致不等：

| 场景 | rh 长度 vs row.Fields 长度 | 是否会越界 |
|------|---------------------------|-----------|
| 内置渲染器（Pod, Service 等） | 严格一致 | 否 |
| Table/Generic 渲染器 | `defaultHeader()` 来自 ServerSideTable 列定义，`defaultRow()` 来自 `row.Cells`，Cells 可能少于 ColumnDefinitions | **可能** |
| CRD 资源 | 同上，动态列定义 | **可能** |

在 [Table.defaultRow()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/table.go#L112-L163) 中：

```go
r.Fields = make(model1.Fields, 0, len(th))
for i, c := range row.Cells {
    // ...逐个 append，长度 = len(row.Cells)
}
```

当 `row.Cells` 少于 `t.table.ColumnDefinitions`（即 `th` 的来源）时，`r.Fields` 长度 < `len(th)`。此时阶段二的 `row.Fields[idx]` 可能越界，触发 **panic: runtime error: index out of range**。

### 14.3 修正建议（未实施，仅供理解）

若要修复此问题，阶段二应增加与 hydrate 相同的越界检查：

```go
var v string
if idx >= len(row.Fields) {
    v = NAValue
} else {
    v = row.Fields[idx]
}
rc := RenderedCol{Header: hc, Value: v}
```

---

## 十五、校准：单管道列定义的正则分组归属

### 15.1 正则重新审视

[fullRX](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_col.go#L17)：

```regex
^([\w\s%/-]+):?([\w\W]*?)\|?([NTWSLRH]{0,3})$
```

关键点：
- 组2 `([\w\W]*?)` 是**非贪婪**匹配（`*?`），消费越少越优先
- 组3 `([NTWSLRH]{0,3})$` 必须**恰好匹配到行尾**（`$` 锚定 + 字符集严格限定）
- `\|?` 是可选的管道分隔符

**分组成败判定的核心约束**：组3 `[NTWSLRH]{0,3}$` 的**尾锚定不可违反**。即：第三组捕获的内容必须正好是字符串的最后 0~3 个字符，且每个字符都 ∈ `{N,T,W,S,L,R,H}`。只要管道后出现任何非合法标志字符（或合法标志超过 3 个），引擎就必须让组2回溯扩展，把多余部分纳入表达式段。

### 15.2 逐场景推演

**情况 A：`"fred|W"`**（无冒号，单管道，后跟合法 FLAGS 且 ≤3 字符）

```
正则匹配过程:
  尝试最小消费: mm[1]="fred", mm[2]="", \|?="|", mm[3]="W"
    → mm[3]="W" ∈ [NTWSLRH], 长度 1 ≤ 3, 后面正好是 $ → 匹配成功
  mm[1] = "fred"
  mm[2] = ""
  mm[3] = "W"

结论: |W 落入标志段 ✓
```

测试用例 `"plain-wide"` 证实。

**情况 B：`"fred:.metadata.name|W"`**（有冒号，单管道，后跟合法 FLAGS 且 ≤3 字符）

```
正则匹配过程:
  冒号消费后剩余 ".metadata.name|W"
  组2非贪婪开始尝试空值 → 尾部剩余 ".metadata.name|W"
  mm[3] 至少需要消费到字符串末尾且全为合法字符 → 无法满足，回溯
  组2逐步扩展直到剩余 "|W"
  此时 \|?="|", mm[3]="W" → 尾锚定满足
  mm[1] = "fred"
  mm[2] = ".metadata.name"
  mm[3] = "W"

结论: |W 落入标志段 ✓
```

测试用例 `"partial-no-type-wide"` 证实。

**情况 C：`"fred|T"`**（无冒号，单管道，后跟合法 FLAGS）

```
mm[1] = "fred"
mm[2] = ""
mm[3] = "T"

结论: |T 落入标志段 ✓
```

测试用例 `"partial-no-spec-no-wide"` 证实。

**情况 D：`"fred:.spec.foo|bar"`**（有冒号，单管道，后跟非法 FLAGS 尾段）

```
正则匹配过程 —— 关键在于尾锚定:
  冒号消费后剩余 ".spec.foo|bar"
  尝试组2=".spec.foo", \|?="|", mm[3]="bar"
    → 'b' ∉ [NTWSLRH] → 第三组无法匹配（{0,3} 必须匹配到末尾，
                        mm[3] 只能接受全合法字符，否则只能匹配 0 次，
                        但 "bar" 还没到末尾，0次匹配也不行）
    → 尾锚定 + 字符集双重约束失败，必须回溯
  组2扩展为 ".spec.foo|b" → 尾部剩 "ar" → 'a' 非合法字符 → 回溯
  组2扩展为 ".spec.foo|ba" → 尾部剩 "r" → 'r' 非合法字符 → 回溯
  组2扩展为 ".spec.foo|bar" → 尾部剩 ""（空）
    \|? 消费 ""（无管道），mm[3] = ""（匹配 0 次）
    空字符串正好到 $ → 尾锚定满足 ✓
  mm[1] = "fred"
  mm[2] = ".spec.foo|bar"     ← 关键：管道和 bar 全部回溯进入表达式段
  mm[3] = ""                  ← 标志段为空

结论: |bar 回退进入表达式段，mm[3]="", newColFlags("") 不做任何处理
```

> **之前的错误结论校准**：不能认为 `mm[3]="bar"`，因为 `'b'` 不在 `[NTWSLRH]` 中，组3的字符集约束不允许。真正行为是 **mm[3] 只能取 0 次匹配（空串），整段 `|bar` 回溯进组2**。

**情况 E：`"fred||.metadata.name|W"`**（双管道，有冒号，末尾合法 FLAGS）

```
正则匹配过程:
  冒号消费后剩余 "||.metadata.name|W"
  尝试组2="", \|?="|", mm[3] 需消费 "|.metadata.name|W" 到末尾 → 首字符 | 非合法 → 回溯
  ...
  组2扩展为 "||.metadata.name" → 剩余 "|W"
    \|?="|", mm[3]="W" → 尾锚定 + 字符集满足 ✓
  mm[1] = "fred"
  mm[2] = "||.metadata.name"
  mm[3] = "W"

结论: 前两个 | 留在表达式段，最后一个 |W 为标志段 ✓
```

测试用例 `"toast"` 证实：`spec: "{.||.metadata.name}"`，`wide: true`。

**情况 F：`"fred:.xx|NWLR"`**（有冒号，单管道，合法 FLAGS 超过 3 字符）

```
正则匹配过程:
  冒号消费后剩余 ".xx|NWLR"
  尝试组2=".xx", \|?="|", mm[3]="NWLR"
    N/W/L/R 全合法，但长度 4 > {0,3}上限 → 回溯
  组2扩展为 ".xx|N" → 剩余 "WLR"
    \|? 消费 ""（因为剩余首字符是 W 不是 |），mm[3]="WLR" → 3 字符全合法，正好到末尾 ✓
  mm[1] = "fred"
  mm[2] = ".xx|N"     ← 第一个 | 和多余的 N 被切进表达式段
  mm[3] = "WLR"       ← 标志段取最后 3 个合法字符

结论: 合法 FLAGS 超过 3 个时，多余部分被回溯进表达式段
```

### 15.3 核心规则总结

| 模式 | 管道归属 | mm[2] 表达式段 | mm[3] FLAGS 段 | 判定依据 |
|------|---------|--------------|----------------|---------|
| 无冒号 + 单管道 + 合法FLAGS(≤3) `"fred\|W"` | 管道归分隔符 | `""` | `"W"` | 末尾 "W" 正好匹配组3 |
| 有冒号 + 单管道 + 合法FLAGS(≤3) `"fred:.xx\|W"` | 管道归分隔符 | `".xx"` | `"W"` | 末尾 "W" 正好匹配组3 |
| 无冒号 + 单管道 + 非法尾段 `"fred\|bar"` | **管道+bar 全归表达式段** | `"\|bar"` | `""` | `'b'` 不合法，组3只能 0 次，整体回溯 |
| 有冒号 + 单管道 + 非法尾段 `"fred:.xx\|bar"` | **管道+bar 全归表达式段** | `".xx\|bar"` | `""` | 同上 |
| 有冒号 + 合法FLAGS>3 `"fred:.xx\|NWLR"` | **第一个\|+多余字符归表达式** | `".xx\|N"` | `"WLR"` | 组3最多 3 字符，长度 4 触发回溯 |
| 有冒号 + 多管道 + 末尾合法 `"fred:\|a\|b\|W"` | 最后一个\|为分隔符 | `"\|a\|b"` | `"W"` | 末尾 "W" 匹配组3 |
| 无冒号 + 无管道 `"fred"` | 无 | `""` | `""` | 无管道分隔 |

**通用判定规则（精确版）**：

从字符串末尾向前看，设末尾连续合法标志字符数为 `k`（即 `N/T/W/S/L/R/H` 连续个数，从右往左数，遇到非法字符停止）：

1. **若 `0 < k ≤ 3`，且倒数第 `k+1` 个字符是 `|`**：
   - 该 `|` 是分隔符
   - mm[3] = 末尾 `k` 个字符
   - mm[2] = 剩下的前缀（不含最后那个 `|`）

2. **若 `k = 0`（末尾字符非合法标志）或 倒数第 `k+1` 个字符不是 `|`**：
   - mm[3] = `""`（0 次匹配）
   - 所有管道全部留在 mm[2]（表达式段）

---

## 十六、校准：边界条件对 JQ/JSONPath 判断的实际影响

### 16.1 正则分组 → Spec → isJQSpec 的完整链路

JQ 判定不在正则阶段，而在 [isJQSpec()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-k9s/internal/render/cust_cols.go#L267-L269) 阶段，操作对象是**经过 `RelaxedJSONPathExpression` 包装后的 spec 字符串**：

```
用户输入 → parse() 正则分组 → RelaxedJSONPathExpression(mm[2]) → cc[idx].Spec
                                                                      │
                                                                      ▼
                                                              isJQSpec(cc[idx].Spec)
```

`RelaxedJSONPathExpression` 的包装规则（来自 `kubectl` 包）：
- `.metadata.name` → `{.metadata.name}`（加花括号）
- `{.metadata.name}` → `{.metadata.name}`（已有花括号不变）
- `:.metadata.name` → 报错（以冒号开头的非合法表达式）

### 16.2 各种输入的完整推演

| 用户输入 | mm[2] | Spec (包装后) | isJQSpec? | 实际路径 |
|---------|-------|-------------|-----------|---------|
| `"NAME"` | `""` | `""` | N/A (Spec="", parser=nil) | 默认列引用 |
| `"NAME\|W"` | `""` | `""` | N/A (Spec="", parser=nil) | 默认列引用 |
| `"IP:.status.hostIP"` | `".status.hostIP"` | `"{.status.hostIP}"` | Split=1段 → false | JSONPath |
| `"IP:.status.hostIP\|W"` | `".status.hostIP"` | `"{.status.hostIP}"` | Split=1段 → false | JSONPath |
| `"X:.spec\|bar"` | `".spec\|bar"` | `"{.spec\|bar}"` | Split=2段 → false | JSONPath (但 RelaxedJSONPath 可能拒绝\|bar) |
| `"X:\|\|.foo\|W"` | `"\|\|.foo"` | `"{\|\|.foo}"` | Split=3段 → true | JQ 优先 |
| `"X:.items[]\|select(.x)\|name\|W"` | `".items[]\|select(.x)\|name"` | `"{.items[]\|select(.x)\|name}"` | Split=3段 → true | JQ 优先 |
| `"X:.xx\|NWLR"` (合法FLAGS>3) | `".xx\|N"` | `"{.xx\|N}"` | Split=2段 → false | JSONPath (但 RelaxedJSONPath 可能拒绝\|N) |

### 16.3 单管道列定义的 JQ 判定分情况讨论

**校准（推翻之前"永远不能"的绝对结论）**：单管道是否触发 JQ，取决于那个管道最终落入哪个分组，以及落入表达式段后 spec 内总管道数是否 >2。

| 单管道场景 | mm[2] 内容 | 包装后 spec | 管道数判断 | isJQSpec 结果 |
|-----------|-----------|------------|-----------|--------------|
| 单管道 + 后跟合法FLAGS(≤3) `"fred:.xx\|W"` | `".xx"` | `"{.xx}"` | 0 个管道 → 1段 | false → JSONPath |
| 单管道 + 后跟非法尾段 `"fred:.spec\|bar"` | `".spec\|bar"` | `"{.spec\|bar}"` | 1 个管道 → 2段 | false（>2才是 true）→ JSONPath |
| 单管道 + 后跟合法FLAGS>3 `"fred:.xx\|NWLR"` | `".xx\|N"` | `"{.xx\|N}"` | 1 个管道 → 2段 | false → JSONPath |
| 多管道 + 末尾合法 `"fred:\|a\|b\|W"` | `"\|a\|b"` | `"{.\|a\|b}"` | 2 个管道 → 3段 | true → JQ 优先 |

**关键结论**：
1. **单管道无论什么情况都不会触发 JQ**——因为即使管道回退入表达式段，spec 内也只有 1 个管道，`Split(spec, "|")` 得到 2 段，不满足 `>2` 的 JQ 判定阈值。
2. JQ 触发的必要条件：**包装后的 spec 内至少包含 2 个管道符**（即表达式段 mm[2] 含 ≥2 个 `|`），使得 `Split` 结果 ≥ 3 段。
3. 用户输入中想要启用 JQ，表达式部分必须显式写 ≥2 个管道，例如 JQ 管道链 `|filter|transform`。

### 16.4 测试用例 "toast" 的 JQ 判定校准

测试用例 `"fred||.metadata.name|W"` 的完整链路：

```
正则: mm[1]="fred", mm[2]="||.metadata.name", mm[3]="W"
RelaxedJSONPathExpression("||.metadata.name")
  → spec = "{.||.metadata.name}"
isJQSpec("{.||.metadata.name}")
  → Split by "|" → ["{.", "", ".metadata.name}"] → 3段 > 2 → true
```

**校准结论**：此 spec 会被判定为 JQ（因为 2 个管道 → 3段），但 `{.||.metadata.name}` 作为 JQ 表达式（去掉首尾花括号后为 `.||.metadata.name`）语法是无效的。`gojq.Parse` 会失败，`jqParse` 返回 false，降级到 JSONPath，而 JSONPath parser 在 realize() 预处理时已 Parse 此 spec（`{.||.metadata.name}`），大概率也失败。由于 `isJQSpec=true`，Parse 失败不告警。最终运行时 JQ 失败 + JSONPath FindResults 也失败 → hydrate 返回 error。

### 16.5 "toast-no-name" 匹配失败的根因

测试用例 `":.metadata.name.fred|TW"` → 正则匹配**失败**（`len(mm) != 4`）。

原因：组1 `([\w\s%/-]+)` 要求至少一个字符，而冒号前为空字符串，`+` 量词不满足，整个正则不匹配。**这无关管道归属，而是名称段为空导致的整体匹配失败**。

### 16.6 单管道 + 非法尾段场景的完整路径推演

用户输入 `"X:.spec.foo|bar"`（单管道 + 非法尾段 `bar`）的全链路：

```
① 正则分组（尾锚定约束导致回退）:
   mm[1] = "X"
   mm[2] = ".spec.foo|bar"   ← |bar 回退进入表达式段
   mm[3] = ""                ← 标志段空

② newColFlags(""): 无标志字符，不做任何处理（default 分支也触发不了）

③ RelaxedJSONPathExpression(".spec.foo|bar"):
   kubectl 函数的行为是把形如 ".xxx" 的字符串包装成 "{.xxx}"，
   不会检查内部是否含非法字符（| 对 JSONPath 语法是非法的）
   → spec = "{.spec.foo|bar}"

④ isJQSpec("{.spec.foo|bar}"):
   Split("{.spec.foo|bar}", "|") = ["{.spec.foo", "bar}"] → 长度2
   → 2 > 2? false → isJQSpec = false
   结论: 单管道 + 非法尾段 → 仍然走 JSONPath 路径，不触发 JQ

⑤ realize() 预处理 parser.Parse("{.spec.foo|bar}"):
   isJQSpec=false，Parse 若失败会打 slog.Warn，但 parser 仍非 nil

⑥ hydrate() 运行时：
   parser.FindResults(o) → JSONPath 语法不支持 "|bar" → 报错返回
   → hydrate 返回 error，整行渲染失败
```

**核心校准**：单管道 + 非法尾段虽然让管道进入 spec，但因为只贡献 1 个管道，isJQSpec 判定为 false，**不会触发 JQ**。最终走向是 JSONPath 语法报错而非 JQ 求值。
