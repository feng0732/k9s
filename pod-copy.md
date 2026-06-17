# K9s 容器文件 CP 传输路径代码分析

依赖版本依据：[go.mod:41](go.mod#L41-L41) 声明 `k8s.io/kubectl v0.35.1`；本文中 kubectl cp 的源代码分析片段来自该版本（用于说明 tar 流管道的内部原理）。**注意：K9s 实际运行时调用的外部 kubectl 二进制版本可能与此不同**，详见下文「零、两个 kubectl 版本的区分」。

---

## 零、两个 kubectl 版本的区分

K9s 代码中同时存在两个完全独立的"kubectl"概念，二者版本可能不一致，需要明确区分：

### 0.1 依赖声明的 kubectl（编译时链接）

这是 [go.mod:41](go.mod#L41-L41) 中声明的 Go 模块依赖 `k8s.io/kubectl v0.35.1`。

**性质**：编译期依赖，与 K9s 二进制静态链接。

**使用场景**：全库共有 11 处 import `k8s.io/kubectl` 包，全部用于 CP 之外的功能：

| 文件 | import 路径 | 用途 |
|------|------------|------|
| [internal/render/sc.go](internal/render/sc.go#L15-L15) | `k8s.io/kubectl/pkg/util/storage` | StorageClass 容量渲染 |
| [internal/render/cust_col.go](internal/render/cust_col.go#L14-L14) | `k8s.io/kubectl/pkg/cmd/get` | 自定义列渲染 |
| [internal/dao/rs.go](internal/dao/rs.go#L20-L21) | `k8s.io/kubectl/pkg/cmd/util` + `polymorphichelpers` | ReplicaSet 操作 |
| [internal/dao/port_forwarder.go](internal/dao/port_forwarder.go#L25-L25) | `k8s.io/kubectl/pkg/cmd/util` | 端口转发 |
| [internal/dao/node.go](internal/dao/node.go#L22-L23) | `k8s.io/kubectl/pkg/drain` + `scheme` | Node 排水操作 |
| [internal/dao/dynamic.go](internal/dao/dynamic.go#L20-L20) | `k8s.io/kubectl/pkg/cmd/util` | 动态资源操作 |
| [internal/dao/dp.go](internal/dao/dp.go#L23-L24) | `k8s.io/kubectl/pkg/polymorphichelpers` + `scheme` | Deployment 滚动更新 |
| [internal/dao/describe.go](internal/dao/describe.go#L11-L11) | `k8s.io/kubectl/pkg/describe` | 资源描述 |

**关键证据**：全库 grep `k8s.io/kubectl/pkg/cmd/cp` 零匹配——**依赖声明的 kubectl v0.35.1 从未被用于 CP 功能**。

### 0.2 运行时的外部 kubectl（进程边界调用）

这是通过 `exec.LookPath("kubectl")` 在用户系统 PATH 中找到的外部二进制。

**性质**：运行期依赖，独立进程，与 K9s 通过 stdin/stdout 通信。

**代码定位**：
- [internal/view/exec.go:58](internal/view/exec.go#L58-L58)：`runK()` 中的 `bin, err := exec.LookPath("kubectl")`
- [internal/view/exec.go:242](internal/view/exec.go#L242-L242)：`runKu()` 中的 `bin, err := exec.LookPath("kubectl")`

**实际走外部 kubectl 的命令（代码证据）**：

| 命令 | 入口函数 | 构造参数的函数 | 最终调用 runK |
|------|---------|--------------|-------------|
| CP | [pod.go:109-115](internal/view/pod.go#L109-L115) T 键 → transferCmd | 直接构造 `["cp", ...]` | [pod.go:293-332](internal/view/pod.go#L293-L332) ack 回调 |
| Shell | [pod.go:222](internal/view/pod.go#L222-L238) shellCmd | [pod.go:461-468](internal/view/pod.go#L461-L468) computeShellArgs → `["exec", "-it", ..., "--", "sh", "-c", ...]` | [pod.go:412-416](internal/view/pod.go#L412-L416) shellIn() |
| Attach | [pod.go:240-256](internal/view/pod.go#L240-L256) attachCmd | [pod.go:478-499](internal/view/pod.go#L478-L499) buildShellArgs → `["attach", "-it", ...]` | [pod.go:456](internal/view/pod.go#L456-L456) attachIn() |

**不走外部 kubectl 的命令（代码证据）**：

| 命令 | 入口函数 | 实现方式 |
|------|---------|---------|
| Dir | [app.go:679-696](internal/view/app.go#L679-L696) App.dirCmd | K9s 内置 `Dir` 视图组件，直接 `os.Stat(path)` 读取本地文件系统，完全不涉及 kubectl |
| PortForward | [dao/port_forwarder.go:121-171](internal/dao/port_forwarder.go#L121-L171) PortForwarder.Start | 直接使用 `k8s.io/client-go/tools/portforward` 库 + SPDY 连接，**不调用外部 kubectl 二进制**。代码证据：[dao/port_forwarder.go:23](internal/dao/port_forwarder.go#L23-L23) import `k8s.io/client-go/tools/portforward`，[dao/port_forwarder.go:152-170](internal/dao/port_forwarder.go#L152-L170) 直接构造 `rest.RESTClientFor` 并调用 `forwardPorts()` |

**版本**：完全由用户安装决定，可能是 v1.27、v1.28、v1.29、v1.30 等任意版本，**与 go.mod 中的 v0.35.1 没有绑定关系**。

> **k8s 版本号说明**：kubectl 模块使用 `v0.x.y` 版本方案（如 v0.35.1），对应 Kubernetes 发行版的 `v1.x.y`（如 v1.35.1）。但这只是模块命名约定，不意味着运行时的外部 kubectl 也必须是这个版本。

### 0.3 二者对比与版本不一致风险

| 维度 | 依赖声明的 kubectl | 运行时的外部 kubectl |
|------|-------------------|---------------------|
| 位置 | go.mod 声明，编译进 K9s 二进制 | 用户 PATH 中的独立二进制 |
| 版本 | 固定 v0.35.1 | 用户安装的任意版本 |
| 绑定方式 | Go 模块静态链接 | `os/exec` 进程间调用 |
| 用于 CP | ❌ 从未 | ✅ 是 CP 功能的实际执行者 |
| 用于 Shell | ❌ | ✅ 走 `kubectl exec` |
| 用于 Attach | ❌ | ✅ 走 `kubectl attach` |
| 用于 PortForward | ✅ `kubectl/pkg/cmd/util` 仅用于 Factory，真正传输走 client-go SPDY | ❌ 完全不走外部二进制，直连 kube-apiserver |
| 用于 Dir | N/A（纯本地文件系统） | ❌ 完全不涉及 kubectl |
| 用于其他功能 | ✅ drain、describe、render 等 11 处 import | — |
| 升级方式 | 修改 go.mod 重新编译 K9s | 用户自行 `brew install kubectl` 等 |

**版本不一致风险**：
- 只要 `kubectl cp` 的 CLI 参数（`-c`、`--no-preserve`、`--retries`、`[[ns/]pod:]path` 格式）保持兼容，不同版本可正常工作
- kubectl cp 的这些 flag 自 v1.12 引入以来基本稳定
- 若未来 kubectl 更改 cp 子命令的 flag 语义或输出格式，就会出现 K9s 与外部 kubectl 的版本不兼容问题

**进程边界再次确认**：
CP 功能的执行路径是 `K9s Go 代码 → exec.CommandContext → kubectl 二进制进程`。K9s 的 CP 相关代码在 [internal/view/pod.go](internal/view/pod.go#L286-L353) 和 [internal/view/exec.go](internal/view/exec.go#L57-L97) 中构造完命令行参数、调用完 `cmd.Run()` 之后就结束了，后续的 tar 流管道、断点续传等逻辑完全在外部 kubectl 进程内执行，K9s 代码不可见。

### 0.4 命令边界总览（cp / shell / attach / dir / port-forward）

将 K9s 中的 5 个常见命令按调用边界分类：

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  类别 A：通过外部 kubectl 二进制（进程边界，依赖用户 PATH 中的 kubectl）            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  cp     → kubectl cp        runK([pod.go:293-332]) → exec.CommandContext        │
│  shell  → kubectl exec -it  shellIn([pod.go:412-416]) → runK                    │
│  attach → kubectl attach -i attachIn([pod.go:456]) → runK                       │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│  类别 B：使用 client-go 库直接调用 kube-apiserver（库调用边界，无外部进程）         │
├──────────────────────────────────────────────────────────────────────────────────┤
│  port-forward → SPDY 升级连接  PortForwarder.Start([dao/port_forwarder.go:121]) │
│                 → rest.RESTClientFor → forwardPorts()                            │
│                 → k8s.io/client-go/tools/portforward                              │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│  类别 C：K9s 内置实现（纯本地，完全不涉及 Kubernetes 网络）                         │
├──────────────────────────────────────────────────────────────────────────────────┤
│  dir → os.Stat([app.go:681]) → NewDir 视图组件 → 本地文件系统遍历                 │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**为什么容易混淆**：
- PortForward 在 [dao/port_forwarder.go:25](internal/dao/port_forwarder.go#L25-L25) 有一行 `import cmdutil "k8s.io/kubectl/pkg/cmd/util"`，但这只是用 kubectl 的 Factory 创建客户端，**绝不意味着调用外部 kubectl port-forward**
- `k8s.io/kubectl/pkg/cmd/util` 里的 Factory 是共享工具，既可用于构造内部 client，也可被 kubectl 自身的子命令使用
- 判断是否走外部进程的唯一依据是：**代码中是否调用了 `exec.LookPath("kubectl")` + `exec.CommandContext` + `runK()` 这条链路**

---

## 一、K9s 与 kubectl cp 的调用边界

**核心事实**（已在「零」章通过代码证明）：K9s 不直接调用 kubectl cp 的 Go API，而是通过 `os/exec` 启动外部 `kubectl` 二进制进程。

调用链路：

```
用户按键 T
  └─▶ K9s Go 代码（本地源码可见）
        ├─ 构造 CLI 参数字符串数组
        ├─ 注入认证参数（--context、--kubeconfig、--as 等）
        └─ os/exec 启动外部 kubectl 进程
              └─▶ kubectl cp（外部进程，二进制形式，非 Go API 调用）
                    ├─ 解析参数
                    ├─ tar 流管道
                    └─ exec 通道传输
```

K9s 与 kubectl cp 之间的调用边界是 **进程边界**，边界数据是命令行参数（`[]string`）和进程 stdout/stderr 字节流。

---

## 二、命令构造：从按键到 kubectl cp 参数

### 2.1 按键绑定

[pod.go:109-115](internal/view/pod.go#L109-L115) 将 `T` 键标记为危险操作并绑定 `transferCmd`：

```go
ui.KeyT: ui.NewKeyActionWithOpts(
    "Transfer",
    p.transferCmd,
    ui.ActionOpts{
        Visible:   true,
        Dangerous: true,
    }),
```

绑定入口由 [bindDangerousKeys](internal/view/pod.go#L86-L124) 管理，仅在非只读模式下注册（[pod.go:126-134](internal/view/pod.go#L126-L134)）。

### 2.2 transferCmd：对话框弹出 + ack 回调

[transferCmd](internal/view/pod.go#L286-L353) 是全部 CP 逻辑的入口。它做两件事：

**(1) 拉取 Pod 信息并准备对话框选项** ([pod.go:334-350](internal/view/pod.go#L334-L350))

```go
pod, err := fetchPod(p.App().factory, path)
opts := dialog.TransferDialogOpts{
    Title:      "Transfer",
    Containers: fetchContainers(&pod.ObjectMeta, &pod.Spec, false),
    Message:    "Download Files",
    Pod:        fmt.Sprintf("%s/%s:", ns, n),
    Ack:        ack,
    Retries:    defaultTxRetries,
    Cancel:     func() {},
}
dialog.ShowUploads(&d, p.App().Content.Pages, &opts)
```

默认重试次数由常量 `defaultTxRetries = 999` 控制（[pod.go:41](internal/view/pod.go#L41-L41)）。

**(2) ack 回调构造 kubectl cp 参数** ([pod.go:293-332](internal/view/pod.go#L293-L332))

```go
ack := func(args dialog.TransferArgs) bool {
    // 上传时校验本地文件存在
    local := args.To
    if !args.Download {
        local = args.From
    }
    if _, err := os.Stat(local); !args.Download && errors.Is(err, fs.ErrNotExist) {
        p.App().Flash().Err(err)
        return false
    }

    // 构造 kubectl cp 的 args
    opts := make([]string, 0, 10)
    opts = append(opts,
        "cp",
        strings.TrimSpace(args.From),
        strings.TrimSpace(args.To),
        fmt.Sprintf("--no-preserve=%t", args.NoPreserve),
        fmt.Sprintf("--retries=%d", args.Retries),
    )
    if args.CO != "" {
        opts = append(opts, "-c="+args.CO)
    }
    opts = append(opts, fmt.Sprintf("--retries=%d", args.Retries))  // 注意：这里重复追加了 --retries

    cliOpts := shellOpts{
        background: true,
        args:       opts,
    }
    // ...
    if err := runK(p.App(), &cliOpts); err != nil { ... }
    return true
}
```

注意 [pod.go:314](internal/view/pod.go#L314-L314) 存在一个参数重复 bug：`--retries` 在 [pod.go:309](internal/view/pod.go#L309-L309) 和 [pod.go:314](internal/view/pod.go#L314-L314) 被追加了两次。

### 2.3 对话框参数收集

[ShowUploads](internal/ui/dialog/transfer.go#L34-L123) 是纯 UI 层，负责收集用户输入：

| 字段 | 类型 | 默认值 | 对应 kubectl 参数 |
|------|------|--------|-------------------|
| From | string | `ns/podname:` | `<file-spec-src>` |
| To | string | 空 | `<file-spec-dest>` |
| Download | bool | `true` | 决定 From/To 语义方向 |
| NoPreserve | bool | `false` | `--no-preserve` |
| CO | string | 首个容器名 | `-c=<container>` |
| Retries | int | 999 | `--retries` |

**方向切换逻辑** ([transfer.go:55-65](internal/ui/dialog/transfer.go#L55-L65))：

```go
f.AddCheckbox("Download:", args.Download, func(_ string, flag bool) {
    args.Download = flag
    args.From, args.To = args.To, args.From  // 交换源和目标
    fromField.SetText(args.From)
    toField.SetText(args.To)
})
```

默认 `Download=true`，即从容器复制到本地。切换为上传时，From/To 自动互换。

[TransferArgs](internal/ui/dialog/transfer.go#L19-L23) 结构体：

```go
type TransferArgs struct {
    From, To, CO         string
    Download, NoPreserve bool
    Retries              int
}
```

### 2.4 shellOpts：命令执行的内部结构

[shellOpts](internal/view/exec.go#L45-L51) 是 K9s 执行外部命令的统一选项载体：

```go
type shellOpts struct {
    clear, background bool
    pipes             []string
    binary            string
    banner            string
    args              []string
}
```

CP 场景只用到 `background=true` 和 `args` 两个字段。

---

## 三、runK：参数注入 + 进程调度边界

[runK](internal/view/exec.go#L57-L97) 是 K9s 与外部 kubectl 进程之间的最后一道代码。它完成：

### 3.1 查找 kubectl 二进制

```go
bin, err := exec.LookPath("kubectl")
if errors.Is(err, exec.ErrDot) {
    return fmt.Errorf("kubectl command must not be in the current working directory: %w", err)
}
if err != nil {
    return fmt.Errorf("kubectl command is not in your path: %w", err)
}
```

### 3.2 注入集群认证参数 ([exec.go:65-81](internal/view/exec.go#L65-L81))

```go
args := []string{opts.args[0]}  // "cp"
if u, err := a.Conn().Config().ImpersonateUser(); err == nil {
    args = append(args, "--as", u)
}
if g, err := a.Conn().Config().ImpersonateGroups(); err == nil {
    args = append(args, "--as-group", g)
}
if isInsecure := a.Conn().Config().Flags().Insecure; isInsecure != nil && *isInsecure {
    args = append(args, "--insecure-skip-tls-verify")
}
args = append(args, "--context", a.Config.K9s.ActiveContextName())
if cfg := a.Conn().Config().Flags().KubeConfig; cfg != nil && *cfg != "" {
    args = append(args, "--kubeconfig", *cfg)
}
if len(args) > 0 {
    opts.args = append(args, opts.args[1:]...)  // 合并：认证参数 + 原始参数
}
opts.binary = bin
```

**最终 kubectl 命令行参数顺序**：

```
kubectl cp \
    --as <user> \
    --as-group <group> \
    --insecure-skip-tls-verify \
    --context <active-context> \
    --kubeconfig <path> \
    <from> <to> \
    --no-preserve=<bool> \
    --retries=<n> \
    -c=<container> \
    --retries=<n>   ← 重复
```

### 3.3 run：后台/前台调度

[run](internal/view/exec.go#L99-L122) 根据 `opts.background` 决定执行模式：

```go
if opts.background {
    if err := execute(opts, statusChan); err != nil {
        errChan <- err
    }
    close(errChan)
    return true, errChan, statusChan
}
// 前台模式：挂起 UI → execute → 恢复 UI
```

CP 场景下 `background=true`，因此不会挂起 UI，命令在 goroutine 中异步执行。

### 3.4 execute：构造 exec.Cmd

[execute](internal/view/exec.go#L172-L239) 创建 Go 的 `os/exec.Cmd`：

```go
cmds := make([]*exec.Cmd, 0, 1)
cmd := exec.CommandContext(ctx, opts.binary, opts.args...)
cmds = append(cmds, cmd)
// CP 场景 opts.pipes 为空，所以 cmds 只有一个命令
err := pipe(ctx, opts, statusChan, &o, &e, cmds...)
```

### 3.5 pipe：真正运行命令

[pipe](internal/view/exec.go#L549-L615) 中单命令后台模式代码路径 ([exec.go:556-572](internal/view/exec.go#L556-L572))：

```go
if opts.background {
    go func() {
        cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, w, e
        if err := cmd.Run(); err != nil {
            slog.Error("Command exec failed", slogs.Error, err)
        } else {
            for _, l := range strings.Split(w.String(), "\n") {
                if l != "" {
                    statusChan <- fmt.Sprintf("%s %s", outputPrefix, l)
                }
            }
            statusChan <- fmt.Sprintf("Command completed successfully: %q", ...)
        }
        close(statusChan)
    }()
    return nil
}
```

**进程边界至此完全清晰**：K9s 的最后一行 Go 代码是 `cmd.Run()`，之后进入外部 `kubectl` 二进制进程。K9s 对 kubectl cp 的 tar 管道逻辑完全不可见，只能通过 stdout/stderr 字节流被动接收输出。

---

## 四、kubectl cp (v0.35.1) 内部：tar 流管道机制

K9s 通过进程边界调用 kubectl cp 后，真正的文件传输发生在 kubectl 内部。以下代码逻辑分析基于 `k8s.io/kubectl v0.35.1` 的 `pkg/cmd/cp/cp.go` 源码（仅用于说明 tar 流管道原理，**实际运行的外部 kubectl 二进制版本可能不同**）。

### 4.1 CopyOptions 与命令定义

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
type CopyOptions struct {
    Container  string
    Namespace  string
    NoPreserve bool
    MaxTries   int
    ClientConfig      *restclient.Config
    Clientset         kubernetes.Interface
    ExecParentCmdName string
    args []string
    genericiooptions.IOStreams
}
```

`NewCmdCp` 注册了与 K9s 构造的参数一一对应的 flag：

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
cmd.Flags().StringVarP(&o.Container, "container", "c", o.Container, "...")
cmd.Flags().BoolVarP(&o.NoPreserve, "no-preserve", "", false, "...")
cmd.Flags().IntVarP(&o.MaxTries, "retries", "", 0, "...")
```

参数对照：

| K9s 构造 | kubectl CopyOptions 字段 |
|----------|--------------------------|
| `<from> <to>` | `o.args[0]`, `o.args[1]` |
| `-c=<container>` | `Container` |
| `--no-preserve=<bool>` | `NoPreserve` |
| `--retries=<n>` | `MaxTries` |
| `--context` | 由 `cmdutil.Factory` 解析，不进 CopyOptions |
| `--kubeconfig` | 由 `cmdutil.Factory` 解析 |
| `--as`, `--as-group` | 由 `cmdutil.Factory` 解析 |

### 4.2 路径解析：extractFileSpec

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func extractFileSpec(arg string) (fileSpec, error) {
    i := strings.Index(arg, ":")
    if i == 0 { return fileSpec{}, errFileSpecDoesntMatchFormat }
    if i == -1 {
        return fileSpec{File: newLocalPath(arg)}, nil  // 无冒号 → 本地
    }
    pod, file := arg[:i], arg[i+1:]
    pieces := strings.Split(pod, "/")
    switch len(pieces) {
    case 1:
        return fileSpec{PodName: pieces[0], File: newRemotePath(file)}, nil
    case 2:
        return fileSpec{PodNamespace: pieces[0], PodName: pieces[1], File: newRemotePath(file)}, nil
    default:
        return fileSpec{}, errFileSpecDoesntMatchFormat
    }
}
```

这就是路径格式 `[[namespace/]pod:]file/path` 的实现依据。

### 4.3 Run 方法：方向分派

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func (o *CopyOptions) Run() error {
    srcSpec, err := extractFileSpec(o.args[0])
    destSpec, err := extractFileSpec(o.args[1])
    // 必须一端本地、一端远端
    if len(srcSpec.PodName) != 0 && len(destSpec.PodName) != 0 {
        return fmt.Errorf("one of src or dest must be a local file specification")
    }
    if len(srcSpec.PodName) != 0 {
        return o.copyFromPod(srcSpec, destSpec)  // 下载
    }
    if len(destSpec.PodName) != 0 {
        return o.copyToPod(srcSpec, destSpec, &exec.ExecOptions{})  // 上传
    }
    return fmt.Errorf("one of src or dest must be a remote file specification")
}
```

### 4.4 上传（本地 → 容器）：copyToPod

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func (o *CopyOptions) copyToPod(src, dest fileSpec, options *exec.ExecOptions) error {
    if _, err := os.Stat(src.File.String()); err != nil {
        return fmt.Errorf("%s doesn't exist in local filesystem", src.File)
    }
    reader, writer := io.Pipe()  // 建立管道

    // 如果目标是目录，则自动追加源文件名
    if err := o.checkDestinationIsDir(dest); err == nil {
        destFile = destFile.Join(srcFile.Base())
    }

    // goroutine: 本地 → tar → pipe 写端
    go func(src localPath, dest remotePath, writer io.WriteCloser) {
        defer writer.Close()
        cmdutil.CheckErr(makeTar(src, dest, writer))
    }(srcFile, destFile, writer)

    // 容器端命令
    var cmdArr []string
    if o.NoPreserve {
        cmdArr = []string{"tar", "--no-same-permissions", "--no-same-owner", "-xmf", "-"}
    } else {
        cmdArr = []string{"tar", "-xmf", "-"}
    }
    destFileDir := destFile.Dir().String()
    if len(destFileDir) > 0 {
        cmdArr = append(cmdArr, "-C", destFileDir)
    }

    // exec: 将 pipe 读端挂到 stdin，容器内执行 tar 解压
    options.StreamOptions = exec.StreamOptions{
        IOStreams: genericiooptions.IOStreams{
            In:     reader,
            Out:    o.Out,
            ErrOut: o.ErrOut,
        },
        Stdin: true,
        Namespace: dest.PodNamespace,
        PodName:   dest.PodName,
    }
    options.Command = cmdArr
    options.Executor = &exec.DefaultRemoteExecutor{}
    return o.execute(options)
}
```

**上传管道拓扑**：

```
本地文件
  │
  ▼
recursiveTar()  递归遍历目录，写 tar header + data
  │
  ▼
tar.NewWriter(writer)
  │
  ▼
io.Pipe.Writer ──────► io.Pipe.Reader
                             │
                             ▼
                    exec API stdin (kubernetes SPDY/WS 通道)
                             │
                             ▼
                    容器内 tar -xmf - -C <destDir>
                             │
                             ▼
                    容器文件系统
```

### 4.5 makeTar / recursiveTar：本地归档

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func makeTar(src localPath, dest remotePath, writer io.Writer) error {
    tarWriter := tar.NewWriter(writer)
    defer tarWriter.Close()
    srcPath := src.Clean()
    destPath := dest.Clean()
    return recursiveTar(srcPath.Dir(), srcPath.Base(), destPath.Dir(), destPath.Base(), tarWriter)
}
```

`recursiveTar` 处理三种文件类型：
- 目录：递归遍历子文件，空目录写空 header
- 符号链接：写 `tar.FileInfoHeader` + `hdr.Linkname = target`
- 普通文件：写 header + `io.Copy(tw, f)` 传输文件内容

### 4.6 下载（容器 → 本地）：copyFromPod

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func (o *CopyOptions) copyFromPod(src, dest fileSpec) error {
    reader := newTarPipe(src, o)  // TarPipe 是支持断点续传的 io.Reader
    srcFile := src.File.(remotePath)
    destFile := dest.File.(localPath)
    prefix := stripPathShortcuts(srcFile.StripSlashes().Clean().String())
    return o.untarAll(src.PodNamespace, src.PodName, prefix, srcFile, destFile, reader)
}
```

### 4.7 TarPipe：断点续传读取器

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
type TarPipe struct {
    src       fileSpec
    o         *CopyOptions
    reader    *io.PipeReader
    outStream *io.PipeWriter
    bytesRead uint64
    retries   int
}

func newTarPipe(src fileSpec, o *CopyOptions) *TarPipe {
    t := new(TarPipe)
    t.src = src
    t.o = o
    t.initReadFrom(0)
    return t
}

func (t *TarPipe) initReadFrom(n uint64) {
    t.reader, t.outStream = io.Pipe()
    options := &exec.ExecOptions{
        StreamOptions: exec.StreamOptions{
            IOStreams: genericiooptions.IOStreams{
                In:     nil,
                Out:    t.outStream,
                ErrOut: t.o.Out,
            },
            Namespace: t.src.PodNamespace,
            PodName:   t.src.PodName,
        },
        Command:  []string{"tar", "cf", "-", t.src.File.String()},
        Executor: &exec.DefaultRemoteExecutor{},
    }
    if t.o.MaxTries != 0 {
        // 有重试时使用 tail -c+N 跳过已传字节
        options.Command = []string{"sh", "-c", fmt.Sprintf("tar cf - %s | tail -c+%d", t.src.File, n)}
    }
    go func() {
        defer t.outStream.Close()
        cmdutil.CheckErr(t.o.execute(options))
    }()
}

// io.Reader 接口实现：读失败时自动重试
func (t *TarPipe) Read(p []byte) (n int, err error) {
    n, err = t.reader.Read(p)
    if err != nil {
        if t.o.MaxTries < 0 || t.retries < t.o.MaxTries {
            t.retries++
            fmt.Printf("Resuming copy at %d bytes, retry %d/%d\n", t.bytesRead, t.retries, t.o.MaxTries)
            t.initReadFrom(t.bytesRead + 1)  // 从断点重建流
            err = nil
        } else {
            fmt.Printf("Dropping out copy after %d retries\n", t.retries)
        }
    } else {
        t.bytesRead += uint64(n)
    }
    return
}
```

**下载管道拓扑**：

```
容器文件系统
  │
  ▼
容器内 tar cf - <path>
  │  (如果 MaxTries≠0，则加 | tail -c+<bytesRead+1>)
  ▼
exec API stdout (kubernetes SPDY/WS 通道)
  │
  ▼
io.Pipe.Writer ──────► io.Pipe.Reader (TarPipe.reader)
                             │
                             ▼
                    TarPipe.Read()  (记录 bytesRead，失败时重试)
                             │
                             ▼
                    untarAll() → tar.NewReader
                             │
                             ▼
                    本地文件系统
```

### 4.8 untarAll：本地解压 + 安全校验

```go
// k8s.io/kubectl v0.35.1 pkg/cmd/cp/cp.go
func (o *CopyOptions) untarAll(ns, pod string, prefix string, src remotePath, dest localPath, reader io.Reader) error {
    tarReader := tar.NewReader(reader)
    for {
        header, err := tarReader.Next()
        if err != nil {
            if err != io.EOF { return err }
            break
        }
        // 安全校验 1：所有 tar 条目必须以 prefix 开头（防路径穿越）
        if !strings.HasPrefix(header.Name, prefix) {
            return fmt.Errorf("tar contents corrupted")
        }
        mode := header.FileInfo().Mode()
        destFileName := dest.Join(newRemotePath(header.Name[len(prefix):]))
        // 安全校验 2：目标路径必须在 dest 目录内（相对路径）
        if !isRelative(dest, destFileName) {
            fmt.Fprintf(o.IOStreams.ErrOut, "warning: file %q is outside target destination, skipping\n", destFileName)
            continue
        }
        // 创建父目录
        if err := os.MkdirAll(destFileName.Dir().String(), 0755); err != nil { return err }
        if header.FileInfo().IsDir() {
            os.MkdirAll(destFileName.String(), 0755)
            continue
        }
        if mode&os.ModeSymlink != 0 {
            // 符号链接：输出警告（默认跳过，提示用户用 tar 管道）
        }
        // 普通文件：写入本地
    }
}
```

两层安全校验防止恶意 tar 包将文件写到目标目录之外。

---

## 五、进度展示机制

### 5.1 K9s 层：三态通知

K9s 没有实现传输进度条，只有"发起 → 等待 → 结束"三态：

- **发起**：无特殊通知，对话框关闭即表示已提交
- **等待**：用户完全看不到进度，UI 上无任何反馈
- **结束**：成功走 [pod.go:329](internal/view/pod.go#L329-L329) `Flash().Infof(...)`，失败走 [pod.go:327](internal/view/pod.go#L327-L327) `cowCmd(err.Error())`

```go
if err := runK(p.App(), &cliOpts); err != nil {
    p.App().cowCmd(err.Error())      // 失败：cow 弹窗显示错误
} else {
    p.App().Flash().Infof("%s successful on %s!", op, fqn)  // 成功：顶部 Flash
}
```

后台模式下，[pipe](internal/view/exec.go#L556-L572) 会通过 `statusChan` 发送 stdout 行，但 K9s 的 [runK](internal/view/exec.go#L88-L90) 只做了 debug 日志，没有推给用户 UI：

```go
for v := range stChan {
    slog.Debug("stdout", slogs.Line, v)  // 仅 debug 日志，用户不可见
}
```

### 5.2 kubectl cp 层：续传时输出字节数

kubectl cp 的 TarPipe 只在断点续传时打印进度信息到 stdout：

```go
// TarPipe.Read() 中
fmt.Printf("Resuming copy at %d bytes, retry %d/%d\n", t.bytesRead, t.retries, t.o.MaxTries)
```

正常无中断传输时，**无任何输出**。这就是为什么 K9s 侧看不到进度——上游本身就不产生进度事件。

### 5.3 为什么没有实时进度条

根本原因在 tar 流协议和 exec 通道的限制：

1. **tar 格式不带总大小**：tar 是顺序流式归档，每个 header 只带单文件大小，全局没有"归档总字节数"字段
2. **exec 通道是裸字节流**：`kubectl exec` 的 stdout 只传原始字节，不携带元数据
3. **容器端不报告进度**：容器内的 `tar cf -` 只输出流，不输出百分比

即使 K9s 想实现进度条，在不修改 kubectl 源码的前提下也无法获知"总大小"和"已传百分比"。

---

## 六、完整调用链路（带文件行号）

```
用户按键 T
  │
  ├─ [pod.go:109-115] KeyT 绑定 transferCmd (Dangerous=true)
  │
  ├─ [pod.go:286-353] transferCmd()
  │     ├─ fetchPod() 取 Pod 对象
  │     ├─ [transfer.go:34-123] dialog.ShowUploads() 弹出对话框
  │     │     └─ 用户填完后点 OK
  │     │
  │     └─ [pod.go:293-332] ack(TransferArgs)
  │           ├─ os.Stat() 校验本地文件（上传时）
  │           ├─ 构造 opts = ["cp", from, to, --no-preserve, --retries, -c=..., --retries]
  │           └─ shellOpts{background: true, args: opts}
  │
  ├─ [exec.go:57-97] runK()
  │     ├─ exec.LookPath("kubectl")
  │     ├─ 注入 --as, --as-group, --insecure-skip-tls-verify, --context, --kubeconfig
  │     └─ opts.binary = kubectl 路径
  │
  ├─ [exec.go:99-122] run()
  │     └─ background=true → 直接 execute()，不挂起 UI
  │
  ├─ [exec.go:172-239] execute()
  │     ├─ context.WithCancel + 信号处理
  │     ├─ exec.CommandContext(ctx, kubectl, args...)
  │     └─ 调用 pipe()
  │
  ├─ [exec.go:549-615] pipe() 单命令后台模式
  │     └─ goroutine { cmd.Run() → statusChan }
  │
  │  ═══════════════════════════════ 进程边界 ═══════════════════════════════
  │
  └─▶ 外部 kubectl cp 进程（用户 PATH 中的任意版本）
        │  （以下代码逻辑基于 k8s.io/kubectl v0.35.1 版本分析）
        ├─ extractFileSpec() 解析 [[ns/]pod:]path
        ├─ Run() 分派方向
        │
        ├─ 上传：copyToPod()
        │     ├─ io.Pipe() 建立管道
        │     ├─ goroutine: makeTar() → recursiveTar() → pipe.Writer
        │     └─ exec API stdin → 容器 tar -xmf - -C <dir>
        │
        ├─ 下载：copyFromPod()
        │     ├─ TarPipe (io.Reader) 包装 exec stdout
        │     │   └─ Read() 失败时 initReadFrom(bytesRead+1) 断点续传
        │     └─ untarAll() → 安全校验 → 写本地文件
        │
        └─ 进程退出 → K9s goroutine 捕获 stdout/stderr → statusChan
              └─ K9s 仅 debug 日志，用户侧只看到成功/失败 Flash
```

---

## 七、关键代码索引

### K9s 本地代码

| 文件 | 行号 | 功能 |
|------|------|------|
| [go.mod](go.mod#L41-L41) | L41 | `k8s.io/kubectl v0.35.1` 依赖声明 |
| [internal/view/exec.go](internal/view/exec.go#L58-L58) | L58 | `runK()` 中 `exec.LookPath("kubectl")` 查找外部二进制 |
| [internal/view/exec.go](internal/view/exec.go#L242-L242) | L242 | `runKu()` 中 `exec.LookPath("kubectl")` 查找外部二进制 |
| [internal/render/sc.go](internal/render/sc.go#L15-L15) | L15 | import `k8s.io/kubectl/pkg/util/storage`（依赖声明用途） |
| [internal/dao/node.go](internal/dao/node.go#L22-L23) | L22-L23 | import `k8s.io/kubectl/pkg/drain` + `scheme`（依赖声明用途） |
| [internal/dao/port_forwarder.go](internal/dao/port_forwarder.go#L23-L24) | L23-L24 | PortForward 走 client-go（import 而非外部 kubectl） |
| [internal/dao/port_forwarder.go](internal/dao/port_forwarder.go#L121-L171) | L121-L171 | PortForwarder.Start：直连 SPDY，不调外部进程 |
| [internal/view/app.go](internal/view/app.go#L679-L696) | L679-L696 | dirCmd：纯本地 os.Stat()，不涉及 kubectl |
| [internal/view/pod.go](internal/view/pod.go#L403-L417) | L403-L417 | shellIn：构造 kubectl exec 参数 → runK |
| [internal/view/pod.go](internal/view/pod.go#L453-L459) | L453-L459 | attachIn：构造 kubectl attach 参数 → runK |
| [internal/view/pod.go](internal/view/pod.go#L461-L468) | L461-L468 | computeShellArgs：`exec -it ... sh -c` |
| [internal/view/pod.go](internal/view/pod.go#L478-L499) | L478-L499 | buildShellArgs：exec/attach 参数构造通用逻辑 |
| [pod.go](internal/view/pod.go#L41-L42) | L41 | `defaultTxRetries = 999` |
| [pod.go](internal/view/pod.go#L109-L115) | L109-L115 | T 键绑定 transferCmd |
| [pod.go](internal/view/pod.go#L286-L353) | L286-L353 | transferCmd 主函数 |
| [pod.go](internal/view/pod.go#L293-L332) | L293-L332 | ack 回调：参数构造 + runK 调用 |
| [transfer.go](internal/ui/dialog/transfer.go#L19-L23) | L19-L23 | TransferArgs 结构 |
| [transfer.go](internal/ui/dialog/transfer.go#L34-L123) | L34-L123 | ShowUploads 对话框 |
| [exec.go](internal/view/exec.go#L45-L51) | L45-L51 | shellOpts 结构 |
| [exec.go](internal/view/exec.go#L57-L97) | L57-L97 | runK：查找 kubectl + 注入认证 |
| [exec.go](internal/view/exec.go#L99-L122) | L99-L122 | run：后台/前台调度 |
| [exec.go](internal/view/exec.go#L172-L239) | L172-L239 | execute：构造 exec.Cmd |
| [exec.go](internal/view/exec.go#L549-L615) | L549-L615 | pipe：执行命令 + 管道 |

### kubectl cp (v0.35.1)

| 结构/函数 | 功能 |
|-----------|------|
| `CopyOptions` | cp 命令参数载体 |
| `NewCmdCp()` | 注册 `-c`、`--no-preserve`、`--retries` flag |
| `extractFileSpec()` | 解析 `[[ns/]pod:]file/path` 路径格式 |
| `Run()` | 分派 copyToPod / copyFromPod |
| `checkDestinationIsDir()` | exec 调用 `test -d` 判断目标是否为目录 |
| `copyToPod()` | 上传：本地 tar → io.Pipe → exec stdin → 容器 tar -x |
| `copyFromPod()` | 下载：容器 tar cf → exec stdout → TarPipe → 本地 untar |
| `TarPipe` | 带断点续传的 io.Reader，记录 bytesRead、retries |
| `TarPipe.initReadFrom()` | 建立 exec stdout → io.Pipe 通道，MaxTries≠0 时用 `tail -c+N` |
| `TarPipe.Read()` | 读失败时自动重建流实现续传 |
| `makeTar()` / `recursiveTar()` | 本地递归归档为 tar 流 |
| `untarAll()` | 本地解压 tar 流，含 prefix 校验和相对路径校验 |

---

## 八、传输路径难以理解的原因总结

1. **两个 kubectl 版本混淆**：go.mod 声明的 `k8s.io/kubectl v0.35.1` 是编译期依赖（用于 drain、describe、render 等），而 CP 实际调用的是用户 PATH 中的外部 kubectl 二进制，二者版本可能不同且无绑定关系
2. **命令分类边界混淆**：cp/shell/attach 走外部 kubectl 二进制（进程边界），但 port-forward 走 client-go SPDY 直连（库调用边界），dir 更是纯本地实现，三类边界混在一起容易误判
3. **进程边界不透明**：K9s 与 kubectl cp 之间是进程边界，用户按下 T 键后 K9s 代码就走完了，后面的 tar 管道全在外部进程里，K9s UI 层无任何中间状态可见
4. **冒号分隔歧义**：`ns/pod:/path` 格式中，`:` 是 Pod 与路径的分隔符，与 Windows 盘符 `C:\path` 形态相似，容易混淆
5. **From/To 语义方向依赖**：同一个 From 字段，下载时是远端路径，上传时是本地路径，由 Download bool 反转决定
6. **tar 是隐式前置条件**：容器必须有 `tar` 二进制，这个前提既不在对话框提示，也不在错误信息中明确
7. **参数重复 bug**：`--retries` 被追加两次（[pod.go:309](internal/view/pod.go#L309-L309) 和 [pod.go:314](internal/view/pod.go#L314-L314)），虽然 kubectl 通常取最后值，但增加了理解成本
8. **无中间进度反馈**：传输过程是 UI 黑盒，只有成功/失败两端状态，长时间传输时用户无法判断是否卡死
