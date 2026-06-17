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

### 2.3 权限合并的三个层级

代码中存在三个不同层级的"合并"操作，它们的范围、触发条件和信息保留行为完全不同。必须精确区分才能判断来源信息何时保留、何时丢失。

#### 层级一：parseRules 内部合并——同一角色的规则之间

位于 [parseRules](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L151-L174)

`parseRules` 接收**单个角色的所有 Rules**，在一个 `Policies` 列表内通过 `Upsert` 合并。

```go
func parseRules(ns, binding string, rules []rbacv1.PolicyRule) render.Policies {
    pp := make(render.Policies, 0, len(rules))
    for _, rule := range rules {
        for _, grp := range rule.APIGroups {
            for _, res := range rule.Resources {
                for _, na := range rule.ResourceNames {
                    pp = pp.Upsert(render.NewPolicyRes(ns, binding, FQN(res, na), grp, rule.Verbs))
                }
                pp = pp.Upsert(render.NewPolicyRes(ns, binding, FQN(grp, res), grp, rule.Verbs))
            }
        }
    }
    return pp
}
```

**关键特征**：同一角色内，ns 和 binding 参数完全相同。

**合并行为**：因为所有 PolicyRes 的 Namespace 和 Binding 都相同，即使 GR 匹配触发 Merge，Namespace 和 Binding 的值也不会丢失——先插入的和后插入的值相同。

**何时触发 GR 匹配**：同一角色中，不同 PolicyRule 可能覆盖相同的 Group/Resource。例如：

```yaml
# Rule 1
apiGroups: [""]
resources: ["pods"]
verbs: ["get", "list"]
# Rule 2
apiGroups: [""]
resources: ["pods"]
verbs: ["create"]
```

这两条 Rule 展开后 GR 都是 `"core/core/pods"`，Upsert 触发 Merge，Verbs 合并为 [get, list, create]。Namespace 和 Binding 不受影响。

**结论**：✅ **层级一不会丢失来源信息**——同一角色内的合并，Namespace 和 Binding 始终相同。

#### 层级二：loadClusterRoleBinding / loadRoleBinding 内部追加——不同角色之间

`loadClusterRoleBinding` 和 `loadRoleBinding` 都在函数内维护一个 `rows render.Policies`，通过 `rows = append(rows, parseRules(...)...)` 追加不同角色的解析结果。

**append 本身是纯追加**，不会触发合并。但 parseRules 返回的 Policies 内部已经过层级一的合并。追加后 rows 中可能存在 GR 相同但来自不同角色的 PolicyRes——**此时没有跨角色的 Upsert/Merge 调用**。

```go
// loadClusterRoleBinding
rows := make(render.Policies, 0, len(nn))
for i := range crs {
    rows = append(rows, parseRules(client.NotNamespaced, "CR:"+crs[i].Name, crs[i].Rules)...)
}
return rows, nil
```

```go
// loadRoleBinding
rows := make(render.Policies, 0, len(crs))
for i := range crs {
    rows = append(rows, parseRules(rbNs, "CR:"+crs[i].Name, crs[i].Rules)...)
}
for i := range ros {
    rows = append(rows, parseRules(ros[i].Namespace, "RO:"+ros[i].Name, ros[i].Rules)...)
}
return rows, nil
```

**关键特征**：`append` 是纯列表拼接，不调用 Upsert/Merge。不同角色的解析结果作为独立条目共存于同一个 Policies 列表中。

**GR 冲突但未合并**：如果 ClusterRole/A 对 pods 有 [get] 且 ClusterRole/B 对 pods 有 [delete]，rows 中会存在两条 GR 相同的 PolicyRes：

```
rows[0] = {Namespace:"*", Binding:"CR:A", Resource:"core/pods", Verbs:[get]}
rows[1] = {Namespace:"*", Binding:"CR:B", Resource:"core/pods", Verbs:[delete]}
```

**结论**：✅ **层级二保留来源信息**——append 不触发合并，不同角色生成独立的 PolicyRes 条目，各自保留自己的 Namespace 和 Binding。

#### 层级三：List 方法汇总——跨绑定类型追加

位于 [List](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L32-L60)

```go
func (p *Policy) List(ctx context.Context, _ string) ([]runtime.Object, error) {
    crps, _ := p.loadClusterRoleBinding(kind, name)
    rps, _  := p.loadRoleBinding(kind, name)

    oo := make([]runtime.Object, 0, len(crps)+len(rps))
    for _, p := range crps {
        oo = append(oo, p)
    }
    for _, p := range rps {
        oo = append(oo, p)
    }
    return oo, nil
}
```

**关键特征**：这里将 Policies 转为 `[]runtime.Object`，是最终返回给视图层的数据。`append` 仍然是纯列表拼接。

**结论**：✅ **层级三保留来源信息**——将 crps 和 rps 简单追加为 runtime.Object 列表，不触发合并。

### 2.4 来源信息何时保留、何时丢失

#### 2.4.1 合并机制的唯一入口：Upsert

位于 [Upsert](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L158-L172)

```go
func (pp Policies) Upsert(p *PolicyRes) Policies {
    idx, ok := pp.find(p.GR())
    if !ok {
        return append(pp, p)
    }
    p, err := pp[idx].Merge(p)
    if err != nil {
        slog.Error("Policy upsert failed", slogs.Error, err)
        return pp
    }
    pp[idx] = p
    return pp
}
```

Upsert 只在 `parseRules` 内部被调用。**跨角色追加（层级二、层级三）使用的是原生 `append`，不经过 Upsert**。

因此：**来源信息丢失只发生在 parseRules 内部的 Upsert/Merge 中，且仅当 GR 碰撞时。**

#### 2.4.2 保留来源的精确场景

| 场景 | 合并层级 | 是否丢失来源 | 原因 |
|------|----------|-------------|------|
| 同一角色内不同 Rule 覆盖同一 GR | 层级一（parseRules 内 Upsert） | ❌ 不丢失 | Namespace 和 Binding 相同 |
| 不同 ClusterRole 对同一 GR 有权限 | 层级二（append 追加） | ❌ 不丢失 | append 不合并，各自独立 |
| ClusterRoleBinding + RoleBinding 各有同 GR 权限 | 层级三（append 追加） | ❌ 不丢失 | append 不合并，各自独立 |
| 同一角色内不同 Rule 对同 GR 授予不同 Verbs | 层级一（parseRules 内 Upsert） | ❌ 不丢失 | Namespace 和 Binding 相同 |
| 不同 Role 对同一 GR 有权限 | 层级二（append 追加） | ❌ 不丢失 | append 不合并，各自独立 |

#### 2.4.3 丢失来源的精确场景

来源丢失需要两个条件**同时**满足：
1. **GR 碰撞**：两条 PolicyRes 的 `Group + "/" + Resource` 相同
2. **在同一 Policies 列表内经过 Upsert**

当前代码中，**这两个条件只在 parseRules 内部同一角色的规则之间才同时满足**，而此时 Namespace 和 Binding 参数相同，不会丢失。

**但是，存在一个隐含的 GR 碰撞路径**：同一角色内，PolicyRule 中有 ResourceNames 时，资源名规则和通配规则会生成不同的 Resource 字段值，导致不同的 GR，不会碰撞。然而，如果同一角色的两条 PolicyRule 分别以不同方式引用同一资源（如一条用 `apiGroups:[""]` + `resources:["pods"]`，另一条用 `apiGroups:[""]` + `resources:["pods"]` + `resourceNames:["my-pod"]`），它们的 GR 不同（`core/core/pods` vs `core/pods/my-pod`），不会碰撞。

**结论：在当前代码的实际路径下，来源信息不会因 Merge 而丢失。**

#### 2.4.4 界面呈现中来源信息"看起来丢失"的原因

虽然 Merge 本身没有丢失来源信息，但用户在 Policy 视图中仍然可能看不到完整的来源，原因是**不同角色的同 GR 条目作为独立行共存**：

```
rows[0] = {Namespace:"*", Binding:"CR:admin", Resource:"core/pods", Verbs:[get,list]}
rows[1] = {Namespace:"default", Binding:"RO:writer", Resource:"core/pods", Verbs:[create,patch]}
```

这两行在视图中渲染为：

| NAMESPACE | NAME | API-GROUP | BINDING | GET | LIST | CREATE | PATCH |
|-----------|------|-----------|---------|-----|------|--------|-------|
| * | pods | core | CR:admin | ✓ | ✓ | × | × |
| default | pods | core | RO:writer | × | × | ✓ | ✓ |

**来源信息完整保留**，用户可以看到两个不同的来源。但 Verbs 被分散到不同行，需要用户自行在脑中合并才能理解"对 pods 的完整权限是什么"。

**与之前分析结论的修正**：之前认为"跨角色 Merge 会丢失 Binding 信息"，这是不准确的。跨角色追加使用的是 append 而非 Upsert，不会触发 Merge。来源信息实际是保留的，只是分散在不同行中。

### 2.5 同名角色误纳入问题（独立类别）

此问题与合并/丢失无关，是**角色选择阶段的匹配缺陷**，属于独立的 Bug 类别。

#### 2.5.1 问题根因

位于 [loadRoleBinding](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L93-L129)

```go
// Role 处理片段
ros, _ := p.fetchRoles()  // 返回所有命名空间的所有 Role
for i := range ros {
    if _, ok := rbsMap["Role:"+ros[i].Name]; !ok {
        continue
    }
    // ⚠️ 只要 rbsMap 中存在 "Role:<name>" 键，就纳入
    // 不检查 Role 所在命名空间是否与 RoleBinding 的命名空间一致
    rows = append(rows, parseRules(ros[i].Namespace, "RO:"+ros[i].Name, ros[i].Rules)...)
}
```

`rbsMap` 的 Key 是 `RoleRef.Kind + ":" + RoleRef.Name`（如 `"Role:admin"`），不包含命名空间。`fetchRoles` 返回所有命名空间的 Role。当遍历 Role 列表时，只要 `rbsMap` 中存在该名称的键，所有命名空间中同名的 Role 都会被纳入。

#### 2.5.2 误纳入的精确条件

同时满足以下三个条件才会触发误纳入：

1. **主体在某个命名空间存在 RoleBinding**，引用了一个 Role（如 `Role/admin`）
2. **另一个命名空间存在同名 Role**（如另一个命名空间也有 `Role/admin`）
3. **该主体在另一个命名空间没有对应的 RoleBinding**

条件 2 和 3 同时成立意味着：另一个命名空间的同名 Role 从未绑定给该主体，但因为名称匹配而被纳入。

#### 2.5.3 误纳入场景详解

**场景构造**：User/alice 只在 ns-a 被授予 Role/admin，但 ns-b 也有名为 admin 的 Role

```
# 绑定关系（alice 仅在 ns-a 有绑定）
ns-a/RoleBinding/rb-a → Role:admin (ns-a) → Subject: User/alice

# Role 定义（两个命名空间都有同名 Role，但权限不同）
ns-a/Role/admin → pods [get, list]
ns-b/Role/admin → pods [delete, create]  # 这个 Role 从未绑定给 alice！
```

**执行流程**：

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | fetchRoleBindingNamespaces | `rbsMap = {"Role:admin": "ns-a"}` |
| 2 | fetchRoles | 返回 `[ns-a/admin, ns-b/admin]` |
| 3 | 遍历 ns-a/admin | `rbsMap["Role:admin"]` 存在 → 纳入 ✅ |
| 4 | 遍历 ns-b/admin | `rbsMap["Role:admin"]` 存在 → **误纳入** ❌ |

**最终显示**：

| NAMESPACE | NAME | API-GROUP | BINDING | GET | LIST | CREATE | DELETE |
|-----------|------|-----------|---------|-----|------|--------|--------|
| ns-a | pods | core | RO:admin | ✓ | ✓ | × | × |
| ns-b | pods | core | RO:admin | × | × | ✓ | ✓ |

alice 实际上从未被授予 ns-b/Role/admin 的权限，但界面显示她在 ns-b 有 create/delete 权限。

#### 2.5.4 不会误纳入的场景

**ClusterRole 不受此问题影响**：

```go
// ClusterRole 处理片段
for i := range crs {
    if rbNs, ok := rbsMap["ClusterRole:"+crs[i].Name]; ok {
        rows = append(rows, parseRules(rbNs, "CR:"+crs[i].Name, crs[i].Rules)...)
    }
}
```

ClusterRole 是集群范围的，名称全局唯一。不存在"另一个命名空间有同名 ClusterRole"的情况。

**不同名称的 Role 不受此问题影响**：如果 ns-b 的 Role 叫 `writer` 而不是 `admin`，`rbsMap["Role:writer"]` 不存在，不会被纳入。

#### 2.5.5 ClusterRole 命名空间偏移问题

**这是另一个独立问题**，不属于误纳入，也不属于合并丢失，而是**map.Value 覆盖导致命名空间显示偏移**。

位于 [fetchRoleBindingNamespaces](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L167-L184)

```go
ss[rbs[i].RoleRef.Kind+":"+rbs[i].RoleRef.Name] = rbs[i].Namespace
// ⚠️ Value 是单个 string，多次写入同一 Key 会导致后者覆盖前者
```

当多个命名空间的 RoleBinding 引用同一个 ClusterRole 时：

```
ns-a/RoleBinding/crb-a → ClusterRole:cluster-admin → Subject: User/alice
ns-b/RoleBinding/crb-b → ClusterRole:cluster-admin → Subject: User/alice
```

| 循环 | rb[i] | Map Key | Map Value |
|------|-------|---------|-----------|
| i=0 | ns-a/crb-a | `"ClusterRole:cluster-admin"` | `"ns-a"` |
| i=1 | ns-b/crb-b | `"ClusterRole:cluster-admin"` | **"ns-b"（覆盖！）** |

后续 `loadRoleBinding` 处理时使用 `rbNs` 作为 parseRules 的 ns 参数：

```go
rows = append(rows, parseRules(rbNs, "CR:"+crs[i].Name, crs[i].Rules)...)
// rbNs = "ns-b"，ns-a 的信息丢失
```

**结果**：ClusterRole 的权限行只显示一个命名空间（最后覆盖的），遗漏了其他命名空间。

**与 loadClusterRoleBinding 的对比**：

```go
// ClusterRoleBinding 路径
rows = append(rows, parseRules(client.NotNamespaced, "CR:"+crs[i].Name, crs[i].Rules)...)
// 使用 NotNamespaced = "*"，显示正确
```

#### 2.5.6 三个独立问题的关系

| 问题 | 类别 | 触发条件 | 影响 |
|------|------|----------|------|
| 同名 Role 误纳入 | 角色选择缺陷 | 主体在某命名空间有 RoleBinding + 另一命名空间有同名 Role | 显示了主体实际没有的权限 |
| ClusterRole 命名空间偏移 | map.Value 覆盖 | 多命名空间 RoleBinding 引用同一 ClusterRole | NAMESPACE 列只显示一个命名空间 |
| fetchRoleBindingNamespaces 的 map 覆盖 | map 设计缺陷 | 多个 RoleBinding 引用同名 Role | Map 只保留最后一个绑定命名空间 |

三个问题的根因不同，但都源自 `fetchRoleBindingNamespaces` 的 `map[string]string` 设计——Key 不包含命名空间，Value 是单值而非列表。

### 2.6 主体权限聚合完整流程

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
            cns = bns
        }
        return client.IsAllNamespaces(ns) || cns == ns
    }
    return true
}
```

边界处理：ServiceAccount 未显式指定 Namespace 时，回退到 RoleBinding 所在命名空间。

### 2.7 角色/绑定查看 (Rbac DAO)

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

### 6.1 权限合并层级与来源保留

| 层级 | 范围 | 合并方式 | 来源信息 | 说明 |
|------|------|----------|----------|------|
| 层级一 | parseRules 内部（同一角色） | Upsert/Merge | ✅ 保留 | 同一角色内 Namespace 和 Binding 相同，即使 GR 碰撞也不会丢失 |
| 层级二 | loadClusterRoleBinding/loadRoleBinding 内部（跨角色） | append | ✅ 保留 | append 是纯追加，不触发合并，不同角色生成独立 PolicyRes 条目 |
| 层级三 | List 方法（跨绑定类型） | append | ✅ 保留 | append 不触发合并，ClusterRoleBinding 和 RoleBinding 的结果各自独立 |

**核心结论**：当前代码中，Merge 只在 parseRules 内部触发（层级一），且此时 Namespace 和 Binding 参数相同，不会丢失来源信息。跨角色和跨绑定类型的追加使用原生 append，不会合并。

**用户感知问题**：虽然来源信息保留，但同一 GR 的权限分散在不同行中，用户需要自行在脑中合并才能理解完整权限。这不属于"信息丢失"，而是"信息分散"。

### 6.2 同名角色误纳入（独立 Bug 类别）

| 条件 | 说明 |
|------|------|
| 触发条件 1 | 主体在某命名空间存在 RoleBinding，引用了一个 Role |
| 触发条件 2 | 另一个命名空间存在同名 Role |
| 触发条件 3 | 该主体在另一个命名空间没有对应的 RoleBinding |
| 根因 | `rbsMap["Role:"+name]` 只按名称匹配，不检查命名空间 |
| 影响 | 显示了主体实际没有的权限 |
| 不受影响 | ClusterRole（名称全局唯一）、不同名称的 Role |

### 6.3 ClusterRole 命名空间偏移（独立问题）

| 条件 | 说明 |
|------|------|
| 触发条件 | 多个命名空间的 RoleBinding 引用同一 ClusterRole |
| 根因 | `map[string]string` 的 Value 被后写入的覆盖 |
| 影响 | ClusterRole 的权限行只显示一个命名空间（最后覆盖的），遗漏其他命名空间 |
| 对比 | ClusterRoleBinding 路径使用 NotNamespaced="*"，显示正确 |

### 6.4 权限缓存的边界

| 方面 | 设计 | 潜在问题 |
|------|------|----------|
| LRU 容量 | 100 条 | 资源多时频繁淘汰 |
| 过期时间 | 固定 5 分钟 | 权限变更后最长 5 分钟才生效 |
| 动词顺序 | Key 包含 verb 顺序 | ["get","list"] 和 ["list","get"] 不共享缓存 |
| 拒绝结果缓存 | 缓存 false | 被拒绝后管理员立刻授权，用户仍看到拒绝 |
| 连接状态 | 断开不清缓存 | 连接恢复后旧缓存可能无效 |

### 6.5 权限不足展示的异常路径

| 问题 | 位置 | 影响 |
|------|------|------|
| VALID 列未使用 | Render 方法 | 用户永远看到空列 |
| HasSynced error 被丢弃 | TableNoData | 权限不足时界面永久显示 "Synchronizing..." |
| 动作按钮不检查用户权限 | refreshActions | Edit/Delete 按钮显示但点击后才报错 |
| CanI 缓存丢失详细 Reason | switchNamespaceCmd | 缓存命中拒绝时错误信息不完整 |
| editRes 覆盖原始错误 | editRes | API Server 返回的详细原因被替换为通用消息 |
| Init 错误取决于调用方 | App.inject | 部分入口可能静默失败无 Flash |

### 6.6 渐进式权限检查的设计权衡

1. **视图初始化检查**：确保有 list 权限才能打开视图（粗粒度前置检查）
2. **动作按钮不检查**：基于资源类型而非用户权限（避免大量 API 调用）
3. **操作时再次检查**：点击 Edit/Delete 时才执行精确权限检查（懒检查）
4. **命名空间切换检查**：不同命名空间权限可能不同

这种设计在"用户体验（响应速度）"和"精确性"之间做了权衡：优先保证界面流畅，实际操作时才做精确校验。
