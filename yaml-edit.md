# K9s 资源描述与 YAML 编辑器链路分析

> 代码证据路径：所有代码引用均使用相对于仓库根目录 `internal/` 的相对路径，如 `internal/view/exec.go#L57-L97`

---

## 一、整体架构概览

K9s 的资源描述和 YAML 编辑器功能采用分层架构，核心设计哲学是：
**尽可能复用 Kubernetes 生态成熟组件（kubectl、printers、describer），自身只负责 UI 展示和交互编排。**

```
UI 层 (view)         →  Model 层 (model)  →  DAO 层 (dao)  →  外部/底层
─────────────────────────────────────────────────────────────────────────────
workload.go           yaml.go              resource.go       client.Connection
live_view.go          describe.go          describe.go       kubectl (外部进程)
yaml.go               -                    patch.go
exec.go               -                    helpers.go
browser.go            -                    dp.go/sts.go/ds.go
helpers.go            -                    -
screen_dump.go        -                    -
dir.go                -                    -
```

---

## 二、外部命令调用路径全景（★ 核心修正）

K9s 调用外部命令有三条独立路径，其中只有部分路径会注入 `K9S_EDITOR`：

```
                      用户操作
                         │
         ┌───────────────┴───────────────┐
         ▼                               ▼
  走通用执行入口                   绕开通用执行入口
  (经过 execute())                (不经过 execute())
         │                               │
  ┌──────┴──────┐                  ┌─────┴──────┐
  ▼             ▼                  ▼            ▼
runK()       edit()             runKu()    DAO 层 client API
  │             │                  │            │（不创建外部进程）
  └──────┬──────┘                  │            ├─ Get()/List()
         ▼                         │            ├─ ToYAML()
       run()                       │            └─ Describe()
         │                         │
         ▼                         ▼
     execute()                 oneShoot()
         │（K9S_EDITOR 注入点）     │（无 K9S_EDITOR 注入）
         ▼                         ▼
       pipe()               exec.Command 直接运行
  stdin/stdout/stderr 直通
```

### 2.1 路径一：走通用执行入口（会注入 K9S_EDITOR）

**调用链**：`XXX → runK()/edit() → run() → execute() → pipe()`

**K9S_EDITOR 注入位置**：`execute()` 函数中无条件执行（`internal/view/exec.go#L203-L214`）

**具体调用点（共 6 个场景）**：

| 场景 | 触发 | 入口函数 | 调用链 | 代码位置 |
|------|------|---------|--------|----------|
| 资源编辑 (kubectl edit) | 列表/YAML页按 `e` | `editRes()` | `editRes() → runK() → run() → execute()` | `internal/view/browser.go#L533-L558` |
| 容器 exec | Pod 按 `s` | `shellIn()` | `shellIn() → runK() → run() → execute()` | `internal/view/pod.go#L412-L416` |
| 容器 attach | Pod 按 `a` | `attachIn()` | `attachIn() → runK() → run() → execute()` | `internal/view/pod.go#L453-L458` |
| Node shell | Node 按 `s` | `sshIn()` | `sshIn() → runK() → run() → execute()` | `internal/view/exec.go#L369-L373` |
| Xray 编辑 | Xray 按 `e` | `editCmd()` | `editCmd() → runK() → run() → execute()` | `internal/view/xray.go#L458-L484` |
| 本地文件编辑 | ScreenDump/Dir 按 `e` | `edit()` | `edit() → run() → execute()` | `internal/view/screen_dump.go#L48-L56`<br/>`internal/view/dir.go#L132-L137` |

**★ 关键细节**：
- `execute()` 中的 K9S_EDITOR 注入逻辑是**无条件**的，没有 `if opts.binary == "kubectl"` 判断
- 所有经过 `execute()` 的命令（包括 `vim`、`code` 等编辑器）都会被注入 `KUBE_EDITOR` 环境变量
- 对于非 kubectl 命令，注入 `KUBE_EDITOR` 无实际影响（它们不读这个环境变量）

### 2.2 路径二：绕开通用执行入口（不会注入 K9S_EDITOR）

**调用链**：`XXX → runKu() → oneShoot() → exec.Command 直接运行`

**为什么不注入？** `oneShoot()` 直接创建 `exec.Command`，不经过 `execute()` 函数，所以不会执行 K9S_EDITOR 注入逻辑。

**具体调用点（共 2 个场景）**：

| 场景 | 触发 | 入口函数 | 调用链 | 代码位置 |
|------|------|---------|--------|----------|
| Dir 页面 Apply | Dir 选中文件按 `a` | `applyCmd()` | `applyCmd() → runKu() → oneShoot()` | `internal/view/dir.go#L217-L232` |
| Dir 页面 Delete | Dir 选中文件按 `d` | `deleteCmd()` | `deleteCmd() → runKu() → oneShoot()` | `internal/view/dir.go#L258-L273` |

**oneShoot() 实现**（`internal/view/exec.go#L270-L288`）：
```go
func oneShoot(ctx context.Context, opts *shellOpts) (string, error) {
    cmd := exec.CommandContext(ctx, opts.binary, opts.args...)
    buff := bytes.NewBufferString("")
    cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, buff, buff  // 输出收集到 buffer，不直通
    err := cmd.Run()
    return strings.Trim(buff.String(), "\n"), err
}
```

### 2.3 路径三：DAO 层读取（完全不调用外部 kubectl）

**特点**：直接使用 Go client 库调用 Kubernetes API，不创建外部进程，因此**完全不会触发 K9S_EDITOR 注入**。

**包含的操作**：
- `Get()` / `List()` - 资源读取
- `ToYAML()` - YAML 序列化
- `Describe()` - 资源描述
- `Patch()` - 资源更新（如重启）
- 所有 DAO 层数据访问

**代码位置**：
- `internal/dao/resource.go#L41-L53` - `ToYAML()`
- `internal/dao/describe.go#L14-L58` - `Describe()`
- `internal/dao/helpers.go#L85-L104` - 通用 `ToYAML()`
- `internal/dao/dp.go#L398-L444` - `restartRes()`

---

## 三、K9S_EDITOR 注入边界总结表

| 功能场景 | 调用入口 | 执行路径 | 是否注入 K9S_EDITOR | 代码位置 |
|---------|---------|---------|---------------------|----------|
| K8s 资源编辑 | `e` 键 | `editRes() → runK() → execute()` | ✅ 是 | `internal/view/browser.go#L553` |
| 容器 exec | `s` 键 | `shellIn() → runK() → execute()` | ✅ 是 | `internal/view/pod.go#L412` |
| 容器 attach | `a` 键 | `attachIn() → runK() → execute()` | ✅ 是 | `internal/view/pod.go#L456` |
| Node shell | `s` 键 | `sshIn() → runK() → execute()` | ✅ 是 | `internal/view/exec.go#L369` |
| Xray 编辑 | `e` 键 | `editCmd() → runK() → execute()` | ✅ 是 | `internal/view/xray.go#L478` |
| ScreenDump 编辑 | `e` 键 | `edit() → run() → execute()` | ✅ 是 | `internal/view/screen_dump.go#L53` |
| Dir 本地文件编辑 | `e` 键 | `edit() → run() → execute()` | ✅ 是 | `internal/view/dir.go#L135` |
| Dir Apply Manifest | `a` 键 | `applyCmd() → runKu() → oneShoot()` | ❌ 否 | `internal/view/dir.go#L222` |
| Dir Delete Manifest | `d` 键 | `deleteCmd() → runKu() → oneShoot()` | ❌ 否 | `internal/view/dir.go#L263` |
| YAML 查看 (y 键) | `y` 键 | DAO 层 ToYAML(), client API | ❌ 否 | `internal/dao/resource.go#L41` |
| 资源描述 (d 键) | `d` 键 | DAO 层 Describe(), client API | ❌ 否 | `internal/dao/describe.go#L14` |
| 资源列表刷新 | 自动/手动 | DAO 层 List(), client API | ❌ 否 | `internal/dao/` 各文件 |
| 重启 Deployment | `ctrl-r` | DAO 层 Patch(), client API | ❌ 否 | `internal/dao/dp.go#L398` |
| 本地保存 YAML | `Ctrl+S` | os.OpenFile() 写本地文件 | ❌ 否 | `internal/view/yaml.go#L72` |

---

## 四、资源描述链路 (Describe)

### 4.1 触发入口

资源描述功能通过快捷键 `d` 触发，入口在 `internal/view/workload.go#L156-L169` 的 `describeCmd` 函数。

### 4.2 View 层处理

`describeResource` 函数在 `internal/view/helpers.go#L126-L131`：

```go
func describeResource(app *App, _ ui.Tabular, gvr *client.GVR, path string) {
    v := NewLiveView(app, "Describe", model.NewDescribe(gvr, path))
    app.inject(v, false)
}
```

创建 `LiveView` 并注入 `Describe` 模型。

### 4.3 Model 层 - Describe 模型

`model.NewDescribe` 创建描述模型，通过 `Watch` 方法定期刷新数据。

### 4.4 DAO 层 - 核心描述逻辑

实际的资源描述在 `internal/dao/describe.go#L14-L58` 的 `Describe` 函数：

```go
func Describe(c client.Connection, gvr *client.GVR, path string) (string, error) {
    mapper := RestMapper{Connection: c}
    m, _ := mapper.ToRESTMapper()          // 1. 获取 REST mapper
    gvk, _ := m.KindFor(gvr.GVR())         // 2. 获取 GVK
    ns, n := client.Namespaced(path)       // 3. 解析命名空间和名称
    mapping, _ := mapper.ResourceFor(...)  // 4. 获取资源映射
    d, _ := describe.Describer(...)        // 5. 使用 kubectl describer 包（非外部命令）
    return d.Describe(ns, n, describe.DescriberSettings{ShowEvents: true})
}
```

**关键点**：
- 使用 `k8s.io/kubectl/pkg/describe` 包（Go 库），**不调用外部 kubectl 命令**
- 支持显示事件信息 (`ShowEvents: true`)

---

## 五、YAML 读取链路

### 5.1 触发入口

YAML 查看通过快捷键 `y` 触发，入口在 `internal/view/workload.go#L192-L208` 的 `yamlCmd` 函数：

```go
v := NewLiveView(w.App(), yamlAction, model.NewYAML(gvr, fqn))
```

### 5.2 Model 层 - YAML 模型

YAML 模型在 `internal/model/yaml.go#L26-L36` 中定义，**精确字段类型**如下：

```go
// internal/model/types.go#L32: ViewerToggleOpts 是 map[string]bool 的类型别名
type ViewerToggleOpts map[string]bool

type YAML struct {
    gvr       *client.GVR              // 指针：资源 GVR
    inUpdate  int32                    // int32：原子操作锁，防止并发刷新
    path      string                   // 字符串：资源路径 (ns/name)
    query     string                   // 字符串：搜索过滤查询
    lines     []string                 // 字符串切片：YAML 按行分割
    listeners []ResourceViewerListener // 接口切片：数据变更监听器
    options   ViewerToggleOpts         // map[string]bool：显示选项（如 ManagedFields）
    decode    bool                     // 布尔：Secret 是否解码显示
}
```

**相关接口定义**（`internal/model/types.go#L26-L46`）：

```go
type ResourceViewerListener interface {
    ResourceChanged(lines []string, matches fuzzy.Matches)
    ResourceFailed(error)
}

type ResourceViewer interface {
    GetPath() string
    Filter(string)
    GVR() *client.GVR
    ClearFilter()
    Peek() []string
    SetOptions(context.Context, ViewerToggleOpts)
    Watch(context.Context) error
    Refresh(context.Context) error
    AddListener(ResourceViewerListener)
    RemoveListener(ResourceViewerListener)
}
```

**数据刷新流程**：

1. `Watch` → 启动后台协程 `updater`，按 `defaultReaderRefreshRate` (5秒) 周期刷新
2. `refresh` → `CompareAndSwapInt32(&inUpdate, 0, 1)` 原子锁防止并发更新冲突
3. `reconcile` → 调用 `ToYAML` 获取数据，`reflect.DeepEqual` 比较无变化则跳过通知
4. `ToYAML` → 通过 DAO 层接口获取 YAML

特殊处理：Secret 支持解码，通过 `EncDecResourceViewer` 接口的 `Toggle()` 切换。

### 5.3 DAO 层 - YAML 序列化

分为两层，**均不调用外部 kubectl 命令**：

**第一层：Resource.ToYAML**（`internal/dao/resource.go#L41-L53`）
```go
func (r *Resource) ToYAML(path string, showManaged bool) (string, error) {
    o, _ := r.Get(context.Background(), path)  // 从 API 获取资源（client-go）
    return ToYAML(o, showManaged)              // 序列化为 YAML
}
```

**第二层：ToYAML 通用函数**（`internal/dao/helpers.go#L85-L104`）
```go
func ToYAML(o runtime.Object, showManaged bool) (string, error) {
    var p printers.ResourcePrinter = &printers.YAMLPrinter{}
    if !showManaged {
        o = o.DeepCopyObject()   // 隐藏管理字段时先深拷贝，避免修改原对象
        p = &printers.OmitManagedFieldsPrinter{Delegate: p}
    }
    var buff bytes.Buffer
    p.PrintObj(o, &buff)        // 使用 Kubernetes 官方 YAMLPrinter（Go 库）
    return buff.String(), nil
}
```

### 5.4 View 层 - LiveView 展示

`LiveView`（`internal/view/live_view.go`）核心功能：
- 自动刷新（`autoRefresh` 开关，默认启用）
- 搜索过滤（模糊/正则）
- YAML 语法高亮（`internal/view/yaml.go#L33-L62`）
- 快捷键：
  - `Ctrl+S` → 本地保存 YAML
  - `E` → 编辑资源（调用 `editRes`，走 `runK()` 路径）
  - `M` → 切换 ManagedFields 显示
  - `X` → 切换 Secret 解码

---

## 六、编辑器拉起链路：两条完全独立的编辑路径

**重要澄清**：代码中存在两个命名相似但用途**完全不同**的编辑入口，之前常被混淆：

| 函数 | 调用路径 | 用途 | 编辑对象 | K9S_EDITOR 处理 |
|------|----------|------|----------|----------------|
| `editRes()` + `runK()` | 资源编辑链路 | 编辑 K8s 资源 | kubectl 管理的临时 YAML | execute() 中转成 KUBE_EDITOR 传给 kubectl |
| `edit()` 函数 | 本地文件编辑链路 | 编辑本地文件 | ScreenDump / 目录中的本地文件 | edit() 中自己查 + execute() 中再次注入 |

---

### 6.1 路径一：资源编辑（kubectl edit）—— 快捷键 `e`

这是用户编辑 K8s 资源的主链路。

#### 6.1.1 触发入口

两个入口最终都调用 `editRes`：
1. **资源列表页**：`internal/view/workload.go#L172-L189`
2. **YAML 查看页**：`internal/view/live_view.go#L182-L194`

#### 6.1.2 权限检查与命令构建

`editRes`（`internal/view/browser.go#L533-L558`）：

```go
func editRes(app *App, gvr *client.GVR, path string) error {
    ns, n := client.Namespaced(path)

    // 权限检查：Patch 权限（注意不是 Update 权限）
    if ok, err := app.Conn().CanI(ns, gvr, n, client.PatchAccess); !ok || err != nil {
        return fmt.Errorf("current user can't edit resource %s", gvr)
    }

    // 构建 kubectl edit 命令参数
    args := []string{"edit", gvr.FQN(n)}
    if ns != client.BlankNamespace {
        args = append(args, "-n", ns)
    }

    // ★ 关键：调用 runK，不是调用 edit 函数
    return runK(app, &shellOpts{clear: true, args: args})
}
```

#### 6.1.3 runK：kubectl 命令包装

`runK`（`internal/view/exec.go#L57-L97`）负责找到 kubectl 并注入 K8s 连接参数：

```go
func runK(a *App, opts *shellOpts) error {
    bin, _ := exec.LookPath("kubectl")     // 1. 查找 kubectl
    args := []string{opts.args[0]}         // 取第一个子命令（edit）

    // 2. 注入认证参数
    if u, _ := a.Conn().Config().ImpersonateUser(); u != "" {
        args = append(args, "--as", u)
    }
    if g, _ := a.Conn().Config().ImpersonateGroups(); len(g) > 0 {
        args = append(args, "--as-group", g)
    }
    if *a.Conn().Config().Flags().Insecure {
        args = append(args, "--insecure-skip-tls-verify")
    }

    // 3. 注入上下文参数
    args = append(args, "--context", a.Config.K9s.ActiveContextName())
    if cfg := a.Conn().Config().Flags().KubeConfig; *cfg != "" {
        args = append(args, "--kubeconfig", *cfg)
    }

    // 4. 拼回剩余参数（资源类型、名称、命名空间）
    opts.args = append(args, opts.args[1:]...)
    opts.binary = bin

    // 5. 调用 run 挂起终端执行
    suspended, errChan, stChan := run(a, opts)
    // ... 收集输出和错误
}
```

#### 6.1.4 run：终端挂起/恢复管理

`run`（`internal/view/exec.go#L99-L122`）：

```go
func run(a *App, opts *shellOpts) (ok bool, errC chan error, outC chan string) {
    // background 模式不挂起（kubectl edit 不是 background）
    if !opts.background {
        a.Halt()          // 挂起 K9s UI 事件循环
        defer a.Resume()  // 命令完成后恢复 UI
    }

    return a.Suspend(func() {   // 挂起终端，切换到原始模式
        execute(opts, statusChan)  // 实际执行命令
    }), errChan, statusChan
}
```

#### 6.1.5 execute：进程执行 + K9S_EDITOR 传递（★ 关键细节）

`execute`（`internal/view/exec.go#L172-L239`）是**所有外部命令**的通用执行入口，包含**K9S_EDITOR 传递给 kubectl** 的关键逻辑：

```go
func execute(opts *shellOpts, statusChan chan<- string) error {
    clearScreen()
    ctx, cancel := context.WithCancel(context.Background())
    defer func() { clearScreen(); cancel() }()

    // 信号处理：Ctrl+C → 取消命令
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    // ...

    // ========== ★ K9S_EDITOR 注入逻辑（无条件执行！） ★ ==========
    cmd := exec.CommandContext(ctx, opts.binary, opts.args...)
    // opts.binary = kubectl，opts.args = [edit, resource/name, --context, ...]

    if env := os.Getenv("K9S_EDITOR"); env != "" {
        // 用户设置了 K9S_EDITOR → 转成 KUBE_EDITOR 传给子进程
        binTokens := strings.Split(env, " ")

        // 解析编辑器二进制路径（支持 "code -w" 这种带参数的配置）
        if bin, err := exec.LookPath(binTokens[0]); err == nil {
            binTokens[0] = bin  // 替换成绝对路径
            // 把解析后的完整编辑器命令设置为子进程的 KUBE_EDITOR 环境变量
            cmd.Env = append(os.Environ(),
                fmt.Sprintf("KUBE_EDITOR=%s", strings.Join(binTokens, " ")))
        }
    }
    // 注意：这段逻辑没有 if opts.binary == "kubectl" 判断，
    // 意味着所有经过 execute() 的命令（包括 vim/code/其他）都会被注入 KUBE_EDITOR！
    // ==============================================================

    // 单命令分支：stdin/stdout/stderr 直通终端
    // （多命令管道分支走另一个逻辑，kubectl edit 走单命令分支）
    cmds := []*exec.Cmd{cmd}
    return pipe(ctx, opts, statusChan, &o, &e, cmds...)
}
```

**pipe 函数中单命令分支**（`internal/view/exec.go#L554-L595`）：

```go
func pipe(...) error {
    if len(cmds) == 1 {
        cmd := cmds[0]
        if !opts.background {
            // ★ stdin/stdout/stderr 直接连到终端，k9s 不拦截任何数据
            cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
            _, _ = cmd.Stdout.Write([]byte(opts.banner))

            slog.Debug("Exec started")
            err := cmd.Run()  // ★ 阻塞等待 kubectl edit 完全结束
            // ...
        }
    }
}
```

---

### 6.2 K9S_EDITOR 注入的完整时序（★ 再次核准）

#### 6.2.1 资源编辑路径（kubectl edit）

```
用户按 'e'
    ↓
[browser.go:533] editRes()
    ├─ 权限检查
    ├─ 构建 args = ["edit", "pod/nginx", "-n", "default"]
    └─ 调用 runK(app, opts{clear:true, args})
        ↓
[exec.go:57] runK()
    ├─ LookPath("kubectl") → "/usr/local/bin/kubectl"
    ├─ 注入 --as/--as-group/--context/--kubeconfig
    ├─ opts.binary = "kubectl"
    │  opts.args   = ["edit", "pod/nginx", "--context", "xxx", ...]
    └─ 调用 run(a, opts)
        ↓
[exec.go:99] run()
    ├─ a.Halt()      // 挂起 K9s UI
    ├─ a.Suspend(...) // 挂起终端到原始模式
    └─ 调用 execute(opts)
        ↓
[exec.go:172] execute()  ★ K9S_EDITOR 注入点
    ├─ 创建 cmd = exec.Command("kubectl", ["edit", ...])
    │
    ├─ ★ 读取 os.Getenv("K9S_EDITOR")
    │   假设 K9S_EDITOR="code -w"
    │   ├─ strings.Split → ["code", "-w"]
    │   ├─ LookPath("code") → "/usr/bin/code"
    │   ├─ binTokens = ["/usr/bin/code", "-w"]
    │   └─ cmd.Env = append(os.Environ(),
    │                  "KUBE_EDITOR=/usr/bin/code -w")
    │
    └─ 调用 pipe(..., [cmd])
        ↓
[exec.go:554] pipe()
    ├─ cmd.Stdin = os.Stdin    ★ 直通
    ├─ cmd.Stdout = os.Stdout  ★ 直通
    ├─ cmd.Stderr = os.Stderr  ★ 直通
    └─ cmd.Run()  ← 阻塞等待 kubectl edit 进程结束
        ↓
        │  ┌─────────────── 黑盒：kubectl edit 内部 ───────────────┐
        │  │  1. GET /api/v1/namespaces/default/pods/nginx        │
        │  │  2. 写 /tmp/kubectl-edit-abc123.yaml                  │
        │  │  3. 读 KUBE_EDITOR → "/usr/bin/code -w"                │
        │  │  4. exec /usr/bin/code -w /tmp/kubectl-edit-abc123.yaml│
        │  │  5. 用户编辑保存 → code 退出                            │
        │  │  6. 读回 YAML → 校验 → PATCH API Server                 │
        │  │  7. 成功 → 输出 "pod/nginx edited"                      │
        │  │     失败 → 输出错误 + 保存临时文件 + 非0退出码            │
        │  └───────────────────────────────────────────────────────┘
        ↓
cmd.Run() 返回
    ↓
[exec.go:99] run() defer a.Resume() → K9s UI 恢复
    ↓
刷新资源列表
```

#### 6.2.2 本地文件编辑路径（edit 函数）

```
用户在 ScreenDump/Dir 页按 'e'
    ↓
[screen_dump.go:53 / dir.go:135] 调用 edit(app, opts{args: [localPath]})
    ↓
[exec.go:124] edit()  ★ k9s 自己查编辑器（不经过 kubectl）
    ├─ 遍历 editorEnvVars = ["K9S_EDITOR", "KUBE_EDITOR", "EDITOR"]
    │   假设 K9S_EDITOR="code -w"
    │   ├─ LookPath("code") → "/usr/bin/code"
    │   ├─ opts.args 从 [localPath] 变成 ["-w", localPath]
    │   └─ break
    ├─ opts.binary = "/usr/bin/code"
    └─ 调用 run(a, opts)
        ↓
[exec.go:99] run()
    ├─ a.Halt()
    ├─ a.Suspend(...)
    └─ 调用 execute(opts)
        ↓
[exec.go:172] execute()  ★ 再次处理 K9S_EDITOR！
    ├─ 创建 cmd = exec.Command("/usr/bin/code", ["-w", localPath])
    │
    ├─ ★ 再次读取 os.Getenv("K9S_EDITOR") = "code -w"
    │   ├─ LookPath("code") → "/usr/bin/code"
    │   └─ cmd.Env = append(os.Environ(),
    │                  "KUBE_EDITOR=/usr/bin/code -w")
    │   （给 code 子进程设 KUBE_EDITOR 环境变量，无实际影响）
    │
    └─ 调用 pipe(..., [cmd])
        ↓
[exec.go:554] pipe()
    ├─ stdin/stdout/stderr 直通
    └─ cmd.Run() 等待编辑器退出
```

**★ 重要发现：K9S_EDITOR 在本地文件编辑路径中被处理了两次！**
1. 第一次在 `edit()` 函数中：用于实际启动编辑器（有用）
2. 第二次在 `execute()` 函数中：给编辑器子进程注入 KUBE_EDITOR 环境变量（无用，副作用）

---

#### 6.2.3 KUBE_EDITOR / EDITOR 由谁处理？

| 场景 | 处理主体 | 优先级 | 代码位置 |
|------|---------|--------|----------|
| **资源编辑** (kubectl edit) | **kubectl 自身** | `KUBE_EDITOR` → `EDITOR` → 默认(vi/notepad) | kubectl 源码（不在 k9s 中），k9s 只负责把 K9S_EDITOR 转成 KUBE_EDITOR 注入 |
| **本地文件编辑** (edit 函数) | **k9s 自身** | `K9S_EDITOR` → `KUBE_EDITOR` → `EDITOR` | `internal/view/exec.go#L129-L152`，k9s 自己遍历环境变量 |

**k9s 的作用仅限于**（资源编辑时）：
- 如果用户设置了 `K9S_EDITOR`，把它转成 `KUBE_EDITOR` 注入 kubectl 子进程
- 如果用户没设置 `K9S_EDITOR`，不做任何处理，kubectl 自己按 `KUBE_EDITOR → EDITOR → 默认` 优先级找

---

### 6.3 两条编辑路径对比表

| 维度 | 资源编辑 (kubectl edit) | 本地文件编辑 (edit 函数) |
|------|------------------------|------------------------|
| 触发场景 | 列表页/YAML页按 `e` 编辑 K8s 资源 | ScreenDump/Dir 页面编辑本地文件 |
| 入口函数 | `editRes()` → `runK()` | `edit()` 函数直接调用 |
| 执行的命令 | `kubectl edit <resource>` | 用户配置的编辑器（vim/code/...） |
| 编辑对象 | kubectl 创建的临时 YAML 文件 | 本地磁盘上的已有文件 |
| K9S_EDITOR 处理次数 | 1 次（execute 中） | 2 次（edit 中 + execute 中） |
| K9S_EDITOR 处理位置 | execute() 中转成 KUBE_EDITOR 传给 kubectl | edit() 中直接用 + execute 中无效注入 |
| KUBE_EDITOR/EDITOR 处理 | **由 kubectl 自身处理** | edit() 函数按优先级查找 |
| 拉起编辑器的主体 | **kubectl** | **k9s 自己** |
| 终端 I/O | stdin/stdout/stderr 直通 | stdin/stdout/stderr 直通 |

---

## 七、YAML 回写与保存链路

### 7.1 模式一：本地保存 (Ctrl+S / ScreenDump)

用于把当前 LiveView 中显示的 YAML/描述内容保存到本地文件，**不涉及集群回写**。

**入口**：`internal/view/live_view.go#L379-L388`
```go
func (v *LiveView) saveCmd(*tcell.EventKey) *tcell.EventKey {
    name := fmt.Sprintf("%s--%s",
        strings.Replace(v.model.GetPath(), "/", "-", 1),
        strings.ToLower(v.title))
    // 保存到 ContextScreenDumpDir() 目录
    saveYAML(v.app.Config.K9s.ContextScreenDumpDir(),
        name, sanitizeEsc(v.text.GetText(true)))
}
```

**saveYAML 实现**：`internal/view/yaml.go#L72-L101`
```go
func saveYAML(dir, name, raw string) (string, error) {
    ensureDir(dir)
    // 文件名：{sanitized-name}--{unix-timestamp}.yaml
    fName := fmt.Sprintf("%s--%d.yaml", data.SanitizeFileName(name), time.Now().Unix())
    fpath := filepath.Join(dir, fName)
    // 文件权限 0600：仅所有者可读写，防止敏感信息泄露
    file, _ := os.OpenFile(fpath, os.O_CREATE|os.O_WRONLY, 0600)
    defer file.Close()
    file.WriteString(raw)
    return fpath, nil
}
```

---

### 7.2 模式二：资源编辑回写 (kubectl edit)

这是真正把编辑后的 YAML 写回 K8s 集群的模式。

#### 7.2.1 K9s 是否参与回写？

**答案：完全不参与。** 回写流程 100% 由 kubectl 完成。

**代码证据**（来自 `pipe` 函数的单命令分支，`internal/view/exec.go#L554-L595`）：
```go
if len(cmds) == 1 {
    cmd := cmds[0]
    if !opts.background {
        // ★ stdin/stdout/stderr 直接连到终端，k9s 不拦截任何数据
        cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
        _, _ = cmd.Stdout.Write([]byte(opts.banner))

        slog.Debug("Exec started")
        err := cmd.Run()  // ★ 阻塞等待 kubectl edit 完全结束
        // ...
    }
}
```

**k9s 在 kubectl edit 执行期间：**
- ❌ 不读取 kubectl 创建的临时 YAML 文件路径
- ❌ 不拦截编辑器的输入输出
- ❌ 不读取或处理编辑后的 YAML 内容
- ❌ 不参与发送 Patch/Update 请求到 API Server
- ❌ 不处理任何版本冲突逻辑
- ✅ 只负责：挂起/恢复终端、等待进程退出、收集退出状态

#### 7.2.2 kubectl edit 的完整流程（k9s 视角下的黑盒）

```
k9s 挂起终端
    ↓
kubectl edit 启动（在终端内独立运行）
    ├─ 1. GET /apis/.../resources/name  ← 从 API Server 拉取最新资源
    ├─ 2. 写入临时文件（如 /tmp/kubectl-edit-xxxxx.yaml）
    ├─ 3. 根据 KUBE_EDITOR / EDITOR 拉起编辑器
    ├─ 4. 用户编辑后保存，编辑器退出
    ├─ 5. 读取临时文件，做 YAML 校验
    ├─ 6. 对比 resourceVersion：
    │     ├─ 匹配 → PATCH/PUT 请求
    │     │          └─ 成功 → kubectl 输出 "xxx edited"
    │     │          └─ 失败 → 显示 API 返回的错误
    │     └─ 不匹配（版本冲突）→ 进入冲突处理：
    │                ├─ 显示 "the object has been modified" 错误
    │                ├─ 把用户修改保存到临时文件（带冲突提示）
    │                ├─ 告知用户手动处理后重试
    │                └─ 非 0 退出码
    └─ 7. 删除临时文件
    ↓
kubectl edit 进程退出
    ↓
k9s 恢复终端，刷新资源列表
```

---

## 八、版本冲突处理

### 8.1 kubectl edit 的版本冲突：K9s 完全不参与

**核心结论**：编辑 K8s 资源时，从获取资源、版本比对、冲突提示到重试，**全部由 kubectl 处理**，k9s 代码中没有任何处理版本冲突的逻辑。

**代码证据**（k9s 源码中搜索以下关键词，结果为零匹配）：
- `IsConflict` → 零匹配
- `StatusReasonConflict` → 零匹配
- `kerrors.IsConflict` → 零匹配
- `resourceVersion` → 仅出现在测试 JSON 数据中，业务代码零匹配
- `Conflict`（排除测试数据和日志）→ 零匹配

**进一步证据**：
1. k9s 不拦截 kubectl 的 stdout/stderr，冲突信息直接由 kubectl 打印到终端
2. 没有任何重试、重新拉取资源、合并 diff 的代码
3. 没有读取 kubectl 临时文件的代码

**用户看到的版本冲突流程**：
```
用户在 k9s 中按 e
    ↓
k9s 挂起，终端切到 kubectl edit
    ↓
（用户编辑期间，该资源被其他人修改）
    ↓
用户保存退出编辑器
    ↓
kubectl 检测到 resourceVersion 不匹配 → 在终端打印：
    error: Apply failed with 1 conflict: ...
    Please update the following fields and try again:
    ...
    Your changes have been saved to a temporary file: /tmp/...
    ↓
kubectl 以非 0 码退出
    ↓
k9s 恢复终端 → flash 显示 "Edit command failed: ..."
    ↓
（用户自行判断是否重新按 e 重试）
```

---

### 8.2 K9s 内部 Patch 操作：也无显式冲突处理

K9s 自身有一些 Patch 操作（如重启 Deployment/StatefulSet/DaemonSet），流程在 `internal/dao/dp.go#L398-L444`：

```go
func restartRes[T runtime.Object](ctx context.Context, f Factory, gvr *client.GVR, path string, opts *metav1.PatchOptions) error {
    o, _ := f.Get(gvr, path, true, labels.Everything())    // 1. 拉取当前资源（client API）
    // ... 权限检查
    before, _ := runtime.Encode(..., *r)                    // 2. 序列化当前状态
    after, _ := polymorphichelpers.ObjectRestarterFn(*r)    // 3. 生成重启后的状态（加 annotation）
    diff, _ := strategicpatch.CreateTwoWayMergePatch(...)   // 4. 生成 Strategic Merge Patch
    dial.AppsV1().Deployments(ns).Patch(                    // 5. 直接发送 Patch 请求（client API）
        ctx, n, types.StrategicMergePatchType, diff, *opts)
}
```

**特点**：
- 使用 `StrategicMergePatchType`，不是 `Update`
- 走 client-go API，**不调用外部 kubectl 命令**
- Patch 操作不强制匹配 resourceVersion（取决于 API Server 的配置）
- **没有任何冲突重试或冲突提示逻辑**，出错直接返回给上层 UI

---

### 8.3 YAML Model 的并发控制：仅防止刷新冲突

唯一的"冲突"处理在 YAML Model 的刷新逻辑中，使用原子锁防止**同一个 model 被并发刷新**：

`internal/model/yaml.go#L149-L161`：
```go
func (y *YAML) refresh(ctx context.Context) error {
    // CAS 原子操作：确保同一时刻只有一个协程在刷新
    if !atomic.CompareAndSwapInt32(&y.inUpdate, 0, 1) {
        slog.Debug("Dropping update...", slogs.GVR, y.gvr)
        return nil  // 如果正在刷新中，直接丢弃本次刷新请求
    }
    defer atomic.StoreInt32(&y.inUpdate, 0)
    return y.reconcile(ctx)
}
```

这**不是** K8s 资源的版本冲突处理，只是 UI 内部的刷新去抖。

---

## 九、K9s 源码证据 vs kubectl 黑盒边界

### 9.1 源码证据边界表

| 功能模块 | K9s 源码中有证据 | kubectl 黑盒推断 | 代码位置 |
|---------|-----------------|-----------------|----------|
| 挂起 K9s UI | ✅ | ❌ | `internal/view/exec.go#L99-L122` |
| 挂起终端到原始模式 | ✅ | ❌ | `a.Suspend()`（tview 库） |
| 查找 kubectl 路径 | ✅ | ❌ | `internal/view/exec.go#L57-L97` |
| 注入 --as/--context 等参数 | ✅ | ❌ | `internal/view/exec.go#L57-L97` |
| K9S_EDITOR → KUBE_EDITOR 注入 | ✅ | ❌ | `internal/view/exec.go#L203-L214` |
| stdin/stdout/stderr 直通终端 | ✅ | ❌ | `internal/view/exec.go#L554-L595` |
| 等待进程退出 + 收集退出码 | ✅ | ❌ | `cmd.Run()` in pipe() |
| 恢复 K9s UI 和终端 | ✅ | ❌ | `defer a.Resume()` in run() |
| 从 API Server GET 资源 | ❌ | ✅ | kubectl 内部（edit 命令） |
| 写入临时 YAML 文件 | ❌ | ✅ | kubectl 内部（editor 包） |
| 读取 KUBE_EDITOR/EDITOR 环境变量 | ❌ | ✅ | kubectl 内部（editor 包） |
| 拉起编辑器进程 | ❌ | ✅ | kubectl 内部（editor 包） |
| 读取编辑后的 YAML 内容 | ❌ | ✅ | kubectl 内部（edit 命令） |
| YAML 语法校验 | ❌ | ✅ | kubectl 内部 |
| resourceVersion 比对 | ❌ | ✅ | kubectl 内部（edit 命令） |
| 发送 PATCH/PUT 请求 | ❌ | ✅ | kubectl 内部 |
| 版本冲突处理 + 提示 | ❌ | ✅ | kubectl 内部（edit 命令） |
| 删除临时文件 | ❌ | ✅ | kubectl 内部 |

### 9.2 关键边界结论

1. **k9s 不读取 kubectl 产生的任何临时文件**
2. **k9s 不解析或处理编辑后的 YAML 内容**
3. **k9s 不向 API Server 发送任何编辑相关的 HTTP 请求**
4. **k9s 不处理任何版本冲突逻辑**
5. **k9s 的角色是透明的终端切换器**：把终端交给 kubectl → 等 kubectl 结束 → 把终端切回来

---

## 十、关键数据流转图

### 10.1 资源编辑（kubectl edit）链路

```
用户按 'e' 编辑 K8s 资源
    │
    ▼
[workload.go / live_view.go] editCmd()
    │  取 path → 解析 gvr, fqn
    ▼
[browser.go:533] editRes(app, gvr, fqn)
    │  1. CanI(PatchAccess) 权限检查
    │  2. 构建 args: ["edit", "pod/nginx", "-n", "default"]
    ▼
[exec.go:57] runK(app, opts)
    │  1. LookPath("kubectl")
    │  2. 注入 --as / --as-group / --context / --kubeconfig
    │  3. opts.binary = "kubectl"
    │     opts.args   = ["edit", "pod/nginx", "--context", "xxx", ...]
    ▼
[exec.go:99] run(app, opts)
    │  background=false，所以：
    │  1. app.Halt()   // 挂起 K9s UI
    │  2. app.Suspend(...)  // 挂起终端到原始模式
    │  3. defer app.Resume()
    ▼
[exec.go:172] execute(opts)
    │  clearScreen()
    │  ┌───────────────────────────────────────────────┐
    │  │ ★ K9S_EDITOR 处理（无条件执行！）                │
    │  │ if env := os.Getenv("K9S_EDITOR"); env != "" { │
    │  │   解析编辑器路径 + LookPath                    │
    │  │   cmd.Env = append(...,                        │
    │  │     "KUBE_EDITOR=/usr/bin/code -w")            │
    │  │ }  // 传给 kubectl 子进程                      │
    │  └───────────────────────────────────────────────┘
    │  cmd.Stdin = os.Stdin   ★ 直通
    │  cmd.Stdout = os.Stdout ★ 直通
    │  cmd.Stderr = os.Stderr ★ 直通
    │  cmd.Run()  ← 阻塞等待 kubectl edit 完全结束
    │
    ▼  （此时终端被 kubectl edit 接管，k9s 完全不参与）
    │
    │  ┌──────────────────── 黑盒：kubectl edit ────────────────────┐
    │  │  1. GET resource，带 resourceVersion                        │
    │  │  2. 写 /tmp/kubectl-edit-xxx.yaml                           │
    │  │  3. 读 KUBE_EDITOR(←k9s注入) / EDITOR → 拉起编辑器           │
    │  │  4. 用户编辑保存                                            │
    │  │  5. 读回 YAML → 校验                                        │
    │  │  6. PATCH 请求（带 resourceVersion）                        │
    │  │     → 成功：输出 "edited"                                   │
    │  │     → 冲突：输出错误信息 + 保存临时文件 + 非0退出码           │
    │  └────────────────────────────────────────────────────────────┘
    │
    ▼
kubectl edit 进程退出
    │
    ▼
execute() 清理屏幕 → 返回
    │
    ▼
app.Resume() → K9s UI 恢复
    │
    ▼
刷新资源列表，显示最新状态
```

### 10.2 本地文件编辑（edit 函数）链路

```
用户在 ScreenDump/Dir 页按 e 编辑本地文件
    │
    ▼
[screen_dump.go / dir.go] 调用 edit(app, opts{args: [localPath]})
    │
    ▼
[exec.go:124] edit(a *App, opts)   ★ 注意：不是 editRes！
    │  ┌────────────────────────────────────────────┐
    │  │ ★ k9s 自己按优先级查找编辑器（不经过 kubectl）│
    │  │ for e in [K9S_EDITOR, KUBE_EDITOR, EDITOR]: │
    │  │   LookPath + 拆分参数                        │
    │  │   opts.args = [编辑器参数..., 文件路径]       │
    │  └────────────────────────────────────────────┘
    │  opts.binary = 编辑器可执行文件
    │
    ▼
[exec.go:99] run(app, opts)
    │  Halt → Suspend → execute → Resume
    │
    ▼
[exec.go:172] execute(opts)
    │  stdin/stdout/stderr 直通终端
    │  ★ K9S_EDITOR 的再次处理逻辑会执行，但：
    │    - 此时 opts.binary 是 vim/code/... 不是 kubectl
    │    - 给 vim 子进程设 KUBE_EDITOR 环境变量无实际影响
    │  cmd.Run() 等待编辑器退出
    │
    ▼
编辑器退出
    │
    ▼
K9s UI 恢复
```

---

## 十一、安全与设计要点

### 11.1 安全设计

| 措施 | 位置 | 说明 |
|------|------|------|
| Patch 权限检查 | `internal/view/browser.go#L544` | 编辑前检查 `PatchAccess`，不是 `UpdateAccess` |
| 文件权限 0600 | `internal/view/yaml.go#L80` | 本地保存的 YAML 仅所有者可读写 |
| Secret 默认编码 | `internal/model/yaml.go#L209-L211` | Secret 不解码，用户按 `X` 手动切换 |
| 只读模式 | `internal/view/live_view.go#L160-L162` | `IsReadOnly()` 时不注册 `E` 键绑定 |
| ManagedFields 默认隐藏 | `internal/dao/helpers.go#L92-L94` | 避免大段管理字段干扰阅读，`M` 键切换 |

### 11.2 并发控制

- YAML Model 刷新：`atomic.CompareAndSwapInt32(&inUpdate, 0, 1)` 防并发刷新（仅 UI 内部去抖）
- 后台刷新协程：`context.WithCancel` + select，退出时及时释放

### 11.3 扩展点

1. **K9S_EDITOR 环境变量**：支持带参数，如 `K9S_EDITOR="code -w"`
2. **`EncDecResourceViewer` 接口**：Secret 解码的扩展点
3. **`resource.GenerateOptions`**：不同资源可实现自己的 `ToYAML`

---

## 十二、总结表

| 功能 | 实现方式 | 处理主体 | 代码位置 |
|------|----------|----------|----------|
| 资源描述 | kubectl describer 包（Go 库） | k9s 调用 Go 库 | `internal/dao/describe.go` |
| YAML 读取 | K8s YAMLPrinter（Go 库） | k9s 调用 Go 库 | `internal/dao/helpers.go#L85` |
| K8s 资源编辑 | kubectl edit 子进程 | **kubectl**（k9s 只挂起终端） | `internal/view/browser.go#L533` → `internal/view/exec.go#L57` |
| 本地文件编辑 | exec 拉起编辑器 | **k9s** 自己查环境变量 | `internal/view/exec.go#L124` |
| K9S_EDITOR → kubectl | 子进程 Env 设置 KUBE_EDITOR | k9s 中转 | `internal/view/exec.go#L203-L214` |
| K9S_EDITOR 注入范围 | **所有经过 execute() 的命令**（包括非 kubectl） | k9s（无条件） | `internal/view/exec.go#L203-L214` |
| KUBE_EDITOR/EDITOR 查找（资源编辑时） | kubectl 内部实现 | **kubectl** | kubectl 源码（不在 k9s 中） |
| KUBE_EDITOR/EDITOR 查找（本地文件时） | k9s 遍历 env vars | **k9s** | `internal/view/exec.go#L129-L152` |
| 版本冲突处理（kubectl edit 时） | resourceVersion 比对 + 提示 | **kubectl**（k9s 零参与） | kubectl 源码（不在 k9s 中） |
| 本地 YAML 保存 | os.OpenFile + 0600 | k9s | `internal/view/yaml.go#L72` |
| K9s 内部 Patch 冲突处理 | 无，直接透传错误 | k9s（无重试） | `internal/dao/dp.go#L398` |

**核心设计哲学**：编辑 K8s 资源时，k9s 的角色是「**透明的终端切换器**」—— 把终端交给 kubectl，等 kubectl 跑完再把终端切回来。编辑器选择、YAML 读写、版本比对、冲突处理、API 请求，**全部由 kubectl 完成**，k9s 完全不介入。
