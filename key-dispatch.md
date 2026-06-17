# K9s 按键分发流程深度解析

本文档详细分析 k9s 项目中按键事件从用户输入到最终视图动作执行的完整分发链路，包括全局键、视图键和动作注册的协作机制。

---

## 一、核心数据结构

### 1.1 键值定义层

**文件**: `internal/ui/key.go`

k9s 定义了一套完整的按键常量体系，将 rune（字符码）映射为 `tcell.Key` 类型：

```
数字键: Key0 ~ Key9 (ASCII 48-57)
字母键: KeyA ~ KeyZ (ASCII 97-122)
大写键: KeyShiftA ~ KeyShiftZ (ASCII 65-90)
特殊键: KeyHelp(?) / KeySlash(/) / KeyColon(:) / KeySpace(32) 等
```

通过 `init()` → `initKeys()` 函数在包加载时将这些键名注册到 tcell 的 `KeyNames` 映射表中，用于后续在菜单提示中显示可读名称。

**工具函数**: `AsKey()` —— 将 `*tcell.EventKey` 转换为统一的 `tcell.Key` 键值，处理普通键、Rune 字符、Alt 组合键等各种情况。

---

### 1.2 动作注册层

**文件**: `internal/ui/action.go`

核心类型定义：

| 类型 | 说明 |
|------|------|
| `ActionHandler` | `func(*tcell.EventKey) *tcell.EventKey` —— 按键处理器，返回 nil 表示已消费事件 |
| `ActionOpts` | 动作选项：Visible(可见) / Shared(共享) / Plugin(插件) / HotKey(热键) / Dangerous(危险) |
| `KeyAction` | 单个按键动作：描述 + 处理器 + 选项 |
| `KeyMap` | `map[tcell.Key]KeyAction` —— 键到动作的映射表 |
| `KeyActions` | 带读写锁的动作集合，提供 Add/Bulk/Merge/Get/Delete/Clear 等操作 |

**动作构造函数**:
- `NewKeyAction(d, a, visible)`: 基础动作，指定可见性
- `NewSharedKeyAction(d, a, visible)`: 共享动作（Shared=true，菜单提示中不重复显示）
- `NewKeyActionWithOpts(d, a, opts)`: 完整选项动作

**关键方法**:
- `Add(k, ka)`: 添加单个键动作
- `Bulk(km)`: 批量添加键映射
- `Merge(aa)`: 合并另一个 KeyActions（同名键会被覆盖）
- `Get(k)`: 根据键查找动作
- `ClearDanger()`: 清除所有标记为危险的动作
- `Hints()`: 生成菜单提示（排除 Shared=true 的共享键）

---

## 二、按键分发流程（两层拦截机制）

按键事件采用 **tview 输入捕获链** 实现分层处理，依次经过两层拦截：

```
用户按键 → tcell 终端输入 → tview 事件循环
                           ↓
              ┌─────────────────────────┐
              │  第一层: App 级全局键    │ → view.App.keyboard()
              │  (通过 SetInputCapture)  │
              └─────────────────────────┘
                           ↓ 未匹配则透传（返回原 evt）
              ┌─────────────────────────┐
              │  第二层: View 级视图键   │ → Table.keyboard()
              │  (通过 SetInputCapture)  │
              └─────────────────────────┘
                           ↓ 未匹配则透传（返回原 evt）
                    tview 组件内部默认处理
```

> **事件透传规则**: 每层的输入捕获函数如果返回 `nil` 表示事件已被消费，终止分发；如果返回原 `evt` 表示未处理，事件继续向下传递。

---

### 2.1 第一层：App 级全局键处理

**入口函数**: `internal/view/app.go` 中的 `keyboard()` 方法

```go
func (a *App) keyboard(evt *tcell.EventKey) *tcell.EventKey {
    if k, ok := a.HasAction(ui.AsKey(evt)); ok && !a.Content.IsTopDialog() {
        return k.Action(evt)
    }
    return evt
}
```

**设置捕获点**: `App.Init()` 中调用 `a.SetInputCapture(a.keyboard)`

#### 全局键注册来源

全局键存储在 `ui.App.actions` 中，由两部分合并组成：

**(1) ui.App 基础绑定** —— `internal/ui/app.go` 的 `bindKeys()` 方法

| 键 | 动作 | 说明 |
|----|------|------|
| `KeyColon(:)` | Cmd | 激活命令模式 |
| `CtrlR` | Redraw | 重绘界面 |
| `CtrlP` | Persist | 保存配置 |
| `CtrlU/CtrlQ` | Clear Filter | 清除过滤器（共享键） |

**(2) view.App 扩展绑定** —— `internal/view/app.go` 的 `bindKeys()` 方法

| 键 | 动作 | 说明 |
|----|------|------|
| `CtrlE` | ToggleHeader | 切换头部显示 |
| `CtrlG` | ToggleCrumbs | 切换面包屑显示 |
| `KeyHelp(?)` | Help | 帮助视图（共享键） |
| `KeyLeftBracket` | Go Back | 后退到历史视图 |
| `KeyRightBracket` | Go Forward | 前进到历史视图 |
| `KeyDash(-)` | Last View | 切换到上一个视图 |
| `CtrlA` | Aliases | 别名列表 |
| `Enter` | Goto | 执行命令栏跳转 |
| `CtrlC` | Quit | 退出程序 |

> **对话框屏蔽**: 只有当顶层不是对话框（`!IsTopDialog()`）时，全局键才生效。对话框期间会屏蔽全局快捷键，避免误操作。

---

### 2.2 第二层：View 级视图键处理

**入口函数**: `internal/view/table.go` 中的 `keyboard()` 方法

```go
func (t *Table) keyboard(evt *tcell.EventKey) *tcell.EventKey {
    key := evt.Key()
    // Shift+左右: 选择列（特殊处理）
    if evt.Modifiers()&tcell.ModShift != 0 { ... }
    // Up/Down: 直接透传给 tcell
    if key == tcell.KeyUp || key == tcell.KeyDown { return evt }
    // 查找并执行视图级动作
    if a, ok := t.Actions().Get(ui.AsKey(evt)); ok && !t.app.Content.IsTopDialog() {
        return a.Action(evt)
    }
    return evt
}
```

**设置捕获点**: `Table.Init()` 中调用 `t.SetInputCapture(t.keyboard)`

视图级按键存储在每个视图自己的 `KeyActions` 中（通过 `Actions()` 方法获取）。每个视图实例维护自己独立的键绑定集合。

---

## 三、视图键注册的六层叠加模型

视图键采用 **洋葱式叠加注册**，从底层到顶层依次经过 6 个层次，每层都可以向同一个 `KeyActions` 中添加或覆盖键绑定。

> **覆盖规则**: 后注册的键会覆盖先注册的同名键（本质是 `map[k] = v` 赋值）。

```
┌───────────────────────────────────────────────────────────┐
│  第 6 层: 动态加载层（热键 + 插件）                         │  ← 优先级最高
│    hotKeyActions() / pluginActions() —— 运行时从配置加载   │
├───────────────────────────────────────────────────────────┤
│  第 5 层: 具体资源层（Pod/Deploy/...）                     │
│    例如 Pod.bindKeys() —— 通过 AddBindKeysFn 注入           │
├───────────────────────────────────────────────────────────┤
│  第 4 层: 功能扩展层（Extender 装饰器链）                   │
│    外层 Extender 后注入 → 可以覆盖内层 Extender 的键        │
│    LogsExtender / PortForwardExtender / ...                │
├───────────────────────────────────────────────────────────┤
│  第 3 层: 浏览器动态层                                     │
│    Browser.refreshActions() —— 每次数据刷新时重建           │
├───────────────────────────────────────────────────────────┤
│  第 2 层: 浏览器基础层                                     │
│    Browser.bindKeys() —— 过滤/重置等浏览器通用键            │
├───────────────────────────────────────────────────────────┤
│  第 1 层: 表格基础层                                       │
│    Table.bindKeys() —— 标记/排序/过滤模式等通用表格键       │  ← 优先级最低
└───────────────────────────────────────────────────────────┘
```

---

### 3.1 第 1 层：表格基础层

**文件**: `internal/view/table.go` 的 `bindKeys()` 方法

**注册时机**: `Table.Init()` 中调用 `t.bindKeys()` —— 只执行一次

这是所有表格视图的基础键，提供表格通用功能：

| 键 | 动作 | 说明 |
|----|------|------|
| `KeyHelp(?)` | Help | 帮助（共享键） |
| `KeySpace` | Mark | 标记/取消标记当前行 |
| `CtrlSpace` | Mark Range | 范围标记 |
| `Ctrl\` | Marks Clear | 清除所有标记 |
| `CtrlS` | Save | 保存表格到文件 |
| `KeySlash(/)` | Filter Mode | 激活过滤模式 |
| `CtrlZ` | Toggle Faults | 切换故障显示 |
| `CtrlW` | Toggle Wide | 切换宽模式 |
| `ShiftN` | Sort Name | 按名称排序 |
| `ShiftA` | Sort Age | 按年龄排序 |
| `ShiftS` | Sort Status | 按状态排序 |
| `ShiftO` | Sort Selected Column | 按选中列排序 |

---

### 3.2 第 2 层：浏览器基础层

**文件**: `internal/view/browser.go` 的 `bindKeys()` 方法

**注册时机**: `Browser.Init()` 中调用 `b.bindKeys(b.Actions())` —— 只执行一次

提供浏览器视图的通用过滤/导航键：

| 键 | 动作 | 说明 |
|----|------|------|
| `Escape` | Filter Reset | 重置过滤/退出视图 |
| `KeyQ` | Filter Reset | 重置过滤/退出视图（共享键） |
| `Enter` | Filter | 应用过滤条件（共享键） |
| `KeyHelp(?)` | Help | 帮助（共享键） |

> **注意**: 这里的 `Enter` 绑定到 `filterCmd`，但在后续 `refreshActions()` 中会被覆盖为 `enterCmd`。`enterCmd` 内部会先调用 `filterCmd` 处理过滤模式场景，所以两者是协作关系而非冲突。

---

### 3.3 第 3 层：浏览器动态层

**文件**: `internal/view/browser.go` 的 `refreshActions()` 方法

**注册时机**: 每次数据刷新时调用（`TableDataChanged` / `TableNoData` 事件触发）—— 动态重建

这是最核心的动态绑定层，会根据运行时条件 **动态决定** 注册哪些键。每次刷新时先创建一个新的临时 `KeyActions`（变量名 `aa`），填充完毕后 Merge 到视图的实际 `KeyActions` 中。

**动态条件判断**:

1. **基础动作**（始终注册）:
   - `KeyC` → Copy（复制资源名）
   - `Enter` → View（查看详情或进入子视图，覆盖 Browser 基础层的 Filter 绑定）
   - `CtrlR` → Refresh（手动刷新）

2. **连接正常时** (`ConOK()`):
   - `KeyN` → Copy Namespace（复制命名空间名）
   - `KeyW` → Warp To Namespace（跳转到选中资源所在命名空间）
   - `Key0~Key9` → 切换到收藏命名空间（根据用户配置动态生成数字快捷键）

3. **非只读模式 + 权限检查**:
   - 有 `edit` 权限: `KeyE` → Edit（编辑资源，危险操作）
   - 有 `delete` 权限: `CtrlD` → Delete（删除资源，危险操作）
   - 只读模式下: 调用 `ClearDanger()` 清除所有危险操作键

4. **非 K9s 内部资源** (`!IsK9sMeta`):
   - `KeyY` → YAML 查看
   - `KeyD` → Describe

5. **回调 Extender 和具体资源层**（`bindKeysFn` 钩子链）:
   ```go
   for _, f := range b.bindKeysFn {
       f(aa)  // 依次调用所有注册的绑定函数，作用于临时 aa
   }
   b.Actions().Merge(aa)  // 最后一次性合并到视图 actions
   ```

6. **最后加载动态扩展**:
   - `hotKeyActions()`: 从热键配置文件加载
   - `pluginActions()`: 从插件配置文件加载

---

### 3.4 第 4 层：功能扩展层（Extender 装饰器模式）

Extender 采用 **装饰器模式**，通过包装 `ResourceViewer` 接口来增强功能并注入键绑定。

#### 工作机制

每个 Extender 都遵循相同模式：
1. 嵌入 `ResourceViewer` 接口（Go 语言的装饰器写法）
2. 在构造函数中调用 `AddBindKeysFn(extender.bindKeys)` 注入自己的绑定函数
3. `bindKeys()` 函数向 `KeyActions` 中添加该 Extender 特有的快捷键

> **关键点**: Extender 不重写 `AddBindKeysFn` 方法，所有 Extender 对 `AddBindKeysFn` 的调用最终都会传递到最内层的 `Table.AddBindKeysFn`，将绑定函数追加到 `Table.bindKeysFn` 切片的末尾。

#### 绑定顺序（以 Pod 为例）

构造函数调用顺序是从内向外逐层包装：

```go
// internal/view/pod.go
func NewPod(gvr *client.GVR) ResourceViewer {
    var p Pod
    p.ResourceViewer = NewPortForwardExtender(   // 第6个调用 AddBindKeysFn → 最后执行
        NewOwnerExtender(                         // 第5个
            NewVulnerabilityExtender(             // 第4个
                NewImageExtender(                 // 第3个
                    NewLogsExtender(              // 第2个
                        NewBrowser(gvr),          // 最内层，不调用 AddBindKeysFn
                        p.logOptions,
                    ),
                ),
            ),
        ),
    )
    p.AddBindKeysFn(p.bindKeys)  // 第7个 → 最后执行，可以覆盖所有 Extender 的键
    return &p
}
```

`bindKeysFn` 切片中的执行顺序（从先到后）：

| 顺序 | 来源 | 说明 |
|------|------|------|
| 1 | LogsExtender | 日志功能：KeyL / KeyP |
| 2 | ImageExtender | 镜像相关功能 |
| 3 | VulnerabilityExtender | 漏洞扫描功能 |
| 4 | OwnerExtender | 所有者查看功能 |
| 5 | PortForwardExtender | 端口转发功能：KeyF / KeyShiftF |
| 6 | Pod（具体资源） | Pod 专属功能：CtrlK/KeyS/KeyA/KeyT/KeyZ/KeyO |

> **优先级结论**: 外层 Extender > 内层 Extender，具体资源 > 所有 Extender。后执行的绑定会覆盖先执行的同名键。

#### 典型 Extender 示例

**(1) LogsExtender** —— `internal/view/logs_extender.go`
```go
func (l *LogsExtender) bindKeys(aa *ui.KeyActions) {
    aa.Bulk(ui.KeyMap{
        ui.KeyL: ui.NewKeyAction("Logs", l.logsCmd(false), true),
        ui.KeyP: ui.NewKeyAction("Logs Previous", l.logsCmd(true), true),
    })
}
```

**(2) PortForwardExtender** —— `internal/view/pf_extender.go`
```go
func (p *PortForwardExtender) bindKeys(aa *ui.KeyActions) {
    aa.Bulk(ui.KeyMap{
        ui.KeyF:      ui.NewKeyAction("Show PortForward", p.showPFCmd, true),
        ui.KeyShiftF: ui.NewKeyAction("Port-Forward", p.portFwdCmd, true),
    })
}
```

**其他 Extender**: `ImageExtender`、`VulnerabilityExtender`、`OwnerExtender`、`ScaleExtender`、`RestartExtender`、`ValueExtender` 等。

---

### 3.5 第 5 层：具体资源层

以 Pod 为例 —— `internal/view/pod.go` 的 `bindDangerousKeys()` 和 `bindKeys()` 方法

通过 `AddBindKeysFn(p.bindKeys)` 注入到钩子链的末尾：

| 键 | 动作 | 条件 |
|----|------|------|
| `CtrlK` | Kill（删除 Pod） | 非只读模式（危险操作） |
| `KeyS` | Shell（进入容器） | 非只读模式（危险操作） |
| `KeyA` | Attach（附加到容器） | 非只读模式（危险操作） |
| `KeyT` | Transfer（文件传输） | 非只读模式（危险操作） |
| `KeyZ` | Sanitize（清理异常 Pod） | 非只读模式（危险操作） |
| `KeyO` | Show Node（跳转到所在节点） | 始终注册 |

> 不同的资源视图（Deploy/StatefulSet/Job 等）有各自不同的专属键。

---

### 3.6 第 6 层：动态加载层（热键 + 插件）

这两层都在 `refreshActions()` 的最后直接操作 `b.Actions()`（不是临时 `aa`），所以优先级最高。

#### 热键系统 —— `hotKeyActions()` in `internal/view/actions.go`

- **加载来源**: 从 `hotkeys.yaml` 配置文件加载
- **加载时机**: 每次数据刷新时重新加载
- **清理机制**: 加载前先 `Range` 遍历并删除所有 `HotKey=true` 的旧动作，确保热键变化能即时生效
- **功能实质**: 快速跳转的快捷方式，内部调用 `gotoResource(cmd, path, clearStack)`
- **标记**: `HotKey=true`、`Shared=true`
- **冲突处理**: 键冲突时根据 `Override` 配置决定是报错还是覆盖原有绑定

#### 插件系统 —— `pluginActions()` in `internal/view/actions.go`

- **加载来源**: 从 `plugins.yaml` 配置文件加载
- **加载时机**: 每次数据刷新时重新加载
- **清理机制**: 加载前先删除所有 `Plugin=true` 的旧动作
- **范围匹配**: 插件的 `Scopes` 必须包含视图别名（通过 `inScope()` 检查），`"all"` 表示适用于所有视图
- **权限检查**: 只读模式下跳过 `Dangerous=true` 的插件
- **冲突处理**: 键冲突时根据 `Override` 配置决定是报错还是覆盖
- **输入支持**: 插件可定义 `Inputs`，执行时弹出输入对话框收集参数

---

## 四、初始化与刷新的完整时序

### 4.1 初始化阶段（视图创建时，执行一次）

```
NewPod(gvr)
  → NewBrowser(gvr)
    → NewTable(gvr)
      → Table 创建，actions 为空
  → 各层 Extender 构造
    → 每个 Extender 调用 AddBindKeysFn(自身的 bindKeys)
    → 全部追加到 Table.bindKeysFn 切片
  → Pod 自身也调用 AddBindKeysFn(p.bindKeys)
  
App.inject(component) → 触发 Component.Init()
  → Browser.Init(ctx)
    → Table.Init(ctx)
      → t.SetInputCapture(t.keyboard)   // 设置视图层按键捕获
      → t.bindKeys()                   // 第1层：表格基础键注册
    → b.bindKeys(b.Actions())          // 第2层：浏览器基础键注册
    → for f in b.bindKeysFn {          // 第4+5层：Extender 和具体资源键注册
        f(b.Actions())
      }
    → 其他初始化工作...
```

### 4.2 刷新阶段（每次数据变化，反复执行）

```
数据到达 → TableDataChanged / TableNoData 事件
  → Browser.refreshActions()
    → 创建新的临时 KeyActions (aa)
    → 添加动态基础键（Copy/Enter/Refresh）  // 第3层：浏览器动态键
    → 连接正常时添加命名空间相关键
    → 有权限时添加 Edit/Delete 等危险键
    → 非内部资源添加 YAML/Describe 键
    → for f in b.bindKeysFn { f(aa) }     // 第4+5层：Extender 和具体资源键重建
    → b.Actions().Merge(aa)               // 合并到视图（覆盖同名键）
    → hotKeyActions(b, b.Actions())       // 第6层a：热键（先删旧再加新）
    → pluginActions(b, b.Actions())       // 第6层b：插件（先删旧再加新）
    → 更新菜单提示 HydrateMenu(Hints())
```

### 4.3 键的生命周期对比

| 层次 | 注册时机 | 更新频率 | 是否可能被覆盖 |
|------|----------|----------|----------------|
| 表格基础层 | Init 时一次 | 从不 | 是（被上层覆盖） |
| 浏览器基础层 | Init 时一次 | 从不 | 是（被上层覆盖） |
| 浏览器动态层 | 每次刷新 | 每次刷新重建 | 是（被上层覆盖） |
| Extender 层 | Init + 每次刷新 | 每次刷新重建 | 是（被外层覆盖） |
| 具体资源层 | Init + 每次刷新 | 每次刷新重建 | 是（被插件/热键覆盖） |
| 热键层 | 每次刷新 | 每次刷新重建 | 是（被插件覆盖） |
| 插件层 | 每次刷新 | 每次刷新重建 | 否（最顶层） |

> **注意**: Init 阶段 `bindKeysFn` 直接作用在 `b.Actions()` 上；Refresh 阶段 `bindKeysFn` 作用在临时 `aa` 上，然后 Merge。两种方式最终效果一致，但 Refresh 阶段的做法更利于实现"动态条件判断 + 原子更新"。

---

## 五、完整调用示例（Pod 视图中按 L 键查看日志）

```
1. 用户按下 'l' 键
   ↓
2. tcell 捕获终端输入，生成 *tcell.EventKey（KeyRune, Rune='l'）
   ↓
3. tview 事件循环，事件沿组件树向上传递
   ↓
4. 第一层拦截: view.App.keyboard()
   ├─ ui.AsKey(evt) → tcell.Key(108) = KeyL
   ├─ 在 ui.App.actions 中查找 KeyL → 未找到（全局键没有 L）
   └─ 返回 evt 原样透传
   ↓
5. 第二层拦截: Table.keyboard()
   ├─ 排除 Up/Down/Shift+方向键等特殊键
   ├─ ui.AsKey(evt) → KeyL (108)
   ├─ 在 t.Actions() 中查找 KeyL → 找到！
   │   （这个键是 LogsExtender.bindKeys 注册的，在 refreshActions 中被重建）
   ├─ 检查 !IsTopDialog() → 当前不是对话框
   └─ 调用 a.Action(evt) → LogsExtender.logsCmd(false)(evt)
   ↓
6. logsCmd 闭包执行:
   ├─ 获取选中的 Pod path（如 default/nginx-xxx）
   ├─ 构造 LogOptions
   └─ 调用 app.inject(NewLog(...)) 打开日志视图
   ↓
7. 返回 nil → 事件被消费，终止分发链
```

---

## 六、关键协作机制总结

### 6.1 BindKeysFn 钩子链

`BindKeysFunc` 类型: `func(*ui.KeyActions)`

`Table.bindKeysFn` 是一个函数切片，所有 Extender 和具体资源都通过 `AddBindKeysFn()` 将自己的绑定函数追加到这个切片中。

**执行两次**:
- **Init 时**: 直接作用在 `b.Actions()` 上（`Browser.Init` 第 103-105 行）
- **Refresh 时**: 作用在临时 `aa` 上，然后 Merge（`Browser.refreshActions` 第 662-664 行）

这种设计使得 **键绑定可以动态变化**（例如根据权限、只读模式、连接状态等条件决定是否注册某些键）。

### 6.2 键冲突与覆盖优先级

从低到高排列（后者覆盖前者）：

| 优先级 | 层次 | 覆盖方式 |
|--------|------|----------|
| 最低 | Table 基础层 | map 赋值覆盖 |
| ↑ | Browser 基础层 | map 赋值覆盖 |
| ↑ | Browser 动态层 | map 赋值覆盖 |
| ↑ | 内层 Extender | map 赋值覆盖 |
| ↑ | 外层 Extender | map 赋值覆盖 |
| ↑ | 具体资源（Pod 等） | map 赋值覆盖 |
| ↑ | 热键 | 先删旧再加新 |
| 最高 | 插件 | 先删旧再加新 |

实际合并使用 `Merge()` / `Bulk()` / `Add()`，本质都是 `map[k] = v` 赋值，后写入的值会覆盖先写入的。

### 6.3 危险操作保护

`ActionOpts.Dangerous` 标记用于多层次安全防护：

1. **绑定层防护**: 只读模式下 `ClearDanger()` 清除所有危险操作的键绑定
2. **插件层防护**: 只读模式下跳过加载 `Dangerous=true` 的插件
3. **执行层防护**: 部分危险操作触发时弹出确认对话框（如 Delete 删除、Sanitize 清理）
4. **权限层防护**: 根据 K8s RBAC 权限动态决定是否添加 Edit/Delete 等键

### 6.4 共享键机制

`ActionOpts.Shared=true` 表示该动作是 **跨视图共享的全局功能键**（如 Help、Quit、Clear Filter 等）。

在 `KeyActions.Hints()` 生成菜单提示时，共享键会被排除在视图专属提示之外，避免每个视图的菜单都重复显示相同的全局键。

---

## 七、视图注册与 GVR 映射

视图通过 `MetaViewers` 注册表关联 GVR（Group/Version/Resource）与视图构造函数。

**文件**: `internal/view/registrar.go`

```go
MetaViewers map[*client.GVR]MetaViewer

type MetaViewer struct {
    viewerFn ViewerFunc   // 视图构造函数: func(*GVR) ResourceViewer
    enterFn  EnterFunc    // Enter 键回调（可选，用于自定义进入行为）
}
```

按类别分组注册：
- `coreViewers()`: 核心资源（Pod/Service/Node/ConfigMap 等）
- `appsViewers()`: 工作负载（Deployment/StatefulSet/DaemonSet 等）
- `rbacViewers()`: RBAC 相关（Role/ClusterRole/Binding 等）
- `batchViewers()`: 批处理（CronJob/Job）
- `miscViewers()`: 杂项（Context/Container/Pulse/Alias 等）
- `helmViewers()`: Helm 相关
- `crdViewers()`: CRD 相关

当用户输入 `pod` 命令时，`Command` 解释器通过别名解析找到 GVR，再查注册表得到 `NewPod` 构造函数，创建视图实例并注入到页面栈中。

---

## 八、架构设计亮点

1. **装饰器模式实现功能正交**: Extender 机制将日志、端口转发、镜像扫描等功能实现为可插拔装饰器，任意组合而不修改基础代码，符合开闭原则。

2. **输入捕获链实现分层解耦**: 利用 tview 的 `SetInputCapture` 构建 App→View 两层拦截，全局键与视图键各司其职，层级清晰。

3. **动态绑定适应运行时状态**: `refreshActions()` 在每次数据刷新时重建动作集合，能够响应连接状态、权限配置、只读模式等运行时变化。

4. **配置驱动的扩展性**: 插件和热键系统允许用户通过 YAML 配置完全自定义快捷键，无需修改代码。

5. **危险操作多层次防护**: 从键绑定（只读模式清除）→ 权限检查（RBAC）→ 对话框确认 → 资源实际操作，形成多层安全网。

6. **两阶段初始化设计**: Init 阶段设置静态基础键，Refresh 阶段动态重建条件相关的键，兼顾了初始化效率和运行时灵活性。
