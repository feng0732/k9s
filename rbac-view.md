# RBAC 视图权限预检与资源聚合分析

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

---

## 二、角色合并与权限聚合

### 2.1 核心数据结构

#### PolicyRes - 单条权限规则

位于 [policy.go#L95-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L95-L113)

```go
type PolicyRes struct {
    Namespace, Binding string
    Resource, Group    string
    ResourceName       string
    NonResourceURL     string
    Verbs              []string
}
```

- **GR()** - Group/Resource 组合，用于唯一标识一种资源

#### Policies - 权限规则集合

位于 [policy.go#L155-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L155-L183)

```go
type Policies []*PolicyRes
```

### 2.2 权限合并策略

#### Upsert 方法

位于 [policy.go#L158-L172](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L158-L172)

**核心逻辑：**

1. 以 `Group/Resource` (GR) 为键查找现有策略

2. 若未找到则追加新策略

3. 若找到则合并两个策略的 Verbs 列表

```go
func (pp Policies) Upsert(p *PolicyRes) Policies {
    idx, ok := pp.find(p.GR())
    if !ok {
        return append(pp, p)
    }
    p, err := pp[idx].Merge(p)
    // ...
    pp[idx] = p
    return pp
}
```

#### Merge 方法

位于 [policy.go#L120-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/policy.go#L120-L133)

**合并规则：**

- 仅当 GR (Group/Resource) 相同时才能合并

- 合并时去重 Verbs 列表

- 不合并 Namespace 和 Binding 字段（保留原始的）

### 2.3 规则解析流程 (parseRules)

位于 [rbac.go#L151-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L151-L174)

将 Kubernetes 的 `rbacv1.PolicyRule` 解析为 `render.Policies：

```
PolicyRule
├── APIGroups []string     → 遍历每个 API Group
│   └── Resources []string    → 遍历每个 Resource
│       ├── ResourceNames []string → 每个 ResourceName 生成一条 PolicyRes
│       └── (无 ResourceNames 生成一条 PolicyRes
└── NonResourceURLs []string → 每个非资源 URL 生成一条 PolicyRes
```

**关键点：**

- 空 APIGroup 映射为 `"core"`

- 有 ResourceNames 时会生成多条规则（每个资源名一条）+ 无 ResourceNames 时生成一条通配规则

- NonResourceURLs 以 `/` 开头，Group 设为 `n/a`

### 2.4 主体权限聚合 (Policy DAO)

位于 [rbac_policy.go#L32-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L32-L60)

**聚合流程：**

```
主体 (User/Group/SA)
├── 查找所有 ClusterRoleBinding
│   └── 匹配主体 → 找到对应的 ClusterRole
│       └── 解析 ClusterRole.Rules → Policies
└── 查找所有 RoleBinding
    ├── 匹配主体 → 找到对应的 ClusterRole (通过 RoleRef)
    │   └── 解析 ClusterRole.Rules → Policies
    └── 匹配主体 → 找到对应的 Role
        └── 解析 Role.Rules → Policies
```

**关键函数：**

- **loadClusterRoleBinding**: 加载集群角色绑定的权限

- **loadRoleBinding**: 加载命名空间角色绑定的权限

- **isSameSubject**: 判断主体是否匹配（特别处理 ServiceAccount 的命名空间校验）

#### isSameSubject 逻辑

位于 [rbac_policy.go#L189-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac_policy.go#L189-L202)

```go
func isSameSubject(kind, ns, bns, name string, subject *rbacv1.Subject) bool {
    // Kind 和 Name 必须匹配
    if subject.Kind != kind || subject.Name != name {
        return false
    }
    // ServiceAccount 还需要校验命名空间
    if kind == rbacv1.ServiceAccountKind {
        cns := subject.Namespace
        if cns == "" {
            cns = bns  // 绑定的命名空间
        }
        return client.IsAllNamespaces(ns) || cns == ns
    }
    return true
}
```

### 2.5 角色/绑定查看 (Rbac DAO)

位于 [rbac.go#L30-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/dao/rbac.go#L30-L52)

根据上下文的 GVR 类型加载不同的资源：

| GVR 类型 | 加载方法 |
|-----------|----------|
| clusterrolebindings | loadClusterRoleBinding |
| rolebindings | loadRoleBinding |
| clusterroles | loadClusterRole |
| roles | loadRole |

**RoleBinding 的特殊处理：**

- RoleRef.Kind 为 `ClusterRole` 时，加载对应的 ClusterRole 规则

- RoleRef.Kind 为 `Role` 时，加载对应命名空间的 Role 规则

---

## 三、权限预检机制

### 3.1 权限预检层次

系统有两层权限预检：

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

### 3.2 APIClient.CanI - 核心权限检查

位于 [client.go#L154-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/client.go#L154-L216)

#### 核心流程

```
CanI(ns, gvr, name, verbs)
│
├── 连接状态检查 → 未连接直接返回 false
│
├── 特殊资源处理
│   ├── NsGVR: ns = name (命名空间资源用资源名当命名空间
│   ├── HmGVR: gvr = SecGVR (Helm 用 Secret 存储)
│   └── 集群范围: ns = BlankNamespace
│
├── 缓存检查
│   └── 缓存命中 → 直接返回缓存结果
│
├── 构造 SelfSubjectAccessReview
│
└── 逐个动词检查
    └── 任一动词不允许 → 缓存 false + 返回错误
    └── 全部允许 → 缓存 true + 返回 true
```

#### 缓存机制

- **缓存类型**：LRUExpireCache (LRU 带过期缓存)

- **缓存大小**：100 条

- **过期时间**：5 分钟 (cacheExpiry)

- **缓存 Key**：`ns:gvr:name::verb1,verb2,...`

#### SelfSubjectAccessReview 构造

位于 [client.go#L91-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/client.go#L91-L108)

```go
func makeSAR(ns string, gvr *GVR, name string) *authorizationv1.SelfSubjectAccessReview {
    return &authorizationv1.SelfSubjectAccessReview{
        Spec: authorizationv1.SelfSubjectAccessReviewSpec{
            ResourceAttributes: &authorizationv1.ResourceAttributes{
                Namespace:   ns,
                Group:       res.Group,
                Version:     res.Version,
                Resource:    res.Resource,
                Subresource: gvr.SubResource(),
                Name:        name,
            },
        },
    }
}
```

### 3.3 Factory.CanForResource - Informer 级预检

位于 [factory.go#L202-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go#L202-L222)

**作用：** 在创建 Informer 之前检查权限

```go
func (f *Factory) CanForResource(ns string, gvr *GVR, verbs []string) (informers.GenericInformer, error) {
    // 1. 调用 APIClient.CanI 检查权限
    auth, err := f.Client().CanI(ns, gvr, resName, verbs)
    if err != nil {
        return nil, err
    }
    if !auth {
        return nil, fmt.Errorf("%v access denied on resource %q:%q", verbs, ns, gvr)
    }
    // 2. 权限通过后才创建/获取 Informer
    return f.ForResource(ns, gvr)
}
```

### 3.4 Factory.CanForInstance - 实例级预检

位于 [factory.go#L224-L248](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/watch/factory.go#L224-L248)

**作用：** 检查单个资源实例的访问权限

**特殊处理：** 对 Namespace 资源，用资源名作为 RBAC 检查的命名空间

### 3.5 预定义权限组

位于 [types.go#L57-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/client/types.go#L57-L72)

| 权限组 | 包含动词 | 用途 |
|--------|----------|------|
| PatchAccess | patch | 编辑资源 |
| GetAccess | get | 读取单个资源 |
| ListAccess | list | 列出资源 |
| MonitorAccess | list, watch | 监控资源 |
| ReadAllAccess | get, list, watch | 全部读权限 |

---

## 四、权限不足时的展示策略

### 4.1 视图初始化时的权限检查

位于 [browser.go#L89-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L89-L94)

```go
if dao.IsK8sMeta(b.meta) && b.app.ConOK() {
    if _, e := b.app.factory.CanForResource(ns, b.GVR(), client.ListAccess); e != nil {
        return e  // 初始化失败，返回错误
    }
}
```

**行为：** 初始化时如果没有 list 权限，视图无法打开，错误向上传播。

### 4.2 命名空间切换时的权限检查

位于 [browser.go#L560-L596](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L560-L596)

```go
auth, err := b.App().factory.Client().CanI(ns, b.GVR(), "", client.ListAccess)
if !auth {
    if err == nil {
        err = fmt.Errorf("access denied for user on: %s/%s", ns, b.GVR())
    }
    b.App().Flash().Err(err)  // Flash 红色错误提示
    return nil
}
```

**展示策略：**

- 检查不通过时不切换命名空间

- 通过 Flash 消息展示红色错误提示

### 4.3 编辑操作前的权限检查

位于 [browser.go#L533-L558](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L533-L558)

```go
if ok, err := app.Conn().CanI(ns, gvr, n, client.PatchAccess); !ok || err != nil {
    return fmt.Errorf("current user can't edit resource %s", gvr)
}
```

**展示策略：** 编辑前检查 patch 权限，无权限则不打开编辑器，通过 Flash 展示错误。

### 4.4 动作按钮的可见性控制

位于 [browser.go#L627-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L627-L676)

```go
if !b.app.Config.IsReadOnly() {
    if client.Can(b.meta.Verbs, "edit") {
        aa.Add(ui.KeyE, ui.NewKeyActionWithOpts("Edit", ...))
    }
    if client.Can(b.meta.Verbs, "delete") {
        aa.Add(tcell.KeyCtrlD, ui.NewKeyActionWithOpts("Delete", ...))
    }
}
```

**策略：**

- 根据资源支持的动词 (从 API 发现的 verbs 显示/隐藏操作按钮

- 只读模式下隐藏所有危险操作

- 基于 `b.meta.Verbs` 判断资源支持哪些操作

### 4.5 无数据时的展示策略

位于 [browser.go#L298-L335](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/view/browser.go#L298-L335)

**TableNoData 处理：**

1. **首次视图**：不显示警告（初始化中）

2. **缓存未同步**：显示 "Synchronizing..." 中性提示

3. **同步后无数据**：显示 "No resources found" 警告

4. **权限不足导致加载失败**：TableLoadFailed 显示红色错误

### 4.6 权限错误展示层级总结

| 场景 | 展示方式 | 颜色 |
|------|----------|------|
| 视图初始化失败 | 返回错误，视图打不开 | - |
| 命名空间切换失败 | Flash 错误消息 | 红色 |
| 编辑/删除无权限 | Flash 错误消息 | 红色 |
| 资源加载失败 | Flash 错误消息 | 红色 |
| Informer 无权限 | Flash 错误消息 | 红色 |
| 无数据 | Flash 警告消息 | 黄色/橙色 |
| 缓存同步中 | Flash 信息消息 | 中性 |

---

## 五、RBAC 视图渲染

### 5.1 动词图标展示

位于 [rbac.go#L79-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/rbac.go#L79-L101)

```go
func asVerbs(verbs []string) []string {
    // 标准8个动词：get, list, watch, create, patch, update, delete, deletecollection
    // 有权限：绿色 ✓
    // 无权限：橙红色 ×
    // 额外动词：显示在 EXTRAS 列
}
```

### 5.2 动词映射

位于 [rbac.go#L17-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/18-k9s/internal/render/rbac.go#L17-L33)

- HTTP 动词到 K8s 动词的映射：
  - `post` → `create`
  - `put` → `update`

- `*` 表示所有权限

### 5.3 显示列结构

**Rbac 视图列：**
```
NAME | API-GROUP | GET | LIST | WATCH | CREATE | PATCH | UPDATE | DELETE | DEL-LIST | EXTRAS | VALID
```

**Policy 视图列：**
```
NAMESPACE | NAME | API-GROUP | BINDING | GET | LIST | ... | VALID
```

---

## 六、关键设计要点

### 6.1 权限聚合的设计

1. **以 GR (Group/Resource) 为聚合键**：相同资源的权限会被合并到一起

2. **动词去重合并**：多个角色授予同一资源的相同动词会被合并

3. **Binding 字段保留来源信息**：Policy 视图中显示权限来自哪个角色

### 6.2 缓存设计

1. **LRU + 过期**：兼顾性能和时效性

2. **分层检查**：Factory 层和 Client 层都有缓存逻辑

3. **连接断开时清缓存**：保证权限变化后重新检查

### 6.3 渐进式权限检查

1. **视图初始化检查**：确保有基本的 list 权限才能打开视图

2. **操作前检查**：编辑/删除等危险操作前再次检查

3. **命名空间切换检查**：切换命名空间时重新检查该命名空间的权限

### 6.4 主体匹配的精确性

1. **ServiceAccount 命名空间校验**：防止同名不同命名空间的 SA 权限混淆

2. **ClusterRole 与 Role 区分**：正确处理 RoleBinding 引用 ClusterRole 的情况

3. **集群角色绑定与命名空间角色绑定**：两种绑定都要考虑
