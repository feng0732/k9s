# Pulses 健康指示链路分析

## 一、整体架构

Pulses 健康指示系统采用分层架构，从数据采集到界面展示分为四个主要层级：

```
┌─────────────────────────────────────────────────────────┐
│                  界面展示层 (View)                       │
│  [view/pulse.go]  Pulse 视图组件                        │
│  ├─ Gauge 仪表盘 (资源健康状态)                         │
│  └─ SparkLine 折线图 (CPU/内存时序)                     │
├─────────────────────────────────────────────────────────┤
│                  数据模型层 (Model)                      │
│  [model/pulse.go]       Pulse 模型 (监听器管理)          │
│  [model/pulse_health.go] PulseHealth 健康检查器          │
├─────────────────────────────────────────────────────────┤
│                  健康检查层 (Health)                     │
│  [render/*.go]   各资源 Healthy() 方法                  │
│  [health/check.go] Check 健康检查数据结构               │
├─────────────────────────────────────────────────────────┤
│                  数据采集层 (DAO)                        │
│  [dao/recorder.go]  Recorder 指标采集器                 │
│  [dao/factory.go]   Factory 资源访问工厂                │
└─────────────────────────────────────────────────────────┘
```

## 二、核心数据流

### 2.1 健康检查数据流向

```
Pulse.Start() [view/pulse.go:308-339]
        │
        ▼
model.Watch() [model/pulse.go:43-56]
        ├───────────────────────────────────┐
        │                                   │
        ▼                                   ▼
PulseHealth.Watch()               Recorder.Watch()
[model/pulse_health.go:76-98]     [dao/recorder.go:104-139]
        │                                   │
        ▼                                   ▼
checkPulse() 每10秒轮询              record*Metrics() 每1分钟采集
[model/pulse_health.go:100-110]     [dao/recorder.go]
        │                                   │
        ▼                                   ▼
check() -> Renderer.Healthy()        时间序列缓存 (LRU 600条)
[model/pulse_health.go:112-148]      [dao/recorder.go:54]
        │                                   │
        ▼                                   ▼
HealthPoint{GVR, Total, Faults}      TimeSeries{Time, Value, Tags}
        │                                   │
        ▼                                   ▼
Pulse.PulseChanged()              Pulse.SeriesChanged()
[view/pulse.go:245-266]           [view/pulse.go:180-242]
        │                                   │
        ▼                                   ▼
Gauge.Add(ok, fault)                SparkLine.AddMetric(t, f)
[tchart/gauge.go:63-69]             [tchart/sparkline.go:78-82]
        │                                   │
        ▼                                   ▼
Gauge.Draw() 点阵数字显示           SparkLine.Draw() 折线图渲染
[tchart/gauge.go:79-113]            [tchart/sparkline.go:118-158]
```

## 三、指标窗口（时间序列）实现

### 3.1 指标采集与缓存

**采集入口**：[dao/recorder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go)

```go
// 关键常量定义
const (
    seriesCacheSize   = 600            // 缓存最大条目数
    seriesCacheExpiry = 3 * time.Hour  // 缓存过期时间
    seriesRecordRate  = 1 * time.Minute // 采集频率
)
```

**数据结构**：

```go
type TimeSeries []Point

type Point struct {
    Time  time.Time          // 采集时间戳
    Tags  map[string]string  // 标签：type(node/pod), namespace
    Value client.NodeMetrics // 指标值：CurrentCPU, CurrentMEM, AllocatableCPU/MEM
}
```

**采集逻辑**：

1. **集群级别指标**（All Namespaces 模式）：[dao/recorder.go:148-207](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L148-L207)
   - 聚合所有 Node 的 CPU/内存使用量
   - 计算总量：CurrentCPU, CurrentMEM, AllocatableCPU, AllocatableMEM

2. **命名空间级别指标**：[dao/recorder.go:209-273](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go#L209-L273)
   - 按命名空间聚合 Pod 的 CPU/内存使用量
   - 遍历 Pod 内所有容器的 Usage 累加

### 3.2 时间序列窗口管理

**SparkLine 组件**：[tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go)

```go
type SparkLine struct {
    *Component
    series     MetricSeries  // map[time.Time]float64 时间序列数据
    max        float64       // Y轴最大值（动态调整）
    unit       string        // 单位："c" (CPU), "Gi" (内存)
    colorIndex int           // 颜色索引（根据阈值）
}
```

**窗口裁剪逻辑**：[tchart/sparkline.go:176-183](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go#L176-L183)

```go
func (s *SparkLine) cutSet(width int) {
    if width <= 0 || s.series.Empty() {
        return
    }
    if len(s.series) > width {
        s.series.Truncate(width)  // 保留最新的 width 个数据点
    }
}
```

**动态缩放**：
- `SetMax(float64)`：只增不减，确保图表不会频繁跳动
- 缩放因子：`scale = float64(len(sparks)*(rect.Dy()-pad)) / float64(s.max)`
- 使用 8 级灰度块：`▁▂▃▄▅▆▇█` 表示不同高度

## 四、阈值判断逻辑

### 4.1 阈值配置

**阈值定义**：[config/threshold.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go)

```go
type SeverityLevel int
const (
    SeverityLow    SeverityLevel = iota  // 0: 正常
    SeverityMedium                       // 1: 警告
    SeverityHigh                         // 2: 严重
)

type Severity struct {
    Critical int  // 默认 90
    Warn     int  // 默认 70
}

type Threshold map[string]*Severity  // key: "cpu", "memory"
```

**阈值判断**：[config/threshold.go:78-91](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L78-L91)

```go
func (t Threshold) LevelFor(k string, v int) SeverityLevel {
    s, ok := t[k]
    if !ok || v < 0 || v > 100 {
        return SeverityLow
    }
    if v >= s.Critical {   // >= 90: 严重
        return SeverityHigh
    }
    if v >= s.Warn {       // >= 70: 警告
        return SeverityMedium
    }
    return SeverityLow     // < 70: 正常
}
```

**颜色映射**：[config/threshold.go:94-104](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go#L94-L104)

```go
func (t *Threshold) SeverityColor(k string, v int) string {
    switch t.LevelFor(k, v) {
    case SeverityHigh:   return "red"       // 严重：红色
    case SeverityMedium: return "orangered" // 警告：橙红色
    default:             return "green"     // 正常：绿色
    }
}
```

### 4.2 阈值在 UI 中的应用

**CPU/内存指标**：[view/pulse.go:180-242](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L180-L242)

```go
func (p *Pulse) SeriesChanged(tt dao.TimeSeries) {
    // ...
    perc := client.ToPercentage(last.Value.CurrentCPU, int64(cpu.GetMax()))
    index := int(p.app.Config.K9s.Thresholds.LevelFor("cpu", perc))
    cpu.SetColorIndex(index)  // 设置颜色索引
    
    // 图例显示：CPU 85%(1200m/4000m) - 85%显示为对应阈值颜色
    cpu.SetLegend(fmt.Sprintf(cpuFmt,
        "Cpu",
        p.app.Config.K9s.Thresholds.SeverityColor("cpu", perc),  // 百分比颜色
        render.PrintPerc(perc),
        nn[index],  // 折线颜色
        render.AsThousands(last.Value.CurrentCPU),
        "white",
        render.AsThousands(int64(cpu.GetMax())),
    ))
}
```

**资源健康状态**：[view/pulse.go:245-266](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L245-L266)

```go
func (p *Pulse) PulseChanged(pt model.HealthPoint) {
    // ...
    if pt.Faults > 0 {
        v.SetBorderColor(tcell.ColorDarkRed)        // 有故障：红色边框
    } else {
        v.SetBorderColor(tcell.ColorDarkOliveGreen) // 无故障：绿色边框
    }
    v.Add(pt.Total, pt.Faults)
}
```

## 五、健康检查实现

### 5.1 健康检查入口

**健康检查调度**：[model/pulse_health.go:76-98](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L76-L98)

```go
const pulseRate = 10 * time.Second  // 健康检查频率：每10秒一次

func (h *PulseHealth) Watch(ctx context.Context, ns string) HealthChan {
    go func(ctx context.Context, ns string, c HealthChan) {
        h.checkPulse(ctx, ns, c)  // 立即执行一次
        for {
            select {
            case <-ctx.Done():
                close(c); return
            case <-time.After(pulseRate):  // 每10秒轮询
                h.checkPulse(ctx, ns, c)
            }
        }
    }(ctx, ns, c)
    return c
}
```

**检查的资源类型**：[model/pulse_health.go:26-46](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L26-L46)

```go
var PulseGVRs = client.GVRs{
    client.NodeGVR, client.NsGVR, client.SvcGVR, client.EvGVR,
    client.PodGVR, client.DpGVR, client.StsGVR, client.DsGVR,
    client.JobGVR, client.CjGVR, client.PvGVR, client.PvcGVR,
    client.HpaGVR, client.IngGVR, client.NpGVR, client.SaGVR,
}  // 共16种资源类型
```

### 5.2 单资源健康检查

**检查逻辑**：[model/pulse_health.go:112-148](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go#L112-L148)

```go
func (h *PulseHealth) check(ctx context.Context, ns string, gvr *client.GVR) (HealthPoint, error) {
    meta := Registry[gvr]  // 获取资源元数据（DAO + Renderer）
    meta.DAO.Init(h.factory, gvr)
    oo, _ := meta.DAO.List(ctx, ns)  // 列出该命名空间下的所有资源
    
    c := HealthPoint{GVR: gvr, Total: len(oo)}
    for _, o := range oo {
        // 调用具体资源的 Healthy() 方法判断健康状态
        if err := meta.Renderer.Healthy(ctx, o); err != nil {
            c.Faults++  // 统计故障数
        }
    }
    return c, nil
}
```

### 5.3 具体资源 Healthy 实现

**Pod 健康检查**：[render/pod.go:214-253](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go#L214-L253)

```go
func (p *Pod) Healthy(_ context.Context, o any) error {
    // 转换为 PodWithMetrics
    pwm, _ := o.(*PodWithMetrics)
    // 解析 status 和 spec
    var st v1.PodStatus
    runtime.DefaultUnstructuredConverter.FromUnstructured(..., &st)
    
    dt := pwm.Raw.GetDeletionTimestamp()
    phase := p.Phase(dt, spec, &st)           // Pod 阶段
    cr, ct, _, _ := p.ContainerStats(...)     // 就绪容器数/总数
    ready := hasPodReadyCondition(st.Conditions) // Ready 条件
    rgr, rgt := p.readinessGateStats(...)     // ReadinessGate 统计
    
    return p.diagnose(phase, cr, ct, ready, rgr, rgt)
}

func (*Pod) diagnose(phase string, cr, ct int, ready bool, rgr, rgt int) error {
    if phase == Completed { return nil }                    // 已完成的 Pod 算健康
    if cr != ct || ct == 0 {                                // 容器未全部就绪
        return fmt.Errorf("container ready check failed: %d of %d", cr, ct)
    }
    if rgt > 0 && rgr != rgt {                              // ReadinessGate 未满足
        return fmt.Errorf("readiness gate check failed: %d of %d", rgr, rgt)
    }
    if !ready { return fmt.Errorf("pod not ready") }        // 未就绪
    return nil
}
```

**Node 健康检查**：[render/node.go:171-218](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go#L171-L218)

```go
func (n Node) Healthy(_ context.Context, o any) error {
    nwm, _ := o.(*NodeWithMetrics)
    var no v1.Node
    runtime.DefaultUnstructuredConverter.FromUnstructured(nwm.Raw.Object, &no)
    
    ss := make([]string, 10)
    status(no.Status.Conditions, no.Spec.Unschedulable, ss)  // 解析节点状态
    
    return n.diagnose(ss)
}

func (Node) diagnose(ss []string) error {
    var ready, cordoned bool
    for _, s := range ss {
        if s == "SchedulingDisabled" { cordoned = true }
        if s == "Ready" { ready = true }
    }
    if !ready { return notReadyErr }    // 节点未就绪
    if cordoned { return cordonErr }    // 节点被封锁
    return nil
}
```

**Namespace 健康检查**：[render/ns.go:100-122](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ns.go#L100-L122)

```go
func (n Namespace) Healthy(_ context.Context, o any) error {
    res, _ := o.(*unstructured.Unstructured)
    var ns v1.Namespace
    runtime.DefaultUnstructuredConverter.FromUnstructured(res.Object, &ns)
    
    return n.diagnose(ns.Status.Phase)
}

func (Namespace) diagnose(phase v1.NamespacePhase) error {
    if phase != v1.NamespaceActive && phase != v1.NamespaceTerminating {
        return errors.New("namespace not ready")  // 非 Active/Terminating 状态
    }
    return nil
}
```

**Event 健康检查**：[render/ev.go:20-32](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ev.go#L20-L32)

```go
func (*Event) Healthy(_ context.Context, o any) error {
    r, _ := o.(metav1.TableRow)
    idx := 2  // 事件类型列（Normal/Warning）
    if idx < len(r.Cells) && r.Cells[idx] != "Normal" {
        return fmt.Errorf("event is not normal: %s", r.Cells[idx])
    }
    return nil
}
```

**默认实现**：[render/base.go:69-72](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/base.go#L69-L72)

```go
func (*Base) Healthy(context.Context, any) error {
    return nil  // 未实现的资源默认视为健康
}
```

## 六、降级显示机制

### 6.1 Gauge 仪表盘降级

**Gauge 数据结构**：[tchart/gauge.go:30-36](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L30-L36)

```go
type Gauge struct {
    *Component
    state               State  // {OK: 正常数, Fault: 故障数}
    resolution          int
    deltaOK, deltaFault delta  // 变化趋势（上升/下降/不变）
}
```

**数字降级显示**：[tchart/gauge.go:115-139](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L115-L139)

```go
func (g *Gauge) drawNum(sc tcell.Screen, o image.Point, n number, style tcell.Style) {
    // 前导零显示为灰色（降级）
    significant := n.val == 0
    for i := range len(n.str) {
        if n.str[i] == '0' && !significant {
            g.drawDial(sc, dm.Print(...), o, g.dimmed)  // 前导零：灰色暗淡
        } else {
            significant = true
            g.drawDial(sc, dm.Print(...), o, style)      // 有效数字：正常颜色
        }
        o.X += 3
    }
}
```

**趋势指示**：[tchart/gauge.go:173-181](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go#L173-L181)

```go
func printDelta(sc tcell.Screen, d delta, o image.Point, s tcell.Style) {
    s = s.Dim(false)
    switch d {
    case DeltaLess:
        sc.SetContent(o.X-1, o.Y+1, '↓', nil, s)  // 下降：↓
    case DeltaMore:
        sc.SetContent(o.X-1, o.Y+1, '↑', nil, s)  // 上升：↑
    // DeltaSame: 无指示
    }
}
```

### 6.2 SparkLine 折线图降级

**颜色降级**：[view/pulse.go:180-242](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L180-L242)

```go
func (p *Pulse) SeriesChanged(tt dao.TimeSeries) {
    // CPU 颜色降级
    nn := cpu.GetSeriesColorNames()
    if last.Value.CurrentCPU == 0 {
        nn[0] = "gray"  // 当前CPU为0：第一条线灰色
    }
    if last.Value.AllocatableCPU == 0 {
        nn[1] = "gray"  // 可分配CPU为0：第二条线灰色
    }
    
    // 内存同理...
}
```

**Gauge 颜色降级**：[view/pulse.go:245-266](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L245-L266)

```go
func (p *Pulse) PulseChanged(pt model.HealthPoint) {
    nn := v.GetSeriesColorNames()
    if pt.Total == 0 {
        nn[0] = "gray"  // 总数为0：OK颜色变灰
    }
    if pt.Faults == 0 {
        nn[1] = "gray"  // 故障数为0：Fault颜色变灰
    }
    // ...
}
```

### 6.3 边框颜色指示

[view/pulse.go:260-264](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go#L260-L264)

```go
if pt.Faults > 0 {
    v.SetBorderColor(tcell.ColorDarkRed)        // 有故障：红色边框
} else {
    v.SetBorderColor(tcell.ColorDarkOliveGreen) // 无故障：绿色边框
}
```

## 七、关键代码位置索引

| 功能模块 | 文件 | 关键行 |
|---------|------|--------|
| 健康检查调度 | [model/pulse_health.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go) | L76-L148 |
| 监控资源列表 | [model/pulse_health.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/model/pulse_health.go) | L26-L46 |
| Pod 健康检查 | [render/pod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/pod.go) | L214-L253 |
| Node 健康检查 | [render/node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/node.go) | L171-L218 |
| Namespace 健康检查 | [render/ns.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ns.go) | L100-L122 |
| Event 健康检查 | [render/ev.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/render/ev.go) | L20-L32 |
| 阈值配置 | [config/threshold.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/config/threshold.go) | L1-L104 |
| 指标采集 | [dao/recorder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/dao/recorder.go) | L1-L300 |
| 时间序列 | [tchart/series.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/series.go) | L1-L50 |
| Gauge 组件 | [tchart/gauge.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/gauge.go) | L1-L181 |
| SparkLine 组件 | [tchart/sparkline.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/tchart/sparkline.go) | L1-L197 |
| Pulse 视图 | [view/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go) | L1-L549 |
| 阈值应用 | [view/pulse.go](file:///d:/fz/0601-2/solo-dogfeeding/code/8-k9s/internal/view/pulse.go) | L180-L266 |

## 八、总结

Pulses 健康指示系统采用**分层架构**和**事件驱动**设计：

1. **采集层**：每10秒进行健康检查，每1分钟采集指标，使用 LRU 缓存保存3小时数据
2. **检查层**：每种资源实现独立的 `Healthy()` 方法，通过错误返回值标记故障
3. **阈值层**：CPU/内存使用三级阈值（70%警告，90%严重），映射到不同颜色
4. **展示层**：
   - Gauge 仪表盘显示 OK/Fault 计数，点阵数字 + 变化趋势箭头
   - SparkLine 折线图显示 CPU/内存时序，8级高度块
   - 多重降级：前导零灰化、零值灰化、边框颜色指示
5. **数据流**：通过 channel 异步传递 HealthPoint 和 TimeSeries，UI 层通过 QueueUpdateDraw 线程安全更新

整个链路清晰，各层职责明确，通过 Renderer 接口实现资源健康检查的可扩展性，通过图表组件的降级显示提供丰富的视觉反馈。
