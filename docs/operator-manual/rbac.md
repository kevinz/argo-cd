<!-- TRANSLATED by md-translate -->
# RBAC 配置

RBAC 功能可以限制对 Argo CD 资源的访问。 Argo CD 没有自己的用户管理系统，只有一个内置用户 "admin"。 "admin "用户是超级用户，它对系统的访问不受限制。 RBAC 需要[SSO 配置](user-management/index.md) 或[一个或多个本地用户设置](user-management/index.md)。 一旦配置了 SSO 或本地用户，就可以定义其他 RBAC 角色，然后将 SSO 组或本地用户映射到角色。

## 基本内置角色

Argo CD 有两个预定义角色，但 RBAC 配置允许定义角色和组（见下文）。

* 角色:只读"- 对所有资源的只读访问权限
* 角色:admin`-不受限制地访问所有资源

这些默认内置角色定义可以在 [builtin-policy.csv](https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv) 中看到

### RBAC 权限结构

在 Argo CD 中，应用程序和其他资源类型的权限定义略有不同。

* 除应用程序特定权限外的所有资源（请参阅下一条）：
    p、&lt;角色/用户/组&gt;、<resource>,<action>,<object>`
* 应用程序、应用程序集、logging 和 exec（属于`AppProject`）：
    P、&lt;角色/用户/组&gt;、<resource>,<action>,<appproject>/<object>`

#### RBAC 资源和操作

资源："集群"、"项目"、"应用程序"、"应用程序集"、"存储库"、"证书"、"账户"、"gpgkeys"、"日志"、"执行"、"扩展

操作：`get`、`create`、`update`、`delete`、`sync`、`override`、`action/&lt;group/kind/action-name&gt;`。

请注意，"sync"、"override "和 "action/&lt;group/kind/action-name&gt;"仅对 "applications "资源有意义。

#### 应用资源

应用程序对象的资源路径格式为 `<project-name>/<application-name>`。

删除对项目子资源（如 rollout 或 pod）的访问权限无法进行细粒度管理。`<project-name>/<application-name>` 授予对应用程序所有子资源的访问权限。

#### `action`行动

action "动作对应于[Argo CD 资源库中](https://github.com/argoproj/argo-cd/tree/master/resource_customizations)定义的内置资源定制，或您定义的[自定义资源动作](resource_actions.md#custom-resource-actions)。"action "路径的形式为 "action/<api-group>/<Kind>/<action-name>"。例如，资源定制路径 "resource_customizations/extensions/DaemonSet/actions/restart/action.lua "对应于 "action "路径 "action/extensions/DaemonSet/restart"。您也可以在action路径中被引用glob模式："action/*"（或 "regex "模式，如果您已[启用 "regex "匹配模式](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-rbac-cm.yaml)）。

如果资源不属于某个组（例如 Pod 或 ConfigMaps），则在 RBAC 配置中省略组名：

```csv
p, example-user, applications, action//Pod/maintenance-off, default/*, allow
```

#### `exec` 资源

exec "是一种特殊资源。 当使用 "create "操作时，该权限允许用户通过 Argo CD 用户界面 "exec "进入 Pod。 其功能类似于 "kubectl exec"。

更多信息请参阅 [基于网络的终端](web_based_terminal.md)。

#### "Applicationsets "资源

[ApplicationSet](applicationset/index.md)提供了一种声明式的自动创建/更新/删除应用程序的方法。

授予 "Applicationsets, create "有效地授予了创建应用程序的能力。 虽然它不允许用户直接创建应用程序，但用户可以通过 ApplicationSet 创建应用程序。

在 v2.5，无法通过 API（或 CLI）创建带有模板项目字段（如`project: {{path.basename}}`）的 ApplicationSet。 禁止模板项目可确保通过 RBAC 进行的项目限制安全：

```csv
p, dev-group, applicationsets, *, dev-project/*, allow
```

有了这条规则，"dev-group "用户将无法创建能够在 "dev-project "项目之外创建应用程序的 ApplicationSet。

#### `扩展名`资源

通过 "扩展 "资源，可以配置调用[代理扩展]的权限（.../developer-guide/extensions/proxy-extensions.md）。 扩展 "RBAC 验证与 "应用 "资源协同工作。 登录 Argo CD（用户界面或 CLI）的用户至少需要拥有请求所属项目、名称空间和应用的读取权限。

请看下面的例子：

```csv
g, ext, role:extension
p, role:extension, applications, get, default/httpbin-app, allow
p, role:extension, extensions, invoke, httpbin, allow
```

解释：

* _line1_:定义了与主题 `ext` 相关联的组 `role:extension` 。
* _line2_：定义了一个策略，允许此角色读取（`获取`）`default`项目中的`httpbin-app`应用程序。
* _line3_：定义了另一条策略，允许该角色 "调用"`httpbin`扩展。

**注意 1**：要允许扩展请求，还需要_line2_中定义的策略。

**注 2**："invoke "是专门为与 "扩展名 "资源一起使用而引入的新操作。 目前 "扩展名 "的操作是 "*"或 "invoke"。

## Tying It All Together

可以在 `argocd-rbac-cm` 配置表中配置其他角色和组。 下面的示例配置了一个名为 `org-admin` 的自定义角色。 该角色会分配给属于 `your-github-org:your-team` 组的任何用户。 所有其他用户都会获得默认的 `role:readonly` 策略，不能修改 Argo CD 设置。

警告 所有已通过身份验证的用户至少可获得默认策略授予的权限。 不能通过 "拒绝 "规则阻止这种访问。 相反，应限制默认策略，然后根据需要向个别角色授予权限。

_ArgoCD 配置地图 `argocd-rbac-cm` 示例：_

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, get, *, allow
    p, role:org-admin, repositories, get, *, allow
    p, role:org-admin, repositories, create, *, allow
    p, role:org-admin, repositories, update, *, allow
    p, role:org-admin, repositories, delete, *, allow
    p, role:org-admin, projects, get, *, allow
    p, role:org-admin, projects, create, *, allow
    p, role:org-admin, projects, update, *, allow
    p, role:org-admin, projects, delete, *, allow
    p, role:org-admin, logs, get, *, allow
    p, role:org-admin, exec, create, */*, allow

    g, your-github-org:your-team, role:org-admin
```

---

另一个 `policy.csv` 示例可能如下：

```csv
p, role:staging-db-admin, applications, create, staging-db-project/*, allow
p, role:staging-db-admin, applications, delete, staging-db-project/*, allow
p, role:staging-db-admin, applications, get, staging-db-project/*, allow
p, role:staging-db-admin, applications, override, staging-db-project/*, allow
p, role:staging-db-admin, applications, sync, staging-db-project/*, allow
p, role:staging-db-admin, applications, update, staging-db-project/*, allow
p, role:staging-db-admin, logs, get, staging-db-project/*, allow
p, role:staging-db-admin, exec, create, staging-db-project/*, allow
p, role:staging-db-admin, projects, get, staging-db-project, allow
g, db-admins, role:staging-db-admin
```

本示例定义了一个名为 "staging-db-admin "的_角色，该角色有九个_权限，允许拥有该角色的用户执行以下_操作：

* 为 "staging-db-project "项目中的应用程序提供 "创建"、"删除"、"获取"、"覆盖"、"同步 "和 "更新 "功能、
* 为`staging-db-project`项目中的对象生成`get`logging、
* 为 "staging-db-project "项目中的对象创建 "执行"，以及
* 获取`staging-db-project`项目的日志。

注意 `scopes`字段控制在执行 rbac 时要检查哪些 OIDC 作用域（除了 `sub` 作用域）。 如果省略，默认为：`'[组]'`。 作用域值可以是字符串，也可以是字符串列表。

下面的示例显示了从 OIDC Provider 针对 "电子邮件 "和 "群组 "的操作。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
data:
  policy.csv: |
    p, my-org:team-alpha, applications, sync, my-project/*, allow
    g, my-org:team-beta, role:admin
    g, user@example.org, role:admin
  policy.default: role:readonly
  scopes: '[groups, email]'
```

有关 "范围 "的更多信息，请查阅[用户管理文档](user-management/index.md)。

## 政策 CSV 组成

可以在 `argocd-rbac-cm` configmaps 中提供附加条目，以组成最终的策略 csv。在这种情况下，密钥必须遵循 `policy.<any string>.csv`。Argo CD 会将找到的所有附加策略按此模式连接到主策略（"policy.csv"）下面。 附加策略的顺序由密钥字符串决定。 例如：如果提供了两个附加策略，密钥分别为 `policy.A.csv` 和 `policy.B.csv`，则会首先连接 `policy.A.csv` 然后连接 `policy.B.csv`。

这对于在 kustomize、Helm 等配置管理工具中合成策略非常有用。

下面的示例展示了如何在叠加中 Provider 补丁，为现有 RBAC 策略添加额外配置。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.tester-overlay.csv: |
    p, role:tester, applications, *, */*, allow
    p, role:tester, projects, *, *, allow
    g, my-org:team-qa, role:tester
```

## 匿名访问

可以使用 `argocd-cm` 中的`users.anonymous.enabled`字段启用对 Argo CD 的匿名访问（参见 [argocd-cm.yaml](argocd-cm.yaml)）。 匿名用户获得由 `argocd-rbac-cm.yaml` 中的`policy.default`指定的默认角色权限。 若要实现只读访问，则需要如上所示使用`policy.default: role:readonly` 。

## 验证和测试你的 RBAC 策略

如果要确保 RBAC 策略按预期运行，可以使用 "argocd admin settings rbac "命令对其进行验证。 通过该工具，可以测试特定角色或主体是否可以使用系统中尚未生效的策略（即来自本地文件或 config maps 的策略）执行所请求的操作。 此外，还可以针对 Argo CD 所运行集群中的生效策略使用该工具。

要检查新策略是否有效并被 Argo CD 的 RBAC 实现所理解，可以引用 `argocd admin settings rbac validate` 命令。

### 验证政策

验证存储在本地文本文件中的策略：

```shell
argocd admin settings rbac validate --policy-file somepolicy.csv
```

要验证 YAML 文件中存储在本地 k8s ConfigMap 定义中的策略：

```shell
argocd admin settings rbac validate --policy-file argocd-rbac-cm.yaml
```

要验证存储在 k8s 中、被 Argo CD 引用在 namespace `argocd` 中的策略，请确保 `~/.kube/config` 中的当前上下文指向 Argo CD 集群，并给出适当的 namespace：

```shell
argocd admin settings rbac validate --namespace argocd
```

### 测试政策

要测试某个角色或主体（组或本地用户）是否有足够的权限在某些资源上执行某些操作，可以使用 `argocd admin settings rbac can` 命令。 它的一般语法是

```shell
argocd admin settings rbac can SOMEROLE ACTION RESOURCE SUBRESOURCE [flags]
```

鉴于上述配置表中的示例定义了角色 `role:org-admin` 并以 `argocd-rbac-cm-yaml` 的形式存储在本地系统中，您可以测试该角色是否可以执行如下操作：

```console
$ argocd admin settings rbac can role:org-admin get applications --policy-file argocd-rbac-cm.yaml
Yes

$ argocd admin settings rbac can role:org-admin get clusters --policy-file argocd-rbac-cm.yaml
Yes

$ argocd admin settings rbac can role:org-admin create clusters 'somecluster' --policy-file argocd-rbac-cm.yaml
No

$ argocd admin settings rbac can role:org-admin create applications 'someproj/someapp' --policy-file argocd-rbac-cm.yaml
Yes
```

另一个例子，给定上面来自 `policy.csv` 的策略，其中定义了角色 `role:staging-db-admin` 并将组 `db-admins` 与之关联。 策略在本地存储为 `policy.csv`：

您可以根据角色进行测试：

```console
$ # Plain policy, without a default role defined
$ argocd admin settings rbac can role:staging-db-admin get applications --policy-file policy.csv
No

$ argocd admin settings rbac can role:staging-db-admin get applications 'staging-db-project/*' --policy-file policy.csv
Yes

$ # Argo CD augments a builtin policy with two roles defined, the default role
$ # being 'role:readonly' - You can include a named default role to use:
$ argocd admin settings rbac can role:staging-db-admin get applications --policy-file policy.csv --default-role role:readonly
Yes
```

或反对所定义的群体：

```console
$ argocd admin settings rbac can db-admins get applications 'staging-db-project/*' --policy-file policy.csv
Yes
```