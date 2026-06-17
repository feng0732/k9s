# Helm Release 回滚 UI 回调——补正分析

本文深入分析确认按钮点击后的执行上下文、UI 阻塞情况、以及 Flash 提示与表格刷新的时序。

---

## 关键前提：tview 事件循环模型

k9s 使用 derailed 分支的 tview 框架，其核心模型是**单线程事件循环**。

### 1.1 主 goroutine 与事件循环

应用启动时，[view/app.go:570](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/app.go#L570) 在**主 goroutine**中调用：

```go
func (a *App) Run() error {
    // ...
    a.SetRunning(true)
    if err := a.Application.Run(); err != nil {  // ← 主 goroutine 阻塞在这里
        return err
    }
    return nil
}
```

`tview.Application.Run()` 内部是一个死循环：
```
for {
    PollEvent()      // 阻塞等待键盘/鼠标/重绘事件
    处理事件          // 调用控件的 InputHandler
    Draw()           // 重绘界面
}
```

**核心规则**：**所有 UI 事件回调都在主 goroutine 中同步执行**。除非回调内部显式 `go func()`，否则会阻塞整个事件循环。

### 1.2 QueueUpdate/QueueUpdateDraw 的作用

从 [ui/app.go:64-82](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/ui/app.go#L64-L82) 可以看出跨 goroutine 更新 UI 的标准方式：

```go
func (a *App) QueueUpdate(f func()) {
    go func() {
        a.Application.QueueUpdate(f)  // 入队到主事件循环
    }()
}
```

- `QueueUpdate`：将函数排入主 goroutine 的执行队列，在下一次事件循环迭代时执行
- `QueueUpdateDraw`：排入队列并在执行后触发重绘
- 这两个函数是**线程安全**的，可以从任意 goroutine 调用

---

## 问题一：确认按钮回调是否另起 goroutine？

### 结论：**没有另起 goroutine，回调直接在主 goroutine 中同步执行**

### 证据链

**第 1 步：tview 按钮注册回调**

在 [dialog/confirm.go:41-48](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/ui/dialog/confirm.go#L41-L48)：

```go
f.AddButton("OK", func() {
    if !accept {
        return
    }
    ack()              // ← 这就是我们传入的回滚回调
    dismissConfirm(pages)
    cancel()
})
```

`tview.Form.AddButton()` 的回调是**闭包函数引用**，注册时不会执行，只是存起来。

**第 2 步：按钮点击时的调用链**

当用户按下 Enter 键或鼠标点击 OK 按钮时：
```
主 goroutine PollEvent() 收到按键事件
    ↓
tview.Form.InputHandler() 处理
    ↓
查找焦点按钮（OK）
    ↓
直接调用按钮的注册回调函数  ← 同步执行，没有 go 关键字！
    ↓
ack() → rollbackCmd 回调 → h.rollback() → hm.Rollback()
```

**关键证据**：整个调用链中**没有任何 `go func()`**，全部是同步函数调用。

### 类比理解

想象你在写一个 JavaScript 浏览器应用：
```javascript
button.addEventListener('click', async () => {
    await longRunningOperation();  // 会阻塞事件循环吗？
})
```
在 JS 中，如果 `longRunningOperation` 是 CPU 密集型同步操作，会卡住整个页面。k9s 的 tview 模型与此完全相同，只是用 Go 实现。

---

## 问题二：回滚期间界面是否会被阻塞？

### 结论：**会！而且是完全卡住，不能响应任何按键，直到回滚完成。**

### 完整阻塞时间线

```
t0: 用户移动焦点到 OK 按钮，按下 Enter 键
 │
 ▼
主 goroutine: 事件循环 PollEvent() 收到按键事件
 │
 ▼
t0.01: 调用按钮回调 ack()
 │
 ├─► 创建 ctx, cancel（无用，见之前分析）
 │
 ├─► 调用 h.rollback(ctx, path, rev)
 │     │
 │     ├─► hm.Init(...)  ← 快速
 │     │
 │     ├─► hm.Rollback(ctx, path, rev)
 │     │     │
 │     │     ├─► ensureHelmConfig(...)  ← 快速
 │     │     ├─► strconv.Atoi(rev)      ← 快速
 │     │     ├─► action.NewRollback(cfg) ← 快速
 │     │     └─► clt.Run(n)
 │     │           │
 │     │           └─► [Helm SDK] 与 K8s apiserver 通信
 │     │                ├─ 读取 Secret/ConfigMap 中的 release 数据
 │     │                ├─ 执行回滚：更新 release 版本号
 │     │                ├─ 重建 Kubernetes 资源（Pod/Service/ConfigMap 等）
 │     │                └─ 等待资源就绪？（取决于 Helm 配置）
 │     │
 │     │           ╔═══════════════════════════════════════════════╗
 │     │           ║  这里可能阻塞 2 秒 ~ 几分钟！                  ║
 │     │           ║  期间主 goroutine 被占着，什么都做不了        ║
 │     │           ║  - 屏幕不会重绘                               ║
 │     │           ║  - 按键不会响应                               ║
 │     │           ║  - 鼠标点击不会响应                           ║
 │     │           ║  - 光标停止闪烁                               ║
 │     │           ╚═══════════════════════════════════════════════╝
 │     │
 │     └─► h.Refresh() ← 回滚完成后才执行
 │
 ├─► Flash.Infof(...) ← 回滚完成后才执行
 │
 └─► dismissConfirm(pages) ← 关闭对话框
 │
t0 + 阻塞时间: 回调返回，主 goroutine 回到事件循环
 │
 ▼
处理队列中的 UI 更新，屏幕重绘，用户看到提示
```

### 阻塞期间的状态

| 组件 | 状态 |
|------|------|
| **键盘输入** | ❌ 完全无响应，按什么键都没用 |
| **屏幕重绘** | ❌ 停在最后一帧，对话框还在，按钮保持按下状态 |
| **鼠标输入** | ❌ 无响应 |
| **Flash 消息** | ❌ 看不到，因为 Flash.Watch goroutine 发了 QueueUpdateDraw，但主 goroutine 没空处理 |
| **Watch 数据轮询** | ✅ 后台 goroutine 仍在运行，新数据缓存在 model 里，但 UI 不刷新 |
| **Cancel 按钮** | ❌ 点击 Cancel 也没用，事件循环卡住了 |

### 更严重的问题

如果回滚操作需要 30 秒，那么用户看到的现象是：
1. 点击 OK 后，按钮"陷下去"不弹起来
2. 整个界面"死了"
3. 用户可能以为程序崩溃了，强制关掉终端
4. 但实际上 Helm 回滚仍在 K8s 后台运行，只是 k9s 不响应了

---

## 问题三：成功提示和数据刷新的先后关系

### 结论：**h.Refresh() 先执行，Flash.Infof() 后执行；但两者都要等回滚阻塞结束后才会运行。**

### 精确的代码执行顺序

回到 [helm_history.go:108-116](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L108-L116) 的回调：

```go
dialog.ShowConfirmAck(..., func() {
    ctx, cancel := context.WithTimeout(...)
    defer cancel()
    
    // ① 调用 rollback（注意这个函数的内部顺序）
    if err := h.rollback(ctx, client.FQN(ns, n), rev); err != nil {
        // 失败路径
        h.App().Flash().Err(err)           // ③ 失败：Flash 错误
    } else {
        // 成功路径
        h.App().Flash().Infof("Rollout restart in progress...")  // ③ 成功：Flash 信息
    }
}, ...)
```

关键在 [h.rollback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L121-L130) 内部：

```go
func (h *History) rollback(ctx context.Context, path, rev string) error {
    var hm dao.HelmHistory
    hm.Init(h.App().factory, h.GVR())
    
    // ② 先执行回滚（阻塞！）
    if err := hm.Rollback(ctx, path, rev); err != nil {
        return err  // 失败直接返回，不刷新
    }
    
    // ②' 回滚成功后，同步刷新表格
    h.Refresh()
    
    return nil
}
```

### 时序图（成功路径）

```
时间轴 →

  阻塞区（回滚执行中）
┌─────────────────────────┐
│                         │
│  clt.Run(n) 阻塞中...    │
│                         │
└─────────────────────────┘
                         │
                         ▼
                      h.Refresh() ────────────┐
                                              │ 同步
                         ┌────────────────────┘
                         │
                         ▼
                      ui.Table.Refresh()
                         │
                         ├─► t.model.Peek()        读取缓存数据
                         ├─► t.Update(data)         应用排序/过滤
                         └─► t.UpdateUI(cdata)      更新 tview Table 单元格
                                              （直接修改内存，立即生效）
                         │
                         ▼
                      Flash.Infof("Rollout restart...")
                         │
                         ├─► Flash.SetMessage(FlashInfo, msg)
                         │    ├─► 设置 f.msg
                         │    ├─► fireFlashChanged()
                         │    │   └─► f.msgChan <- msg  ──► 发送到 channel
                         │    └─► go f.refresh(ctx)     ──► 启动6秒后清除的 goroutine
                         │
                         ▼
                      ack() 回调返回
                         │
                         ▼
                      主 goroutine 回到事件循环
                         │
                         ├─► 重绘表格（因为 UpdateUI 修改了单元格）
                         │
                         └─► 处理 QueueUpdateDraw 队列
                              │
                              ▼
                   Flash.Watch goroutine 之前发来的更新
                              │
                              ├─► f.app.QueueUpdateDraw(fn)
                              │
                              ▼
                   主 goroutine 执行 fn，Flash 消息显示

用户感知时间：
───────────────────────────────────────────────────────────────
点击 OK ──（卡住 N 秒）──► 表格刷新 ──（约 10~50ms）──► Flash 提示出现
```

### 关键区别：h.Refresh() vs Flash.Infof()

| 特性 | `h.Refresh()` | `Flash.Infof()` |
|------|--------------|-----------------|
| **执行时机** | 回滚成功后，回调内同步执行 | 回滚成功后，回调内同步调用，但实际显示异步 |
| **Goroutine** | 主 goroutine 直接执行 | 发送 channel，Flash.Watch goroutine 接收，再 QueueUpdateDraw 到主 goroutine |
| **数据来源** | 从 model.Peek() 读取已缓存的数据 | 新消息内容 |
| **生效速度** | 立即修改内存，下次重绘可见 | 经过两次 goroutine 切换 + 队列调度，稍有延迟 |
| **用户感知** | 表格内容变化 | 底部状态栏出现消息 |

### 重要说明：h.Refresh() 不拉新数据

`ui.Table.Refresh()` [ui/table.go:598-605](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/ui/table.go#L598-L605) 的实现是：

```go
func (t *Table) Refresh() {
    data := t.model.Peek()  // 从 model 读已缓存的数据
    if data.HeaderCount() == 0 {
        return
    }
    cdata := t.Update(data, t.hasMetrics)  // 应用排序过滤
    t.UpdateUI(cdata, data)                // 更新 UI
}
```

它**不会调用 DAO.List()**，不会去 Helm 后端拉取最新数据。它只是把 model 中已有的数据重新渲染到表格上。

真正拉取新数据的是 Watch 轮询：
- Watch goroutine 每 2 秒（默认 RefreshRate）调用一次 DAO.List()
- 新数据存入 model
- 触发 Table 的 ResourceChanged 回调，异步更新 UI

所以回滚成功后：
1. `h.Refresh()` 立即渲染 model 中已有的**旧数据**（可能已经过期了几毫秒到几秒）
2. 下一次 Watch 轮询（最多 2 秒后）才会拉取回滚后的新版本记录
3. 这时候表格才会真正显示新增的 revision 行

### 失败路径的差异

如果 `hm.Rollback()` 返回 error：
```
hm.Rollback() → 返回 error
    │
    ▼
h.rollback() → return err
    │
    ▼
h.App().Flash().Err(err)
    │
    ├─► slog.Error(...)  记录错误日志
    └─► Flash.SetMessage(FlashErr, err.Error())
          └─► msgChan <- 错误消息
    │
    ▼
ack() 返回
    │
    ▼
主 goroutine 回到事件循环
    │
    └─► 显示 Flash 错误消息（红色）
```

**失败时不会调用 `h.Refresh()`**，表格保持原样。

---

## 总结：之前分析的补正点

| 问题 | 之前的分析 | 补正后的真实情况 |
|------|-----------|-----------------|
| **回调 goroutine** | 未明确说明，暗示可能不阻塞 | ✅ **同步执行在主 goroutine**，没有另起 goroutine |
| **UI 阻塞** | "不会阻塞整个 UI 主线程"（之前 followup 的错误结论！） | ❌ **完全阻塞**，回滚期间界面卡死，无法响应任何输入 |
| **Flash 与 Refresh 顺序** | 未明确区分 | ✅ 代码顺序：先 `h.Refresh()` 刷新表格，再 `Flash.Infof()` 发消息；但两者都在回滚阻塞结束后才运行；Flash 显示稍晚 |
| **Refresh 效果** | "Refresh() 是 TView 组件重绘" | ✅ 更精确：是 `ui.Table.Refresh()`，从已缓存的 model 数据重绘表格，**不拉新数据**；新数据要等下一次 Watch 轮询 |

---

## 代码改进建议

### 1. 避免阻塞 UI：回滚操作另起 goroutine

```go
func (h *History) rollbackCmd(evt *tcell.EventKey) *tcell.EventKey {
    // ... 解析 path ...
    
    h.Stop()
    defer h.Start()
    
    msg := fmt.Sprintf("RollingBack chart [yellow::b]%s[-::-] to release <[orangered::b]%s[-::-]>?", n, rev)
    dialog.ShowConfirmAck(..., func() {
        // ✅ 另起 goroutine 执行回滚，不阻塞主事件循环
        go func() {
            ctx, cancel := context.WithTimeout(context.Background(), h.App().Conn().Config().CallTimeout())
            defer cancel()
            
            h.App().Flash().Info("Rollback in progress...")  // 先提示正在进行
            
            var hm dao.HelmHistory
            hm.Init(h.App().factory, h.GVR())
            if err := hm.Rollback(ctx, client.FQN(ns, n), rev); err != nil {
                h.App().Flash().Err(err)
            } else {
                h.App().Flash().Infof("Rollback succeeded for `%s`", n)
                // ✅ 从 goroutine 更新 UI 必须用 QueueUpdateDraw
                h.App().QueueUpdateDraw(func() {
                    h.Refresh()
                })
            }
        }()
    }, ...)
    
    return nil
}
```

### 2. 让 context 超时真正生效（配合 goroutine）

```go
func (h *HelmHistory) Rollback(ctx context.Context, path, rev string) error {
    ns, n := client.Namespaced(path)
    cfg, err := ensureHelmConfig(h.Client().Config().Flags(), ns)
    if err != nil {
        return err
    }
    ver, err := strconv.Atoi(rev)
    if err != nil {
        return fmt.Errorf("could not convert revision to a number: %w", err)
    }
    clt := action.NewRollback(cfg)
    clt.Version = ver
    
    // ✅ 通过 channel + select 支持超时
    done := make(chan error, 1)
    go func() {
        _, err := clt.Run(n)  // Helm SDK Run 返回 *release.Release
        done <- err
    }()
    
    select {
    case err := <-done:
        return err
    case <-ctx.Done():
        return fmt.Errorf("rollback timed out after %s: %w", 
            h.Client().Config().CallTimeout(), ctx.Err())
    }
}
```

### 3. 立即刷新最新数据（可选）

如果想让用户看到最新的历史版本，回滚成功后应该主动触发一次数据拉取：

```go
// 在回滚成功后
if err := h.GetModel().Refresh(h.GetContext()); err != nil {
    slog.Error("Failed to refresh history after rollback", slogs.Error, err)
}
```

而不是只调用 `h.Refresh()` 重绘旧数据。

---

## 为什么之前会误解"不阻塞 UI"？

在之前的 followup 分析中，我错误地认为"操作在确认对话框的回调 goroutine 中执行，不会阻塞整个 UI 主线程"。这个错误的来源是：

1. **混淆了"ShowConfirmAck 是异步的"和"回调是异步的"**：ShowConfirmAck 函数本身是立即返回的（非阻塞），但它注册的 ack 回调函数是在用户点击 OK 时**同步执行**的。

2. **没有理解 tview 的单线程事件循环模型**：tview 和大多数 UI 框架（Win32、Cocoa、Qt、JS 浏览器）一样，所有 UI 操作都必须在主线程（事件循环线程）中执行，回调自然也在这个线程。

3. **忽略了"回滚是同步函数调用"**：`clt.Run(n)` 本身就是同步阻塞的，没有 `go` 关键字就不会并发。

这个补正非常关键——如果开发者基于"不阻塞"的误解编写更多同步耗时操作，整个应用的响应性会非常差。
