<!-- TRANSLATED by md-translate -->
# 任意名称空间中的 ApplicationSet

**当前功能状态**：测试版

警告 在启用此功能之前，请仔细阅读本文档。 配置错误可能会导致潜在的安全问题。

## 简介

从 2.8 版开始，Argo CD 支持在控制平面名称空间（通常是 `argocd`）以外的名称空间中管理 `ApplicationSet` 资源，但必须显式启用此功能并进行适当配置。

Argo CD 管理员可以定义一组特定的 namespace，在这些 namespace 中可以创建、更新和调节 `ApplicationSet` 资源。

由于 ApplicationSet 生成的应用程序与 ApplicationSet 本身在同一命名空间中生成，因此可与 [App in any namespace](../app-any-namespace.md) 结合使用。

## 先决条件

#### 应用程序在任何 namespace 中配置

此功能需要激活 [App in any namespace](../app-any-namespace.md) 功能。 命名空间列表必须相同。

### 集群范围的 Argo CD 安装

该功能只有在 Argo CD ApplicationSet 控制器作为集群范围的实例安装时才能启用和使用，因此它拥有在集群范围内列出和操作资源的权限。 如果 Argo CD 以名称空间范围模式安装，则无法使用该功能。

### SCM Providers secrets consideration

如果允许在任何名称空间中使用 ApplicationSet，就必须注意任何秘密都可能被`scmProvider`或`pullRequest`生成器外泄。

下面就是一个例子：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - scmProvider:
      gitea:
        # The Gitea owner to scan.
        owner: myorg
        # With this malicious setting, user can send all request to a Pod that will log incoming requests including headers with tokens
        api: http://my-service.my-namespace.svc.cluster.local
        # If true, scan every branch of every repository. If false, scan only the default branch. Defaults to false.
        allBranches: true
        # By changing this token reference, user can exfiltrate any secrets
        tokenRef:
          secretName: gitea-token
          key: token
  template:
```

因此，管理员必须通过将环境变量 `ARGOCD_APPLICATIONSET_CONTROLLER_ALLOWED_SCM_PROVIDERS` 设置为 argocd-cmd-params-cm `applicationsetcontroller.allowed.scm.providers` 来限制允许的 SCM Providers 的 url（例如：`https://git.mydomain.com/,https://gitlab.mydomain.com/`）。如果使用其他 url，则会被 ApplicationSet 控制器引用。

例如

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  applicationsetcontroller.allowed.scm.providers: https://git.mydomain.com/,https://gitlab.mydomain.com/
```

注意 请注意，"ApplicationSet "的 "api "字段中使用的 url 必须与管理员声明式的 url（包括协议）一致。

允许列表仅适用于用户可以配置自定义 `api` 的 SCM Provider。 如果 SCM 或 PR 生成器不接受自定义 API URL，则隐式允许该 Provider。

如果不打算允许用户使用 SCM 或 PR 生成器，可以通过将环境变量 `ARGOCD_APPLICATIONSET_CONTROLLER_ALLOW_SCM_PROVIDERS` 设为 argocd-cmd-params-cm `applicationsetcontroller.allow.scm.Provider` 设为 `false` 来完全禁用它们。

#### 概览

要在 Argo CD 控制平面名称空间之外管理和调节 ApplicationSet，必须满足两个前提条件：

1.必须使用环境变量 `ARGOCD_APPLICATIONSET_CONTROLLER_NAMESPACES` 或参数 `--applicationset-namespaces` 来显式设置`argocd-applicationset-controller`可以从中获取`ApplicationSets`的命名空间列表。
2.启用的名称空间必须完全包含在 [App in any namespace](../app-any-namespace.md) 中，否则在允许的应用程序名称空间之外生成的应用程序将无法对账。

可以通过将环境变量 `ARGOCD_APPLICATIONSET_CONTROLLER_NAMESPACES` 设置为 argocd-cmd-params-cm `applicationsetcontroller.namespace` 来实现。

不同命名空间中的 `ApplicationSet` 可以像之前 `argocd` 命名空间中的其他 `ApplicationSet` 一样，通过声明式或 Argo CD API（例如使用 CLI、Web UI、REST API 等）来创建和管理。

#### 重新配置 Argo CD 以允许使用某些 namespace

#### 更改工作负载启动参数

要启用此功能，Argo CD 管理员必须重新配置和 `argocd-applicationset-controller` 工作负载，在容器的启动命令中添加 `--applicationset-namespace` 参数。

#### 安全模板项目

由于 [App in any namespace](../app-any-namespace.md) 是前提条件，因此可以安全地进行模板项目。

让我们以两个团队和一个基础设施项目为例：

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: infra-project
  namespace: argocd
spec:
  destinations:
    - namespace: '*'
```

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: team-one-project
  namespace: argocd
spec:
  sourceNamespaces:
  - team-one-cd
```

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: team-two-project
  namespace: argocd
spec:
  sourceNamespaces:
  - team-two-cd
```

创建以下 `ApplicationSet` 会生成两个应用程序 `infra-escalation` 和 `team-two-escalation`。 这两个应用程序都将被拒绝，因为它们在 `argocd` 名称空间之外，因此将检查 `sourceNamespaces

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-one-product-one
  namespace: team-one-cd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    list:
    - name: infra
      project: infra-project
    - name: team-two
      project: team-two-project
  template:
    metadata:
      name: '{{.name}}-escalation'
    spec:
      project: "{{.project}}"
```

###应用程序集名称

对于 CLI，ApplicationSet 现在以 `<namespace>/<name>` 的格式引用和显示。

为实现向后兼容性，如果 ApplicationSet 的命名空间是控制平面的命名空间（即 `argocd`），则在引用应用程序集名称时，可以省略应用程序集名称中的 `<namespace>`。例如，应用程序名称 `argocd/someappset` 和 `someappset` 在语义上是相同的，在 CLI 和 UI 中引用的是同一个应用程序。

###Applicationsets RBAC

应用程序对象的 RBAC 语法已从 `<project>/<applicationset>` 改为 `<project>/<namespace>/<applicationset>`，以适应根据要管理的应用程序的源名称空间限制访问的需要。

为了向后兼容，argocd 名称空间中的应用程序在 RBAC 策略规则中仍可称为 `<project>/<applicationset>`。

通配符尚未区分项目和 ApplicationSet 名称空间。 例如，下面的 RBAC 规则将匹配属于项目 foo 的任何应用程序，而不管它是在哪个名称空间创建的：

```
p, somerole, applicationsets, get, foo/*, allow
```

如果要限制只对名称空间 `bar` 中项目 `foo` 的 `ApplicationSet` 进行访问，则需要对规则作如下调整：

```
p, somerole, applicationsets, get, foo/bar/*, allow
```

## 在其他 namespace 中管理 ApplicationSet

### 使用 CLI

您可以使用所有现有的 Argo CD CLI 命令来管理其他 namespace 中的应用程序，就像使用 CLI 管理控制平面 namespace 中的应用程序一样。

例如，要检索名称空间 `bar` 中名为 `foo` 的 `ApplicationSet` ，可以使用以下 CLI 命令：

```shell
argocd appset get foo/bar
```

同样，为了管理这个 ApplicationSet，请继续将其称为 `foo/bar`：

```bash
# Delete the application
argocd appset delete foo/bar
```

创建命令没有任何变化，因为它被引用的是一个文件。 你只需在 `metadata.namespace` 字段中添加 namespace。

如前所述，对于 Argo CD 控制平面名称空间中的 ApplicationSet，可以在应用程序名称中省略名称空间。

### 使用 REST API

如果使用的是 REST API，则不能将 `ApplicationSet` 的命名空间指定为应用程序名称，需要使用可选的 `appNamespace` 查询参数指定资源。 例如，要处理命名空间 `bar` 中名为 `foo` 的 `ApplicationSet` 资源，请求如下：

```bash
GET /api/v1/applicationsets/foo?appsetNamespace=bar
```

对于其他操作，如 `POST` 和 `PUT`，`appNamespace` 参数必须是请求有效载荷的一部分。

对于控制平面命名空间中的 `ApplicationSet` 资源，此参数可以省略。

## 考虑的集群秘密

允许在任何命名空间中使用 ApplicationSet，就必须意识到集群可以被发现和引用。

例如

以下将发现所有集群

```yaml
spec:
  generators:
  - clusters: {} # Automatically use all clusters defined within Argo CD
```

如果不想让用户从其他命名空间发现所有带有 ApplicationSet 的集群，可以考虑在命名空间范围内部署 ArgoCD 或使用 OPA 规则。