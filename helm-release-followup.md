# Helm Release 回滚失败处理——更正分析（Follow-up）

本文针对之前分析中三个理解偏差点进行代码级的精准验证和更正。

---

## 问题一：超时控制是否真的传到后端？

### 结论：**完全没有传递到后端，超时 context 是"死代码"**

### 代码调用链条追踪

从 View 层到 DAO 层的完整调用链：

**第 1 层（View）：创建带超时的 context**
[helm_history.go:108-116](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L108-L116)

```go
dialog.ShowConfirmAck(..., func() {
    // ① 创建了带超时的 context（看起来很专业）
    ctx, cancel := context.WithTimeout(
        context.Background(),
        h.App().Conn().Config().CallTimeout(), // 例如 10s
    )
    defer cancel()
    // ② 把 ctx 传下去
    if err := h.rollback(ctx, client.FQN(ns, n), rev); err != nil {
        h.App().Flash().Err(err)
    }
    ...
}, ...)
```

**第 2 层（View 内部包装）：原样透传**
[helm_history.go:121-130](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L121-L130)

```go
func (h *History) rollback(ctx context.Context, path, rev string) error {
    var hm dao.HelmHistory
    hm.Init(h.App().factory, h.GVR())
    // ③ 继续把 ctx 传给 DAO
    if err := hm.Rollback(ctx, path, rev); err != nil {
        return err
    }
    h.Refresh()
    return nil
}
```

**第 3 层（DAO）：第一个参数直接写 `_` 忽略！**
[helm_history.go:138-153](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L138-L153)

```go
// ④ 注意第一个参数是 _ ！！！直接把 context 丢了
func (h *HelmHistory) Rollback(_ context.Context, path, rev string) error {
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

    // ⑤ Helm SDK 的 Run 方法本身也不接受 context
    return clt.Run(n)
}
```

### 关键证据对比

| 位置 | 代码 | 含义 |
|------|------|------|
| 函数签名 | `Rollback(_ context.Context, ...)` | 用 `_` 明确表示"这个参数我不用" |
| 函数体内部 | 没有任何地方引用 context | 不存在超时取消逻辑 |
| Helm SDK 调用 | `clt.Run(n)` | Helm v3 SDK 的 `Rollback.Run()` 本身不接受 context 参数 |

### 深层原因

Helm v3 SDK 的 `action.Rollback.Run()` 方法签名就是 `func (r *Rollback) Run(name string) (*release.Release, error)`，**本身不支持 context 取消**。这是 Helm SDK 的设计限制，k9s 目前的版本也没有通过其他方式（如 goroutine + select + 超时 channel）来弥补这个限制。

### 实际影响

当 kube-apiserver 或 Helm 存储后端（如 Secret/ConfigMap）响应缓慢时：
- ❌ 回滚操作会**无限期阻塞**，不会因配置的 `CallTimeout` 而超时取消
- ❌ UI 上的 Flash 提示永远不会出现（因为一直卡在 `clt.Run(n)`）
- ✅ 但由于操作是在确认对话框的回调 goroutine 中执行，不会阻塞整个 UI 主线程

---

## 问题二：版本号解析失败的完整提示路径

### 结论：**空 revision 不会在 View 层提前拦截，而是一路漏到 DAO 层才报错**

### 解析失败时的执行流程

**场景**：假设历史列表的行 ID 格式异常（不含冒号），如 `default/my-release` 而不是 `default/my-release:3`

**第 1 步：View 层解析（不严谨）**
[helm_history.go:98-103](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L98-L103)

```go
ns, nrev := client.Namespaced(path) // path = "default/my-release"
tt := strings.Split(nrev, ":")       // tt = ["my-release"]，长度=1
n, rev := nrev, ""                   // 默认值：n = "my-release", rev = ""
if len(tt) == 2 {                    // ❌ 条件不满足，跳过
    n, rev = tt[0], tt[1]
}
// 执行完后：n = "my-release", rev = ""（空字符串！）
```

**第 2 步：确认对话框显示奇怪内容**
[helm_history.go:107](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L107)

```go
msg := fmt.Sprintf("RollingBack chart [yellow::b]%s[-::-] to release <[orangered::b]%s[-::-]>?", n, rev)
// 实际渲染出的消息："RollingBack chart my-release to release <>?"
//                                       rev 是空字符串 → 尖括号里什么都没有 ^^
```

用户会看到一个空白的版本号 `<>`，但仍可以点击 OK。

**第 3 步：DAO 层 Atoi 转换失败**
[helm_history.go:145-148](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/dao/helm_history.go#L145-L148)

```go
ver, err := strconv.Atoi(rev) // rev = ""，strconv.Atoi("") 返回错误
if err != nil {
    // 包装后的错误信息：
    // "could not convert revision to a number: strconv.Atoi: parsing "": invalid syntax"
    return fmt.Errorf("could not convert revision to a number: %w", err)
}
```

**第 4 步：Flash 展示给用户**
[helm_history.go:111-112](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/view/helm_history.go#L111-L112)

```go
if err := h.rollback(ctx, client.FQN(ns, n), rev); err != nil {
    h.App().Flash().Err(err) // 在底部状态栏显示红色错误
}
```

### 用户实际看到的错误提示

```
（红色）could not convert revision to a number: strconv.Atoi: parsing "": invalid syntax
```

这个提示包含了 Go 标准库的原始错误信息，对用户不太友好。理想情况下应该在 View 层就检查 `rev == ""` 并给出更清晰的提示，如 "无法从选中行解析出版本号，请检查历史列表数据"。

### 完整的失败路径图

```
path 中缺少冒号
    │
    ▼
View 层 strings.Split → len(tt) != 2 → rev = ""
    │
    ▼
ShowConfirmAck 显示空版本号 <...>（用户可能忽略并点 OK）
    │
    ▼
DAO 层 strconv.Atoi("") → 返回 error
    │
    ▼
View 层 Flash.Err() 红色提示 "could not convert revision to a number..."
```

---

## 问题三：确认后的刷新路径——Stop/Start/Refresh 的真实时序

### 结论：**Stop() 和 Start() 是在对话框弹出前同步执行的，Refresh() 是在用户点击 OK 后异步调用的。Stop/Start 对是"多余但无害"的。**

### 完整时序分析

首先必须理解 `ShowConfirmAck` 的本质——它是**异步非阻塞**的：

[confirm.go:16-67](file:///d:/fz/0601-2/solo-dogfeeding/code/14-k9s/internal/ui/dialog/confirm.go#L16-L67)

```go
func ShowConfirmAck(..., ack confirmFunc, cancel cancelFunc) {
    ...
    f.AddButton("OK", func() {      // ← 这只是注册一个回调函数
        if !accept { return }
        ack()                        // ← 用户点击 OK 时才执行
        dismissConfirm(pages)
        cancel()
    })
    ...
    pages.AddPage(confirmKey, modal, false, false)
    pages.ShowPage(confirmKey)       // ← 只是把对话框加到页面栈上，立即返回！
}
```

### rollbackCmd 的精确执行时序

时间轴（从上到下按执行顺序）：

```
t0  用户按下 R 键
 │
 ▼
t1  rollbackCmd() 被调用
 │
 ├─► L93-96:  检查选中行是否为空
 │
 ├─► L98-103: 解析 path → n, rev
 │
 ├─► L105:    h.Stop() ──────────────────────────────┐
 │     │                                              │
 │     └─► Browser.Stop()                             │
 │          ├─► cancelFn() 取消模型 Watch             │ 同步执行
 │          ├─► RemoveListener(b)                     │
 │          ├─► CmdBuff.RemoveListener                │
 │          └─► Table.Stop()                          │
 │                                                     │
 ├─► L106:    defer h.Start() ──► 注册到函数返回时执行 │
 │                                                     │
 ├─► L107-117: ShowConfirmAck(...)                    │
 │     │                                              │
 │     ├─► 构建 Form + Modal                          │
 │     ├─► 注册 OK 按钮回调 ack()                     │
 │     └─► pages.ShowPage(confirmKey) ← 立即返回      │
 │                                                     │
 ├─► 函数 rollbackCmd() return ───────────────────────┘
 │     │
 │     └─► defer h.Start() 被执行 ────────────────────┐
 │          │                                         │
 │          └─► Browser.Start()                       │ 同步执行
 │               ├─► switchNS 切换命名空间            │
 │               ├─► b.Stop() 再次清理（幂等？）      │
 │               ├─► AddListener(b)                   │
 │               ├─► Table.Start()                    │
 │               └─► GetModel().Watch(...) ──► 启动 Watch │
 │                                                     │
 t2  UI 线程继续运行，显示对话框                        │
     │                                                 │
     ├─► （用户看到对话框，思考中...）                  │
     │                                                 │
 t3  用户点击 OK 按钮（异步事件）                      │
     │                                                 │
     └─► ack() 回调被触发 ─────────────────────────────┘
          │
          ├─► 创建带超时的 context（但实际无效，参见问题一）
          │
          ├─► h.rollback(ctx, path, rev)
          │     │
          │     ├─► hm.Rollback(...) ──► Helm SDK 执行回滚
          │     │
          │     └─► L127: h.Refresh() ────────────────────────┐
          │          │                                         │
          │          └─► Browser 嵌入的 Table.Refresh()        │
          │               └─► ui.Table.Refresh()               │ 异步事件回调中执行
          │                    └─► 重绘表格组件 UI             │
          │
          ├─► Flash.Infof("Rollout restart in progress...")
          │
          └─► dismissConfirm() 关闭对话框
```

### Stop/Start 对的作用分析

在对话框弹出前调用 `h.Stop()`，本意应该是**暂停历史列表的自动刷新**，防止用户在操作过程中数据发生变化。

但是由于 `ShowConfirmAck` 是立即返回的：
- `h.Stop()` 在 t1 执行（关闭 Watch、移除 Listener）
- `defer h.Start()` 在 t1.5 执行（恢复 Watch、加回 Listener）

两者之间的时间差只有**几毫秒**（仅仅是构建对话框 UI 的时间），**根本覆盖不到用户思考和点击按钮的时间（t2 → t3）**。

**实际效果**：Stop/Start 这对调用基本等于什么都没做，只是短暂地重启了一下 Watch 循环，对用户操作期间没有任何保护。

### Refresh() 的真正作用

`t3` 时刻回滚成功后调用的 `h.Refresh()`，追溯其实现链路：

```
h.Refresh()
  → History 嵌入 ResourceViewer
    → ValueExtender 嵌入 Browser（NewValueExtender(NewBrowser(gvr))）
      → Browser 嵌入 *Table
        → Table 嵌入 ui.Table
          → ui.Table.Refresh()  [ui/table.go:598]
```

`ui.Table.Refresh()` 只是**触发 TView 组件的重绘**，并不会重新从 DAO 层拉取数据。

真正刷新数据的逻辑是 `b.Start()` → `model.Watch()` 启动后台轮询，它会在 RefreshRate（默认 2 秒）周期到达时自动调用 DAO.List() 更新表格。

所以：
- ✅ `h.Refresh()` 可以立即让已有数据重新渲染（但数据没变）
- ❌ 它不会立即重新查询 Helm 的历史版本列表
- ✅ 但由于 `h.Start()` 已经在 t1.5 恢复了 Watch，所以下一个轮询周期会自动拉取到回滚后的新版本

### 用户感知的刷新延迟

```
用户点击 OK → 回滚成功 → Flash 提示 → 表格仍显示旧数据
                                                          ↓ 等待 ~2 秒（默认 RefreshRate）
                                                    下一次 Watch 轮询触发
                                                          ↓
                                                    调用 HelmHistory.List()
                                                          ↓
                                                    列表更新，显示新增的 revision 记录
```

用户会看到"Rollout restart in progress"提示，但表格可能要等几秒才更新。这是正常的，因为不是实时推送。

---

## 总结：三处认知偏差更正

| 问题 | 之前的分析（错误/不完整） | 更正后的实际情况 |
|------|--------------------------|-----------------|
| **超时控制** | "回滚操作使用 context 超时控制" | ❌ context 参数名是 `_`，完全没用到；Helm SDK Run() 本身也不支持 context；超时设置是死代码 |
| **版本号解析失败** | "版本号解析失败 → 返回 error"的笼统描述 | ✅ View 层不提前拦截空 rev → 对话框显示空版本号 `<>` → 用户确认后才在 DAO 层 Atoi 报错 → 用户看到不友好的 Go 库原始错误信息 |
| **确认后刷新** | "回滚成功后调用 Refresh() 刷新历史列表" | ✅ Stop/Start 是在对话框弹出前同步执行的（仅间隔几毫秒），没有覆盖用户思考时间；Refresh() 是 TView 组件重绘，不重新拉数据；实际数据更新靠 Watch 轮询周期（~2 秒延迟） |

---

## 代码改进建议（可选）

### 1. 修复超时：让 context 真正生效

```go
// DAO 层改为使用 context
func (h *HelmHistory) Rollback(ctx context.Context, path, rev string) error {
    // ...
    // 通过 channel + goroutine 模拟超时取消
    done := make(chan error, 1)
    go func() {
        done <- clt.Run(n)
    }()
    select {
    case err := <-done:
        return err
    case <-ctx.Done():
        return fmt.Errorf("rollback timed out: %w", ctx.Err())
    }
}
```

### 2. View 层提前拦截 rev 空值

```go
// 在 rollbackCmd 解析完成后立即检查
tt := strings.Split(nrev, ":")
if len(tt) != 2 {
    h.App().Flash().Errf("invalid history row format %q (missing revision)", path)
    return evt
}
n, rev := tt[0], tt[1]
```

### 3. 把 Stop/Start 放到正确的位置

```go
// 应该在对话框回调内 Stop/Start，而不是在 ShowConfirmAck 外
dialog.ShowConfirmAck(..., func() {
    h.Stop()          // 确认后才暂停刷新
    defer h.Start()   // 操作完恢复刷新
    
    ctx, cancel := context.WithTimeout(...)
    defer cancel()
    if err := h.rollback(ctx, ...); err != nil {
        h.App().Flash().Err(err)
    }
    // 想立即刷新数据？应该调用 b.refresh() → b.Start() 重新 Watch
    b.refresh()
}, ...)
// 外面不要调用 Stop/Start
```

