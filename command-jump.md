# K9s 命令面板 → 资源跳转：解析、补全与历史回填协作机制

## 全景概览

用户在 k9s 命令面板中输入文本，最终跳转到目标资源视图，经历了以下核心阶段：

```
用户按键 → Prompt UI → FishBuff(命令缓冲+补全) → CmdBuff(底层缓冲) 
    → BufferCompleted 事件 → App.gotoCmd → Command.run 
    → Interpreter 解析 → Alias 解析 → 视图创建与注入 → 页面展示
```

---

## 阶段一：输入捕获与缓冲

### 1.1 命令面板的激活

入口在 [app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/app.go#L240-L248) 的 `activateCmd` 方法。当用户按 `:` 键时触发：

```go
func (a *App) activateCmd(evt *tcell.EventKey) *tcell.EventKey {
    if a.InCmdMode() {
        return evt
    }
    a.ResetPrompt(a.cmdBuff)     // 清空缓冲、设置 Prompt 模型为 FishBuff、激活
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
  - `BufferCompleted(text, suggestion)`：输入完成（延时触发）
  - `BufferChanged(text, suggestion)`：输入变化
  - `BufferActive(state, kind)`：缓冲激活/失活

**延时完成机制**：`Add` 方法在用户输入后启动 100ms 定时器，定时器到期才触发 `BufferCompleted`。这防止了每次按键都触发命令执行，等待用户停止输入后才视为"完成"。

### 1.4 Prompt UI —— 用户界面层

[prompt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go) 是终端中的命令行输入框，实现了 `BuffWatcher` 和 `SuggestionListener` 接口。

键盘处理在 [keyboard](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go#L144-L193) 方法中：

| 按键 | 行为 |
|------|------|
| 字符键 | `model.Add(r)` 添加字符 |
| Backspace/Delete | `model.Delete()` 删除字符 |
| Escape | 清空文本、失活 |
| Enter/CtrlE | 将当前文本设为最终文本、失活 |
| CtrlW/CtrlU | 清空文本 |
| ↑/↓ | 切换补全建议 |
| Tab/→/CtrlF | 接受当前补全，将建议追加到文本 |

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
输入为空 → 返回命令历史列表
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

### 3.1 gotoCmd —— Enter 键触发跳转

[view/app.go#L664-L672](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L664-L672)：

```go
func (a *App) gotoCmd(evt *tcell.EventKey) *tcell.EventKey {
    if a.CmdBuff().IsActive() && !a.CmdBuff().Empty() {
        a.gotoResource(a.GetCmd(), "", true, true)
        a.ResetCmd()
        return nil
    }
    return evt
}
```

### 3.2 gotoResource → Command.run

[gotoResource](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go#L791-L797) 是简单委托：

```go
func (a *App) gotoResource(c, path string, clearStack, pushCmd bool) {
    err := a.command.run(cmd.NewInterpreter(c), path, clearStack, pushCmd)
    ...
}
```

### 3.3 Command.run —— 命令调度中心

[command.go#L176-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L176-L242) 是整个跳转的核心调度：

```
run(p *Interpreter, fqn string, clearStack, pushCmd bool)
  ├─ specialCmd(p) → 处理特殊命令（cow/quit/help/alias/xray/rbac/context/ns/dir）
  ├─ viewMetaFor(p) → 通过 Alias 解析获取 GVR + MetaViewer
  ├─ 处理 context 切换（@ctxName 语法）
  ├─ 处理 namespace 切换
  ├─ 应用 filter/fuzzy/label selector
  └─ exec(p, gvr, component, clearStack, pushCmd)
```

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

### 4.2 grok() 解析流程

[grok](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go#L62-L86) 方法：

1. 用 `strings.Fields` 分词
2. 第一个词作为 `cmd`（转小写）
3. 在剩余部分中提取单引号包裹的 label selector
4. 将剩余词传给 `newArgs` 解析为参数

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

### 6.2 exec —— 执行命令并管理历史

[command.go#L352-L387](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go#L352-L387)：

```go
func (c *Command) exec(p *cmd.Interpreter, gvr *client.GVR, comp model.Component, clearStack, pushCmd bool) error {
    comp.SetCommand(p)
    if clearStack {
        v := contextRX.ReplaceAllString(p.GetLine(), "")
        c.app.Config.SetActiveView(v)
    }
    if err := c.app.inject(comp, clearStack); err != nil {
        return err
    }
    if pushCmd {
        c.app.cmdHistory.Push(p.GetLine())
    }
    ...
}
```

关键步骤：
1. `comp.SetCommand(p)`：将解析后的 Interpreter 传递给组件
2. `Config.SetActiveView(v)`：持久化当前视图到配置（去掉 @context 部分）
3. `app.inject(comp, clearStack)`：将组件推入页面栈
4. `cmdHistory.Push(p.GetLine())`：将命令推入历史

### 6.3 inject —— 页面栈管理

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

## 阶段七：历史回填（History）

### 7.1 History 结构

[history.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/history.go) 实现了一个命令历史栈：

- `commands []string`：历史命令列表
- `currentIdx int`：当前位置索引
- `limit int`：最大容量（默认 20）

### 7.2 历史写入

命令执行成功后在 `exec` 中调用 `cmdHistory.Push(p.GetLine())`。

Push 规则：
- 空命令不推入
- 超过 limit 不推入
- 与栈顶相同的命令不推入（去重）
- 推入后截断当前位置之后的历史（类似浏览器前进栈清除）

### 7.3 历史回填触发

回填有两个入口：

**入口一：命令面板空输入时**

在 `suggestCommand` 中，当输入为空时返回历史列表作为补全建议：

```go
if s == "" {
    if a.cmdHistory.Empty() {
        return
    }
    return a.cmdHistory.List()
}
```

用户按 ↑/↓ 可在历史建议中切换，按 Tab 接受。

**入口二：快捷键导航**

| 快捷键 | 方法 | 行为 |
|--------|------|------|
| `[` | `previousCommand` | `cmdHistory.Back()` + `gotoResource` |
| `]` | `nextCommand` | `cmdHistory.Forward()` + `gotoResource` |
| `-` | `lastCommand` | `cmdHistory.Top()` + `gotoResource` |

这些方法直接跳转到历史命令对应的资源视图，不经过命令面板。

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

## 完整数据流图

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
Prompt.keyboard → FishBuff.Add(rune)
  │                     │
  │                     ├── CmdBuff.Add → BufferChanged 事件
  │                     │                    └── Prompt.BufferChanged → 重绘
  │                     │
  │                     └── 100ms 后 → BufferCompleted 事件
  │                                        └── Prompt.BufferCompleted → 重绘
  │
  │                FishBuff.Notify → suggestionFn(当前文本)
  │                     │
  │                     ├── 文本为空 → 返回 cmdHistory.List()
  │                     └── 文本非空 → 遍历 alias 前缀匹配 + SuggestSubCommand
  │                                       │
  │                                       ▼
  │                               fireSuggestionChanged → Prompt.SuggestionChanged
  │                                       │
  │                                       ▼ 灰色建议文本显示
  │
  ▼ 用户按 Tab
Prompt.keyboard → 接受建议追加到文本
  │
  ▼ 用户按 Enter
Prompt.keyboard → SetText(最终文本, "", true) + SetActive(false)
  │                                     │
  │                                     ▼ BuffWatcher.BufferActive(false)
  │                                        Prompt.Deactivate → 隐藏输入框
  │
  ▼ App.gotoCmd (tcell.KeyEnter 绑定)
App.gotoResource(文本, "", true, true)
  │
  ▼
Command.run(NewInterpreter(文本), ...)
  │
  ├── Interpreter.grok() 解析命令
  │     ├── 分词 → cmd + args
  │     └── newArgs → {ns, topic, filter, fuzzy, labels, context}
  │
  ├── specialCmd() → 处理特殊命令
  │
  ├── Alias.Resolve(interpreter) → GVR
  │     ├── 查别名表 → GVR
  │     ├── IsK8sRes → 替换短名为 GVR 全称
  │     └── IsAlias → 递归展开，Merge 合并参数
  │
  ├── viewMetaFor → GVR + MetaViewer
  │     └── customViewers[GVR] 或 默认 Browser
  │
  ├── 处理 @context 切换
  ├── 处理 namespace 切换
  ├── 应用 filter/fuzzy/labels
  │
  └── exec(interpreter, gvr, component, ...)
        ├── comp.SetCommand(interpreter)
        ├── Config.SetActiveView(命令行)
        ├── app.inject(comp, clearStack)
        │     └── comp.Init → PageStack.Push → comp.Start
        └── cmdHistory.Push(命令行)
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [ui/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/app.go) | UI 应用框架，FishBuff 创建、命令面板激活 |
| [ui/prompt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/ui/prompt.go) | 命令输入框 UI，键盘事件处理，补全建议渲染 |
| [model/cmd_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/cmd_buff.go) | 底层命令缓冲，事件分发，延时完成机制 |
| [model/fish_buff.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/fish_buff.go) | Fish-style 自动补全缓冲 |
| [model/history.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/model/history.go) | 命令历史栈，前/后导航 |
| [view/cmd/interpreter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/interpreter.go) | 命令解析器，将输入字符串结构化 |
| [view/cmd/args.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/args.go) | 参数解析，按前缀和命令类型分类 |
| [view/cmd/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/types.go) | 命令类型关键字定义 |
| [view/cmd/helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/cmd/helpers.go) | 补全辅助，SuggestSubCommand |
| [view/app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/app.go) | 应用视图，gotoCmd/gotoResource/历史导航/suggestCommand |
| [view/command.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/command.go) | 命令调度中心，run/specialCmd/viewMetaFor/exec |
| [config/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/alias.go) | 别名配置，Resolve 解析与别名链 |
| [dao/alias.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/dao/alias.go) | 别名数据访问，动态生成 K8s 资源别名 |
| [view/browser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/browser.go) | 资源浏览器视图，enterCmd/CustomJump |
| [view/custom_jump.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/custom_jump.go) | 自定义跳转，模板渲染 label/field selector |
| [config/jumps.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/config/jumps.go) | 自定义跳转规则配置 |
| [view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/registrar.go) | GVR → 自定义视图注册表 |
| [view/page_stack.go](file:///d:/fz/0601-2/solo-dogfeeding/code/3-k9s/internal/view/page_stack.go) | 页面栈，Push/Pop 管理 Start/Stop 生命周期 |
