# Pulses 健康指示链路分析

## 一、整体架构

Pulses 健康指示系统分为四个层级：

```
┌─────────────────────────────────────────────────────────┐
│                  界面展示层 (View)                       │
│  view/pulse.go  Pulse 视图组件                          │
│  ├─ Gauge 仪表盘 (资源健康状态: OK/Fault 计数)          │
│  └─ SparkLine 折线图 (CPU/内存时序)                     │
├─────────────────────────────────────────────────────────┤
│                  数据模型层 (Model)                      │
│  model/pulse.go        Pulse 模型 (Watch 分发)          │
│  model/pulse_health.go PulseHealth 健康检查器            │
├─────────────────────────────────────────────────────────┤
│                  健康检查层 (Render)                     │
│  render/*.go   各资源 Healthy() 方法                    │
│  health/check.go Check 健康检查数据结构                 │
├─────────────────────────────────────────────────────────┤
│                  数据采集层 (DAO)                        │
│  dao/recorder.go  Recorder 指标采集器                   │
│  dao/factory.go   Factory 资源访问工厂                  │
└─────────────────────────────────────────────────────────┘
```

## 二、健康检查入口：调用链路详解

### 2.1 PulseListener 接口定义与实际调用

[model/pulse.go:14-23](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse.go#L14-L23) 定义了 `PulseListener` 接口：

```go
type PulseListener interface {
    PulseChanged(*health.Check)  // 健康 data 变更
    PulseFailed(error)           // 健康检查失败
    MetricsChanged(dao.TimeSeries) // 指标时间序列变更
}
```

**关键理解**：`PulseListener` 接口虽然定义了，但 `view.Pulse` **并没有通过 `AddListener` 注册**。实际的调用链路是 channel 直连模式，而非观察者模式。

### 2.2 实际调用链路（channel 直连）

[view/pulse.go:308-339](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L308-L339) 中 `Start()` 方法直接消费两个 channel：

```
view.Pulse.Start()                          [view/pulse.go:308]
    │
    ├── model.Watch(ctx)                     [model/pulse.go:43]
    │       │
    │       ├── PulseHealth.Watch(ctx, ns)   [model/pulse_health.go:76]
    │       │       返回 HealthChan           (chan HealthPoint)
    │       │
    │       └── Recorder.Watch(ctx, ns)      [dao/recorder.go:104]
    │               返回 MetricsChan          (chan TimeSeries)
    │
    └── 启动 goroutine 消费两个 channel：
            select {
            case check := <-gaugeChan:       ← HealthPoint
                p.app.QueueUpdateDraw(func() {
                    p.PulseChanged(check)    [view/pulse.go:245]
                })
            case mx := <-metricsChan:        ← TimeSeries
                p.app.QueueUpdateDraw(func() {
                    p.SeriesChanged(mx)      [view/pulse.go:180]
                })
            }
```

**核心发现**：
1. `model.Pulse` 有 `listeners []PulseListener` 字段和 `AddListener/RemoveListener` 方法，但在 `Start()` 中**完全没有使用**
2. `view.Pulse` 的 `PulseChanged` 方法签名是 `PulseChanged(pt model.HealthPoint)`，而 `PulseListener` 接口要求的是 `PulseChanged(*health.Check)` —— **两者类型不同**，`view.Pulse` 并未实现 `PulseListener` 接口
3. `view.Pulse` 实际是通过 `Start()` 中的 goroutine 直接消费 channel，不经过 listener 机制

### 2.3 HealthPoint 与 health.Check 的关系

[model/pulse_health.go:19-22](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L19-L22) 传输的是 `HealthPoint`：

```go
type HealthPoint struct {
    GVR           *client.GVR
    Total, Faults int
}
```

[health/check.go:13-17](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/health/check.go#L13-L17) 定义的是 `Check`：

```go
type Check struct {
    Counts
    GVR *client.GVR
}
```

两者是不同的数据结构。`HealthPoint` 是健康检查链路**实际使用**的传输类型，`health.Check` 是 `PulseListener` 接口声明的类型，但在当前 pulses 链路中**未被使用**。

### 2.4 健康检查执行流程

[model/pulse_health.go:76-98](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L76-L98)：

```go
const pulseRate = 10 * time.Second

func (h *PulseHealth) Watch(ctx context.Context, ns string) HealthChan {
    c := make(HealthChan, 2)
    ctx = context.WithValue(ctx, internal.KeyWithMetrics, false)  // 关闭 metrics 请求
    go func(ctx context.Context, ns string, c HealthChan) {
        h.checkPulse(ctx, ns, c)  // 立即执行一次
        for {
            select {
            case <-ctx.Done():
                close(c); return
            case <-time.After(pulseRate):
                h.checkPulse(ctx, ns, c)  // 每10秒执行
            }
        }
    }(ctx, ns, c)
    return c
}
```

**注意**：`context.WithValue(ctx, internal.KeyWithMetrics, false)` 在健康检查请求中**关闭了 metrics 采集**，避免健康检查请求也去拉取 metrics 数据。

### 2.5 单资源检查：Registry 查找与 Table 分支

[model/pulse_health.go:112-148](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L112-L148)：

```go
func (h *PulseHealth) check(ctx context.Context, ns string, gvr *client.GVR) (HealthPoint, error) {
    meta, ok := Registry[gvr]
    if !ok {
        // 未注册的资源使用 Table DAO + Table Renderer（Base.Healthy 返回 nil）
        meta = ResourceMeta{
            DAO:      new(dao.Table),
            Renderer: new(render.Table),
        }
    }
    if meta.DAO == nil {
        meta.DAO = &dao.Resource{}
    }

    meta.DAO.Init(h.factory, gvr)
    oo, err := meta.DAO.List(ctx, ns)
    if err != nil {
        return HealthPoint{}, err
    }
    c := HealthPoint{GVR: gvr, Total: len(oo)}
    if isTable(oo) {
        ta := oo[0].(*metav1.Table)
        c.Total = len(ta.Rows)             // Table 模式取 Rows 数
        for _, row := range ta.Rows {
            if err := meta.Renderer.Healthy(ctx, row); err != nil {
                c.Faults++
            }
        }
    } else {
        for _, o := range oo {
            if err := meta.Renderer.Healthy(ctx, o); err != nil {
                c.Faults++
            }
        }
    }
    return c, nil
}
```

**关键分支**：
- `isTable(oo)` 判断：只有当返回**恰好1个** `metav1.Table` 对象时走 Table 分支，此时 `Total` 用 `len(ta.Rows)` 覆盖
- 非 Table 分支用 `len(oo)` 作为 Total
- `Registry` 中未注册的 GVR 使用 `render.Table`，其 `Healthy` 继承自 `Base`，**永远返回 nil**（即永远视为健康）

### 2.6 已注册 Healthy 实现的资源

| 资源 | 文件 | 行号 | 不健康判定 |
|------|------|------|-----------|
| Pod | [render/pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L214-L253) | L214 | 容器未全就绪 / ReadinessGate 不满足 / Pod 未 Ready |
| Node | [render/node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L171-L218) | L171 | Ready 条件缺失 / 被封锁(SchedulingDisabled) |
| Namespace | [render/ns.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ns.go#L100-L122) | L100 | Phase 非 Active 且非 Terminating |
| Event | [render/ev.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ev.go#L20-L32) | L20 | Cells[2] 不是 "Normal"（即 Warning 事件） |
| Helm Chart | [render/helm/chart.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/helm/chart.go#L75-L90) | L75 | Release 状态非 "deployed" |
| 其他（默认） | [render/base.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/base.go#L69-L72) | L69 | 永远返回 nil（永远健康） |

## 三、指标窗口（时间序列）实现

### 3.1 两条采集路径

[dao/recorder.go:104-139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L104-L139) 中 `Watch()` 根据命名空间选择采集路径：

```go
func (r *Recorder) Watch(ctx context.Context, ns string) MetricsChan {
    go func() {
        kind := podMetrics
        if client.IsAllNamespaces(ns) {
            kind = nodeMetrics  // 全命名空间 → 采集 Node 级别
        }
        switch kind {
        case podMetrics:
            r.recordPodMetrics(ctx, ns)    // 特定命名空间 → Pod 级别
        case nodeMetrics:
            r.recordNodeMetrics(ctx)       // 全集群 → Node 级别
        }
        r.dispatchSeries(kind, ns)         // 初始加载：发送缓存中的最近1小时数据
        <-ctx.Done()
        // ...
    }()
    return r.mxChan
}
```

**采集参数**：[dao/recorder.go:23-29](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L23-L29)

| 参数 | 值 | 含义 |
|------|-----|------|
| `seriesCacheSize` | 600 | LRU 缓存最大条目 |
| `seriesCacheExpiry` | 3h | 单条缓存过期时间 |
| `seriesRecordRate` | 1min | 采集间隔 |

### 3.2 Node 级别采集

[dao/recorder.go:148-207](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L148-L207)：

- 聚合所有 Node 的 `CurrentCPU/MEM` 和 `AllocatableCPU/MEM`
- 每次采集通过 `r.mxChan <- TimeSeries{pt}` 直接推送到 channel
- 同时写入 LRU 缓存 `r.series.Add(pt.Time, pt, seriesCacheExpiry)`

### 3.3 Pod 级别采集

[dao/recorder.go:230-273](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L230-L273)：

- 遍历命名空间下所有 Pod 的所有容器，累加 `CurrentCPU/MEM`
- **关键**：[dao/recorder.go:262-263](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L262-L263) 中 `AllocatableCPU = CurrentCPU`，即 **Pod 模式下 Allocatable 被设为与 Current 相同**，因此百分比永远接近 100%

```go
if len(pp.Items) > 0 {
    pt.Value.AllocatableCPU = pt.Value.CurrentCPU   // 注意：等于 Current
    pt.Value.AllocatableMEM = pt.Value.CurrentMEM   // 注意：等于 Current
    // ...
}
```

### 3.4 初始加载：dispatchSeries

[dao/recorder.go:75-102](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L75-L102)：

`Watch()` 启动后会先调用 `dispatchSeries`，从 LRU 缓存中筛选最近1小时的数据一次性推送给 UI，确保界面有历史数据可展示：

```go
func (r *Recorder) dispatchSeries(kind, ns string) {
    kk := r.series.Keys()
    hour := time.Now().Add(-1 * time.Hour)
    // 筛选：type 匹配 + 时间在最近1小时内 + namespace 匹配
    // ...
    if len(ts) > 0 {
        r.mxChan <- ts
    }
}
```

### 3.5 SparkLine 窗口裁剪

[tchart/sparkline.go:118-128](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L118-L128)：

`Draw()` 方法在每次绘制时调用 `cutSet(rect.Dx() - padX)` 裁剪数据：

```go
func (s *SparkLine) cutSet(width int) {
    if width <= 0 || s.series.Empty() {
        return
    }
    if len(s.series) > width {
        s.series.Truncate(width)  // 保留最新的 width 个点
    }
}
```

[tchart/series.go:68-77](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/series.go#L68-L77) `Truncate` 实现：

```go
func (mm MetricSeries) Truncate(size int) {
    kk := mm.Keys()           // 按时间排序
    kk = kk[0 : len(kk)-size] // 取出要删除的旧数据（保留最后 size 个）
    for t := range mm {
        if kk.Includes(t) {
            continue
        }
        delete(mm, kk[0])
    }
}
```

**窗口大小 = 终端宽度**：`width` 由 `rect.Dx() - padX` 决定，即图表可用宽度减去1列 padding。所以时间窗口的数据点数量完全取决于终端宽度，不是固定时间窗口。

### 3.6 SparkLine.SetMax 只增不减

[tchart/sparkline.go:65-69](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L65-L69)：

```go
func (s *SparkLine) SetMax(m float64) {
    if m > s.max {
        s.max = m
    }
}
```

Y轴最大值只增不减。但如果 `view.Pulse.SeriesChanged` 每次都调用 `cpu.SetMax(float64(t.Value.AllocatableCPU))`，当 Allocatable 不变时 `s.max` 保持不变。**缩放因子**：

```go
scale = float64(len(sparks)*(rect.Dy()-pad)) / float64(s.max)
```

当 `s.max == 0` 时会导致除零，但实际上 `AddMetric` 只在 `max > 0` 时才能产生有意义的缩放。

## 四、阈值判断逻辑

### 4.1 阈值配置与判断

[config/threshold.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go)

```go
type Severity struct {
    Critical int  // 默认 90
    Warn     int  // 默认 70
}

type Threshold map[string]*Severity  // key 来自 config/types.go: CPU="cpu", MEM="memory"
```

[config/threshold.go:78-91](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L78-L91) `LevelFor` 返回 `SeverityLevel`（0=Low, 1=Medium, 2=High）：

```go
func (t Threshold) LevelFor(k string, v int) SeverityLevel {
    s, ok := t[k]
    if !ok || v < 0 || v > 100 {
        return SeverityLow   // 未知 key 或越界 → Low
    }
    if v >= s.Critical {     // >= 90 → High
        return SeverityHigh
    }
    if v >= s.Warn {         // >= 70 → Medium
        return SeverityMedium
    }
    return SeverityLow       // < 70 → Low
}
```

[config/threshold.go:94-104](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L94-L104) `SeverityColor` 将 SeverityLevel 映射为颜色字符串：

```go
func (t *Threshold) SeverityColor(k string, v int) string {
    switch t.LevelFor(k, v) {
    case SeverityHigh:   return "red"
    case SeverityMedium: return "orangered"
    default:             return "green"
    }
}
```

### 4.2 阈值在 SeriesChanged 中的应用（仅 SparkLine）

[view/pulse.go:180-242](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L180-L242)：

```go
func (p *Pulse) SeriesChanged(tt dao.TimeSeries) {
    // ...遍历 tt 更新 series 和 max...

    last := tt[len(tt)-1]
    perc := client.ToPercentage(last.Value.CurrentCPU, int64(cpu.GetMax()))

    // 1) LevelFor 获取级别 → 转为 int 作为 colorIndex
    index := int(p.app.Config.K9s.Thresholds.LevelFor("cpu", perc))

    // 2) SetColorIndex 设置到 SparkLine（Gauge 的 SetColorIndex 是空实现）
    cpu.SetColorIndex(index)

    // 3) 获取颜色名称列表
    nn := cpu.GetSeriesColorNames()

    // 4) 零值覆盖（见第五节）

    // 5) 构建 legend：SeverityColor 控制百分比文字颜色，nn[index] 控制图例中数值颜色
    cpu.SetLegend(fmt.Sprintf(cpuFmt,
        cases.Title(language.English).String(client.CpuGVR.R()),
        p.app.Config.K9s.Thresholds.SeverityColor("cpu", perc),  // 百分比文字颜色
        render.PrintPerc(perc),
        nn[index],  // 图例中 "1200m" 的颜色
        render.AsThousands(last.Value.CurrentCPU),
        "white",    // 斜杠颜色
        render.AsThousands(int64(cpu.GetMax())),
    ))
}
```

**阈值生效的三个路径**：

| 路径 | 作用 | 对应代码 |
|------|------|---------|
| `SetColorIndex(index)` | 控制 SparkLine.Draw() 中折线柱体的颜色 | [tchart/sparkline.go:145](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L145) `colors[s.colorIndex%len(colors)]` |
| `SeverityColor()` | 控制 legend 中百分比文字的颜色 | [view/pulse.go:215](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L215) |
| `nn[index]` | 控制 legend 中数值的颜色名 | [view/pulse.go:217](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L217) |

### 4.3 Gauge 不受阈值影响

[tchart/gauge.go:55](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L55)：

```go
func (*Gauge) SetColorIndex(int) {}  // 空实现！
```

Gauge 的颜色完全由 `Component.seriesColors` 决定，通过 `Gauge.Draw()` → `g.colorForSeries()` → `colors[0]`/`colors[1]` 分别用于 OK 和 Fault 数字。阈值系统对 Gauge **没有影响**。

Gauge 的颜色变更只来自两个地方：
1. [view/pulse.go:162-177](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L162-L177) `StylesChanged`：皮肤/样式变更时重新设置 `SetSeriesColors`
2. [view/pulse.go:245-266](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L245-L266) `PulseChanged`：通过修改 `GetSeriesColorNames()` 返回值中的零值条目来灰化

## 五、零值显示机制

### 5.1 SparkLine 的零值灰化（legend 中生效）

[view/pulse.go:206-212](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L206-L212)：

```go
nn := cpu.GetSeriesColorNames()
if last.Value.CurrentCPU == 0 {
    nn[0] = grayC     // "gray"
}
if last.Value.AllocatableCPU == 0 {
    nn[1] = grayC     // "gray"
}
```

**生效路径**：`nn` 被用于 `cpuFmt` 格式字符串中的 `nn[index]`，控制的是 **legend 文字中数值部分的颜色名**。当 `CurrentCPU == 0` 时，`nn[0]` 被改为 "gray"，这意味着 legend 中当前使用量的文字颜色变灰。

**不影响折线柱体颜色**：折线柱体颜色由 `SparkLine.Draw()` 中 `colors[s.colorIndex%len(colors)]` 控制，直接从 `Component.seriesColors` 读取，与 `nn` 无关。

### 5.2 Gauge 的零值灰化（legend 中生效）

[view/pulse.go:251-257](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L251-L257)：

```go
nn := v.GetSeriesColorNames()
if pt.Total == 0 {
    nn[0] = grayC     // OK 数字颜色变灰
}
if pt.Faults == 0 {
    nn[1] = grayC     // Fault 数字颜色变灰
}
```

**同样只影响 legend**：`nn` 未被传给 `SetLegend`（Gauge 的 `PulseChanged` 直接用 GVR 名作为 legend），所以这部分灰化**在当前代码中实际上没有视觉生效**。

### 5.3 Gauge 的 Draw 中数字灰化（实际生效的零值显示）

[tchart/gauge.go:115-139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L115-L139)：

```go
func (g *Gauge) drawNum(sc tcell.Screen, o image.Point, n number, style tcell.Style) {
    colors := g.colorForSeries()
    if n.ok {
        style = style.Foreground(colors[0])
        printDelta(sc, n.delta, o, style)
    }

    dm, significant := NewDotMatrix(), n.val == 0
    if significant {        // 值为0 → 整体 dimmed
        style = g.dimmed
    }
    for i := range len(n.str) {
        if n.str[i] == '0' && !significant {
            g.drawDial(sc, dm.Print(...), o, g.dimmed)  // 前导零 → dimmed
        } else {
            significant = true
            g.drawDial(sc, dm.Print(...), o, style)      // 有效数字 → 正常
        }
        o.X += 3
    }
    if !n.ok {
        o.X++
        printDelta(sc, n.delta, o, style)
    }
}
```

**Gauge 零值显示有两种情况**：

| 情况 | 条件 | 效果 |
|------|------|------|
| 值为 0 | `n.val == 0` | `significant = true`，整组数字使用 `g.dimmed` 样式（灰色暗淡） |
| 前导零 | `n.str[i] == '0' && !significant` | 单个零数字使用 `g.dimmed` 样式 |

[component.go:35](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L35) `dimmed` 样式定义：

```go
dimmed: tcell.StyleDefault.
    Background(tview.Styles.PrimitiveBackgroundColor).
    Foreground(tcell.ColorGray).
    Dim(true)
```

### 5.4 Gauge 边框颜色（健康状态直感）

[view/pulse.go:260-264](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L260-L264)：

```go
if pt.Faults > 0 {
    v.SetBorderColor(tcell.ColorDarkRed)        // 有故障 → 红色边框
} else {
    v.SetBorderColor(tcell.ColorDarkOliveGreen) // 无故障 → 绿色边框
}
```

这是 Gauge 最直观的健康状态指示，**独立于阈值系统**，仅根据 Faults 是否大于 0 判断。

### 5.5 SparkLine 折线柱体的颜色选择

[tchart/sparkline.go:141-147](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L141-L147)：

```go
colors := s.colorForSeries()
cY := rect.Max.Y - pad - 1
for _, t := range s.series.Keys() {
    b := s.makeBlock(s.series[t], scale)
    s.drawBlock(rect, screen, cX, cY, b, colors[s.colorIndex%len(colors)])
    cX++
}
```

所有柱体使用**同一个颜色** `colors[s.colorIndex%len(colors)]`：
- `colorIndex = 0`（Low）→ `colors[0]`（默认绿色）
- `colorIndex = 1`（Medium）→ `colors[1]`（默认橙色）
- `colorIndex = 2`（High）→ `colors[2]`（默认橙红色）

## 六、完整数据流图

```
view.Pulse.Start()                              [view/pulse.go:308]
    │
    ├── model.Pulse.Watch(ctx)                  [model/pulse.go:43]
    │       │
    │       ├── PulseHealth.Watch(ctx, ns)       [model/pulse_health.go:76]
    │       │     返回 HealthChan (chan HealthPoint, cap=2)
    │       │       │
    │       │       ├── 立即执行 checkPulse()
    │       │       └── 每 10s 执行 checkPulse()
    │       │             │
    │       │             └── 遍历 PulseGVRs(16种资源)
    │       │                   │
    │       │                   └── check(ctx, ns, gvr)
    │       │                         │                         [model/pulse_health.go:112]
    │       │                         ├── Registry[gvr] 查找 DAO+Renderer
    │       │                         ├── DAO.List(ctx, ns) 获取资源列表
    │       │                         ├── isTable 判断走 Table/普通分支
    │       │                         └── Renderer.Healthy() 逐个检查 → Faults++
    │       │
    │       └── Recorder.Watch(ctx, ns)          [dao/recorder.go:104]
    │             返回 MetricsChan (chan TimeSeries, cap=2)
    │               │
    │               ├── 初始 dispatchSeries → 发送缓存最近1小时
    │               └── 每 1min 采集一次
    │                     │
    │                     ├── Node 模式: recordClusterMetrics    [dao/recorder.go:173]
    │                     │     聚合所有 Node 的 CPU/MEM
    │                     │
    │                     └── Pod 模式: recordPodsMetrics        [dao/recorder.go:230]
    │                           累加 NS 下所有 Pod 容器 Usage
    │                           注意: Allocatable = Current
    │
    └── goroutine select 消费:
            │
            ├── <-gaugeChan → PulseChanged(HealthPoint)   [view/pulse.go:245]
            │     ├── 零值灰化 nn[0]/nn[1] = "gray"       (仅修改 legend 颜色名)
            │     ├── 边框颜色: Faults>0 → Red, 否则 Green
            │     └── Gauge.Add(Total, Faults)            [tchart/gauge.go:63]
            │           → 计算 delta 趋势
            │           → Draw() 时点阵数字 + 前导零 dimmed + 值为0 整体 dimmed
            │
            └── <-metricsChan → SeriesChanged(TimeSeries)  [view/pulse.go:180]
                  ├── SparkLine.SetMax / AddMetric         [tchart/sparkline.go:65,78]
                  ├── 阈值 LevelFor → colorIndex           (仅 SparkLine 有效)
                  ├── SparkLine.SetColorIndex(index)        (Gauge 的 SetColorIndex 为空)
                  ├── 零值灰化 nn[0]/nn[1] = "gray"        (控制 legend 文字颜色)
                  ├── SeverityColor → legend 百分比颜色
                  └── Draw() 时 colors[colorIndex%len]     → 折线柱体颜色
```

## 七、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 健康检查调度 | [model/pulse_health.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L76-L98) | L76-L98 |
| 单资源检查 | [model/pulse_health.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L112-L148) | L112-L148 |
| Watch channel 消费 | [view/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L308-L339) | L308-L339 |
| PulseListener 接口(未使用) | [model/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse.go#L14-L23) | L14-L23 |
| PulseChanged(Gauge) | [view/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L245-L266) | L245-L266 |
| SeriesChanged(SparkLine) | [view/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L180-L242) | L180-L242 |
| 阈值 LevelFor | [config/threshold.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L78-L91) | L78-L91 |
| 阈值 SeverityColor | [config/threshold.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L94-L104) | L94-L104 |
| CPU/MEM 常量 | [config/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/types.go#L10-L14) | L10-L14 |
| Gauge.SetColorIndex 空实现 | [tchart/gauge.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L55) | L55 |
| SparkLine.SetColorIndex | [tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L61-L63) | L61-L63 |
| SparkLine.Draw colorIndex | [tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L141-L147) | L141-L147 |
| Gauge.Draw 数字 dimmed | [tchart/gauge.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L115-L139) | L115-L139 |
| Component.seriesColors 默认 | [tchart/component.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L30-L34) | L30-L34 |
| Component.dimmed 样式 | [tchart/component.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L35) | L35 |
| GetSeriesColorNames | [tchart/component.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/component.go#L98-L115) | L98-L115 |
| Recorder.Watch | [dao/recorder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L104-L139) | L104-L139 |
| Pod 模式 Allocatable=Current | [dao/recorder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L262-L263) | L262-L263 |
| dispatchSeries 初始加载 | [dao/recorder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L75-L102) | L75-L102 |
| Series.Truncate 窗口裁剪 | [tchart/series.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/series.go#L68-L77) | L68-L77 |
| SparkLine.cutSet | [tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L176-L183) | L176-L183 |
| SparkLine.SetMax 只增不减 | [tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L65-L69) | L65-L69 |
| Pod Healthy | [render/pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L214-L253) | L214-L253 |
| Node Healthy | [render/node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L171-L218) | L171-L218 |
| Base Healthy 默认 | [render/base.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/base.go#L69-L72) | L69-L72 |
