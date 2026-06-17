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
