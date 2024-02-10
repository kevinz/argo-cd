<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

## 项目

项目提供了应用程序的逻辑分组，这在 Argo CD 被多个团队引用时非常有用。

限制可部署的内容（受信任的 Git 源代码库） * 限制应用程序可部署的位置（目标集群和名称空间） * 限制可部署或不可部署的对象类型（如 RBAC、CRD、DaemonSets、NetworkPolicy 等......） * 定义项目角色以提供应用程序 RBAC（与 OIDC 组和/或 JWT 标记绑定） * 限制可部署或不可部署的对象类型（如 RBAC、CRD、DaemonSets、NetworkPolicy 等......

### 默认项目

每个应用程序都属于一个项目。 如果未指定，则应用程序属于`default`项目是自动创建的，默认情况下允许从任何源代码 repo 部署到任何集群和所有资源类型。 默认项目可以修改，但不能删除。 初始创建时，其规范被配置为最具许可性：。

```yaml
spec:
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```

### 创建项目

可以创建其他项目，让不同的团队有不同级别的 namespace 访问权限。 以下命令创建了一个新项目`myproject`可将应用程序部署到 namespace`mynamespace`集群`https://kubernetes.default.svc`允许的 Git 源代码库设置为`https://github.com/argoproj/argocd-example-apps.git`存放处。

```bash
argocd proj create myproject -d https://kubernetes.default.svc,mynamespace -s https://github.com/argoproj/argocd-example-apps.git
```

### 管理项目

允许的源 Git 仓库被引用命令进行管理：

```bash
argocd proj add-source <PROJECT> <REPO>
argocd proj remove-source <PROJECT> <REPO>
```

我们还可以对源代码进行否定（即执行不是被引用）。

```bash
argocd proj add-source <PROJECT> !<REPO>
argocd proj remove-source <PROJECT> !<REPO>
```

从声明的角度来说，我们可以这样做

```yaml
spec:
  sourceRepos:
    # Do not use the test repo in argoproj
    - '!ssh://git@GITHUB.com:argoproj/test'
    # Nor any Gitlab repo under group/ 
    - '!https://gitlab.com/group/**'
    # Any other repo is fine though
    - '*'
```

如果符合以下条件，则源资源库被视为有效：

1._Any_ 允许源规则（即不带`!`前缀的规则）允许源 2.AND _no_ 拒绝源规则（即不带`！`前缀的规则）拒绝源

请记住`！*`是一条无效规则，因为禁止一切规则是没有意义的。

允许的目标集群和名称空间通过命令进行管理（对于集群始终提供服务器，名称不被引用用于匹配）：

```bash
argocd proj add-destination <PROJECT> <CLUSTER>,<NAMESPACE>
argocd proj remove-destination <PROJECT> <CLUSTER>,<NAMESPACE>
```

与来源一样，我们也可以对目的地进行否定（即在任何地方安装除了___________________________之外).

```bash
argocd proj add-destination <PROJECT> !<CLUSTER>,!<NAMESPACE>
argocd proj remove-destination <PROJECT> !<CLUSTER>,!<NAMESPACE>
```

从声明的角度来说，我们可以这样做

```yaml
spec:
  destinations:
  # Do not allow any app to be installed in `kube-system`  
  - namespace: '!kube-system'
    server: '*'
  # Or any cluster that has a URL of `team1-*`   
  - namespace: '*'
    server: '!https://team1-*'
    # Any other namespace or server is fine though.
  - namespace: '*'
    server: '*'
```

与信源一样，如果满足以下条件，则认为目的地有效：

1._Any_ 允许目的地规则（即不带`!`前缀的规则）允许目的地 2.AND _no_ 拒绝目的地规则（即不带`!`前缀的规则）拒绝目的地

请记住`！*`是一条无效规则，因为禁止一切规则是没有意义的。

允许的目标 k8s 资源类型通过命令进行管理。 请注意，namespace-scoped 资源通过 deny 列表进行限制，而集群-scoped 资源则通过 allow 列表进行限制。

```bash
argocd proj allow-cluster-resource <PROJECT> <GROUP> <KIND>
argocd proj allow-namespace-resource <PROJECT> <GROUP> <KIND>
argocd proj deny-cluster-resource <PROJECT> <GROUP> <KIND>
argocd proj deny-namespace-resource <PROJECT> <GROUP> <KIND>
```

### 将应用程序分配给一个项目

可通过以下方式更改应用程序项目`app set`要更改应用程序的项目，用户必须拥有访问新项目的权限。

```
argocd app set guestbook-default --project myproject
```

## 项目角色

这些角色可被引用来为 Ci 管道提供一组受限的权限。 例如，Ci 系统可能只能同步单个应用程序（但不能更改其源或目标）。

这些权限被称为策略，它们以策略字符串列表的形式存储在角色中。 角色的策略只能授予该角色访问权限，并且仅限于该角色项目中的应用程序。

要在项目中创建角色并为角色添加策略，用户需要有更新项目的权限。 以下命令可用于管理角色。

```bash
argocd proj role list
argocd proj role get
argocd proj role create
argocd proj role delete
argocd proj role add-policy
argocd proj role remove-policy
```

如果不生成与角色相关联的令牌，项目角色本身就没有用处。 Argo CD 支持将 JWT 令牌作为验证角色的手段。 由于 JWT 令牌与角色的策略相关联，因此对角色策略的任何更改都会立即对该 JWT 令牌生效。

以下命令被用来管理 jwt 标记。

```bash
argocd proj role create-token PROJECT ROLE-NAME
argocd proj role delete-token PROJECT ROLE-NAME ISSUED-AT
```

由于 JWT 标记没有存储在 Argo CD 中，因此只能在创建时才能引用。 用户可以通过使用`--AUTH-token`标记或设置 ARGOCD_AUTH_TOKEN 环境变量。 JWT 令牌可以一直使用，直到过期或被引用。

下面是一个利用 JWT 令牌访问留言簿应用程序的示例，假定用户已经拥有一个名为 myproject 的项目和一个名为 guestbook-default 的应用程序。

```bash
PROJ=myproject
APP=guestbook-default
ROLE=get-role
argocd proj role create $PROJ $ROLE
argocd proj role create-token $PROJ $ROLE -e 10m
JWT=<value from command above>
argocd proj role list $PROJ
argocd proj role get $PROJ $ROLE

# This command will fail because the JWT Token associated with the project role does not have a policy to allow access to the application
argocd app get $APP --auth-token $JWT
# Adding a policy to grant access to the application for the new role
argocd proj role add-policy $PROJ $ROLE --action get --permission allow --object $APP
argocd app get $APP --auth-token $JWT

# Removing the policy we added and adding one with a wildcard.
argocd proj role remove-policy $PROJ $ROLE -a get -o $APP
argocd proj role add-policy $PROJ $ROLE -a get --permission allow -o '*'
# The wildcard allows us to access the application due to the wildcard.
argocd app get $APP --auth-token $JWT
argocd proj role get $PROJ $ROLE

argocd proj role get $PROJ $ROLE
# Revoking the JWT token
argocd proj role delete-token $PROJ $ROLE <id field from the last command>
# This will fail since the JWT Token was deleted for the project role.
argocd app get $APP --auth-token $JWT
```

## 通过项目配置 RBAC

项目角色允许配置项目范围内的 RBAC 规则。 以下示例项目向 Providers 的任何成员提供项目应用程序的只读权限`my-oidc-group`组。

应用项目示例：_

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  roles:
  # A role which provides read-only access to all applications in the project
  - name: read-only
    description: Read-only privileges to my-project
    policies:
    - p, proj:my-project:read-only, applications, get, my-project/*, allow
    groups:
    - my-oidc-group
```

您可以被引用`argocd proj role`CLI 命令或用户界面中的项目详细信息页面来配置策略。 请注意，每个项目角色策略规则的作用域必须仅限于该项目。 使用`argocd-rbac-`cm 中描述的 ConfigMap[RBAC] (./operator-manual/rbac.md)如果要配置跨项目 RBAC 规则，请参阅文档。

## 配置全局项目（v1.8）

可以对全局项目进行配置，以提供其他项目可以继承的配置。

项目，这些项目与`matchExpressions`中规定的`argocd-cm`ConfigMap，从全局项目中继承下字段：

* 名称空间资源黑名单 * 名称空间资源白名单 * 集群资源黑名单 * 集群资源白名单 * 同步窗口 * 源重置 * 目的地

在`argocd-cm`配置地图：

```yaml
data:
  globalProjects: |-
    - labelSelector:
        matchExpressions:
          - key: opt
            operator: In
            values:
              - prod
      projectName: proj-global-test
kind: ConfigMap
```

可被引用的有效运算符有：In、NotIn、Exists、DoesNotExist、Gt 和 Lt。

项目名称：`proj-global-test`应替换为您自己的全局项目名称。

## 项目范围内的存储库和集群

通常情况下，Argo CD 管理员在创建项目时会事先决定定义哪些集群和 Git 仓库。 然而，如果开发人员在创建项目后想要添加仓库或集群，这就会造成问题。 开发人员不得不再次联系 Argo CD 管理员更新项目定义。

可以为开发人员提供自助服务流程，这样他们即使在最初创建项目后，也可以自行在项目中添加存储库和/或集群。

为此，Argo CD 支持项目范围内的存储库和集群。

要开始这一过程，Argo CD 管理员必须配置 RBAC 安全性，以允许这种自助行为。 例如，要允许用户添加项目范围内的存储库，管理员必须添加以下 RBAC 规则： Argo CD 管理员必须配置 RBAC 安全性，以允许这种自助行为。

```
p, proj:my-project:admin, repositories, create, my-project/*, allow
p, proj:my-project:admin, repositories, delete, my-project/*, allow
p, proj:my-project:admin, repositories, update, my-project/*, allow
```

这提供了额外的灵活性，使管理员可以制定更严格的规则：

```
p, proj:my-project:admin, repositories, update, my-project/https://github.example.com/*, allow
```

如果指定了一个项目，那么相应的集群/仓库就会被视为项目范围： 用户界面和 CLI 都可以选择指定一个项目。

`argocd repo add --name stable https://charts.helm.sh/stable --type helm chart --project my-project`

对于声明式设置，存储库和集群都存储为 Kubernetes Secrets，因此会使用一个新字段来表示此资源是项目作用域的： Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-example-apps
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  project: my-project1                                     # Project scoped 
  name: argocd-example-apps
  url: https://github.com/argoproj/argocd-example-apps.git
  username: ****
  password: ****
```

上面的例子都是关于 Git 仓库的，但同样的原则也适用于集群。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: mycluster.example.com
  project: my-project1 # Project scoped 
  server: https://mycluster.example.com
  config: |
    {
      "bearerToken": "<authentication token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64 encoded certificate>"
      }
    }
```

有了项目范围的集群，我们还可以限制项目只允许目的地属于同一项目的应用程序。 默认行为允许将应用程序安装到不属于同一项目的集群上，如下例所示： 项目范围的集群

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: "some-ns"
spec:
  destination:
    # This destination might not actually be a cluster which belongs to `foo-project`
    server: https://some-k8s-server/
    namespace: "some-ns"
  project: foo-project
```

为了防止这种行为，我们可以设置属性`permitOnlyProjectScopedClusters`项目。

```yaml
spec:
  permitOnlyProjectScopedClusters: true
```

有了这一设置，上述应用程序将不再被允许同步到同一项目中的集群以外的任何集群。