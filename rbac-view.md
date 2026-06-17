# RBAC 视图权限预检与资源聚合深度分析

## 一、整体架构

RBAC 视图系统由多个模块协同工作，分为视图层、DAO 数据访问层、渲染层和权限预检层。

### 1.1 模块分层架构

```
┌──────────────────────────────────────────────────────────┐
│                     视图层 (view)                      │
│  Rbac / Policy / Subject / User / Group / SA       │
└─────────────────────┬────────────────────────────────┘
                    │
┌─────────────────────▼────────────────────────────────┐
│                  DAO 层 (dao)                     │
│  Rbac / Policy / Subject                            │
│  (loadClusterRoleBinding / loadRoleBinding / parseRules │
└─────────────────────┬────────────────────────────────┘
                    │
┌─────────────────────▼────────────────────────────────┐
│                 渲染层 (render)                  │
│  PolicyRes / Policies / RbacVerbHeader / asVerbs        │
└─────────────────────┬────────────────────────────────┘
                    │
┌─────────────────────▼────────────────────────────────┐
│              权限预检层 (client + factory)          │
│  APIClient.CanI / Factory.CanForResource      │
│  SelfSubjectAccessReview + LRU 缓存                │
└───────────────────────────────────────────────────┘
```

### 1.2 核心文件

| 文件 | 职责 |
|------|------|
| [rbac.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/rbac.go) | RBAC 规则查看器视图，展示单个角色/绑定的权限规则 |
| [policy.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/policy.go) | 策略视图，按主体（User/Group/SA）聚合展示所有权限 |
| [rbac.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go) | RBAC DAO，加载角色/绑定并解析规则 |
| [rbac_policy.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go) | Policy DAO，按主体聚合所有角色的权限 |
| [rbac_subject.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_subject.go) | Subject DAO，列出所有主体 |
| [policy.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go) | 权限渲染与 PolicyRes/Policies 数据结构 |
| [rbac.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/rbac.go) | RBAC 渲染与动词图标展示 |
| [client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/client.go) | APIClient.CanI 权限预检 |
| [factory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go) | Factory.CanForResource 权限预检 |
| [helpers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/helpers.go) | FQN 等辅助函数 |

---

## 二、角色合并与权限聚合

### 2.1 核心数据结构

#### PolicyRes - 单条权限规则

位于 [policy.go#L95-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L95-L113)

```go
type PolicyRes struct {
    Namespace, Binding string   // 命名空间、来源绑定(如 CR:admin / RO:reader)
    Resource, Group    string   // 资源名、API Group
    ResourceName       string   // ⚠️ 字段存在但 parseRules 中从未使用
    NonResourceURL     string   // ⚠️ 字段存在但 parseRules 中从未使用
    Verbs              []string // 权限动词列表
}
```

**⚠️ 边界发现**：`ResourceName` 和 `NonResourceURL` 字段虽然在结构体中定义，但 `NewPolicyRes` 构造函数从未设置这两个字段。资源名信息被编码进了 `Resource` 字段本身（通过 FQN 拼接），这两个字段始终为空。

- **GR()** - Group/Resource 组合，用于唯一标识一种资源：`p.Group + "/" + p.Resource`

#### Policies - 权限规则集合

位于 [policy.go#L155-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L155-L183)

```go
type Policies []*PolicyRes
```

### 2.2 带资源名(ResourceNames)的规则展开

位于 [rbac.go#L151-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L151-L174)

```go
func parseRules(ns, binding string, rules []rbacv1.PolicyRule) render.Policies {
    pp := make(render.Policies, 0, len(rules))
    for _, rule := range rules {
        for _, grp := range rule.APIGroups {
            if grp == "" {
                grp = "core"
            }
            for _, res := range rule.Resources {
                // 第一层循环：每个 ResourceName 生成一条独立规则
                for _, na := range rule.ResourceNames {
                    pp = pp.Upsert(render.NewPolicyRes(ns, binding, FQN(res, na), grp, rule.Verbs))
                }
                // 第二层：无论有无 ResourceNames，额外生成一条通配规则
                pp = pp.Upsert(render.NewPolicyRes(ns, binding, FQN(grp, res), grp, rule.Verbs))
            }
        }
        // ... NonResourceURLs
    }
    return pp
}
```

#### FQN 函数的作用

位于 [helpers.go#L59-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/helpers.go#L59-L65)

```go
func FQN(ns, n string) string {
    if ns == "" {
        return n
    }
    return ns + "/" + n
}
```

#### 展开示例详解

假设有如下 PolicyRule：

```yaml
apiGroups: [""]
resources: ["pods"]
resourceNames: ["my-pod-1", "my-pod-2"]
verbs: ["get", "list"]
```

**展开过程：**

| 循环 | 调用 | Resource 字段值 | GR() |
|------|------|-----------------|------|
| ResourceNames 循环 na="my-pod-1" | `FQN("pods", "my-pod-1")` | `"pods/my-pod-1"` | `"core/pods/my-pod-1"` |
| ResourceNames 循环 na="my-pod-2" | `FQN("pods", "my-pod-2")` | `"pods/my-pod-2"` | `"core/pods/my-pod-2"` |
| 通配规则 | `FQN("core", "pods")` | `"core/pods"` | `"core/core/pods"` |

**关键边界：**

1. **GR 完全不同**：带资源名的规则 GR 是 `"core/pods/my-pod-1"`，通配规则 GR 是 `"core/core/pods"`，两者**不会被 Upsert 合并**，各自成为独立行。

2. **渲染层的 cleanseResource**：位于 [policy.go#L82-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L82-L93)

   ```go
   func cleanseResource(r string) string {
       tt := strings.Split(r, "/")
       switch len(tt) {
       case 2, 3:
           return strings.TrimPrefix(r, tt[0]+"/")
       default:
           return r
       }
   }
   ```

   - `"pods/my-pod-1"` 分割为 2 段 → TrimPrefix 后显示 `"my-pod-1"`
   - `"core/pods"` 分割为 2 段 → TrimPrefix 后显示 `"pods"`
   - 这样在界面上看起来分别是资源名和资源类型，但本质是两条独立的 PolicyRes

3. **资源名规则与通配规则的重复问题**：同一条 PolicyRule 同时展开为多条规则，Verb 完全相同，用户看到的是一条"my-pod-1"和一条"pods"都有 get/list 权限，但语义不同——前者仅限特定 pod，后者是所有 pod。

4. **ResourceName 字段的冗余**：`PolicyRes.ResourceName` 字段始终为空字符串，资源名被编码进 Resource 字段路径中。这意味着无法通过结构化字段区分"通配规则"和"资源名规则"，只能靠字符串模式推断。

### 2.3 权限合并策略 - Upsert + Merge

#### Upsert 方法

位于 [policy.go#L158-L172](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L158-L172)

```go
func (pp Policies) Upsert(p *PolicyRes) Policies {
    idx, ok := pp.find(p.GR())
    if !ok {
        return append(pp, p)
    }
    p, err := pp[idx].Merge(p)  // 注意：返回值 p 是合并后的 pp[idx]
    if err != nil {
        slog.Error("Policy upsert failed", slogs.Error, err)
        return pp
    }
    pp[idx] = p
    return pp
}
```

#### Merge 方法 - 信息丢失的重灾区

位于 [policy.go#L120-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L120-L133)

```go
func (p *PolicyRes) Merge(p1 *PolicyRes) (*PolicyRes, error) {
    if p.GR() != p1.GR() {
        return nil, fmt.Errorf("policy mismatch %s vs %s", p.GR(), p1.GR())
    }
    // ⚠️ 仅合并 Verbs，其他字段一律不处理
    for _, v := range p1.Verbs {
        if !p.hasVerb(v) {
            p.Verbs = append(p.Verbs, v)
        }
    }
    return p, nil
}
```

#### 合并后 Binding/来源信息的保留与丢失

**合并时的字段处理情况：**

| 字段 | 是否合并 | 行为 |
|------|----------|------|
| Verbs | ✅ 合并去重 | 将 p1 中不在 p 中的动词追加 |
| Namespace | ❌ 不合并 | 保留先插入的 p 的值 |
| Binding | ❌ 不合并 | 保留先插入的 p 的值 |
| Resource | ❌ 不合并 | GR 匹配隐含 Resource 相同 |
| Group | ❌ 不合并 | GR 匹配隐含 Group 相同 |
| ResourceName | ❌ 不合并 | 始终为空 |
| NonResourceURL | ❌ 不合并 | 始终为空 |

**信息丢失示例场景：**

场景：主体 User/alice 通过两个 RoleBinding 分别获得不同角色对同一资源的权限

```
RoleBinding/rb-1 → Role/reader (ns: default) → pods [get, list]
RoleBinding/rb-2 → Role/writer (ns: default) → pods [create, patch]
```

在 Policy 视图中的聚合过程：

```
1. 先加载 parseRules("default", "RO:reader", ...)
   → 生成 PolicyRes{Namespace:"default", Binding:"RO:reader", Resource:"core/pods", Verbs:[get,list]}
   GR = "core/core/pods"

2. 再加载 parseRules("default", "RO:writer", ...)
   → 生成 PolicyRes{Namespace:"default", Binding:"RO:writer", Resource:"core/pods", Verbs:[create,patch]}
   GR = "core/core/pods" ← 相同！

3. Upsert 触发 Merge
   → pp[idx].Merge(p1)
   → Verbs 合并为 [get, list, create, patch] ✅
   → Binding 仍然是 "RO:reader" ❌ ("RO:writer" 信息完全丢失)
   → Namespace 仍然是 "default" (刚好相同)
```

**最终界面显示：**

| NAMESPACE | NAME | API-GROUP | BINDING | GET | LIST | CREATE | PATCH | ... |
|-----------|------|-----------|---------|-----|------|--------|-------|-----|
| default   | pods | core      | RO:reader | ✓ | ✓   | ✓     | ✓    | ... |

用户只能看到权限来自 `RO:reader`，完全不知道 `RO:writer` 也授予了该资源权限。**这是严重的信息丢失，会干扰权限溯源调试。**

#### 跨命名空间同名资源的合并

更极端的情况：

```
RoleBinding/rb-1 (ns: ns-a) → Role/admin → pods [get]
RoleBinding/rb-2 (ns: ns-b) → Role/admin → pods [delete]
```

GR 匹配后的结果：
- Verbs 合并为 [get, delete] ✅
- Namespace 字段保留先插入的（如 "ns-a"） ❌（ns-b 的命名空间信息丢失）
- Binding 可能是 "RO:admin"（两者相同）

最终显示的 NAMESPACE 列仅显示一个命名空间，用户无法知道另一个命名空间也有权限。

### 2.4 同名角色绑定聚合的影响

#### fetchRoleBindingNamespaces 的 Map 覆盖问题

位于 [rbac_policy.go#L167-L184](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L167-L184)

```go
func (p *Policy) fetchRoleBindingNamespaces(kind, name string) (map[string]string, error) {
    // ...
    ss := make(map[string]string, len(rbs))
    for i := range rbs {
        for _, s := range rbs[i].Subjects {
            if isSameSubject(kind, ns, rbs[i].Namespace, n, &s) {
                // ⚠️ Key 是 "Kind:Name"，Value 是单个命名空间
                ss[rbs[i].RoleRef.Kind+":"+rbs[i].RoleRef.Name] = rbs[i].Namespace
            }
        }
    }
    return ss, nil
}
```

**Map 结构：** `map[RoleRef.Kind:RoleRef.Name]Binding.Namespace`

**边界问题：Value 是单个 string，不是 []string！**

#### 场景一：不同命名空间中存在同名 Role

```
ns-a/Role/admin → pods [get, list]
ns-b/Role/admin → pods [delete]

ns-a/RoleBinding/rb-a → Role/admin → Subject: User/alice
ns-b/RoleBinding/rb-b → Role/admin → Subject: User/alice
```

遍历处理：

| 循环 | rb[i] | RoleRef | Map Key | Map Value |
|------|-------|---------|---------|-----------|
| i=0 | ns-a/rb-a | Role:admin | "Role:admin" | "ns-a" |
| i=1 | ns-b/rb-b | Role:admin | "Role:admin" | **"ns-b" (覆盖!)** |

`ss["Role:admin"]` 的最终值是 `"ns-b"`，`ns-a` 的信息完全丢失。

后续 loadRoleBinding 处理：

```go
for i := range ros {
    if _, ok := rbsMap["Role:"+ros[i].Name]; !ok {
        continue  // 只匹配 "Role:admin" 一次
    }
    // parseRules(ros[i].Namespace, "RO:"+ros[i].Name, ros[i].Rules)
    // ros[i].Namespace 可能是 ns-a 也可能是 ns-b，取决于遍历顺序
    rows = append(rows, parseRules(ros[i].Namespace, "RO:"+ros[i].Name, ros[i].Rules)...)
}
```

**结果：** 只会加载其中一个命名空间的 Role 规则，另一个命名空间的权限完全不显示。

#### 场景二：多个 RoleBinding 引用同一个 ClusterRole

```
ClusterRole/cluster-admin → pods [*]

ns-a/RoleBinding/crb-a → ClusterRole:cluster-admin → Subject: User/alice
ns-b/RoleBinding/crb-b → ClusterRole:cluster-admin → Subject: User/alice
```

遍历处理：

| 循环 | rb[i] | RoleRef | Map Key | Map Value |
|------|-------|---------|---------|-----------|
| i=0 | ns-a/crb-a | ClusterRole:cluster-admin | "ClusterRole:cluster-admin" | "ns-a" |
| i=1 | ns-b/crb-b | ClusterRole:cluster-admin | "ClusterRole:cluster-admin" | **"ns-b" (覆盖!)** |

后续 loadRoleBinding 处理：

```go
for i := range crs {
    if rbNs, ok := rbsMap["ClusterRole:"+crs[i].Name]; ok {
        // rbNs = "ns-b"，ns-a 已丢失
        rows = append(rows, parseRules(rbNs, "CR:"+crs[i].Name, crs[i].Rules)...)
    }
}
```

**结果：** ClusterRole 规则只被加载一次，NAMESPACE 列显示为 `"ns-b"`。虽然 Verbs 是完整的（因为同一个 ClusterRole 规则相同），但 NAMESPACE 信息不准确——ClusterRole 的规则实际上是集群范围的，此处显示为 `"ns-b"` 具有误导性。

#### 场景三：loadClusterRoleBinding 中的重复 ClusterRole

位于 [rbac_policy.go#L62-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L62-L91)

```go
func (p *Policy) loadClusterRoleBinding(kind, name string) (render.Policies, error) {
    // ...
    var nn []string  // 普通 slice，允许重复
    for i := range crbs {
        for _, s := range crbs[i].Subjects {
            if isSameSubject(kind, ns, crbs[i].Namespace, n, &s) {
                nn = append(nn, crbs[i].RoleRef.Name)  // 不做去重
            }
        }
    }
    // ...
    for i := range crs {
        if !inList(nn, crs[i].Name) {  // ⚠️ inList 检查但 parseRules 可能被重复调用吗？
            continue
        }
        // 每个 ClusterRole 只会被遍历一次，因为是遍历 crs (所有 ClusterRole)
        rows = append(rows, parseRules(client.NotNamespaced, "CR:"+crs[i].Name, crs[i].Rules)...)
    }
    return rows, nil
}
```

这里 `nn` 允许重复但外层是 `range crs`，所以每个 ClusterRole 只会被处理一次。问题不大，但 `nn` 中存在重复值是无意义的内存浪费。

### 2.5 主体权限聚合完整流程

位于 [rbac_policy.go#L32-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L32-L60)

```
主体 (User/Group/SA)
├── loadClusterRoleBinding → 找 ClusterRoleBinding
│   ├── 遍历所有 ClusterRoleBinding
│   │   └── isSameSubject 匹配
│   │       └── 收集 RoleRef.Name 到 nn (允许重复)
│   └── 遍历所有 ClusterRole
│       └── inList(nn, cr.Name) 匹配 → parseRules("*", "CR:name", rules)
│
└── loadRoleBinding → 找 RoleBinding
    ├── fetchRoleBindingNamespaces
    │   ├── 遍历所有 RoleBinding
    │   │   └── isSameSubject 匹配
    │   │       └── ss["Kind:Name"] = Namespace  (⚠️ 后写覆盖先写)
    │   └── 返回 map[string]string
    ├── 遍历所有 ClusterRole
    │   └── ss["ClusterRole:"+name] 存在 → parseRules(rbNs, "CR:name", rules)
    └── 遍历所有 Role
        └── ss["Role:"+name] 存在 → parseRules(roleNs, "RO:name", rules)
```

**isSameSubject 精确匹配逻辑**：

位于 [rbac_policy.go#L189-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L189-L202)

```go
func isSameSubject(kind, ns, bns, name string, subject *rbacv1.Subject) bool {
    if subject.Kind != kind || subject.Name != name {
        return false
    }
    if kind == rbacv1.ServiceAccountKind {
        cns := subject.Namespace
        if cns == "" {
            cns = bns  // 当 Subject 未指定 Namespace 时使用 Binding 的命名空间
        }
        return client.IsAllNamespaces(ns) || cns == ns
    }
    return true
}
```

边界处理：ServiceAccount 未显式指定 Namespace 时，回退到 RoleBinding 所在命名空间。

### 2.6 角色/绑定查看 (Rbac DAO)

位于 [rbac.go#L30-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L30-L52)

根据上下文的 GVR 类型加载不同的资源：

| GVR 类型 | 加载方法 | 处理方式 |
|-----------|----------|----------|
| clusterrolebindings | loadClusterRoleBinding | 获取 Binding → 加载 RoleRef 指向的 ClusterRole → parseRules |
| rolebindings | loadRoleBinding | 检查 RoleRef.Kind → ClusterRole 加载 ClusterRole；Role 加载 Role → parseRules |
| clusterroles | loadClusterRole | 直接加载 ClusterRole → parseRules |
| roles | loadRole | 直接加载 Role → parseRules |

**⚠️ loadRoleBinding 中的 parseRules 始终传入 client.ClusterScope：**

位于 [rbac.go#L88-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L88-L112)

```go
// RoleBinding 引用 ClusterRole
return asRuntimeObjects(parseRules(client.ClusterScope, "-", cr.Rules)), nil
// RoleBinding 引用 Role
return asRuntimeObjects(parseRules(client.ClusterScope, "-", role.Rules)), nil
```

这意味着在 RBAC 单角色视图中，Namespace 字段恒为 `"-"`（集群范围），不会显示 Role 实际所在的命名空间。

---

## 三、权限预检机制

### 3.1 权限预检层次

```
┌─────────────────────────────────────────┐
│  视图层 (Browser)                   │
│  Init / 操作前检查                      │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  Factory 层 (watch/factory.go)          │
│  CanForResource / CanForInstance      │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  Client 层 (client/client.go)         │
│  APIClient.CanI                    │
│  SelfSubjectAccessReview + 缓存       │
└─────────────────────────────────────────┘
```

### 3.2 APIClient.CanI - 核心权限检查及异常路径

位于 [client.go#L154-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/client.go#L154-L216)

#### 完整流程图（含异常分支）

```
CanI(ns, gvr, name, verbs)
│
├── ① 连接状态检查
│   └── !getConnOK() → return false, errors.New("ACCESS -- No API server connection")
│
├── ② 特殊资源预处理
│   ├── gvr == NsGVR → ns = name（命名空间资源，资源名即为命名空间）
│   ├── IsClusterWide(ns) → ns = BlankNamespace
│   └── gvr == HmGVR → gvr = SecGVR（Helm 用 Secret 存储 Release）
│
├── ③ 缓存命中检查
│   ├── key = makeCacheKey(ns, gvr, name, verbs)
│   │   = "ns:gvr.String():name::verb1,verb2,..."
│   └── cache.Get(key) 命中且类型为 bool → 直接返回缓存值
│       (⚠️ 类型断言失败会静默 fallback 到 API 调用)
│
├── ④ Dial 获取 Client
│   └── 失败 → return false, err (不写入缓存)
│
├── ⑤ 逐个动词检查
│   for v in verbs:
│   │
│   ├── sar.Spec.ResourceAttributes.Verb = v
│   ├── client.Create(ctx, sar, metav1.CreateOptions{})
│   │   │
│   │   ├── 请求错误(网络/超时)
│   │   │   ├── cache.Add(key, false, 5min)
│   │   │   └── return auth, err  (auth 为 false)
│   │   │
│   │   ├── resp.Status.Allowed == false
│   │   │   ├── cache.Add(key, false, 5min)
│   │   │   └── return auth, fmt.Errorf("(%s) access denied for user on resource %q:%s in namespace %q", ...)
│   │   │
│   │   └── resp.Status.Allowed == true → 继续下一个动词
│   │
│   └── 全部通过 → auth = true
│
└── ⑥ 全部通过
    ├── cache.Add(key, true, 5min)
    └── return true, nil
```

#### 缓存 Key 构造

位于 [client.go#L110-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/client.go#L110-L112)

```go
func makeCacheKey(ns string, gvr *GVR, n string, vv []string) string {
    return ns + ":" + gvr.String() + ":" + n + "::" + strings.Join(vv, ",")
}
```

**缓存命中的边界情况：**

1. **动词顺序敏感**：`["get","list"]` 和 `["list","get"]` 生成不同的 key，不会命中同一缓存条目。

2. **不同 verb 组合分别缓存**：对同一资源先检查 `["list"]`，再检查 `["list","watch"]`，会产生两次 API 调用，分别缓存。

3. **缓存容量 100 条**：超过后 LRU 淘汰旧条目。频繁切换命名空间/资源可能导致缓存抖动。

4. **5 分钟固定过期**：在此期间权限变更（如管理员更新 RoleBinding）不会被感知，用户看到的是过期结果。

5. **连接断开不清缓存**：`setConnOK(false)` 时不会主动清除缓存，只有 `reset()` 时才重建缓存。连接断开再恢复后，旧缓存仍然有效。

#### 拒绝响应的缓存写入

```go
if !resp.Status.Allowed {
    a.cache.Add(key, false, cacheExpiry)  // ⚠️ 拒绝结果也被缓存 5 分钟
    return auth, fmt.Errorf("(%s) access denied for user on resource %q:%s in namespace %q", v, name, gvr, ns)
}
```

**边界问题**：管理员在被拒绝后立刻授予权限，用户在 5 分钟内仍然看到权限不足。

### 3.3 Factory.CanForResource - Informer 级预检

位于 [factory.go#L202-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go#L202-L222)

```go
func (f *Factory) CanForResource(ns string, gvr *GVR, verbs []string) (informers.GenericInformer, error) {
    var resName string
    if gvr == client.NsGVR {
        resName = ns  // NsGVR 特殊处理
    }
    auth, err := f.Client().CanI(ns, gvr, resName, verbs)
    if err != nil {
        return nil, err
    }
    if !auth {
        // ⚠️ auth=false 但 err==nil 的场景来自缓存命中（之前被拒绝过）
        // 此时 CanI 返回的 err 为 nil，Factory 层需要自己构造错误信息
        return nil, fmt.Errorf("%v access denied on resource %q:%q", verbs, ns, gvr)
    }
    if gvr == client.NsGVR {
        ns = client.ClusterScope  // Namespace 资源是集群范围的，Informer 用集群范围
    }
    return f.ForResource(ns, gvr)
}
```

**异常路径：**

```
CanForResource → CanI
                ├── err != nil → Factory: return nil, err (原样透传)
                └── err == nil, auth == false
                    (缓存命中拒绝结果) → Factory: return nil, fmt.Errorf(...)
                    (需要重新构造错误，因为 CanI 缓存命中时没有返回 err)
```

### 3.4 Factory.CanForInstance - 实例级预检

位于 [factory.go#L224-L248](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go#L224-L248)

```go
func (f *Factory) CanForInstance(fqn string, gvr *GVR, verbs []string) (informers.GenericInformer, error) {
    ns, n := namespaced(fqn)
    authNs := ns
    if gvr == client.NsGVR {
        authNs = n  // 对 Namespace 资源，用资源名作为 RBAC 检查的命名空间
    }
    auth, err := f.Client().CanI(authNs, gvr, n, verbs)
    if err != nil {
        return nil, err
    }
    if !auth {
        return nil, fmt.Errorf("%v access denied on resource %q:%q", verbs, authNs, gvr)
    }
    return f.ForResource(ns, gvr)  // ⚠️ Informer 仍使用原始 ns，authNs 仅用于权限检查
}
```

**Namespace 资源的特殊处理**：Informer 用 `ns`（可能是 `-` 集群范围），但 RBAC 检查用 `authNs = n`（实际的 namespace 名称），这样能正确检查用户对特定命名空间的权限。

---

## 四、权限不足时的展示策略与异常路径

### 4.1 视图初始化时的权限检查

位于 [browser.go#L89-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L89-L94)

```go
if dao.IsK8sMeta(b.meta) && b.app.ConOK() {
    if _, e := b.app.factory.CanForResource(ns, b.GVR(), client.ListAccess); e != nil {
        return e  // 直接 return error，Init 失败
    }
}
```

**异常传播链：**

```
Browser.Init → return err
    ↓
App.inject (app.go#L799-L802)
    ├── err != nil → slog.Error(...)
    └── (inject 的调用方决定是否展示 Flash)
```

**⚠️ 边界：** inject 函数中出错后只是打了 slog.Error，但部分调用方（如 showRules）有 Flash 处理，其他入口可能静默失败。

```go
func showRules(app *App, _ ui.Tabular, gvr *client.GVR, path string) {
    v := NewRbac(client.RbacGVR)
    v.SetContextFn(rbacCtx(gvr, path))
    if err := app.inject(v, false); err != nil {
        app.Flash().Err(err)  // ✅ 有 Flash 处理
    }
}
```

### 4.2 TableLoadFailed - 数据加载失败展示

位于 [browser.go#L367-L373](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L367-L373)

```go
func (b *Browser) TableLoadFailed(err error) {
    b.app.QueueUpdateDraw(func() {
        b.app.Flash().Err(err)       // Flash 红色错误
        b.App().ClearStatus(false)   // 清除状态栏
    })
}
```

**触发时机**：Model 层 Watch/Refresh 出错时回调。例如：
- Informer List 失败（权限不足被 API Server 拒绝）
- 资源已被删除但缓存未更新
- 网络中断

### 4.3 命名空间切换时的权限检查

位于 [browser.go#L560-L596](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L560-L596)

```go
auth, err := b.App().factory.Client().CanI(ns, b.GVR(), "", client.ListAccess)
if !auth {
    if err == nil {
        // ⚠️ CanI 返回了 auth=false 但 err==nil
        // 这种情况发生在：缓存命中了之前被拒绝的结果（缓存中只有 bool，没有 err）
        // 需要调用方重新构造错误消息
        err = fmt.Errorf("access denied for user on: %s/%s", ns, b.GVR())
    }
    b.App().Flash().Err(err)
    return nil  // 不切换命名空间
}
```

**两种拒绝路径的展示差异：**

| 场景 | CanI 返回值 | 展示的错误消息 |
|------|-------------|----------------|
| 实时 API 拒绝 | auth=false, err=`"(list) access denied for user on resource \"\":\"pods\" in namespace \"default\""` | API 返回的详细消息 |
| 缓存命中拒绝 | auth=false, err=`nil` | 调用方构造 `"access denied for user on: default/pods"` |

缓存命中时丢失了 API Server 返回的 `Status.Reason` 详细原因，消息更简短。

### 4.4 编辑操作前的权限检查

位于 [browser.go#L533-L558](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L533-L558)

```go
func editRes(app *App, gvr *client.GVR, path string) error {
    // ...
    if ok, err := app.Conn().CanI(ns, gvr, n, client.PatchAccess); !ok || err != nil {
        return fmt.Errorf("current user can't edit resource %s", gvr)
    }
    // 权限通过后才执行 edit 命令
}
```

**⚠️ 错误信息被覆盖**：`CanI` 返回的原始错误（如 API Server 详细拒绝原因）被丢弃，统一替换为 `"current user can't edit resource <gvr>"`。

调用方处理：
```go
if err := editRes(b.app, b.GVR(), path); err != nil {
    b.App().Flash().Err(err)  // Flash 红色错误
}
```

### 4.5 动作按钮可见性控制

位于 [browser.go#L627-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L627-L676)

```go
func (b *Browser) refreshActions() {
    aa := ui.NewKeyActionsFromMap(...)
    if b.app.ConOK() {
        if !b.app.Config.IsReadOnly() {
            // ⚠️ 这里检查的是 b.meta.Verbs（API Server 发现的资源能力）
            // 不是当前用户的实际权限！
            if client.Can(b.meta.Verbs, "edit") {
                aa.Add(ui.KeyE, ui.NewKeyActionWithOpts("Edit", b.editCmd, ...))
            }
            if client.Can(b.meta.Verbs, "delete") {
                aa.Add(tcell.KeyCtrlD, ui.NewKeyActionWithOpts("Delete", b.deleteCmd, ...))
            }
        }
    }
    // ...
}
```

**⚠️ 严重边界：** 按钮可见性基于资源类型是否支持该操作（`b.meta.Verbs`），而非当前用户是否有权限。

结果：
- 用户看到 Edit/Delete 按钮显示
- 点击后才执行实际权限检查（editCmd/deleteCmd → CanI）
- 无权限时通过 Flash 报错

这种设计是故意的——避免在每次刷新动作时都发送大量权限检查 API 调用。

### 4.6 无数据与缓存同步中的展示策略

位于 [browser.go#L298-L335](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L298-L335)

```
TableNoData(mdata)
│
├── !app.ConOK() || cancel==nil || !app.IsRunning()
│   └── return (静默，不展示任何消息)
│
├── firstView == 0 || HeaderCount == 0
│   ├── firstView++
│   └── return (首次加载，不警告)
│
├── !factory.HasSynced(gvr, ns)
│   └── Flash.Infof("Synchronizing %s in %q namespace...", gvr, ns)
│       (中性信息，绿色/白色)
│
└── HasSynced == true（已同步但仍无数据）
    └── Flash.Warnf("No resources found for %s in %q namespace", gvr, ns)
        (警告，黄色)
```

**HasSynced 自身也可能因权限失败：**

位于 [factory.go#L101-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go#L101-L109)

```go
func (f *Factory) HasSynced(gvr *client.GVR, ns string) (bool, error) {
    inf, err := f.CanForResource(ns, gvr, client.ListAccess)
    if err != nil {
        return false, err  // ⚠️ TableNoData 中调用 HasSynced 时忽略了这个 error
    }
    return inf.Informer().HasSynced(), nil
}
```

**隐式异常路径**：如果用户在该命名空间无 list 权限，`HasSynced` 返回 `(false, err)`，但 `TableNoData` 中的判断只取第一个返回值：

```go
if synced, _ := b.app.factory.HasSynced(b.GVR(), b.GetNamespace()); !synced {
    // err 被丢弃！
    // 用户永远看到 "Synchronizing..." 而不是权限不足
}
```

这是一个真实的 Bug：权限不足导致 Informer 创建失败时，HasSynced 始终返回 false，界面无限显示 "Synchronizing..." 中性提示，永远不会告知用户真正的原因是权限不足。

### 4.7 权限错误展示层级总览

| 场景 | 触发位置 | 展示方式 | 颜色 | 信息丢失/异常 |
|------|----------|----------|------|--------------|
| 视图初始化失败 | Browser.Init | 调用方 Flash.Err 或静默 slog.Error | 红色 | 取决于调用方是否处理 |
| 数据加载失败 | Browser.TableLoadFailed | Flash.Err + ClearStatus | 红色 | 原始错误直接展示 |
| 命名空间切换权限不足 | Browser.switchNamespaceCmd | Flash.Err | 红色 | 缓存命中时丢失详细 Reason |
| 编辑权限不足 | editRes → Browser.editCmd | Flash.Err | 红色 | 原始错误被替换为通用消息 |
| 删除权限不足 | Browser.resourceDelete | CanI 在 Model.Delete 中，错误透传 | 红色 | 取决于下层实现 |
| Informer 权限不足导致无限同步中 | Browser.TableNoData | Flash.Info "Synchronizing..." | 中性/绿色 | **真正原因被隐藏，Bug** |
| 首次加载无数据 | Browser.TableNoData | 无提示 | - | 静默 |
| 同步后无数据 | Browser.TableNoData | Flash.Warn "No resources found" | 黄色 | 不区分"真的没数据"还是"权限不足看不到" |
| 动作按钮可见性 | Browser.refreshActions | 始终显示（基于 meta.Verbs） | - | 不反映用户实际权限 |

---

## 五、RBAC 视图渲染

### 5.1 动词图标展示

位于 [rbac.go#L79-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/rbac.go#L79-L101)

```go
func asVerbs(verbs []string) []string {
    // 标准8个动词列：get, list, watch, create, patch, update, delete, deletecollection
    for _, v := range k8sVerbs {
        r = append(r, toVerbIcon(hasVerb(verbs, v)))
        // ✓ 绿色: [green::b] ✓ [::]
        // × 橙红色: [orangered::b] × [::]
    }
    // 额外动词列（非标准）：拼接后截断至 30 字符
    return append(r, Truncate(strings.Join(unknowns, ","), unknownLen))
}
```

### 5.2 动词匹配逻辑

位于 [rbac.go#L110-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/rbac.go#L110-L127)

```go
func hasVerb(verbs []string, verb string) bool {
    if len(verbs) == 1 && verbs[0] == allVerbs {
        return true  // "*" 表示所有权限
    }
    for _, v := range verbs {
        if hv, ok := httpTok8sVerbs[v]; ok {
            if hv == verb {
                return true  // HTTP 动词映射：post→create, put→update
            }
        }
        if v == verb {
            return true
        }
    }
    return false
}
```

**边界：** HTTP 动词映射仅单向生效（`post` → `create`），但反向不生效（`create` 不会匹配 `post`）。

### 5.3 显示列结构

**Rbac 视图（单角色/绑定）列：**

```
NAME | API-GROUP | GET | LIST | WATCH | CREATE | PATCH | UPDATE | DELETE | DEL-LIST | EXTRAS | VALID
```

**Policy 视图（按主体聚合）列：**

```
NAMESPACE | NAME | API-GROUP | BINDING | GET | LIST | WATCH | CREATE | PATCH | UPDATE | DELETE | DEL-LIST | EXTRAS | VALID
```

**⚠️ VALID 列始终为空字符串**：`Render` 方法中直接 `append(ro.Fields, "")`，未填充任何有效值。该列预留但未使用。

---

## 六、关键设计要点与边界问题总结

### 6.1 权限聚合设计

| 设计点 | 说明 | 潜在问题 |
|--------|------|----------|
| GR (Group/Resource) 作为聚合键 | 相同 Group/Resource 的权限合并 | 资源名规则和通配规则 GR 不同，无法合并；跨 Namespace 同资源合并时 Namespace 信息丢失 |
| Verbs 去重合并 | Merge 只合并动词列表 | Binding、Namespace 等来源信息完全丢失，无法追踪权限来源 |
| parseRules 双重展开 | 有 ResourceNames 时同时生成资源名规则和通配规则 | 界面上显示两条独立行，但 Verb 相同，用户可能困惑 |
| ResourceName/NonResourceURL 字段冗余 | PolicyRes 结构体定义了但从不赋值 | 无法通过结构化字段区分规则类型，只能靠字符串解析 |

### 6.2 同名角色绑定的信息丢失

位于 [fetchRoleBindingNamespaces](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L167-L184)

- **Map Value 是单 string 而非 slice**：不同命名空间的同名 Role/ClusterRole，后者覆盖前者
- **影响**：Policy 视图中可能只显示一个命名空间的权限，其他命名空间的同名角色权限被静默丢弃

### 6.3 权限缓存的边界

| 方面 | 设计 | 潜在问题 |
|------|------|----------|
| LRU 容量 | 100 条 | 资源多时频繁淘汰 |
| 过期时间 | 固定 5 分钟 | 权限变更后最长 5 分钟才生效 |
| 动词顺序 | Key 包含 verb 顺序 | ["get","list"] 和 ["list","get"] 不共享缓存 |
| 拒绝结果缓存 | 缓存 false | 被拒绝后管理员立刻授权，用户仍看到拒绝 |
| 连接状态 | 断开不清缓存 | 连接恢复后旧缓存可能无效 |

### 6.4 权限不足展示的异常路径

| 问题 | 位置 | 影响 |
|------|------|------|
| VALID 列未使用 | Render 方法 | 用户永远看到空列 |
| HasSynced error 被丢弃 | TableNoData | 权限不足时界面永久显示 "Synchronizing..." |
| 动作按钮不检查用户权限 | refreshActions | Edit/Delete 按钮显示但点击后才报错 |
| CanI 缓存丢失详细 Reason | switchNamespaceCmd | 缓存命中拒绝时错误信息不完整 |
| editRes 覆盖原始错误 | editRes | API Server 返回的详细原因被替换为通用消息 |
| Init 错误取决于调用方 | App.inject | 部分入口可能静默失败无 Flash |

### 6.5 渐进式权限检查的设计权衡

1. **视图初始化检查**：确保有 list 权限才能打开视图（粗粒度前置检查）
2. **动作按钮不检查**：基于资源类型而非用户权限（避免大量 API 调用）
3. **操作时再次检查**：点击 Edit/Delete 时才执行精确权限检查（懒检查）
4. **命名空间切换检查**：不同命名空间权限可能不同

这种设计在"用户体验（响应速度）"和"精确性"之间做了权衡：优先保证界面流畅，实际操作时才做精确校验。
