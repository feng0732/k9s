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

**关键方法**（`internal/ui/action.go`）:
- `Add(k, ka)`: 添加单个键动作（直接赋值 `actions[k] = ka`，已存在则覆盖）
- `Bulk(km)`: 批量添加键映射（遍历 `for k,v := range km { actions[k] = v }`）
- `Merge(aa)`: **增量合并**另一个 KeyActions——只遍历 `aa` 中存在的键并覆盖同名键，**aa 中不存在的旧键会保留在原集合中，不会被清除**
- `Get(k)`: 根据键查找动作
- `Delete(kk...)`: 删除指定键
- `Clear()`: 清空所有动作
- `ClearDanger()`: **选择性清除**——遍历全部动作，只删除 `Opts.Dangerous == true` 的动作
- `Range(f)`: 遍历全部键动作（读锁保护下遍历快照）
- `Reset(aa)`: 等价于 `Clear()` + `Merge(aa)`（先清空再合并，全量替换）
- `Hints()`: 生成菜单提示（排除 `Opts.Shared=true` 的共享键）

> **核心设计差异**: `Merge()` 是**增量覆盖**（保留旧键），`Reset()` 是**全量替换**（先清空再合并）。k9s 刷新时使用的是 `Merge()`，这带来了"静态层保留、动态层覆盖"的特性。

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
│  第 6 层: 动态加载层（插件 + 热键）                         │  ← 优先级最高
│    pluginActions() / hotKeyActions() —— 运行时从配置加载   │
│    插件先加载，热键后加载（热键可覆盖插件）                  │
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

3. **非只读模式 + 权限检查**（`!IsReadOnly()` 且 `ConOK()`）:
   - 有 `edit` 权限: `KeyE` → Edit（编辑资源，危险操作）—— 添加到 **临时 aa**
   - 有 `delete` 权限: `CtrlD` → Delete（删除资源，危险操作）—— 添加到 **临时 aa**
   - **只读模式下**: 执行 `b.Actions().ClearDanger()` —— 直接作用于现有集合 `b.Actions()`，清除其中所有 `Dangerous=true` 的旧动作（注意：此时 Merge 还未执行，清除的是上一次刷新留下的危险动作）。同时只读模式下不会向 aa 中添加 Edit/Delete 等危险键。

> **ClearDanger() 的精确执行位置**: 位于 `refreshActions()` 第 655 行，在 `Merge(aa)` 之前。先清除旧集合中的危险动作，再合并不含危险动作的新集合 aa。同时各具体资源的 `bindKeys()` 内部也会判断 `IsReadOnly()`（如 Pod.bindKeys 第 127 行），只读模式下不向 aa 注入危险动作，两层防护确保只读模式下绝对没有危险操作键。

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
   - `pluginActions()`: 从插件配置文件加载（先执行）
   - `hotKeyActions()`: 从热键配置文件加载（后执行，可覆盖插件）

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

### 3.6 第 6 层：动态加载层（插件 + 热键）

这两层都在 `refreshActions()` 的最后直接操作 `b.Actions()`（不是临时 `aa`），所以优先级最高。

**实际调用顺序**（见 `internal/view/browser.go` 第 667-674 行）：

```go
b.Actions().Merge(aa)                      // 先合并前 5 层的键
if err := pluginActions(b, b.Actions()); err != nil { ... }  // 第 6a 层：插件先加载
if err := hotKeyActions(b, b.Actions()); err != nil { ... }  // 第 6b 层：热键后加载
```

> **关键**: 插件先写入，热键后写入。按 map 赋值覆盖规则，**热键优先级高于插件**——当热键与插件使用相同按键且热键配置了 `Override: true` 时，热键会覆盖插件的动作。

#### 插件系统 —— `pluginActions()` in `internal/view/actions.go`

- **加载来源**: 从 `plugins.yaml` 配置文件加载
- **加载时机**: 每次数据刷新时重新加载
- **清理机制**: 加载前先 `Range` 遍历并删除所有 `Plugin=true` 的旧动作，确保插件变化能即时生效
- **范围匹配**: 插件的 `Scopes` 必须包含视图别名（通过 `inScope()` 检查），`"all"` 表示适用于所有视图
- **权限检查**: 只读模式下跳过 `Dangerous=true` 的插件
- **冲突处理**: 写入前调用 `aa.Get(key)` 检查键是否已被占用——已被占用且 `Override=false` 则报错跳过，`Override=true` 则覆盖
- **标记**: `Plugin=true`
- **输入支持**: 插件可定义 `Inputs`，执行时弹出输入对话框收集参数

#### 热键系统 —— `hotKeyActions()` in `internal/view/actions.go`

- **加载来源**: 从 `hotkeys.yaml` 配置文件加载
- **加载时机**: 每次数据刷新时重新加载
- **清理机制**: 加载前先 `Range` 遍历并删除所有 `HotKey=true` 的旧动作，确保热键变化能即时生效
- **功能实质**: 快速跳转的快捷方式，内部调用 `gotoResource(cmd, path, clearStack)`
- **冲突处理**: 写入前调用 `aa.Get(key)` 检查键是否已被占用（包括刚写入的插件键）——已被占用且 `Override=false` 则报错跳过，`Override=true` 则覆盖
- **标记**: `HotKey=true`、`Shared=true`

#### 插件与热键的交互覆盖

| 场景 | 结果 |
|------|------|
| 插件键与热键冲突，热键 `Override=true` | 热键覆盖插件（热键后写入） |
| 插件键与热键冲突，热键 `Override=false` | 热键报错跳过，保留插件键 |
| 插件键与热键冲突，插件 `Override=true` | 插件先写入成功；热键根据自身 `Override` 决定是否再覆盖 |
| 同一键在插件和热键中都有定义 | 热键后执行，在 `Override=true` 时最终胜出 |

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
    ├─ [守卫] 非当前顶层视图则直接 return
    ├─ 创建全新临时 KeyActions aa（仅含 Copy/Enter/Refresh 3 个基础键）
    ├─ 连接正常时 → 向 aa 添加命名空间相关键
    ├─ 非只读模式且有权限时 → 向 aa 添加 Edit/Delete 危险键
    ├─ 只读模式时 → b.Actions().ClearDanger() 清除旧集合中的危险键（作用于现有集合）
    ├─ 非内部资源时 → 向 aa 添加 YAML/Describe 键
    ├─ 遍历 bindKeysFn 钩子链 → 各 Extender 和具体资源向 aa 注入键（作用于临时 aa）
    ├─ b.Actions().Merge(aa) → 增量合并（同名键覆盖，aa 中无的旧键保留）
    ├─ pluginActions(b, b.Actions()) → 作用于现有集合：
    │   ├─ 先 Range 遍历，删除所有 Opts.Plugin=true 的旧插件动作
    │   └─ 再从配置加载新插件，冲突时检查 Override 决定是否覆盖
    ├─ hotKeyActions(b, b.Actions()) → 作用于现有集合：
    │   ├─ 先 Range 遍历，删除所有 Opts.HotKey=true 的旧热键动作
    │   └─ 再从配置加载新热键，冲突时检查 Override 决定是否覆盖（可覆盖插件键）
    └─ 更新菜单提示 HydrateMenu(Hints())
```

> **关键观察**: 整个刷新过程中，只有 `b.Actions().Merge(aa)` 这一步可能覆盖前 1-2 层（Table/Browser 基础层）的旧键；而 Table/Browser 基础层中不在 aa 里的键（如 Space 标记行、CtrlZ 切换故障、Escape 重置过滤等）会 **永久保留** 在集合中，除非被后续层同名覆盖。

### 4.3 键的生命周期对比

| 层次 | 注册时机 | 更新频率 | 刷新时是否重建 | 刷新时未重建是否保留 | 是否可能被覆盖 |
|------|----------|----------|----------------|----------------------|----------------|
| 表格基础层 | Init 时一次 | 从不 | **否（aa 中不含）** | **是（永久保留）** | 是（被上层覆盖） |
| 浏览器基础层 | Init 时一次 | 从不 | 部分（Enter 被 aa 覆盖为 View） | Escape/Q/Help 永久保留，Enter 每次被覆盖 | 是（被上层覆盖） |
| 浏览器动态层 | 每次刷新 | 每次刷新重建 | 是 | —（每次重建） | 是（被上层覆盖） |
| Extender 层 | Init + 每次刷新 | 每次刷新重建 | 是（通过 bindKeysFn 注入到 aa） | —（每次重建） | 是（被外层覆盖） |
| 具体资源层 | Init + 每次刷新 | 每次刷新重建 | 是（通过 bindKeysFn 注入到 aa） | —（每次重建） | 是（被插件/热键覆盖） |
| 插件层 | 每次刷新 | 每次刷新重建 | 是（先删旧插件再加新） | —（每次重建） | 是（被热键覆盖） |
| 热键层 | 每次刷新 | 每次刷新重建 | 是（先删旧热键再加新） | —（每次重建） | 否（最顶层） |

> **注意**: Init 阶段 `bindKeysFn` 直接作用在 `b.Actions()` 上（此时集合为空，所有键都是新增）；Refresh 阶段 `bindKeysFn` 作用在临时 `aa` 上，然后通过 `Merge(aa)` 增量合并。`Merge` 是增量覆盖而非全量替换，这就是 Table/Browser 基础层能永久保留的原因。

> **Enter 键的特殊覆盖**: Browser 基础层在 Init 时将 Enter 绑定为 `filterCmd`（过滤确认），但每次刷新时 aa 中都会重新绑定 Enter 为 `enterCmd`（查看/进入子视图）。`enterCmd` 内部会先判断是否处于过滤模式，过滤模式下仍然调用 `filterCmd`，所以 Enter 的实际行为始终正确。

---

## 五、动作集合生命周期深度分析

本章深入剖析 `b.Actions()` 这个 `KeyActions` 实例在视图生命周期中的完整变化过程，包括 Init 阶段如何建立初始集合、Refresh 阶段如何增量覆盖、各类清理机制（ClearDanger/插件清理/热键清理）的精确作用范围。

---

### 5.1 初始状态与 Init 阶段全量构建

**起点**: `NewTable()` 创建时，`KeyActions.actions` 是一个空的 `map[tcell.Key]KeyAction`。

**Init 阶段执行流程**（`Browser.Init()`）:

```
b.Actions() = {} （空 map）
   ↓
[Table.Init] t.bindKeys()          → 一次性注入 ~15 个表格基础键
   │   Space / CtrlSpace / Ctrl\ (标记系列)
   │   CtrlS (保存)
   │   KeySlash (过滤模式)
   │   CtrlZ / CtrlW (显示切换)
   │   ShiftN/A/S/O (排序系列)
   │   KeyHelp (帮助，共享键)
   ↓
[Browser.Init] b.bindKeys()        → 注入 4 个浏览器基础键
   │   Escape / KeyQ (过滤重置)
   │   KeyEnter (过滤确认 filterCmd)
   │   KeyHelp (帮助，共享键 —— 与 Table 中同名，但都是 helpCmd 无实质冲突)
   ↓
[Browser.Init] bindKeysFn 钩子链   → 各 Extender 和具体资源依次注入
   │   LogsExtender: KeyL/KeyP
   │   ImageExtender: ...
   │   VulnerabilityExtender: ...
   │   OwnerExtender: ...
   │   PortForwardExtender: KeyF/KeyShiftF
   │   Pod: CtrlK/KeyS/KeyA/KeyT/KeyZ/KeyO （非只读模式下才添加危险键）
   ↓
此时 b.Actions() 包含约 30-40 个键，全部是一次性写入
```

> Init 阶段 **没有任何清理操作**，集合为空，所有绑定都是纯新增。此时如果出现同名键，后执行的绑定会覆盖先执行的（如 Pod 可以覆盖 LogsExtender 的键）。

---

### 5.2 Merge 的增量覆盖机制（核心）

`Merge()` 是理解动作集合生命周期的关键。其实现（`internal/ui/action.go` 第 139-147 行）:

```go
func (a *KeyActions) Merge(aa *KeyActions) {
    a.mx.Lock()
    defer a.mx.Unlock()
    for k, v := range aa.actions {
        a.actions[k] = v   // 只遍历 aa 中的键，逐个覆盖
    }
}
```

**与 Reset 的对比**:

| 方法 | 行为 | 旧键处理 |
|------|------|----------|
| `Merge(aa)` | 遍历 `aa`，逐个赋值覆盖 | **aa 中没有的旧键全部保留** |
| `Reset(aa)` | `Clear()` + `Merge(aa)` | **所有旧键先被清空，再合并 aa** |

k9s 刷新时使用的是 `Merge()`，这带来了 **"静态层保留、动态层覆盖"** 的设计特性。

#### 保留 vs 覆盖的精确分类

以 Pod 视图为例，Refresh 时 aa 中的内容约为：

| aa 中的键（会覆盖/新增） | 来源 |
|-------------------------|------|
| KeyC / KeyEnter / CtrlR | 浏览器动态基础键 |
| KeyN / KeyW / Key0~Key9 | 命名空间相关（连接正常时） |
| KeyE / CtrlD | Edit/Delete（有权限时） |
| KeyY / KeyD | YAML/Describe（非内部资源时） |
| KeyL / KeyP | LogsExtender |
| KeyF / KeyShiftF | PortForwardExtender |
| ...（其他 Extender 键） | |
| CtrlK / KeyS / KeyA / KeyT / KeyZ / KeyO | Pod 具体资源 |

**aa 中没有、但 b.Actions() 中存在的键（会永久保留）**:

| 保留的旧键 | 来源 | 为什么 aa 中不含 |
|-----------|------|-----------------|
| KeySpace / CtrlSpace / Ctrl\ | Table 基础层 | aa 从不重建这些键 |
| CtrlS | Table 基础层 | aa 从不重建 |
| KeySlash | Table 基础层 | aa 从不重建 |
| CtrlZ / CtrlW | Table 基础层 | aa 从不重建 |
| ShiftN / ShiftA / ShiftS / ShiftO | Table 基础层 | aa 从不重建 |
| KeyEscape / KeyQ | Browser 基础层 | aa 从不重建 |
| KeyHelp | Table + Browser 基础层 | aa 从不重建 |

> **设计意图**: Init 时一次性写入的"静态基础键"永远不参与刷新重建，靠 Merge 的保留特性一直存在。而"动态条件键"每次刷新都重新注入，响应运行时状态变化。

---

### 5.3 ClearDanger() 的触发时机与作用域

**触发条件**: 只读模式下（`IsReadOnly() == true`）且连接正常时执行。

**执行位置**: `refreshActions()` 第 655 行，在 **`Merge(aa)` 之前**。

**作用范围**: 直接作用于现有集合 `b.Actions()`（不是临时 aa），遍历所有键，只删除 `Opts.Dangerous == true` 的动作。

#### 完整时序与效果

假设运行时模式从"正常模式"切换为"只读模式"：

```
[切换前] b.Actions() 包含:
    Table 基础键（非危险）
    Browser 基础键（非危险）
    KeyE / CtrlD（危险，Browser 动态层上次写入）
    CtrlK / KeyS / KeyA / KeyT / KeyZ（危险，Pod 上次写入）
    KeyL / KeyF 等 Extender 键（非危险）
    插件键 / 热键
          ↓
[切换后，首次刷新]
1. 创建 aa（不含 KeyE/CtrlD，因为只读模式）
2. 只读模式分支: b.Actions().ClearDanger()
   → 删除 b.Actions() 中所有 Dangerous=true 的键
   → 结果: KeyE/CtrlD/CtrlK/KeyS/KeyA/KeyT/KeyZ 全部被清除
3. bindKeysFn 钩子链执行，作用于 aa
   → Pod.bindKeys 中 IsReadOnly() 判断为 true，不添加危险键到 aa
   → aa 中也没有 CtrlK/KeyS 等
4. b.Actions().Merge(aa)
   → aa 中所有非危险键覆盖到 b.Actions()
   → aa 中不含的 Table/Browser 基础键保留
5. pluginActions: 只读模式下跳过 Dangerous=true 的插件
6. hotKeyActions: 正常加载
          ↓
[最终] b.Actions() 中所有危险操作键已被彻底清除
```

**双重防护机制**:
1. **Browser 层**: 只读模式下不向 aa 添加 Edit/Delete，并清除旧集合中的危险键
2. **具体资源层**: 每个资源的 `bindKeys()` 内部自行判断 `IsReadOnly()`（如 Pod.bindKeys 第 127 行），只读模式下不注入危险键

两层防护确保只读模式下绝对没有危险操作键残留。

---

### 5.4 插件与热键的选择性清理机制

插件和热键使用 **基于标记的选择性清理**，两者互不干扰，也不触碰其他层的键。

#### pluginActions() 的清理逻辑

```go
// internal/view/actions.go 第 121-125 行
aa.Range(func(k tcell.Key, a ui.KeyAction) {
    if a.Opts.Plugin {      // 只检查 Plugin 标记
        aa.Delete(k)        // 只删除插件自己注册的动作
    }
})
```

**作用范围**: 只删除 `Opts.Plugin == true` 的键。
- 不会删除 Table/Browser 基础键
- 不会删除 Extender 和具体资源键
- 不会删除热键（`HotKey=true` 但 `Plugin=false`）

**执行时机**: 每次刷新时先删除所有旧插件键，再从配置文件重新加载并添加新插件键。

#### hotKeyActions() 的清理逻辑

```go
// internal/view/actions.go 第 62-66 行
aa.Range(func(k tcell.Key, a ui.KeyAction) {
    if a.Opts.HotKey {      // 只检查 HotKey 标记
        aa.Delete(k)        // 只删除热键自己注册的动作
    }
})
```

**作用范围**: 只删除 `Opts.HotKey == true` 的键。
- 不会删除 Table/Browser 基础键
- 不会删除 Extender 和具体资源键
- 不会删除插件（`Plugin=true` 但 `HotKey=false`）

**执行时机**: 每次刷新时先删除所有旧热键，再从配置文件重新加载并添加新热键。

#### 插件与热键的清理交互

| 操作 | 清理插件键 | 清理热键键 | 清理其他层键 |
|------|-----------|-----------|-------------|
| pluginActions() 清理阶段 | ✅ 是 | ❌ 否 | ❌ 否 |
| hotKeyActions() 清理阶段 | ❌ 否 | ✅ 是 | ❌ 否 |
| ClearDanger() | 仅当 Dangerous=true | 仅当 Dangerous=true | 仅当 Dangerous=true |

> **隔离性设计**: 插件和热键通过各自的 Option 标记实现自管理，刷新时只会清理自己上一轮注入的动作，不会误删其他层的键。这也意味着用户删除某个插件/热键配置后，下一次刷新就能自动清理掉对应的按键绑定。

---

### 5.5 刷新前后动作集合的完整状态迁移

以 Pod 视图的一次典型刷新为例，展示完整状态变化：

```
┌─────────────────────────────────────────────────────────────┐
│ [刷新前] b.Actions() 的完整内容                              │
├─────────────────────────────────────────────────────────────┤
│ ① Table 基础层（永久保留）                                    │
│    Space / CtrlSpace / Ctrl\ / CtrlS / /                    │
│    CtrlZ / CtrlW / ShiftN/A/S/O / ?(Help)                   │
│ ② Browser 基础层（大部分永久保留）                              │
│    Escape / Q / ?(Help)   ← 永久保留                        │
│    Enter → filterCmd     ← 即将被 aa 覆盖为 enterCmd        │
│ ③ 上一轮刷新的动态层键（即将被 aa 覆盖）                         │
│    C / Enter(→enterCmd) / CtrlR / N / W / 0-9               │
│    E / CtrlD / Y / D                                        │
│ ④ 上一轮 Extender 和资源层键（即将被 aa 覆盖）                   │
│    L / P / F / ShiftF / ... / CtrlK / S / A / T / Z / O     │
│ ⑤ 上一轮插件键（即将被 pluginActions 清理重建）                 │
│    [Plugin=true 标记的若干键]                                 │
│ ⑥ 上一轮热键（即将被 hotKeyActions 清理重建）                   │
│    [HotKey=true 标记的若干键]                                 │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    refreshActions() 执行
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 1] 创建临时 aa，仅含 3 个基础键                           │
├─────────────────────────────────────────────────────────────┤
│   aa = { C → Copy, Enter → View(enterCmd), CtrlR → Refresh }│
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 2] 条件填充 aa（连接正常、有权限、非内部资源）               │
├─────────────────────────────────────────────────────────────┤
│   aa += N / W / 0-9  (命名空间)                              │
│   aa += E / CtrlD    (Edit/Delete，非只读+有权限时)            │
│   aa += Y / D        (YAML/Describe，非内部资源)              │
│   aa += bindKeysFn 钩子链输出                                 │
│        (LogsExtender→L/P, PortForwardExtender→F/ShiftF, ...) │
│        (Pod→CtrlK/S/A/T/Z/O，非只读时注入危险键)               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 3] 只读模式下 ClearDanger()（作用于 b.Actions()）         │
├─────────────────────────────────────────────────────────────┤
│   非只读: 跳过                                               │
│   只读: 从 b.Actions() 删除所有 Dangerous=true 的键           │
│         （清除的是上一轮遗留的 E/CtrlD/CtrlK/S/A/T/Z 等）     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 4] b.Actions().Merge(aa) —— 增量覆盖                    │
├─────────────────────────────────────────────────────────────┤
│   aa 中存在的键: 覆盖 b.Actions() 中同名键                    │
│     → Enter 从 filterCmd 变为 enterCmd                       │
│     → C / CtrlR / N / W / E / CtrlD / Y / D / L / F / ...   │
│       全部被 aa 中的新版本覆盖                                 │
│   aa 中不存在的键: 保留在 b.Actions() 中不变                   │
│     → ① Space / CtrlSpace / ... 等 Table 基础键保留          │
│     → ② Escape / Q 等 Browser 基础键保留                     │
│     → ⑤⑥ 插件键和热键键暂时保留（下一步清理）                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 5] pluginActions(b, b.Actions())                        │
├─────────────────────────────────────────────────────────────┤
│   5a. Range 遍历，删除所有 Opts.Plugin=true 的旧插件键         │
│       （不影响其他层的键）                                     │
│   5b. 从 plugins.yaml 重新加载，逐一键入                       │
│       冲突时检查 Override，可覆盖前 5 层的同名键                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [步骤 6] hotKeyActions(b, b.Actions())                        │
├─────────────────────────────────────────────────────────────┤
│   6a. Range 遍历，删除所有 Opts.HotKey=true 的旧热键键         │
│       （不影响其他层的键，包括插件键）                          │
│   6b. 从 hotkeys.yaml 重新加载，逐一键入                       │
│       冲突时检查 Override，可覆盖前 5 层及插件层的同名键         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ [刷新后] b.Actions() 最终状态                                  │
├─────────────────────────────────────────────────────────────┤
│ ① Table 基础键: 全部保留，从未被修改                            │
│ ② Browser 基础键: Escape/Q 保留，Enter 被 aa 覆盖为 enterCmd   │
│ ③④ 动态层 + Extender + 资源: 全部由 aa 最新重建                │
│ ⑤ 插件: 全部由 pluginActions 最新重建                         │
│ ⑥ 热键: 全部由 hotKeyActions 最新重建（优先级最高）             │
└─────────────────────────────────────────────────────────────┘
```

---

### 5.6 生命周期总结：四类键的不同命运

| 类别 | 代表键 | 写入时机 | 刷新时是否重建 | 刷新时保留策略 |
|------|--------|----------|----------------|----------------|
| **永久静态键** | Space、Escape、CtrlZ、ShiftN | Init 阶段一次 | ❌ 从不 | Merge 保留，永久存在 |
| **条件覆盖键** | Enter | Init（filterCmd）+ 每次刷新（enterCmd） | ✅ 每次 | Merge 时被 aa 覆盖 |
| **动态条件键** | E、CtrlD、C、N、W、L、F、CtrlK 等 | Init + 每次刷新 | ✅ 每次 | Merge 时被 aa 覆盖 |
| **自管理配置键** | 插件键、热键键 | 每次刷新 | ✅ 每次 | 先按 Option 标记选择性清理，再重新加载写入 |

这个分层设计的核心智慧是：**不变的东西永远不重建（减少开销），变化的东西每次重建（保证最新），各自管理自己的生命周期（降低耦合）**。

---

## 六、完整调用示例（Pod 视图中按 L 键查看日志）

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

## 七、关键协作机制总结

### 7.1 BindKeysFn 钩子链

`BindKeysFunc` 类型: `func(*ui.KeyActions)`

`Table.bindKeysFn` 是一个函数切片，所有 Extender 和具体资源都通过 `AddBindKeysFn()` 将自己的绑定函数追加到这个切片中。

**执行两次**:
- **Init 时**: 直接作用在 `b.Actions()` 上（`Browser.Init` 第 103-105 行）
- **Refresh 时**: 作用在临时 `aa` 上，然后 Merge（`Browser.refreshActions` 第 662-664 行）

这种设计使得 **键绑定可以动态变化**（例如根据权限、只读模式、连接状态等条件决定是否注册某些键）。

### 7.2 键冲突与覆盖优先级

从低到高排列（后者覆盖前者）：

| 优先级 | 层次 | 覆盖方式 |
|--------|------|----------|
| 最低 | Table 基础层 | map 赋值覆盖 |
| ↑ | Browser 基础层 | map 赋值覆盖 |
| ↑ | Browser 动态层 | map 赋值覆盖 |
| ↑ | 内层 Extender | map 赋值覆盖 |
| ↑ | 外层 Extender | map 赋值覆盖 |
| ↑ | 具体资源（Pod 等） | map 赋值覆盖 |
| ↑ | 插件 | 先删旧再加新；冲突时检查 Override |
| 最高 | 热键 | 先删旧再加新；冲突时检查 Override；后于插件写入，可覆盖插件 |

实际合并使用 `Merge()` / `Bulk()` / `Add()`，本质都是 `map[k] = v` 赋值，后写入的值会覆盖先写入的。

> **插件与热键的覆盖细节**: 两者都通过 `aa.Get(key)` 检测冲突。插件先写入时可能覆盖前 5 层的键（`Override=true` 时），热键后写入时可能覆盖包括插件在内的所有键（`Override=true` 时）。若 `Override=false`，冲突时不会覆盖而是报错跳过。

### 7.3 危险操作保护

`ActionOpts.Dangerous` 标记用于多层次安全防护：

1. **绑定层防护**: 只读模式下 `ClearDanger()` 清除所有危险操作的键绑定
2. **插件层防护**: 只读模式下跳过加载 `Dangerous=true` 的插件
3. **执行层防护**: 部分危险操作触发时弹出确认对话框（如 Delete 删除、Sanitize 清理）
4. **权限层防护**: 根据 K8s RBAC 权限动态决定是否添加 Edit/Delete 等键

### 7.4 共享键机制

`ActionOpts.Shared=true` 表示该动作是 **跨视图共享的全局功能键**（如 Help、Quit、Clear Filter 等）。

在 `KeyActions.Hints()` 生成菜单提示时，共享键会被排除在视图专属提示之外，避免每个视图的菜单都重复显示相同的全局键。

---

## 八、视图注册与 GVR 映射

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

## 九、架构设计亮点

1. **装饰器模式实现功能正交**: Extender 机制将日志、端口转发、镜像扫描等功能实现为可插拔装饰器，任意组合而不修改基础代码，符合开闭原则。

2. **输入捕获链实现分层解耦**: 利用 tview 的 `SetInputCapture` 构建 App→View 两层拦截，全局键与视图键各司其职，层级清晰。

3. **动态绑定适应运行时状态**: `refreshActions()` 在每次数据刷新时重建动作集合，能够响应连接状态、权限配置、只读模式等运行时变化。

4. **配置驱动的扩展性**: 插件和热键系统允许用户通过 YAML 配置完全自定义快捷键，无需修改代码。

5. **危险操作多层次防护**: 从键绑定（只读模式清除）→ 权限检查（RBAC）→ 对话框确认 → 资源实际操作，形成多层安全网。

6. **两阶段初始化设计**: Init 阶段设置静态基础键，Refresh 阶段动态重建条件相关的键，兼顾了初始化效率和运行时灵活性。
