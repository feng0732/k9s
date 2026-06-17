# Pulses 健康指示链路分析

## 一、整体架构

Pulses 健康指示系统采用四层架构，从数据采集到界面展示的分层结构：

```
┌─────────────────────────────────────────────────────────────────┐
│                  界面展示层 (View)                               │
│  internal/view/pulse.go  Pulse 视图组件                        │
│  ├─ Gauge 仪表盘：资源健康状态 (OK/Fault 计数)                 │
│  └─ SparkLine 折线图：CPU/内存使用时间序列                      │
├─────────────────────────────────────────────────────────────────┤
│                  数据模型层 (Model)                              │
│  internal/model/pulse.go        Pulse 模型 (Watch 分发)        │
│  internal/model/pulse_health.go PulseHealth 健康检查器          │
├─────────────────────────────────────────────────────────────────┤
│                  健康检查层 (Render)                             │
│  internal/render/*.go   各资源 Healthy() 方法                  │
│  internal/health/check.go  Check 数据结构                      │
├─────────────────────────────────────────────────────────────────┤
│                  数据采集层 (DAO)                                │
│  internal/dao/recorder.go  Recorder 指标采集器                 │
│  internal/dao/factory.go   Factory 资源访问工厂                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、健康检查入口：真实调用链路

### 2.1 PulseListener 接口声明但未在 view.Pulse 中使用

[internal/model/pulse.go#L14-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse.go#L14-L23) 定义了 `PulseListener` 接口：

```go
type PulseListener interface {
    PulseChanged(*health.Check)  // 参数类型: *health.Check
    PulseFailed(error)
    MetricsChanged(dao.TimeSeries)
}
```

**关键事实**：`view.Pulse` **没有实现该接口**，也没有通过 `AddListener` 注册。原因：
- `view.Pulse.PulseChanged` 签名是 `PulseChanged(pt model.HealthPoint)`，参数是 `HealthPoint` 而非接口要求的 `*health.Check`
- `internal/model/pulse_health.go#L19-L22` 中传输的 `HealthPoint` 与 `internal/health/check.go#L13-L17` 中 `health.Check` 是**两个独立的数据结构**

### 2.2 实际使用的是 channel 直连模式

[internal/view/pulse.go#L308-L339](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L308-L339) 中 `Start()` 方法直接消费两个独立的 channel：

```
view.Pulse.Start()                          internal/view/pulse.go#L308
    │
    ├── model.Watch(ctx)                     internal/model/pulse.go#L43
    │       │
    │       ├── PulseHealth.Watch(ctx, ns)   internal/model/pulse_health.go#L76
    │       │       返回 HealthChan (chan HealthPoint, cap=2)
    │       │
    │       └── Recorder.Watch(ctx, ns)      internal/dao/recorder.go#L104
    │               返回 MetricsChan (chan TimeSeries, cap=2)
    │
    └── 启动 goroutine 用 select 消费两个 channel：
            ├── gaugeChan   → PulseChanged(HealthPoint)  internal/view/pulse.go#L245
            └── metricsChan → SeriesChanged(TimeSeries)  internal/view/pulse.go#L180
```

### 2.3 健康检查调度：立即执行 + 10秒轮询

[internal/model/pulse_health.go#L76-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L76-L98)：

```go
const pulseRate = 10 * time.Second

func (h *PulseHealth) Watch(ctx context.Context, ns string) HealthChan {
    c := make(HealthChan, 2)
    ctx = context.WithValue(ctx, internal.KeyWithMetrics, false)  // 关闭 metrics 减少开销
    go func() {
        h.checkPulse(ctx, ns, c)  // 立即执行一次
        for {
            select {
            case <-ctx.Done(): close(c); return
            case <-time.After(pulseRate): h.checkPulse(ctx, ns, c)  // 每 10s
            }
        }
    }()
    return c
}
```

### 2.4 单资源检查：Registry 回退与 isTable 分支

[internal/model/pulse_health.go#L112-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L112-L148)：

```go
func (h *PulseHealth) check(ctx context.Context, ns string, gvr *client.GVR) (HealthPoint, error) {
    meta, ok := Registry[gvr]
    if !ok {
        // 未注册 GVR 回退到通用 Table DAO + Table Renderer
        // Table Renderer 的 Healthy 继承 Base → 永远返回 nil (视为健康)
        meta = ResourceMeta{DAO: new(dao.Table), Renderer: new(render.Table)}
    }
    oo, _ := meta.DAO.List(ctx, ns)
    c := HealthPoint{GVR: gvr, Total: len(oo)}

    if isTable(oo) {
        // 恰好返回1个 metav1.Table 对象时，Total 取 len(ta.Rows) 覆盖
        for _, row := range ta.Rows {
            if meta.Renderer.Healthy(ctx, row) != nil { c.Faults++ }
        }
    } else {
        // 普通资源：Healthy 逐个判定，error → Faults++
        for _, o := range oo {
            if meta.Renderer.Healthy(ctx, o) != nil { c.Faults++ }
        }
    }
    return c, nil
}
```

### 2.5 各资源 Healthy() 实现一览

| 资源 | 文件与行号 | 不健康判定逻辑 |
|------|-----------|---------------|
| **Pod** | [internal/render/pod.go#L214-L253](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L214-L253) | 容器就绪数≠总数 / ReadinessGate 未全满足 / Pod Ready 条件为 False / 未 Completed 但 Phase 异常 |
| **Node** | [internal/render/node.go#L171-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L171-L218) | Conditions 中没有 Ready=True / SchedulingDisabled 被封锁 |
| **Namespace** | [internal/render/ns.go#L100-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ns.go#L100-L122) | Phase 既非 Active 也非 Terminating |
| **Event** | [internal/render/ev.go#L20-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ev.go#L20-L32) | Cells[2] (类型列) 不等于 "Normal"（即 Warning 及以上） |
| **Helm Chart** | [internal/render/helm/chart.go#L75-L90](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/helm/chart.go#L75-L90) | Release Status 不是 "deployed" |
| **默认(其他)** | [internal/render/base.go#L69-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/base.go#L69-L72) | 永远返回 nil（永远健康） |

> **备注**：`PulseGVRs` 共 16 种资源，但只有上表 5 种实现了非默认 Healthy()，其余 11 种（Service/Deployment/StatefulSet/DaemonSet/Job/CronJob/PV/PVC/HPA/Ingress/NetworkPolicy/SA）健康检查恒通过。

---

## 三、指标窗口（时间序列）实现

### 3.1 采集参数与缓存策略

[internal/dao/recorder.go#L23-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L23-L29)：

| 参数 | 值 | 含义 |
|------|-----|------|
| `seriesCacheSize` | 600 | LRU 缓存最大条目数 |
| `seriesCacheExpiry` | 3h | 单条缓存的过期时间 |
| `seriesRecordRate` | 1min | 指标采集间隔 |

### 3.2 两条采集路径：Node 级别 vs Pod 级别

[internal/dao/recorder.go#L104-L139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L104-L139)：

- **全命名空间模式** `IsAllNamespaces(ns)` → `recordNodeMetrics()`：聚合所有 Node 的 CPU/MEM，Node 的 Allocatable 是真实的节点容量
- **特定命名空间模式** → `recordPodMetrics()`：累加该命名空间下所有 Pod 的所有容器 Usage

**Pod 模式的特殊处理** [internal/dao/recorder.go#L262-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L262-L263)：

```go
if len(pp.Items) > 0 {
    pt.Value.AllocatableCPU = pt.Value.CurrentCPU   // 注意：Allocatable = Current
    pt.Value.AllocatableMEM = pt.Value.CurrentMEM   // 即百分比永远接近 100%
}
```

该处理导致 Pod 模式下 CPU/MEM 百分比永远触发最高阈值等级。

### 3.3 初始加载：dispatchSeries 推送最近1小时历史

[internal/dao/recorder.go#L75-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L75-L102)：

`Watch()` 启动后先从 LRU 缓存中筛选 `时间 > now - 1h` 且 type/namespace 匹配的条目，一次性推送到 channel，确保 UI 打开后立即有历史数据展示。

### 3.4 SparkLine 窗口裁剪：大小取决于终端宽度

裁剪发生在每次 `Draw()` 调用时 [internal/tchart/sparkline.go#L118-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L118-L128)：

```go
func (s *SparkLine) Draw(screen tcell.Screen) {
    s.Component.Draw(screen)
    rect := s.asRect()
    // ...
    s.cutSet(rect.Dx() - padX)  // width = 矩形宽度 - 1 列 padding
}
```

[internal/tchart/sparkline.go#L176-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L176-L183)：

```go
func (s *SparkLine) cutSet(width int) {
    if len(s.series) > width {
        s.series.Truncate(width)  // 保留最新的 width 个数据点
    }
}
```

[internal/tchart/series.go#L68-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/series.go#L68-L77) `Truncate` 实现：

```go
func (mm MetricSeries) Truncate(size int) {
    kk := mm.Keys()                     // 按时间升序排序
    kk = kk[0 : len(kk)-size]           // 取出要删除的旧 key
    for t := range mm {
        if kk.Includes(t) { continue }  // 跳过保留的 key
        delete(mm, kk[0])               // 删除旧数据
    }
}
```

> **窗口大小 = 终端可用宽度 (列数)**：不是固定时间窗口，而是动态适应终端宽度。例如 80 列终端保留约 80 个数据点。

### 3.5 SetMax 只增不减

[internal/tchart/sparkline.go#L65-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L65-L69)：

```go
func (s *SparkLine) SetMax(m float64) {
    if m > s.max {
        s.max = m  // 只增不减，避免图表上下跳动
    }
}
```

缩放因子 `scale = float64(len(sparks)*(rect.Dy()-pad)) / float64(s.max)`，Y 轴高度只扩大不缩小。

---

## 四、阈值判断：真实生效路径

### 4.1 阈值配置与分级

[internal/config/threshold.go#L78-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L78-L104)：

```go
type Severity struct { Critical int = 90; Warn int = 70 }

func (t Threshold) LevelFor(k string, v int) SeverityLevel {
    // v < 70  → Low(0)
    // v ≥ 70  → Medium(1)
    // v ≥ 90  → High(2)
}

func (t *Threshold) SeverityColor(k string, v int) string {
    // Low    → "green"
    // Medium → "orangered"
    // High   → "red"
}
```

key 取值来自 [internal/config/types.go#L10-L14](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/types.go#L10-L14)：`CPU = "cpu"`，`MEM = "memory"`。

### 4.2 阈值只在 SparkLine 中生效，Gauge 完全不影响

**Gauge 的 SetColorIndex 是空实现** [internal/tchart/gauge.go#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L55)：

```go
func (*Gauge) SetColorIndex(int) {}  // 空实现，阈值对 Gauge 无任何作用
```

**SparkLine 的 SetColorIndex 正常工作** [internal/tchart/sparkline.go#L61-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L61-L63)：

```go
func (s *SparkLine) SetColorIndex(i int) { s.colorIndex = i }
```

### 4.3 阈值的三处生效路径（全部在 SeriesChanged 中）

[internal/view/pulse.go#L180-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L180-L242)：

```go
func (p *Pulse) SeriesChanged(tt dao.TimeSeries) {
    // ...遍历 tt 更新 series 和 max...
    last := tt[len(tt)-1]
    perc := client.ToPercentage(last.Value.CurrentCPU, int64(cpu.GetMax()))

    // 路径1：LevelFor → colorIndex → SparkLine.Draw() 折线柱体颜色
    index := int(p.app.Config.K9s.Thresholds.LevelFor("cpu", perc))
    cpu.SetColorIndex(index)

    nn := cpu.GetSeriesColorNames()  // 返回全新创建的 slice（见 5.2）
    // ...零值灰化见第五节...

    // 路径2：SeverityColor → legend 中百分比文字的颜色
    // 路径3：nn[index] → legend 中数值 (1200m/4000m) 的颜色名
    cpu.SetLegend(fmt.Sprintf(cpuFmt,
        "Cpu",
        p.app.Config.K9s.Thresholds.SeverityColor("cpu", perc),  // 路径2
        render.PrintPerc(perc),
        nn[index],  // 路径3
        render.AsThousands(last.Value.CurrentCPU),
        "white",
        render.AsThousands(int64(cpu.GetMax())),
    ))
}
```

**路径1 — 折线柱体颜色** [internal/tchart/sparkline.go#L141-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L141-L147)：

```go
colors := s.colorForSeries()  // 读取 Component.seriesColors
for _, t := range s.series.Keys() {
    b := s.makeBlock(...)
    s.drawBlock(rect, screen, cX, cY, b, colors[s.colorIndex%len(colors)])
}
```

所有柱体使用**同一种颜色** `colors[colorIndex%3]`，colorIndex 来自阈值的 0/1/2 映射。

---

## 五、零值显示：真实生效 vs 未进入渲染

### 5.1 GetSeriesColorNames() 返回全新 slice

[internal/tchart/component.go#L98-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L98-L115)：

```go
func (c *Component) GetSeriesColorNames() []string {
    c.mx.RLock()
    defer c.mx.RUnlock()

    if len(c.seriesColors) < 3 {
        return []string{"green", "orange", "red"}  // 情况A：默认 slice
    }
    nn := make([]string, 0, len(c.seriesColors))   // 情况B：make 全新 slice
    for _, color := range c.seriesColors {
        // 反向查找 tcell.ColorNames 中的名称
    }
    return nn  // 返回的是副本，修改不影响 c.seriesColors
}
```

**关键事实**：返回值是通过 `make` + `append` 创建的**独立副本**，修改返回切片中的字符串**不会**改变 `Component.seriesColors` 的值。`colorForSeries()` 直接返回 `c.seriesColors` 引用，Draw 中取色走的是这条路径，与 `GetSeriesColorNames()` 的返回值**完全无关**。

### 5.2 Gauge：PulseChanged 中的 nn[] 修改未进入渲染

[internal/view/pulse.go#L245-L266](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L245-L266)：

```go
func (p *Pulse) PulseChanged(pt model.HealthPoint) {
    v, _ := p.charts[pt.GVR]

    nn := v.GetSeriesColorNames()  // 返回独立副本
    if pt.Total == 0 { nn[0] = "gray" }   // 修改副本 → 不影响 seriesColors
    if pt.Faults == 0 { nn[1] = "gray" }  // 修改副本 → 不影响 seriesColors

    v.SetLegend(cases.Title(language.English).String(pt.GVR.R()))  // legend 只用 GVR 名，nn 完全没传入
    // ...边框颜色...
    v.Add(pt.Total, pt.Faults)
}
```

**结论**：这 4 行代码（L251-L257）的 `nn[]` 修改是**死代码**：
1. 修改的是副本，不改变 `Component.seriesColors`
2. `nn` 变量后续完全没有被使用（没有传入 `SetLegend`）
3. `Gauge.SetColorIndex` 是空实现，阈值系统也无法影响 Gauge 颜色

### 5.3 Gauge 真正的零值显示在 Draw() → drawNum() 中

[internal/tchart/gauge.go#L78-L139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L78-L139)：

```go
func (g *Gauge) Draw(sc tcell.Screen) {
    // ...
    colors := g.colorForSeries()  // 真实读取 Component.seriesColors
    g.drawNum(sc, o,
        number{ok: true, val: g.state.OK, delta: g.deltaOK, str: d1},
        style.Foreground(colors[0]).Dim(false))   // 传入基于 seriesColors[0] 的 style
    // ...
    g.drawNum(sc, o,
        number{ok: false, val: g.state.Fault, delta: g.deltaFault, str: d2},
        style.Foreground(colors[1]).Dim(false))   // 传入基于 seriesColors[1] 的 style
}

func (g *Gauge) drawNum(sc tcell.Screen, o image.Point, n number, style tcell.Style) {
    colors := g.colorForSeries()
    if n.ok {
        style = style.Foreground(colors[0])      // OK 数字：覆盖为 seriesColors[0]
        printDelta(sc, n.delta, o, style)
    }

    dm, significant := NewDotMatrix(), n.val == 0
    if significant {                             // ★ 值为 0：整组数字 dimmed
        style = g.dimmed                         // 灰色暗淡样式
    }
    for i := range len(n.str) {
        if n.str[i] == '0' && !significant {     // ★ 前导零：单个数字 dimmed
            g.drawDial(sc, dm.Print(...), o, g.dimmed)
        } else {
            significant = true
            g.drawDial(sc, dm.Print(...), o, style)  // 有效数字：使用正常 style
        }
        o.X += 3
    }
}
```

[internal/tchart/component.go#L35](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L35) `dimmed` 样式：

```go
dimmed: tcell.StyleDefault.
    Background(tview.Styles.PrimitiveBackgroundColor).
    Foreground(tcell.ColorGray).
    Dim(true)  // 灰色 + 暗淡显示
```

**Gauge 零值显示完整判定表**：

| 场景 | 触发条件 | 代码位置 | 效果 |
|------|---------|---------|------|
| 整体灰化 | `n.val == 0`（数值本身就是0） | [internal/tchart/gauge.go#L122-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L122-L125) | 整组数字 `style = g.dimmed`，seriesColors 被完全覆盖 |
| 前导零灰化 | 数值>0，但数字串中有前导0（如 `"0042"` 中的前两个 0） | [internal/tchart/gauge.go#L127-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L127-L128) | 前导零使用 `g.dimmed`，有效数字使用 seriesColors 颜色 |
| 正常显示 | 数值>0，且不是前导零 | [internal/tchart/gauge.go#L129-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L129-L132) | 使用 `style`（OK→seriesColors[0]，Fault→seriesColors[1]） |
| 边框颜色(独立) | `Faults > 0` 或 `Faults == 0` | [internal/view/pulse.go#L260-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L260-L264) | Faults>0 → DarkRed 边框，否则 → DarkOliveGreen 边框 |

### 5.4 SparkLine：SeriesChanged 中的 nn[] 修改仅影响 legend 文字

[internal/view/pulse.go#L206-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L206-L222)：

```go
nn := cpu.GetSeriesColorNames()          // 返回独立副本
if last.Value.CurrentCPU == 0 {
    nn[0] = grayC                         // 修改副本 → 不影响 seriesColors
}
if last.Value.AllocatableCPU == 0 {
    nn[1] = grayC                         // 修改副本 → 不影响 seriesColors
}
// ...
cpu.SetLegend(fmt.Sprintf(cpuFmt,
    "Cpu",
    p.app.Config.K9s.Thresholds.SeverityColor("cpu", perc),
    render.PrintPerc(perc),
    nn[index],  // ★ nn[index] 被传入 cpuFmt 作为 legend 文字颜色名
    // ...
))
```

**SparkLine 灰化判定表**：

| 场景 | 代码位置 | 影响范围 | 是否生效 |
|------|---------|---------|---------|
| legend 数值变灰 | `nn[index]` 传入 cpuFmt | legend 中 `1200m` 文字颜色 | **生效**（通过 tview 颜色标签） |
| 折线柱体颜色 | Draw() 中 `colors[s.colorIndex%len(colors)]` | 折线图柱体 | nn[] 修改 **不生效**（走 seriesColors 原引用） |
| 折线柱体颜色(阈值) | `SetColorIndex(index)` + Draw() | 折线图柱体 | **生效**（colorIndex 取色） |
| legend 百分比颜色 | `SeverityColor()` 返回值 | legend 中 `85%` 文字颜色 | **生效**（通过 tview 颜色标签） |

---

## 六、完整数据流图

```
view.Pulse.Start()                                [internal/view/pulse.go#L308]
    │
    ├── model.Pulse.Watch(ctx)                    [internal/model/pulse.go#L43]
    │       │
    │       ├── PulseHealth.Watch(ctx, ns)         [internal/model/pulse_health.go#L76]
    │       │     ├── KeyWithMetrics=false (关闭metrics)
    │       │     ├── 立即执行一次 checkPulse()
    │       │     └── ticker: 每 10s 轮询
    │       │           │
    │       │           └── 遍历 PulseGVRs(16 种资源)
    │       │                 │
    │       │                 └── check(ctx, ns, gvr)  [internal/model/pulse_health.go#L112]
    │       │                       ├── Registry[gvr] 查找 DAO + Renderer
    │       │                       ├── 未注册 → Table DAO/Renderer (Healthy 恒 nil)
    │       │                       ├── DAO.List() 获取资源
    │       │                       ├── isTable() 选择分支
    │       │                       └── Renderer.Healthy() → Faults++
    │       │
    │       └── Recorder.Watch(ctx, ns)            [internal/dao/recorder.go#L104]
    │             ├── dispatchSeries(最近 1h 历史)  [internal/dao/recorder.go#L75]
    │             └── ticker: 每 1min 采集
    │                   ├── Node 模式(全NS): 聚合节点容量
    │                   └── Pod 模式(特定NS): 累加容器Usage 注意:Allocatable=Current
    │
    └── goroutine select 消费:
            ├── gaugeChan   → QueueUpdateDraw → PulseChanged(HealthPoint)
            │     [internal/view/pulse.go#L245]
            │     ├── nn[] 修改: 死代码(副本+未使用)
            │     ├── SetBorderColor: Faults>0? Red : Green
            │     └── Gauge.Add(Total, Faults)
            │           └── Gauge.Draw()
            │                 ├── colorForSeries() 取 OK/Fault 颜色
            │                 ├── drawNum(): val==0 → 整体 dimmed
            │                 ├── drawNum(): 前导0 → dimmed
            │                 └── tview.Print(legend, 白色居中)
            │
            └── metricsChan → QueueUpdateDraw → SeriesChanged(TimeSeries)
                  [internal/view/pulse.go#L180]
                  ├── SparkLine.SetMax / AddMetric
                  ├── Thresholds.LevelFor → SetColorIndex → 折线柱体颜色
                  ├── nn[] 修改: 副本不影响 seriesColors
                  ├── nn[index] + SeverityColor → legend 文字颜色
                  └── SparkLine.Draw():
                        ├── cutSet(width=终端宽度) → Truncate 旧数据
                        ├── SetMax 只增不减
                        └── colors[colorIndex%3] → 柱体颜色
```

---

## 七、代码索引表（仓库可定位）

### 7.1 健康检查入口

| 功能 | 可定位引用 |
|------|-----------|
| PulseListener 接口定义 | [internal/model/pulse.go#L14-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse.go#L14-L23) |
| model.Pulse.Watch 返回两个 channel | [internal/model/pulse.go#L43-L56](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse.go#L43-L56) |
| view.Pulse.Start 消费 channel | [internal/view/pulse.go#L308-L339](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L308-L339) |
| PulseHealth.Watch 10秒调度 | [internal/model/pulse_health.go#L76-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L76-L98) |
| 单资源 check 主逻辑 | [internal/model/pulse_health.go#L112-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L112-L148) |
| 监控的 16 种 GVR | [internal/model/pulse_health.go#L26-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L26-L46) |
| HealthPoint 数据结构 | [internal/model/pulse_health.go#L19-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L19-L22) |

### 7.2 指标窗口

| 功能 | 可定位引用 |
|------|-----------|
| 采集参数常量 | [internal/dao/recorder.go#L23-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L23-L29) |
| Recorder.Watch 主入口 | [internal/dao/recorder.go#L104-L139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L104-L139) |
| Node 级别采集 | [internal/dao/recorder.go#L148-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L148-L207) |
| Pod 级别采集 Allocatable=Current | [internal/dao/recorder.go#L262-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L262-L263) |
| dispatchSeries 初始加载历史 | [internal/dao/recorder.go#L75-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L75-L102) |
| SparkLine.Draw 调用 cutSet | [internal/tchart/sparkline.go#L118-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L118-L128) |
| cutSet → Truncate | [internal/tchart/sparkline.go#L176-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L176-L183) |
| Series.Truncate 实现 | [internal/tchart/series.go#L68-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/series.go#L68-L77) |
| SetMax 只增不减 | [internal/tchart/sparkline.go#L65-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L65-L69) |

### 7.3 阈值判断

| 功能 | 可定位引用 |
|------|-----------|
| Threshold.LevelFor 三级判断 | [internal/config/threshold.go#L78-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L78-L91) |
| Threshold.SeverityColor 颜色映射 | [internal/config/threshold.go#L94-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L94-L104) |
| CPU/MEM 常量 key | [internal/config/types.go#L10-L14](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/types.go#L10-L14) |
| SeriesChanged 中应用阈值 | [internal/view/pulse.go#L203-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L203-L222) |
| Gauge.SetColorIndex 空实现 | [internal/tchart/gauge.go#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L55) |
| SparkLine.SetColorIndex | [internal/tchart/sparkline.go#L61-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L61-L63) |
| Draw 中 colorIndex 取色画柱体 | [internal/tchart/sparkline.go#L141-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L141-L147) |

### 7.4 零值显示

| 功能 | 可定位引用 |
|------|-----------|
| GetSeriesColorNames 返回独立副本 | [internal/tchart/component.go#L98-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L98-L115) |
| colorForSeries 返回原引用 | [internal/tchart/component.go#L117-L121](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L117-L121) |
| seriesColors 默认值 | [internal/tchart/component.go#L30-L34](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L30-L34) |
| dimmed 样式定义 | [internal/tchart/component.go#L35](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L35) |
| Gauge.PulseChanged nn[] 修改（死代码） | [internal/view/pulse.go#L251-L257](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L251-L257) |
| Gauge.Draw 主流程 | [internal/tchart/gauge.go#L78-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L78-L113) |
| Gauge.drawNum 值为0 → 整体 dimmed | [internal/tchart/gauge.go#L122-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L122-L125) |
| Gauge.drawNum 前导零 → dimmed | [internal/tchart/gauge.go#L127-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L127-L128) |
| Gauge 边框颜色指示 | [internal/view/pulse.go#L260-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L260-L264) |
| SparkLine.SeriesChanged nn[] 修改 | [internal/view/pulse.go#L206-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L206-L212) |

### 7.5 各资源 Healthy 实现

| 资源 | 可定位引用 |
|------|-----------|
| Pod Healthy | [internal/render/pod.go#L214-L253](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L214-L253) |
| Pod diagnose 判定 | [internal/render/pod.go#L242-L253](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L242-L253) |
| Node Healthy | [internal/render/node.go#L171-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L171-L218) |
| Node diagnose 判定 | [internal/render/node.go#L203-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L203-L218) |
| Namespace Healthy | [internal/render/ns.go#L100-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ns.go#L100-L122) |
| Event Healthy | [internal/render/ev.go#L20-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ev.go#L20-L32) |
| Base Healthy 默认实现 | [internal/render/base.go#L69-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/base.go#L69-L72) |
