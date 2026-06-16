# K9s 命令面板 → 资源跳转：解析、补全与历史回填协作机制

## 全景概览

用户在 k9s 命令面板中输入文本，最终跳转到目标资源视图，经历了以下核心阶段（**修正后的正确顺序**）：

```
用户按键 → Prompt UI → FishBuff(命令缓冲+补全) → CmdBuff(底层缓冲)
    → [补全刷新] SuggestionChanged → UI 重绘
    → [执行] App.keyboard → gotoCmd → gotoResource
        ├── NewInterpreter + grok()（分词+参数拆解，在 run 之前完成）
        └── Command.run（收到已解析的 Interpreter）
            ├── specialCmd（处理特殊命令）
            ├── viewMetaFor → Alias.Resolve（别名解析）
            ├── context 切换（@ctx 语法）
            ├── namespace 切换
            ├── 应用 filter/fuzzy/label selector
            └── exec → 视图创建与注入 → 页面展示 → cmdHistory.Push（历史写入）
```

> **核心纠正 1**：`BufferCompleted` 事件是纯 UI 重绘信号，不触发资源跳转。资源跳转的唯一触发器是 App 级别的 `tcell.KeyEnter` 键绑定 → `gotoCmd`。
>
> **核心纠正 2**：历史写入（`cmdHistory.Push`）发生在整个执行流程的**最后一步**——在解析、别名处理、视图创建与注入都成功完成之后，而不是之前。详见「阶段三：命令执行」。
>
> **核心纠正 3**：命令分词和参数拆解在 `NewInterpreter` 构造时通过 `grok()` 一次性完成，**在 `Command.run` 之前**。`run` 收到的 `*Interpreter` 已经是解析完毕的状态（`cmd` + `args` 已就绪），不需要在 `run` 内部再解析。详见「阶段三」。

---

## 关键纠正：BufferCompleted ≠ 跳转触发器

### 错误理解

旧的理解将 `BufferCompleted` 与 `gotoCmd` 混在一起，认为用户按 Enter 时 `Prompt.keyboard` 先处理（SetText + SetActive），然后 `BufferCompleted` 事件驱使跳转。这暗示了一条「事件驱动跳转」的路径。

### 正确理解

Enter 键按下后，**资源跳转由 App 级别的键绑定直接触发**，与 `BufferCompleted` 事件完全无关。

**证据链：**

1. **App 级 InputCapture 先于 Widget 级执行**。tview 的事件分发顺序：Application.InputCapture → 焦点组件的 InputCapture → 组件 InputHandler。

2. [view/app.go#L263](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L263) 注册了 `tcell.KeyEnter` → `gotoCmd`：

   ```go
   tcell.KeyEnter: ui.NewKeyAction("Goto", a.gotoCmd, false),
   ```

3. [view/app.go#L246-L252](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L246-L252) App 的 `keyboard` 方法在 InputCapture 中拦截：

   ```go
   func (a *App) keyboard(evt *tcell.EventKey) *tcell.EventKey {
       if k, ok := a.HasAction(ui.AsKey(evt)); ok && !a.Content.IsTopDialog() {
           return k.Action(evt)  // ← Enter 在此被 gotoCmd 消费，返回 nil
       }
       return evt
   }
   ```

4. `gotoCmd` 返回 nil（消费事件），**Prompt.keyboard 根本看不到 Enter 键**。

### 实际的 Enter 处理链

```
用户按 Enter
  │
  ▼ [tview 事件分发] Application.InputCapture 优先
  │
  ▼ App.keyboard(evt)
  │   └── HasAction(KeyEnter) → gotoCmd
  │       ├── CmdBuff.IsActive() && !CmdBuff.Empty() ? → YES
  │       ├── gotoResource(cmdText, "", true, true)  ← 跳转发生在这里
  │       ├── ResetCmd()
  │       │   └── cmdBuff.Reset()
  │       │       ├── ClearText(true) → fireBufferCompleted("") ← 纯 UI 通知
  │       │       ├── SetActive(false) → fireActive(false)       ← 隐藏 Prompt
  │       │       └── fireBufferCompleted("")                     ← 纯 UI 通知
  │       └── return nil  ← 事件被消费，不再传播
  │
  ✗ Prompt.keyboard 永远不会收到 Enter 事件
```

### Prompt.keyboard 的 Enter 分支在命令面板上下文中是死代码

[prompt.go#L168-L170](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go#L168-L170)：

```go
case tcell.KeyEnter, tcell.KeyCtrlE:
    p.model.SetText(p.model.GetText(), "", true)
    p.model.SetActive(false)
```

当 `cmdBuff` 激活且有内容时，这段代码**永远不会被执行**，因为 `gotoCmd` 已在 App 级 InputCapture 中消费了 Enter 事件。

这段代码只在以下场景生效：当 Prompt 绑定了**非命令缓冲**的模型（如 Browser 的 FilterBuffer），且该缓冲未在 App 级被拦截时。但即便在那条路径上，它也只是做「关闭输入框」的 UI 清理，不做跳转。

---

## 三大边界的精确定义

### 边界一：补全刷新（Suggestion）

**职责**：根据当前输入计算补全建议，更新 UI 显示。

**触发源**：`FishBuff.Add` / `FishBuff.Delete` → `Notify` → `suggestionFn`

**作用范围**：仅修改 `suggestions` 列表和 `suggestion` 字段，通过 `SuggestionChanged` 通知 Prompt 重绘灰色建议文本。**不修改 `buff`（用户实际输入），不触发任何命令执行。**

**时序**：每次字符增删后立即触发（同步调用 `suggestionFn`），与 `BufferCompleted` 的 100ms 延时无关。

**数据流向**：

```
FishBuff.Add(r)
  ├── CmdBuff.Add(r)          → buff 追加字符
  │   ├── fireBufferChanged   → Prompt.BufferChanged → 重绘（显示新字符）
  │   └── 100ms 后 fireBufferCompleted → Prompt.BufferCompleted → 重绘（无视觉差异）
  │
  └── Notify(false)           → suggestionFn(buff text)
      └── fireSuggestionChanged → Prompt.SuggestionChanged → 重绘（显示灰色建议）
```

> `BufferCompleted` 在此边界中仅触发 `Prompt.update`（重绘），语义上等价于 `BufferChanged`，两者在 Prompt 中的实现完全相同。`BufferCompleted` 的"完成"含义仅对 **FilterBuffer** 有语义差异（见下文对比）。

**建议的接受**：用户按 Tab/→/CtrlF 时，Prompt 将 `currentSuggestion` 追加到 `buff`：

```go
case tcell.KeyTab, tcell.KeyRight, tcell.KeyCtrlF:
    if s, ok := m.CurrentSuggestion(); ok {
        p.model.SetText(p.model.GetText()+s, "", true)  // buff 被修改
        m.ClearSuggestions()                              // 建议被清除
    }
```

接受建议是**修改输入内容的操作**，属于补全边界与输入边界的交叉点。

### 边界二：执行（Execution）

**职责**：将用户确认的命令文本解析、路由、创建视图并注入页面。

**触发源**：`App.keyboard` → `gotoCmd`（App 级 `tcell.KeyEnter` 键绑定）

**入口条件**：`CmdBuff.IsActive() && !CmdBuff.Empty()`

**完整执行顺序（精确到代码行）**：

```
gotoCmd
  ├── gotoResource(cmdText, "", true, true)             [app.go#L791]
  │   │
  │   ├── 0. NewInterpreter(cmdText) + grok()           [interpreter.go#L24-L33]
  │   │       ├── strings.Fields 分词                    [interpreter.go#L63]
  │   │       ├── c.cmd = strings.ToLower(ff[0])         [interpreter.go#L67]
  │   │       ├── 提取单引号内 label selector             [interpreter.go#L71-L80]
  │   │       └── newArgs(c, ff) → args map              [interpreter.go#L85]
  │   │   ← 此时 Interpreter 已完全解析，cmd + args 就绪
  │   │
  │   └── Command.run(p, ...)  [command.go#L176]  ← p 是已解析的 Interpreter
  │       ├── 1. specialCmd(p) [L177]                  ← 处理特殊命令
  │       │       (cow/quit/help/alias/xray/rbac/ctx/ns/dir)
  │       │
  │       ├── 2. viewMetaFor(p) [L180]
  │       │       └── alias.Resolve(p) [L319]           ← 别名解析
  │       │
  │       ├── 3. context 切换（@ctxName 语法）[L188-L211]
  │       │
  │       ├── 4. namespace 切换 [L213-L224]
  │       │
  │       ├── 5. 应用 filter/fuzzy/label selector [L226-L239]
  │       │
  │       └── 6. exec(p, gvr, co, clearStack, pushCmd) [L241]
  │               ├── comp.SetCommand(p) [L372]
  │               ├── Config.SetActiveView(v) [L376]
  │               ├── app.inject(comp) [L378]          ← 视图创建与注入
  │               │       └── comp.Init → PageStack.Push → comp.Start
  │               └── cmdHistory.Push(p.GetLine()) [L382]  ← 最后一步：历史写入
  │
  └── ResetCmd()
      └── cmdBuff.Reset()
          ├── ClearText(true) → fireBufferCompleted("")  ← 纯 UI 通知
          ├── SetActive(false) → fireActive(false)        ← 隐藏 Prompt
          └── fireBufferCompleted("")                      ← 纯 UI 通知
```

**关键纠正**：`NewInterpreter` + `grok()` 在 [interpreter.go#L24-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L24-L33) 中一次性完成分词和参数拆解，**发生在 `Command.run` 之前**（在 [gotoResource](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L792) 调用 `run` 时 `p` 已是解析完毕的状态）。`run` 内部的第一步是 `specialCmd`，不是解析。

**关键纠正**：`cmdHistory.Push` [command.go#L382](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L382) 是 `exec` 方法的**最后一个动作**，在 `app.inject`（视图创建与注入）成功完成之后才执行。历史写入是执行边界的收尾动作，而不是开始动作。

### 边界三：历史回填（History Backfill）

**职责**：在命令面板中提供历史命令作为补全建议，或通过快捷键直接跳转到历史命令。

**写入时机**：`Command.exec` → `cmdHistory.Push(p.GetLine())` [command.go#L381-L383](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L381-L383)

**读取时机（四个独立入口，按触发频率排序）**：

#### 入口 A：空输入时作为补全建议

[suggestCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L195-L227) 中：

```go
if s == "" {
    if a.cmdHistory.Empty() {
        return
    }
    return a.cmdHistory.List()  // ← 整个历史列表作为建议
}
```

这属于**补全刷新边界**的数据源之一。用户按 ↑/↓ 选择，按 Tab 接受（进入补全接受流程），再按 Enter 执行（进入执行边界）。

#### 入口 B：`[` 键 — 后退到上一条命令

**键绑定**：[view/app.go#L259](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L259)
```go
ui.KeyLeftBracket: ui.NewSharedKeyAction("Go Back", a.previousCommand, false),
```

**实现**：[previousCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L734-L745)
```go
func (a *App) previousCommand(evt *tcell.EventKey) *tcell.EventKey {
    // 关键：如果在命令模式下，不拦截，让 Prompt 自己处理
    if evt != nil && evt.Rune() == rune(ui.KeyLeftBracket) && a.Prompt().InCmdMode() {
        return evt
    }
    c, ok := a.cmdHistory.Back()
    if !ok {
        a.App.Flash().Warn("Can't go back any further")
        return evt
    }
    // pushCmd=false：避免重复写入历史
    a.gotoResource(c, "", true, false)
    return nil
}
```

**行为**：`cmdHistory.Back()` 移动 `currentIdx` 指针 -1，获取历史命令，调用 `gotoResource(..., pushCmd=false)` 跳转。

#### 入口 C：`]` 键 — 前进到下一条命令

**键绑定**：[view/app.go#L260](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L260)
```go
ui.KeyRightBracket: ui.NewSharedKeyAction("Go Forward", a.nextCommand, false),
```

**实现**：[nextCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L748-L761)
```go
func (a *App) nextCommand(evt *tcell.EventKey) *tcell.EventKey {
    if evt != nil && evt.Rune() == rune(ui.KeyRightBracket) && a.Prompt().InCmdMode() {
        return evt
    }
    c, ok := a.cmdHistory.Forward()
    if !ok {
        a.App.Flash().Warn("Can't go forward any further")
        return evt
    }
    // We go to the resource before updating the history so that
    // gotoResource doesn't add this command to the history
    a.gotoResource(c, "", true, false)
    return nil
}
```

**注释特别说明**：先跳转再更新历史（实际上 `Forward()` 已经移动了指针），核心是 `pushCmd=false`，避免 `gotoResource` 将这条命令再次添加到历史中。

#### 入口 D：`-` 键 — 切换到上一条命令（类似 `cd -`）

**键绑定**：[view/app.go#L261](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L261)
```go
ui.KeyDash: ui.NewSharedKeyAction("Last View", a.lastCommand, false),
```

**实现**：[lastCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L764-L776)
```go
func (a *App) lastCommand(evt *tcell.EventKey) *tcell.EventKey {
    if evt != nil && evt.Rune() == ui.KeyDash && a.Prompt().InCmdMode() {
        return evt
    }
    c, ok := a.cmdHistory.Top()
    if !ok {
        a.App.Flash().Warn("No previous view to switch to")
        return evt
    }
    // pushCmd=false：避免重复写入历史
    a.gotoResource(c, "", true, false)
    return nil
}
```

**行为**：`cmdHistory.Top()` 只取栈顶（不移动指针），在当前命令和上一条命令之间切换。

**历史快捷键的共同特征**：
- 都检查 `a.Prompt().InCmdMode()`，在命令模式下不拦截（让 Prompt 自己处理输入）
- 都调用 `gotoResource(..., pushCmd=false)`，**避免重复写入历史**
- 都直接进入执行边界，绕过命令面板和补全

### 三大边界交互图

```
┌─────────────────────────────────────────────────────────────┐
│                    补全刷新边界（Suggestion）                  │
│  触发：每次字符增删                                           │
│  输入：FishBuff.buff 当前文本                                 │
│  输出：suggestions 列表 → Prompt 灰色建议                      │
│  副作用：无（仅 UI 重绘）                                      │
│  ─────────────────────────────────────────────────────────── │
│  特殊：空输入时读取 cmdHistory.List() 作为建议源               │
│        ← 历史回填边界为补全刷新边界提供数据（入口 A）             │
└──────────────────────┬──────────────────────────────────────┘
                       │ Tab 接受建议
                       │ （修改 buff 内容）
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    执行边界（Execution）                       │
│  触发：App 级 tcell.KeyEnter → gotoCmd                      │
│  输入：CmdBuff.GetText() 确认后的文本                         │
│  输出：视图创建 + 页面注入                                     │
│  副作用：Config.SetActiveView（持久化当前视图）                  │
│         cmdBuff.Reset（清空缓冲、隐藏 Prompt）                 │
│         cmdHistory.Push（写入历史）← 历史回填边界的写入点         │
│  ─────────────────────────────────────────────────────────── │
│  执行顺序：0.NewInterpreter+grok(在run外) → 1.specialCmd → 2.别名解析 → 3.ctx切换     │
│           → 4.ns切换 → 5.filter/fuzzy/label → 6.inject       │
│           → 7.cmdHistory.Push（最后一步）                      │
└──────────────────────┬──────────────────────────────────────┘
                       │ exec → cmdHistory.Push
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    历史回填边界（History）                      │
│  写入：Command.exec → cmdHistory.Push                         │
│  读取：                                                       │
│    A: suggestCommand（空输入时）→ 补全刷新边界                 │
│    B: [ 键 → cmdHistory.Back() → gotoResource → 执行边界       │
│    C: ] 键 → cmdHistory.Forward() → gotoResource → 执行边界    │
│    D: - 键 → cmdHistory.Top() → gotoResource → 执行边界        │
│  副作用：SwitchNS 修正历史命令中的命名空间                      │
└─────────────────────────────────────────────────────────────┘
```

---

## CommandBuffer 与 FilterBuffer 的 BufferCompleted 语义差异

同一个 `BufferCompleted` 事件，在两种缓冲类型下有截然不同的语义：

### CommandBuffer（`:` 命令面板）

| 事件 | 监听者 | 行为 |
|------|--------|------|
| `BufferCompleted` | `Prompt.BufferCompleted` | 重绘 UI（`p.update`） |
| `BufferCompleted` | `App.BufferCompleted` | **空操作**（方法体为空） |
| `SuggestionChanged` | `Prompt.SuggestionChanged` | 重绘 UI（含灰色建议） |

`BufferCompleted` 在命令面板上下文中是**纯 UI 信号**，不驱动任何业务逻辑。

### FilterBuffer（`/` 过滤面板，在 Browser 中）

| 事件 | 监听者 | 行为 |
|------|--------|------|
| `BufferCompleted` | `Browser.BufferCompleted` | **更新 label selector**（`SetLabelSelector`） |
| `BufferCompleted` | `Prompt.BufferCompleted` | 重绘 UI |
| `BufferActive(false)` | `Browser.BufferActive` | **刷新数据模型**（`model.Refresh` + `UpdateUI`） |

在 FilterBuffer 中，`BufferCompleted` 有**业务语义**——它将用户输入的 label selector 应用到数据查询中。`BufferActive(false)` 触发数据刷新。这是 100ms 延时机制的真正意义：等用户停止输入 label selector 后再应用到查询，避免每次按键都触发 API 请求。

---

## 阶段一：输入捕获与缓冲

### 1.1 命令面板的激活

入口在 [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/app.go#L240-L248) 的 `activateCmd` 方法。当用户按 `:` 键时触发：

```go
func (a *App) activateCmd(evt *tcell.EventKey) *tcell.EventKey {
    if a.InCmdMode() {
        return evt
    }
    a.ResetPrompt(a.cmdBuff)
    a.cmdBuff.ClearText(true)
    return nil
}
```

`cmdBuff` 在 [app.go#L41](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/app.go#L41) 创建，类型为 `*model.FishBuff`，以 `:` 为热键，`CommandBuffer` 为缓冲类型：

```go
cmdBuff: model.NewFishBuff(':', model.CommandBuffer)
```

### 1.2 FishBuff —— 带 Fish-style 补全的命令缓冲

[fish_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/fish_buff.go) 内嵌了 `CmdBuff`，扩展了自动建议功能：

- **`suggestionFn`**：补全函数，由外部通过 `SetSuggestionFn` 注入
- **`suggestions`**：当前补全列表
- **`suggestionIndex`**：当前补全项索引

关键方法：
- `Add(r rune)`：添加字符 → 调用 `CmdBuff.Add` + `Notify` 触发补全计算
- `Delete()`：删除字符 → 调用 `CmdBuff.Delete` + `Notify` 触发补全计算
- `Notify(bool)`：调用 `suggestionFn` 得到补全列表，通过 `fireSuggestionChanged` 通知监听者
- `NextSuggestion()`/`PrevSuggestion()`：上下键切换补全项

### 1.3 CmdBuff —— 底层缓冲与事件分发

[cmd_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/cmd_buff.go) 核心职责：

- 维护 `buff []rune`（用户输入）和 `suggestion string`（补全建议）
- 通过 `BuffWatcher` 接口向监听者通知三种事件：
  - `BufferCompleted(text, suggestion)`：输入停顿后触发（100ms 延时），**纯 UI 重绘信号**
  - `BufferChanged(text, suggestion)`：输入即时变化
  - `BufferActive(state, kind)`：缓冲激活/失活

**延时完成机制的真实用途**：在 `CommandBuffer` 上下文中，延时 `BufferCompleted` 仅用于优化 UI 重绘频率（避免每次按键都重绘），**不驱动任何命令执行**。在 `FilterBuffer` 上下文中，延时机制有业务意义——等用户停顿后再应用 label selector 到数据查询。

### 1.4 Prompt UI —— 用户界面层

[prompt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go) 是终端中的命令行输入框，实现了 `BuffWatcher` 和 `SuggestionListener` 接口。

键盘处理在 [keyboard](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go#L144-L193) 方法中：

| 按键 | 行为 | 何时生效 |
|------|------|----------|
| 字符键 | `model.Add(r)` 添加字符 | 始终 |
| Backspace/Delete | `model.Delete()` 删除字符 | 始终 |
| Escape | 清空文本、失活 | 始终 |
| Enter/CtrlE | SetText + SetActive(false) | **仅 FilterBuffer 上下文**（CommandBuffer 下被 App 级 gotoCmd 拦截） |
| CtrlW/CtrlU | 清空文本 | 始终 |
| ↑/↓ | 切换补全建议 | 始终 |
| Tab/→/CtrlF | 接受当前补全，将建议追加到文本 | 始终 |

UI 渲染在 [write](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go#L236-L246) 中，将用户输入和灰色的补全建议拼接显示：

```
🐶> po[d]                    ← 用户输入 "po"，补全建议 "d"（灰色）
```

---

## 阶段二：补全（Suggestion）

### 2.1 补全函数的注册

在 [view/app.go#L133](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L133) 中注册：

```go
a.CmdBuff().SetSuggestionFn(a.suggestCommand())
```

### 2.2 suggestCommand —— 命令补全逻辑

[suggestCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L195-L227) 定义了补全策略：

```
输入为空 → 返回命令历史列表（历史回填边界提供数据）
输入非空 → 1. 匹配所有别名前缀（alias.Alias map keys）
           2. 调用 SuggestSubCommand 补全子命令（namespace/context）
```

### 2.3 SuggestSubCommand —— 子命令补全

[helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/helpers.go#L65-L108) 根据命令类型决定补全内容：

| 命令类型 | 补全内容 |
|----------|----------|
| xray | 命名空间名称 |
| context | 上下文名称 |
| 其他（有 namespace） | 命名空间名称 |
| 其他（有 context 标记 @） | 上下文名称 |
| 默认 | 上下文名称 |

`ShouldAddSuggest` 检查补全项是否以当前输入为前缀，若是则返回差异部分。

### 2.4 补全的接受

当用户按 Tab/→/CtrlF 时，Prompt 将当前建议追加到文本中：

```go
case tcell.KeyTab, tcell.KeyRight, tcell.KeyCtrlF:
    if s, ok := m.CurrentSuggestion(); ok {
        p.model.SetText(p.model.GetText()+s, "", true)
        m.ClearSuggestions()
    }
```

---

## 阶段三：命令执行

### 3.1 gotoCmd —— Enter 键触发跳转（唯一入口）

[view/app.go#L664-L672](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L664-L672)：

```go
func (a *App) gotoCmd(evt *tcell.EventKey) *tcell.EventKey {
    if a.CmdBuff().IsActive() && !a.CmdBuff().Empty() {
        a.gotoResource(a.GetCmd(), "", true, true)  // 先跳转
        a.ResetCmd()                                 // 再清理缓冲
        return nil                                    // 消费事件
    }
    return evt  // 不消费，传递给下层
}
```

### 3.2 gotoResource —— 解析器创建与命令调度

[gotoResource](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L791-L797) 负责**先创建解析器完成分词拆解，再进入命令调度**：

```go
func (a *App) gotoResource(c, path string, clearStack, pushCmd bool) {
    err := a.command.run(cmd.NewInterpreter(c), path, clearStack, pushCmd)
    //                     ^^^^^^^^^^^^^^^^^^^^
    //                     NewInterpreter(c) 在此处构造，构造时自动调用 grok()
    //                     完成分词和参数拆解，返回已解析的 *Interpreter
    ...
}
```

[NewInterpreter](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L24-L33) 构造函数：

```go
func NewInterpreter(s string, aliases ...string) *Interpreter {
    c := Interpreter{
        line:    s,
        args:    make(args),
        aliases: aliases,
    }
    c.grok()     // ← 构造时立即调用，完成分词 + 参数拆解

    return &c    // ← 返回的 Interpreter 已是解析完毕的状态
}
```

**关键**：`grok()` 在构造函数内被调用，因此 `Command.run` 收到的 `p *Interpreter` 已经拥有完整的 `cmd` 和 `args`，不需要在 `run` 内部再做任何解析。

### 3.3 Command.run —— 命令调度中心（**精确顺序，每步都有代码行号**）

[command.go#L176-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L176-L242) 收到的 `p` 已是解析完毕的 Interpreter，**严格按以下顺序执行**：

```
run(p *Interpreter, fqn string, clearStack, pushCmd bool)
  │
  ├── [L177] 1. specialCmd(p)
  │          → 处理特殊命令：cow/quit/help/alias/xray/rbac/context/ns/dir
  │
  ├── [L180] 2. viewMetaFor(p) → Alias.Resolve(p)
  │          → 别名解析，将短名映射为 GVR
  │
  ├── [L188-L211] 3. context 切换（@ctxName 语法）
  │          → 如果命令包含 @ctx 且 ctx 不同，切换上下文
  │
  ├── [L213-L224] 4. namespace 切换
  │          → 如果资源是命名空间级别的，切换到指定 ns
  │
  ├── [L226-L239] 5. 应用 filter/fuzzy/label selector
  │          → co.SetFilter / co.SetLabelSelector
  │
  └── [L241] 6. exec(p, gvr, co, clearStack, pushCmd)
              │
              ├── [L372] comp.SetCommand(p)
              ├── [L376] Config.SetActiveView(v)
              ├── [L378] app.inject(comp, clearStack) → 视图创建与注入
              │       └── comp.Init → PageStack.Push → comp.Start
              │
              └── [L382] cmdHistory.Push(p.GetLine())  ← 最后一步：历史写入
```

> **关键注意**：`cmdHistory.Push` [L382](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L382) 是 `exec` 方法的最后一个动作，在 `app.inject` 成功完成之后才执行。历史写入是**成功执行的确认动作**，而不是开始动作。

### 3.4 exec —— 执行命令并管理历史

[command.go#L352-L387](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L352-L387)：

```go
func (c *Command) exec(p *cmd.Interpreter, gvr *client.GVR, comp model.Component, clearStack, pushCmd bool) (err error) {
    defer func() { /* panic recovery */ }()

    if comp == nil { return ... }
    comp.SetCommand(p)                          // [L372]

    if clearStack {
        v := contextRX.ReplaceAllString(p.GetLine(), "")
        c.app.Config.SetActiveView(v)           // [L376]
    }
    if err := c.app.inject(comp, clearStack); err != nil {  // [L378]
        return err
    }
    if pushCmd {
        c.app.cmdHistory.Push(p.GetLine())      // [L382] 最后一步才写入历史
    }
    slog.Debug("History (exec)", slogs.Stack, strings.Join(c.app.cmdHistory.List(), "|"))

    return
}
```

### 3.5 inject —— 页面栈管理

[app.go#L799-L814](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L799-L814)：

```go
func (a *App) inject(c model.Component, clearStack bool) error {
    ctx := context.WithValue(context.Background(), internal.KeyApp, a)
    if err := c.Init(ctx); err != nil { ... }
    if clearStack {
        a.Content.Clear()
    }
    a.Content.Push(c)
    return nil
}
```

`PageStack.Push` 会触发 `StackPushed` 回调，调用 `c.Start()` 并聚焦。

---

## 阶段四：命令解析（Interpreter）

### 4.1 Interpreter 结构

[interpreter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go) 解析用户输入字符串为结构化命令：

```
Interpreter {
    line    string     // 原始输入
    cmd     string     // 命令部分（第一个词，小写）
    aliases []string   // 命令的别名链
    args    args       // 解析后的参数 map
}
```

### 4.2 grok() 解析流程（由 NewInterpreter 构造函数自动调用）

[grok](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L62-L86) 在 `NewInterpreter` 构造时自动执行（[interpreter.go#L30](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L30)），不需要外部显式调用。解析流程：

1. 用 `strings.Fields` 分词
2. 第一个词作为 `cmd`（转小写）
3. 在剩余部分中提取单引号包裹的 label selector
4. 将剩余词传给 `newArgs` 解析为参数

> **注意**：`grok()` 还在 `Interpreter.Reset`（[interpreter.go#L143-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L143-L152)）中被调用，用于别名链展开后重新解析命令行。

### 4.3 args 参数解析

[args.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/args.go) 中 `newArgs` 根据词的前缀分类：

| 前缀 | 键 | 示例 |
|------|-----|------|
| `-f` | `fuzzy` | `-f nginx` |
| `/` | `filter` | `/running` |
| `@` | `context` | `@prod` |
| 包含 `=`/`==`/`!=`/` in `/` notin ` | `labels` | `'app=nginx'` |
| 其他 | 取决于命令类型 | `default`（namespace） |

命令类型对默认参数的影响：

- **context 命令**：默认参数 → `context`
- **dir 命令**：默认参数 → `topic`（路径）
- **xray 命令**：第一个默认参数 → `topic`，第二个 → `ns`
- **其他**：默认参数 → `ns`

### 4.4 命令类型判定

[types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/types.go) 定义了各类命令的关键字集合：

| 类型 | 关键字 |
|------|--------|
| context | `ctx`, `context`, `contexts` |
| namespace | `ns`, `namespace`, `namespaces` |
| dir | `dir`, `dirs`, `d`, `ls` |
| bail | `q`, `q!`, `qa`, `Q`, `quit`, `exit` |
| help | `?`, `h`, `help` |
| alias | `a`, `alias`, `aliases` |
| xray | `x`, `xr`, `xray` |

---

## 阶段五：别名解析（Alias Resolution）

### 5.1 别名体系

别名是一个多层映射系统：

```
用户输入的短名 → Alias map → GVR（GroupVersionResource）
```

别名来源有三（按加载顺序）：

1. **默认别名**（[config/alias.go#L181-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/alias.go#L181-L199)）：硬编码的基础别名，如 `h`→help, `q`→quit, `ctx`→context 等
2. **全局别名文件**（`AppAliasesFile`）：用户自定义的 YAML 别名
3. **上下文别名文件**（`ContextAliasesPath`）：特定上下文的别名

K8s 资源的别名在 [dao/alias.go#L77-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/dao/alias.go#L77-L122) 中动态生成：
- GVR 全称（如 `v1/pods`）
- 资源名（如 `pods`）
- 单数名（如 `pod`）
- 短名（如 `po`）

### 5.2 Alias.Resolve —— 别名链解析

[Resolve](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/alias.go#L85-L107) 支持别名链（inception）和参数合并：

```
用户输入: "pdal blee"
别名链:   pdal → pdl → "pod @fred 'app=blee' default"
最终解析:  v1/pods @fred 'app=blee' blee
```

解析逻辑：
1. `p.Cmd()` 查别名表得到 GVR
2. 若 GVR 是 K8s 资源（`IsK8sRes()`）：将命令行中的短名替换为 GVR 全称，记录别名
3. 若 GVR 是别名（`IsAlias()`）：循环解析，每次将别名展开为完整命令行，通过 `Merge` 合并原始参数

**参数合并**：`Interpreter.Merge` 用展开后的命令替换 cmd 和 args，但保留用户原始输入的参数值。

### 5.3 viewMetaFor —— GVR → 视图映射

[command.go#L315-L334](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L315-L334)：

```go
func (c *Command) viewMetaFor(p *cmd.Interpreter) (*client.GVR, *MetaViewer, *cmd.Interpreter, error) {
    gvr, ok := c.alias.Resolve(p)
    ...
    v := MetaViewer{
        viewerFn: func(gvr *client.GVR) ResourceViewer {
            return NewScaleExtender(NewOwnerExtender(NewBrowser(gvr)))
        },
    }
    if mv, ok := customViewers[gvr]; ok {
        v = mv
    }
    return gvr, &v, p, nil
}
```

`customViewers` 在 [registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/registrar.go) 中注册，为特定 GVR 提供定制化的查看器（如 Pod→NewPod, Service→NewService 等）。未注册的 GVR 使用默认的 `NewBrowser`。

---

## 阶段六：视图创建与注入

### 6.1 componentFor —— 创建视图组件

[command.go#L336-L350](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L336-L350)：

```go
func (*Command) componentFor(gvr *client.GVR, fqn string, v *MetaViewer) ResourceViewer {
    var view ResourceViewer
    if v.viewerFn != nil {
        view = v.viewerFn(gvr)
    } else {
        view = NewBrowser(gvr)
    }
    view.SetInstance(fqn)
    if v.enterFn != nil {
        view.GetTable().SetEnterFn(v.enterFn)
    }
    return view
}
```

---

## 阶段七：历史回填（History）

### 7.1 History 结构

[history.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/history.go) 实现了一个命令历史栈：

- `commands []string`：历史命令列表
- `currentIdx int`：当前位置索引
- `limit int`：最大容量（默认 20）

### 7.2 历史写入（执行边界的最后一步）

命令执行成功后在 `exec` 的末尾调用 `cmdHistory.Push(p.GetLine())` [command.go#L381-L383](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L381-L383)：

```go
if pushCmd {
    c.app.cmdHistory.Push(p.GetLine())  // 仅当 pushCmd=true 时写入
}
```

**Push 规则**：
- 空命令不推入
- 超过 limit 不推入
- 与栈顶相同的命令不推入（去重）
- 推入后截断当前位置之后的历史（类似浏览器前进栈清除）

### 7.3 历史回填触发的四个入口

#### 入口 A：命令面板空输入时（补全刷新边界）

在 `suggestCommand` 中，当输入为空时返回历史列表作为补全建议：

```go
if s == "" {
    if a.cmdHistory.Empty() {
        return
    }
    return a.cmdHistory.List()
}
```

用户按 ↑/↓ 可在历史建议中切换，按 Tab 接受，按 Enter 执行跳转。

#### 入口 B：`[` 键 — 后退（执行边界快捷入口）

[previousCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L734-L745)：
- 检查 `InCmdMode()` → 命令模式下不拦截
- `cmdHistory.Back()` → 指针 -1
- `gotoResource(c, "", true, false)` → `pushCmd=false` 避免重复写入历史

#### 入口 C：`]` 键 — 前进（执行边界快捷入口）

[nextCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L748-L761)：
- 检查 `InCmdMode()` → 命令模式下不拦截
- `cmdHistory.Forward()` → 指针 +1
- `gotoResource(c, "", true, false)` → `pushCmd=false` 避免重复写入历史
- 代码注释特别说明：先跳转再更新历史（实际上 Forward 已移动指针），目的是不重复写入

#### 入口 D：`-` 键 — 切换（执行边界快捷入口）

[lastCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L764-L776)：
- 检查 `InCmdMode()` → 命令模式下不拦截
- `cmdHistory.Top()` → 取栈顶，不移动指针
- `gotoResource(c, "", true, false)` → `pushCmd=false` 避免重复写入历史

### 7.4 History.SwitchNS —— 命名空间切换时的历史修正

[history.go#L44-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/history.go#L44-L57)：

当命名空间切换时，`SwitchNS` 将栈顶命令中的命名空间替换为新命名空间，并推入一条新记录。这确保历史命令与当前命名空间保持一致。

---

## 阶段八：资源跳转后的 Enter 行为（Custom Jump）

### 8.1 默认 Enter 行为

在 Browser 视图中按 Enter，[enterCmd](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/browser.go#L454-L476) 默认调用 `describeResource` 打开 YAML 详情。

### 8.2 Custom Jump 机制

如果当前 GVR 在 CustomJumps 配置中存在规则，则优先执行自定义跳转：

```go
if rule, ok := b.App().CustomJumps().GetRule(b.GVR()); ok {
    if err := customJump(b.app, b.GVR(), path, rule); err != nil { ... }
    return nil
}
```

[custom_jump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/custom_jump.go) 的处理流程：

1. **获取源资源对象**：`fetchResourceObject` 通过 DAO accessor 获取完整的 Unstructured 对象
2. **解析目标 GVR**：`client.NewGVR(rule.TargetGVR)`
3. **确定目标命名空间**：`determineTargetNamespace` 根据规则中的 `TargetNamespace` 字段
   - 空 → 使用源资源命名空间
   - `"all"` → 所有命名空间
   - 包含 `{{` → Go 模板渲染
   - 其他 → 字面值
4. **构建 Label Selector**：支持 Go 模板，用源资源数据渲染
5. **构建 Field Selector**：同上
6. **创建 Browser 视图并注入**：`app.inject(v, false)`

---

## 完整数据流图（修正版，顺序精确）

```
用户按 ":"
  │
  ▼
App.activateCmd → ResetPrompt(FishBuff) → FishBuff.SetActive(true)
  │                                              │
  │                                              ▼ BuffWatcher.BufferActive(true)
  │                                         Prompt.activate() → 显示输入框
  │
  ▼ 用户输入字符
FishBuff.Add(r)
  │
  ├── CmdBuff.Add(r)
  │   ├── fireBufferChanged → Prompt.BufferChanged → 重绘
  │   └── 100ms 后 fireBufferCompleted → Prompt.BufferCompleted → 重绘（无业务效果）
  │
  └── Notify → suggestionFn(buff)
      ├── 空 → cmdHistory.List()（历史回填入口 A）
      └── 非空 → alias 前缀匹配 + SuggestSubCommand
      └── fireSuggestionChanged → Prompt.SuggestionChanged → 灰色建议显示
  │
  ▼ 用户按 Tab
Prompt.keyboard → SetText(buff+suggestion) → 建议追加到 buff → 重绘
  │
  ▼ 用户按 Enter
  │
  ▼ [tview] Application.InputCapture 优先执行
  │
  ▼ App.keyboard → HasAction(KeyEnter) → gotoCmd
  │   ├── CmdBuff.IsActive() && !Empty()? → YES
  │   ├── gotoResource(cmdText, "", true, true)         ← 执行边界开始
  │   │   ├── NewInterpreter(cmdText) + grok()           ← 步骤 0：分词+参数拆解（在 run 之前）
  │   │   │       ├── strings.Fields 分词 → cmd
  │   │   │       ├── 提取 label selector
  │   │   │       └── newArgs() → args map
  │   │   │   ← Interpreter 已完全解析
  │   │   │
  │   │   └── Command.run(p)  ← p 是已解析的 Interpreter
  │   │       ├── [L177] 1. specialCmd()
  │   │       ├── [L180] 2. viewMetaFor() → Alias.Resolve() → GVR
  │   │       ├── [L188] 3. context 切换（@ctx）
  │   │       ├── [L213] 4. namespace 切换
  │   │       ├── [L226] 5. filter/fuzzy/labels 应用
  │   │       └── [L241] 6. exec()
  │   │           ├── [L372] comp.SetCommand(interpreter)
  │   │           ├── [L376] Config.SetActiveView()
  │   │           ├── [L378] app.inject(comp) → PageStack.Push → comp.Start
  │   │           └── [L382] cmdHistory.Push()              ← 最后一步：历史写入
  │   │
  │   └── ResetCmd()
  │       └── cmdBuff.Reset()
  │           ├── ClearText(true) → fireBufferCompleted("") → Prompt 重绘（清空）
  │           ├── SetActive(false) → fireActive(false) → Prompt 隐藏 + App 移除 Prompt
  │           └── fireBufferCompleted("") → Prompt 重绘（清空）
  │
  └── return nil  ← 事件被消费，Prompt.keyboard 永远看不到 Enter

─────────────────────────────────────────────────────────────
历史快捷键入口（任何时刻可触发，不经过命令面板）：
  │
  ├── 按 [ → previousCommand → cmdHistory.Back() → gotoResource(..., pushCmd=false)
  ├── 按 ] → nextCommand → cmdHistory.Forward() → gotoResource(..., pushCmd=false)
  └── 按 - → lastCommand → cmdHistory.Top() → gotoResource(..., pushCmd=false)
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [ui/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/app.go) | UI 应用框架，FishBuff 创建、命令面板激活、BufferCompleted 空实现 |
| [ui/prompt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go) | 命令输入框 UI，键盘事件处理，补全建议渲染 |
| [model/cmd_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/cmd_buff.go) | 底层命令缓冲，事件分发，延时完成机制 |
| [model/fish_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/fish_buff.go) | Fish-style 自动补全缓冲 |
| [model/history.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/history.go) | 命令历史栈，前/后导航 |
| [view/cmd/interpreter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go) | 命令解析器，将输入字符串结构化 |
| [view/cmd/args.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/args.go) | 参数解析，按前缀和命令类型分类 |
| [view/cmd/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/types.go) | 命令类型关键字定义 |
| [view/cmd/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/helpers.go) | 补全辅助，SuggestSubCommand |
| [view/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go) | 应用视图，gotoCmd/gotoResource/历史导航/suggestCommand |
| [view/command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go) | 命令调度中心，run/specialCmd/viewMetaFor/exec（历史写入在 L382） |
| [config/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/alias.go) | 别名配置，Resolve 解析与别名链 |
| [dao/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/dao/alias.go) | 别名数据访问，动态生成 K8s 资源别名 |
| [view/browser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/browser.go) | 资源浏览器视图，FilterBuffer 的 BufferCompleted 有业务语义 |
| [view/custom_jump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/custom_jump.go) | 自定义跳转，模板渲染 label/field selector |
| [config/jumps.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/jumps.go) | 自定义跳转规则配置 |
| [view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/registrar.go) | GVR → 自定义视图注册表 |
| [view/page_stack.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/page_stack.go) | 页面栈，Push/Pop 管理 Start/Stop 生命周期 |
