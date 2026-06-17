# K9s 按键分发流程深度解析

本文档详细分析 k9s 项目中按键事件从用户输入到最终视图动作执行的完整分发链路，包括全局键、视图键和动作注册的协作机制。

---

## 一、核心数据结构

### 1.1 键值定义层

**文件位置**: [internal/ui/key.go](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/ui/key.go)

k9s 定义了一套完整的按键常量体系，将 rune（字符码）映射为 `tcell.Key` 类型：

```
数字键: Key0 ~ Key9 (ASCII 48-57)
字母键: KeyA ~ KeyZ (ASCII 97-122)
大写键: KeyShiftA ~ KeyShiftZ (ASCII 65-90)
特殊键: KeyHelp(?) / KeySlash(/) / KeyColon(:) / KeySpace(32) 等
```

通过 `init()` → `initKeys()` 函数在包加载时将这些键名注册到 tcell 的 `KeyNames` 映射表中，用于后续在菜单提示中显示可读名称。

**工具函数**: `AsKey()` 将 `*tcell.EventKey` 转换为统一的 `tcell.Key` 键值（处理普通键、Rune 字符、Alt 组合键等情况）。

---

### 1.2 动作注册层

**文件位置**: [internal/ui/action.go](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/ui/action.go)

核心类型定义：

| 类型 | 说明 |
|------|------|
| `ActionHandler` | `func(*tcell.EventKey) *tcell.EventKey` —— 按键处理器，返回 nil 表示已消费事件 |
| `ActionOpts` | 动作选项：Visible(可见) / Shared(共享) / Plugin(插件) / HotKey(热键) / Dangerous(危险) |
| `KeyAction` | 单个按键动作：描述 + 处理器 + 选项 |
| `KeyMap` | `map[tcell.Key]KeyAction` —— 键到动作的映射表 |
| `KeyActions` | 带读写锁的动作集合，提供 Add/Bulk/Merge/Get/Delete/Clear 等操作 |

**动作注册方法**:
- `NewKeyAction()`: 基础动作，指定可见性
- `NewSharedKeyAction()`: 共享动作（Shared=true，菜单提示中不重复显示）
- `NewKeyActionWithOpts()`: 完整选项动作

---

## 二、按键分发流程（两层拦截机制）

按键事件采用 **tview 输入捕获链** 实现分层处理，依次经过两层拦截：

```
用户按键 → tcell 终端输入 → tview 事件循环
                           ↓
              ┌─────────────────────────┐
              │  第一层: App 级全局键    │ → view.App.keyboard()
              │  (注册在 view.App 上)    │
              └─────────────────────────┘
                           ↓ 未匹配则透传
              ┌─────────────────────────┐
              │  第二层: View 级视图键   │ → Table.keyboard()
              │  (注册在当前视图上)      │
              └─────────────────────────┘
                           ↓ 未匹配则透传
                    tview 内部默认处理
```

---

### 2.1 第一层：App 级全局键处理

**入口函数**: [internal/view/app.go#L246-L252](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/app.go#L246-L252)

```go
func (a *App) keyboard(evt *tcell.EventKey) *tcell.EventKey {
    if k, ok := a.HasAction(ui.AsKey(evt)); ok && !a.Content.IsTopDialog() {
        return k.Action(evt)
    }
    return evt
}
```

**设置捕获点**: [internal/view/app.go#L106](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/app.go#L106)
```go
a.SetInputCapture(a.keyboard)
```

#### 全局键注册来源

全局键存储在 `ui.App.actions` 中，由两部分合并组成：

**(1) ui.App 基础绑定** —— [internal/ui/app.go#L144-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/ui/app.go#L144-L152)
```
KeyColon(:)   → 激活命令模式
CtrlR         → 重绘界面
CtrlP         → 保存配置
CtrlU/CtrlQ   → 清除过滤器（共享键）
```

**(2) view.App 扩展绑定** —— [internal/view/app.go#L254-L266](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/app.go#L254-L266)
```
CtrlE       → 切换头部显示
CtrlG       → 切换面包屑显示
KeyHelp(?)  → 帮助视图
Key[/]      → 后退/前进历史视图
KeyDash(-)  → 切换到上一个视图
CtrlA       → 别名列表
Enter       → 执行命令栏跳转
CtrlC       → 退出程序
```

> **注意**: 只有当顶层不是对话框（`!IsTopDialog()`）时，全局键才生效。对话框期间会屏蔽全局快捷键。

---

### 2.2 第二层：View 级视图键处理

**入口函数**: [internal/view/table.go#L103-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/table.go#L103-L127)

```go
func (t *Table) keyboard(evt *tcell.EventKey) *tcell.EventKey {
    // 特殊处理: Shift+左右 选择列
    // 特殊处理: Up/Down 直接透传给 tcell
    if a, ok := t.Actions().Get(ui.AsKey(evt)); ok && !t.app.Content.IsTopDialog() {
        return a.Action(evt)
    }
    return evt
}
```

**设置捕获点**: [internal/view/table.go#L64](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/table.go#L64)
```go
t.SetInputCapture(t.keyboard)
```

视图级按键存储在每个视图自己的 `KeyActions` 中（通过 `Actions()` 方法获取）。

---

## 三、视图键注册的六层叠加模型

视图键采用 **洋葱式叠加注册**，从底层到顶层依次经过 6 个层次，每层都可以向同一个 `KeyActions` 中添加或覆盖键绑定：

```
┌───────────────────────────────────────────────────────────┐
│  第 6 层: 动态加载层（插件 + 热键）                         │
│    pluginActions() / hotKeyActions()  —— 运行时从配置加载   │
├───────────────────────────────────────────────────────────┤
│  第 5 层: 具体资源层（Pod/Deploy/...）                     │
│    例如 Pod.bindKeys() —— 通过 AddBindKeysFn 注入           │
├───────────────────────────────────────────────────────────┤
│  第 4 层: 功能扩展层（Extender 装饰器链）                   │
│    LogsExtender / PortForwardExtender / ...                │
│    每个 Extender 通过 AddBindKeysFn 注入自己的键             │
├───────────────────────────────────────────────────────────┤
│  第 3 层: 浏览器动态层                                     │
│    Browser.refreshActions() —— 每次数据刷新时动态重建       │
├───────────────────────────────────────────────────────────┤
│  第 2 层: 浏览器基础层                                     │
│    Browser.bindKeys() —— 过滤/重置等浏览器通用键            │
├───────────────────────────────────────────────────────────┤
│  第 1 层: 表格基础层                                       │
│    Table.bindKeys() —— 标记/排序/过滤模式等通用表格键       │
└───────────────────────────────────────────────────────────┘
```

### 3.1 第 1 层：表格基础层

**文件**: [internal/view/table.go#L225-L240](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/table.go#L225-L240)

**时机**: `Table.Init()` → `t.bindKeys()` 时注册

| 键 | 动作 | 说明 |
|----|------|------|
| `KeyHelp(?)` | Help | 帮助（共享键） |
| `KeySpace` | Mark | 标记行 |
| `CtrlSpace` | Mark Range | 范围标记 |
| `Ctrl\` | Marks Clear | 清除标记 |
| `CtrlS` | Save | 保存表格到文件 |
| `KeySlash(/)` | Filter Mode | 激活过滤模式 |
| `CtrlZ` | Toggle Faults | 切换故障显示 |
| `CtrlW` | Toggle Wide | 切换宽模式 |
| `ShiftN/A/S/O` | Sort XXX | 按列排序 |

---

### 3.2 第 2 层：浏览器基础层

**文件**: [internal/view/browser.go#L150-L157](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/browser.go#L150-L157)

**时机**: `Browser.Init()` → `b.bindKeys(b.Actions())` 时注册

| 键 | 动作 | 说明 |
|----|------|------|
| `Escape / KeyQ` | Filter Reset | 重置过滤/退出视图 |
| `Enter` | Filter | 应用过滤条件 |
| `KeyHelp(?)` | Help | 帮助（共享键） |

---

### 3.3 第 3 层：浏览器动态层

**文件**: [internal/view/browser.go#L627-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/browser.go#L627-L676)

**时机**: `TableDataChanged()` / `TableNoData()` 事件触发 → `refreshActions()` 每次数据刷新时动态重建

这是最核心的动态绑定层，会根据以下条件 **动态决定** 注册哪些键：

1. **基础动作**（始终注册）:
   - `KeyC` → Copy（复制资源名）
   - `Enter` → View（查看详情或进入子视图）
   - `CtrlR` → Refresh（手动刷新）

2. **连接正常时**:
   - `KeyN` → Copy Namespace（复制命名空间）
   - `KeyW` → Warp To Namespace（跳转到选中资源所在命名空间）
   - `Key0~Key9` → 切换到收藏命名空间（根据配置动态生成）

3. **非只读模式 + 权限检查**:
   - 有 `edit` 权限: `KeyE` → Edit（危险操作）
   - 有 `delete` 权限: `CtrlD` → Delete（危险操作）

4. **非 K9s 内部资源**:
   - `KeyY` → YAML 查看
   - `KeyD` → Describe

5. **回调 Extender 和具体资源层**:
   ```go
   for _, f := range b.bindKeysFn {
       f(aa)  // 依次调用所有 AddBindKeysFn 注册的函数
   }
   b.Actions().Merge(aa)
   ```

6. **最后加载动态扩展**:
   - `pluginActions()`: 从插件配置文件加载
   - `hotKeyActions()`: 从热键配置文件加载

---

### 3.4 第 4 层：功能扩展层（Extender 装饰器模式）

Extender 采用 **装饰器模式**，通过包装 `ResourceViewer` 接口来增强功能并注入键绑定。

**工作机制**:

每个 Extender 都遵循相同模式：
1. 嵌入 `ResourceViewer` 接口（装饰器模式）
2. 在构造函数中调用 `AddBindKeysFn(l.bindKeys)` 注入自己的绑定函数
3. `bindKeys()` 函数向 `KeyActions` 中添加该 Extender 特有的快捷键

**典型 Extender 示例**:

**(1) LogsExtender** —— [internal/view/logs_extender.go](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/logs_extender.go)
```go
func (l *LogsExtender) bindKeys(aa *ui.KeyActions) {
    aa.Bulk(ui.KeyMap{
        ui.KeyL: ui.NewKeyAction("Logs", l.logsCmd(false), true),
        ui.KeyP: ui.NewKeyAction("Logs Previous", l.logsCmd(true), true),
    })
}
```

**(2) PortForwardExtender** —— [internal/view/pf_extender.go](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/pf_extender.go)
```go
func (p *PortForwardExtender) bindKeys(aa *ui.KeyActions) {
    aa.Bulk(ui.KeyMap{
        ui.KeyF:      ui.NewKeyAction("Show PortForward", p.showPFCmd, true),
        ui.KeyShiftF: ui.NewKeyAction("Port-Forward", p.portFwdCmd, true),
    })
}
```

**其他 Extender**: `ImageExtender`、`VulnerabilityExtender`、`OwnerExtender`、`ScaleExtender`、`RestartExtender`、`ValueExtender` 等。

**组装示例（Pod 视图）** —— [internal/view/pod.go#L51-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/pod.go#L51-L66)
```go
func NewPod(gvr *client.GVR) ResourceViewer {
    var p Pod
    p.ResourceViewer = NewPortForwardExtender(   // 最外层
        NewOwnerExtender(
            NewVulnerabilityExtender(
                NewImageExtender(
                    NewLogsExtender(NewBrowser(gvr), p.logOptions),  // 最内层是 Browser
                ),
            ),
        ),
    )
    p.AddBindKeysFn(p.bindKeys)  // Pod 自己的绑定也通过 AddBindKeysFn 注入
    return &p
}
```

> 调用顺序：外层 Extender 先注册 bindKeysFn，但在 `refreshActions()` 中按添加顺序执行，所以最终效果是 **内层先绑定，外层后绑定，后绑定的可以覆盖先绑定的同名键**。

---

### 3.5 第 5 层：具体资源层

以 Pod 为例 —— [internal/view/pod.go#L86-L134](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/pod.go#L86-L134)

通过 `AddBindKeysFn(p.bindKeys)` 注入：

| 键 | 动作 | 条件 |
|----|------|------|
| `CtrlK` | Kill（删除 Pod） | 非只读模式（危险操作） |
| `KeyS` | Shell（进入容器） | 非只读模式（危险操作） |
| `KeyA` | Attach（附加容器） | 非只读模式（危险操作） |
| `KeyT` | Transfer（文件传输） | 非只读模式（危险操作） |
| `KeyZ` | Sanitize（清理异常 Pod） | 非只读模式（危险操作） |
| `KeyO` | Show Node（跳转到所在节点） | 始终注册 |

---

### 3.6 第 6 层：动态加载层（插件 + 热键）

#### 插件系统 —— `pluginActions()` [internal/view/actions.go#L115-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/actions.go#L115-L174)

- 从 `plugins.yaml` 配置文件加载
- 每个插件定义 `ShortCut`、`Scopes`（适用视图范围）、`Command`、`Args` 等
- **范围匹配**: 插件的 `Scopes` 必须包含视图别名（通过 `inScope()` 检查），`"all"` 表示所有视图
- **权限检查**: 只读模式下跳过 `Dangerous=true` 的插件
- **冲突处理**: 键冲突时根据 `Override` 选项决定是报错还是覆盖原有绑定
- 标记为 `Plugin=true`，下次刷新时会先清除所有旧插件动作再重新加载

#### 热键系统 —— `hotKeyActions()` [internal/view/actions.go#L60-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/actions.go#L60-L106)

- 从 `hotkeys.yaml` 配置文件加载
- 每个热键定义 `ShortCut`、`Command`（要跳转的命令）、`Description`
- 实质是快速跳转的快捷方式：`gotoResource(cmd, path, clearStack)`
- 标记为 `HotKey=true`、`Shared=true`
- 同样支持 `Override` 覆盖原有绑定

---

## 四、完整调用时序（以 Pod 视图中按 L 键查看日志为例）

```
1. 用户按下 'L' 键
   ↓
2. tcell 捕获终端输入，生成 *tcell.EventKey
   ↓
3. tview 事件循环，事件向上传递
   ↓
4. 第一层拦截: view.App.keyboard()
   ├─ 调用 ui.AsKey(evt) → 转换为 tcell.Key = KeyL (108)
   ├─ 在 ui.App.actions 中查找 KeyL → 未找到（全局键没有 L）
   └─ 返回 evt 原样透传
   ↓
5. 第二层拦截: Table.keyboard()
   ├─ 排除 Up/Down/Shift+方向键等特殊键
   ├─ 调用 ui.AsKey(evt) → 转换为 tcell.Key = KeyL (108)
   ├─ 在 t.Actions() 中查找 KeyL → 找到！
   │   （这个键是 LogsExtender.bindKeys 注册的）
   ├─ 检查 !IsTopDialog() → 当前不是对话框
   └─ 调用 a.Action(evt) → LogsExtender.logsCmd(false)(evt)
   ↓
6. logsCmd 执行:
   ├─ 获取选中的 Pod path
   ├─ 构造 LogOptions
   └─ 调用 app.inject(NewLog(...)) 打开日志视图
   ↓
7. 返回 nil → 事件被消费，终止分发
```

---

## 五、关键协作机制总结

### 5.1 BindKeysFn 钩子链

`BindKeysFunc` 类型: `func(*ui.KeyActions)`

`Table.bindKeysFn` 是一个函数切片，所有 Extender 和具体资源都通过 `AddBindKeysFn()` 将自己的绑定函数追加到这个切片中。

**执行时机**: 
- `Browser.Init()` 中先执行一次（第 102-105 行）
- `Browser.refreshActions()` 中每次数据刷新时再次执行（第 662-664 行）

这种设计使得 **键绑定可以动态变化**（例如根据权限、只读模式、连接状态等条件决定是否注册某些键）。

### 5.2 键冲突与覆盖策略

| 层次 | 优先级 | 说明 |
|------|--------|------|
| 插件/热键 | 最高（可配置 Override） | 用户配置可以覆盖系统默认键 |
| 具体资源（如 Pod） | 高 | 可以覆盖 Extender 和基础层的键 |
| Extender 装饰器 | 中 | 外层 Extender 可以覆盖内层的 |
| Browser 动态层 | 低 | 可以覆盖基础层的通用键 |
| Table/Browser 基础层 | 最低 | 提供默认行为，随时可被覆盖 |

实际合并使用 `Merge()` / `Bulk()`（本质是 `map[k] = v` 赋值），后写入的值会覆盖先写入的。

### 5.3 危险操作保护

`ActionOpts.Dangerous` 标记用于：
1. **只读模式**: `ClearDanger()` 会清除所有危险操作的键绑定
2. **插件加载**: 只读模式下跳过 `Dangerous=true` 的插件
3. **对话框确认**: 部分危险操作触发时还会弹出确认对话框（如 Delete、Sanitize）

### 5.4 共享键机制

`ActionOpts.Shared=true` 表示该动作是 **跨视图共享的全局功能键**（如 Help、Quit、Clear Filter 等）。

在 `KeyActions.Hints()` 生成菜单提示时，共享键会被排除在视图专属提示之外，避免每个视图的菜单都重复显示相同的全局键。

---

## 六、视图注册机制

视图通过 `MetaViewers` 注册表关联 GVR 与视图构造函数 —— [internal/view/registrar.go](file:///d:/fz/0601-2/solo-dogfeeding/code/10-k9s/internal/view/registrar.go)

```go
MetaViewers map[*client.GVR]MetaViewer

type MetaViewer struct {
    viewerFn ViewerFunc   // 视图构造函数: func(*GVR) ResourceViewer
    enterFn  EnterFunc    // Enter 键回调（可选）
}
```

按类别分组注册：`coreViewers()`（核心资源）、`appsViewers()`（工作负载）、`rbacViewers()`、`batchViewers()`、`miscViewers()` 等。

当用户输入 `pod` 命令时，`Command` 解释器通过 GVR 查找到 `NewPod` 构造函数，创建 Pod 视图实例并注入到页面栈中。

---

## 七、架构设计亮点

1. **装饰器模式实现功能正交**: Extender 机制将日志、端口转发、镜像扫描等功能实现为可插拔装饰器，任意组合而不修改基础代码。

2. **输入捕获链实现分层解耦**: 利用 tview 的 `SetInputCapture` 构建 App→View 两层拦截，全局键与视图键各司其职。

3. **动态绑定适应运行时状态**: `refreshActions()` 在每次数据刷新时重建动作集合，能够响应连接状态、权限配置、只读模式等运行时变化。

4. **配置驱动的扩展性**: 插件和热键系统允许用户通过 YAML 配置自定义快捷键，无需修改代码。

5. **危险操作多层次防护**: 从键绑定（只读模式清除）→ 执行时对话框确认 → 资源权限检查，形成多层安全网。
