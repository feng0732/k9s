# K9s 插件命令运行机制解析

本文档按代码执行顺序拆解 K9s 插件系统的三大核心流程：配置读取、参数注入、输出处理与进程回收。

---

## 整体执行流程图

```
用户按键触发
    │
    ▼
pluginActions() ── 加载插件配置，注册快捷键
    │
    ▼
pluginAction() ── 获取选中资源，收集输入（如有）
    │
    ▼
executePlugin() ── 环境变量构建 + 参数替换
    │
    ▼
run() ── 暂停 UI（前台模式），准备执行
    │
    ▼
execute() ── 创建进程 + 信号监听 + 管道处理
    │
    ▼
pipe() ── 单命令执行 / 多命令管道连接
    │
    ▼
状态通道返回结果 → Flash 消息提示
```

---

## 一、配置读取流程

### 1.1 加载入口：`pluginActions()`

**位置**: [actions.go#L115-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L115-L174)

这是插件系统的启动入口，在每个视图（Table/Xray 等）绑定键盘快捷键时被调用。

```go
func pluginActions(r Runner, aa *ui.KeyActions) error {
    // 1. 连接检查：无有效 K8s 连接则跳过
    if r.App().Conn() == nil || !r.App().Conn().ConnectionOK() {
        return nil
    }

    // 2. 清理旧的插件快捷键（防止重复）
    aa.Range(func(k tcell.Key, a ui.KeyAction) {
        if a.Opts.Plugin {
            aa.Delete(k)
        }
    })

    // 3. 获取插件配置路径
    path, err := r.App().Config.ContextPluginsPath()

    // 4. 创建插件容器并加载配置（loadExtra=true 加载 XDG 目录）
    pp := config.NewPlugins()
    if err := pp.Load(path, true); err != nil {
        return err
    }

    // 5. 遍历插件，按作用域过滤后注册快捷键
    for k := range pp.Plugins {
        if !inScope(pp.Plugins[k].Scopes, aliases) || (ro && pp.Plugins[k].Dangerous) {
            continue // 不在当前视图作用域内，或只读模式下危险插件被跳过
        }
        key, err := asKey(pp.Plugins[k].ShortCut) // 字符串→tcell.Key
        // ... 快捷键冲突检查 ...
        plugin := pp.Plugins[k]
        aa.Add(key, ui.NewKeyActionWithOpts(
            pp.Plugins[k].Description,
            pluginAction(r, &plugin),  // 绑定处理器
            ui.ActionOpts{Plugin: true, Dangerous: plugin.Dangerous},
        ))
    }
}
```

### 1.2 配置加载核心：`Plugins.Load()`

**位置**: [plugin.go#L122-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/config/plugin.go#L122-L148)

配置加载采用「多源合并」策略，优先级从低到高：

```go
func (p Plugins) Load(path string, loadExtra bool) error {
    // 第1层：全局配置文件 plugins.yaml
    if err := p.load(AppPluginsFile); err != nil { ... }

    // 第2层：集群/上下文专属配置（会覆盖同名全局插件）
    if err := p.load(path); err != nil { ... }

    // 第3层（可选）：XDG 标准目录扫描
    //   - $XDG_DATA_DIRS/k9s/plugins/
    //   - $XDG_DATA_HOME/k9s/plugins/
    //   - $XDG_CONFIG_HOME/k9s/plugins/
    for _, dir := range append(xdg.DataDirs, xdg.DataHome, xdg.ConfigHome) {
        path := filepath.Join(dir, "k9s/plugins")
        p.loadDir(path)  // 递归扫描 .yaml/.yml 文件
    }
}
```

### 1.3 单文件解析：`Plugins.load()`

**位置**: [plugin.go#L150-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/config/plugin.go#L150-L207)

支持三种文件格式（自动检测 JSON Schema）：

| Schema 类型 | 文件结构 | 示例 |
|---|---|---|
| `PluginSchema` | 单个插件（文件名作 key） | `snippet.1.yaml` 直接定义 shortCut/command 等 |
| `PluginsSchema` | `plugins:` 包裹的 map | `plugins.yaml` 中 `plugins: { blah: {...} }` |
| `PluginMultiSchema` | 顶级 map（无 plugins 包裹） | `snippet.multi.yaml` 直接 `crapola: {...}` |

```go
func (p *Plugins) load(path string) error {
    // 1. JSON Schema 校验（防止非法字段）
    scheme, err := data.JSONValidator.ValidatePlugins(bb)

    // 2. 根据 schema 类型选择反序列化方式
    switch scheme {
    case json.PluginSchema:
        var o Plugin
        yaml.Unmarshal(bb, &o)
        o.Validate()  // 校验 inputs 默认值合法性
        // 用文件名（去掉扩展名）作为插件名
        p.Plugins[strings.TrimSuffix(filepath.Base(path), filepath.Ext(path))] = o

    case json.PluginsSchema:
        var oo Plugins
        yaml.Unmarshal(bb, &oo)
        for k := range oo.Plugins {
            oo.Plugins[k].Validate()
            p.Plugins[k] = oo.Plugins[k]  // 直接合并，后加载覆盖先加载
        }

    case json.PluginMultiSchema:
        var oo plugins  // map[string]Plugin
        yaml.Unmarshal(bb, &oo)
        for k := range oo {
            oo[k].Validate()
            p.Plugins[k] = oo[k]
        }
    }
}
```

### 1.4 插件配置数据结构

**位置**: [plugin.go#L43-L67](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/config/plugin.go#L43-L67)

```go
type Plugin struct {
    Scopes          []string      // 作用域：如 ["po", "dp"]，"all" 表示全局
    Args            []string      // 命令参数（可含 $NAMESPACE 等占位符）
    ShortCut        string        // 快捷键：如 "shift-s"
    Override        bool          // 是否允许覆盖已有快捷键
    Pipes           []string      // 管道命令链：如 ["grep foo", "wc -l"]
    Description     string        // 描述文字
    Command         string        // 可执行二进制路径
    Confirm         *bool         // 是否二次确认（nil 时根据 Inputs 推断）
    Background      bool          // 后台执行（不暂停 UI）
    Dangerous       bool          // 危险操作（只读模式下禁用）
    OverwriteOutput bool          // 用插件输出覆盖 Flash 默认提示
    Inputs          []PluginInput // 用户输入表单定义
}
```

---

## 二、参数注入机制

### 2.1 触发处理器：`pluginAction()`

**位置**: [actions.go#L176-L204](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L176-L204)

```go
func pluginAction(r Runner, p *config.Plugin) ui.ActionHandler {
    return func(evt *tcell.EventKey) *tcell.EventKey {
        // 1. 获取当前选中行的资源路径（格式：namespace/name）
        path := r.GetSelectedItem()
        if path == "" { return evt }

        // 2. 检查环境变量函数是否就绪
        if r.EnvFn() == nil { return nil }

        // 3. 如果有 Inputs 定义，先弹出对话框收集用户输入
        if len(p.Inputs) > 0 {
            dialog.ShowPluginInputs(..., func(inputValues dialog.PluginInputValues) {
                executePlugin(r, p, inputValues)  // 收集完成后执行
            }, ...)
            return nil
        }

        // 4. 无 Inputs 直接执行
        executePlugin(r, p, nil)
        return nil
    }
}
```

### 2.2 输入表单：`ShowPluginInputs()`

**位置**: [plugin_inputs.go#L29-L179](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/ui/dialog/plugin_inputs.go#L29-L179)

支持四种输入类型，对应四种表单控件：

| 类型 | 控件 | 校验规则 |
|---|---|---|
| `InputTypeString` | 文本输入框 | 含空格时自动加引号 |
| `InputTypeNumber` | 数字输入框 | 实时校验 `ParseFloat` |
| `InputTypeBool` | 复选框 | 固定 "true"/"false" |
| `InputTypeDropdown` | 下拉选择 | 默认值必须在 Options 中 |

输入值在后续被注入为 `INPUT_<大写名称>` 环境变量。

### 2.3 环境变量构建：`executePlugin()` 前半段

**位置**: [actions.go#L206-L221](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L206-L221)

```go
func executePlugin(r Runner, p *config.Plugin, inputValues dialog.PluginInputValues) {
    // 步骤1：从视图获取基础环境变量
    env := r.EnvFn()()  // 调用 Table.defaultEnv() 或自定义 EnvFn

    // 步骤2：注入用户输入值（加 INPUT_ 前缀 + 大写）
    for name, value := range inputValues {
        env["INPUT_"+strings.ToUpper(name)] = value
    }

    // 步骤3：逐个参数执行占位符替换
    args := make([]string, len(p.Args))
    for i, a := range p.Args {
        arg, err := env.Substitute(a)  // 核心替换逻辑
        args[i] = arg
    }
    // ... 后续执行命令
}
```

### 2.4 环境变量来源分层

从 `Table.defaultEnv()` 开始，环境变量的构建链条：

```
Table.defaultEnv() [table.go#L145-L158]
    │
    ├─► defaultEnv(cfg, path, header, row) [helpers.go#L110-L124]
    │       │
    │       ├─► k8sEnv(cfg) [helpers.go#L75-L108]
    │       │     ├─ CONTEXT    = 当前 kubectl context 名
    │       │     ├─ CLUSTER    = 集群名
    │       │     ├─ USER       = 用户名
    │       │     ├─ GROUPS     = 用户组（逗号分隔）
    │       │     └─ KUBECONFIG = kubeconfig 文件路径
    │       │
    │       ├─ NAMESPACE / NAME = 从选中资源路径解析
    │       │   （client.Namespaced("default/nginx-pod") → ("default", "nginx-pod")）
    │       │
    │       └─ COL-<列名> = 选中行每列的值（如 COL-STATUS, COL-AGE）
    │
    ├─ FILTER           = 当前命令缓冲器文本（搜索过滤词）
    ├─ RESOURCE_GROUP   = GVR Group（如 "apps"）
    ├─ RESOURCE_VERSION = GVR Version（如 "v1"）
    └─ RESOURCE_NAME    = GVR Resource（如 "deployments"）
```

**示例环境变量字典**（选中 default 命名空间的一个 Pod）：
```
CONTEXT=my-cluster
CLUSTER=my-cluster
USER=kubernetes-admin
NAMESPACE=default
NAME=nginx-7c9b8c7d5f-abcde
COL-STATUS=Running
COL-AGE=3d2h
FILTER=nginx
RESOURCE_GROUP=
RESOURCE_VERSION=v1
RESOURCE_NAME=pods
INPUT_MYFIELD=userinput    （如有 Inputs）
```

### 2.5 占位符替换引擎：`Env.Substitute()`

**位置**: [env.go#L38-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/env.go#L38-L70)

正则匹配规则：
```
(\$(!?)([\w\-]+))|(\$\{(!?)([\w\-%/: ]+)})
```

支持四种占位符语法：

| 语法 | 示例 | 含义 |
|---|---|---|
| `$XXX` | `$NAMESPACE` | 基础变量替换 |
| `$!XXX` | `$!READY` | 布尔值取反（READY=true → 替换为 "false"） |
| `${XXX}` | `${COL-STATUS}` | 支持特殊字符（连字符、空格等） |
| `${!XXX}` | `${!COL-READY}` | 特殊字符变量取反 |

替换执行策略：
1. 先按占位符**长度降序**排序，避免短变量误替换（如 `$NAME` 不会先于 `$NAMESPACE` 匹配）
2. 对布尔值（`ParseBool` 成功）自动处理取反逻辑
3. 未找到的变量只打 Warn 日志，不报错（保留原占位符文本）

```go
func (e Env) Substitute(arg string) (string, error) {
    matches := envRX.FindAllStringSubmatch(arg, -1)
    sort.Slice(matches, func(i, j int) bool {
        return len(matches[i][0]) > len(matches[j][0])  // 长度优先，避免前缀冲突
    })
    for _, m := range matches {
        key, inverse := keyFromSubmatch(m)
        v, ok := e[strings.ToUpper(key)]
        if b, err := strconv.ParseBool(v); err == nil {
            if inverse { b = !b }
            v = fmt.Sprintf("%t", b)
        }
        arg = strings.ReplaceAll(arg, m[0], v)
    }
    return arg, nil
}
```

---

## 三、输出处理与进程回收

### 3.1 执行调度：`run()`

**位置**: [exec.go#L99-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L99-L122)

前台/后台模式的分水岭：

```go
func run(a *App, opts *shellOpts) (ok bool, errC chan error, outC chan string) {
    errChan := make(chan error, 1)
    statusChan := make(chan string, 1)

    // 后台模式：直接执行，不挂起 UI
    if opts.background {
        if err := execute(opts, statusChan); err != nil {
            errChan <- err
            a.Flash().Errf("Exec failed %q: %s", opts, err)
        }
        close(errChan)
        return true, errChan, statusChan
    }

    // 前台模式：
    a.Halt()           // 1. 暂停 UI 事件循环（停止刷新）
    defer a.Resume()   // 4. 函数返回时恢复 UI

    // 2. Suspend tview 并切换到终端原始模式
    return a.Suspend(func() {
        if err := execute(opts, statusChan); err != nil {
            errChan <- err
        }
        close(errChan)  // 3. 关闭通道通知外部完成
    }), errChan, statusChan
}
```

### 3.2 进程生命周期管理：`execute()`

**位置**: [exec.go#L172-L239](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L172-L239)

这是最核心的函数，负责三件事：**上下文与信号**、**进程创建**、**管道链构建**。

```go
func execute(opts *shellOpts, statusChan chan<- string) error {
    // ──────────────── 1. 上下文与信号处理 ────────────────
    if opts.clear { clearScreen() }
    ctx, cancel := context.WithCancel(context.Background())
    defer func() {
        if !opts.background {
            cancel()       // 非后台模式：函数退出即取消上下文（回收子进程）
            clearScreen()  // 清屏回到 k9s
        }
    }()

    var interrupted bool
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)  // 监听 Ctrl+C / SIGTERM
    go func(cancel context.CancelFunc) {
        defer slog.Debug("Got signal canceled")
        select {
        case sig := <-sigChan:
            slog.Debug("Command canceled with signal", slogs.Sig, sig)
            cancel()  // 收到信号 → 取消上下文 → CommandContext 杀子进程
        case <-ctx.Done():
            slog.Debug("Signal context canceled!")
        }
        interrupted = true  // 标记中断，后续不把错误当失败
    }(cancel)

    // ──────────────── 2. 创建主进程 ────────────────
    cmds := make([]*exec.Cmd, 0, 1)
    cmd := exec.CommandContext(ctx, opts.binary, opts.args...)

    // 特殊处理：若设置了 K9S_EDITOR，把它同步给 KUBE_EDITOR 环境变量
    if env := os.Getenv("K9S_EDITOR"); env != "" {
        // （编辑器可能带参数，如 "code -w"，需要 LookPath 第一个 token）
        binTokens := strings.Split(env, " ")
        if bin, err := exec.LookPath(binTokens[0]); err == nil {
            binTokens[0] = bin
            cmd.Env = append(os.Environ(), fmt.Sprintf("KUBE_EDITOR=%s", strings.Join(binTokens, " ")))
        }
    }
    cmds = append(cmds, cmd)

    // ──────────────── 3. 构建管道链 ────────────────
    for _, p := range opts.pipes {
        tokens := strings.Split(p, " ")
        if len(tokens) < 2 { continue }
        cmd := exec.CommandContext(ctx, tokens[0], tokens[1:]...)
        cmds = append(cmds, cmd)
    }

    // ──────────────── 4. 调用 pipe() 执行 ────────────────
    var o, e bytes.Buffer
    err := pipe(ctx, opts, statusChan, &o, &e, cmds...)
    if err != nil && !interrupted {
        return errors.Join(err, fmt.Errorf("%s", e.String()))  // 非中断错误返回
    }
    return nil  // Ctrl+C 中断不算错误
}
```

**进程回收关键点**：
- 使用 `exec.CommandContext` 而非 `exec.Command` —— 上下文取消时自动发 kill 信号
- 前台模式 `defer cancel()` 确保函数返回时所有子进程被回收
- 信号监听 goroutine 保证 Ctrl+C 能正确传播到子进程树
- `interrupted` 标记区分「真错误」和「用户主动中断」，避免误报

### 3.3 管道与输出：`pipe()`

**位置**: [exec.go#L549-L615](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L549-L615)

分两条路径：**单命令**和**多命令管道**。

#### 路径 A：单命令（len(cmds) == 1）

```go
if len(cmds) == 1 {
    cmd := cmds[0]

    // 后台模式：异步 goroutine 执行，输出写入 statusChan
    if opts.background {
        go func() {
            cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, w, e  // 缓冲捕获
            if err := cmd.Run(); err != nil {
                slog.Error("Command exec failed", slogs.Error, err)
            } else {
                // 把 stdout 按行拆成 "[output] xxx" 消息
                for _, l := range strings.Split(w.String(), "\n") {
                    if l != "" {
                        statusChan <- fmt.Sprintf("%s %s", outputPrefix, l)
                    }
                }
                statusChan <- fmt.Sprintf("Command completed successfully: %q", ...)
            }
            close(statusChan)
        }()
        return nil  // 立即返回，不等 goroutine
    }

    // 前台模式：直接连接终端 stdin/stdout/stderr
    cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
    _, _ = cmd.Stdout.Write([]byte(opts.banner))  // 打印头部横幅（如 Pod 信息）

    err := cmd.Run()
    var ex *exec.ExitError
    if errors.As(err, &ex) && !ex.Exited() {
        return nil  // 信号终止（非 ExitCode）不算错误
    }
    close(statusChan)
    return err
}
```

#### 路径 B：多命令管道（len(cmds) > 1）

```go
last := len(cmds) - 1
// 连接管道：cmd1.Stdout → io.Pipe → cmd2.Stdin
for i := range cmds {
    cmds[i].Stderr = os.Stderr  // 所有命令的 stderr 直连终端
    if i+1 < len(cmds) {
        r, w := io.Pipe()
        cmds[i].Stdout, cmds[i+1].Stdin = w, r
    }
}
cmds[last].Stdout = os.Stdout  // 最后一个命令 stdout → 终端

// 逐个 Start（不能用 Run，因为要并行跑）
for _, cmd := range cmds {
    if err := cmd.Start(); err != nil {
        return err
    }
}

// 等待最后一个命令完成（管道链自然收尾）
return cmds[last].Wait()
```

**注意**：管道模式下：
- 所有命令**共享同一个 `ctx`**，一个被取消全部终止
- 只有最后一个命令的 stdout 显示到终端
- 中间命令若写满管道缓冲，会被 Go 的 `io.Pipe` 自动流控

### 3.4 完成后状态处理：`executePlugin()` 后半段

**位置**: [actions.go#L223-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L223-L265)

```go
cb := func() {
    opts := shellOpts{
        binary:     p.Command,
        background: p.Background,
        pipes:      p.Pipes,
        args:       args,
    }
    suspend, errChan, statusChan := run(r.App(), &opts)

    // ────── 等待错误通道 ──────
    var errs error
    for e := range errChan {  // 阻塞直到 close(errChan)
        errs = errors.Join(errs, e)
    }
    if errs != nil {
        if !strings.Contains(errs.Error(), "signal: interrupt") {
            r.App().cowCmd(errs.Error())  // 用奶牛图显示错误
            return
        }
    }

    // ────── 异步消费状态通道 ──────
    go func() {
        for st := range statusChan {
            if !p.OverwriteOutput {
                // 默认行为：在 Flash 显示成功提示（含命令摘要）
                r.App().Flash().Infof("Plugin command launched successfully: %q", st)
            } else if strings.Contains(st, outputPrefix) {
                // overwriteOutput=true：截取第一条 "[output] xxx" 作为 Flash 消息
                infoMsg := strings.TrimPrefix(st, outputPrefix)
                r.App().Flash().Info(strings.TrimSpace(infoMsg))
                return  // 只取第一条，后续忽略
            }
        }
    }()
}

// 根据 ShouldConfirm() 判断是否弹确认框
if p.ShouldConfirm() {
    msg := fmt.Sprintf("Run?\n%s %s", p.Command, strings.Join(args, " "))
    dialog.ShowConfirm(..., cb, func() {})  // 确认后才调用 cb()
    return
}
cb()  // 无需确认直接执行
```

`ShouldConfirm()` 的判断逻辑：
```go
// plugin.go#L75-L80
func (p *Plugin) ShouldConfirm() bool {
    if p.Confirm != nil {
        return *p.Confirm          // 用户显式配置优先
    }
    return len(p.Inputs) > 0       // 有 Inputs 时默认需要确认
}
```

---

## 附录：关键类型索引

| 类型/函数 | 文件位置 | 作用 |
|---|---|---|
| `Plugin` | [plugin.go#L54-L67](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/config/plugin.go#L54-L67) | 插件配置结构体 |
| `Plugins.Load()` | [plugin.go#L122-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/config/plugin.go#L122-L148) | 多路径配置加载入口 |
| `pluginActions()` | [actions.go#L115-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L115-L174) | 扫描并注册插件快捷键 |
| `executePlugin()` | [actions.go#L206-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/actions.go#L206-L265) | 构建环境、参数替换、执行回调 |
| `Env.Substitute()` | [env.go#L38-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/env.go#L38-L70) | 占位符正则替换引擎 |
| `defaultEnv()` | [helpers.go#L110-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/helpers.go#L110-L124) | 构建 K8s + 行列环境变量 |
| `run()` | [exec.go#L99-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L99-L122) | 前后台模式分发与 UI 挂起 |
| `execute()` | [exec.go#L172-L239](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L172-L239) | 进程创建、信号监听、管道链 |
| `pipe()` | [exec.go#L549-L615](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/view/exec.go#L549-L615) | 单命令/管道模式下的 I/O 与生命周期 |
| `ShowPluginInputs()` | [plugin_inputs.go#L29-L179](file:///d:/fz/0601-2/solo-dogfeeding/code/11-k9s/internal/ui/dialog/plugin_inputs.go#L29-L179) | 用户输入表单对话框 |

---

## 一句话总结

K9s 插件系统 = **多源 YAML 配置合并** → **`Env` 字典承载 K8s 上下文 + 行列值 + 用户输入** → **`Substitute()` 正则替换占位符** → **`CommandContext`+信号通道确保进程可回收** → **管道链/后台两种 I/O 模式通过 statusChan 回传结果**。
