<!-- TRANSLATED by md-translate -->
# 任何 namespace 中的应用程序

**当前功能状态**：测试版

警告 在启用此功能之前，请仔细阅读本文档。 配置错误可能会导致潜在的安全问题。

## 简介

从 2.5 版开始，Argo CD 支持在控制平面名称空间（通常是 `argocd`）以外的名称空间中管理 `Application` 资源，但必须显式启用此功能并进行适当配置。

Argo CD 管理员可以定义特定的命名空间，在这些命名空间中可以创建、更新和调节 "应用程序 "资源。 但是，这些附加命名空间中的应用程序只能使用 Argo CD 管理员配置的特定 "应用程序项目"。 这样，Argo CD 的普通用户（如应用程序团队）就可以使用声明式管理 "应用程序 "资源、实施应用程序的应用程序等模式，而不会因为使用其他 "应用程序项目 "而产生权限升级的风险，因为这超出了授予应用程序团队的权限。

Argo CD 管理员需要执行一些手动步骤才能启用该功能。

注意：此功能目前还属于测试版。 在升级到稳定版之前，部分实施细节可能会随时间推移而改变。 如果早期用户使用此功能并向我们提供错误报告和反馈，我们将非常高兴。

在任何名称空间中采用应用程序的一个额外优势是允许终端用户在 Argo CD 应用程序所在的名称空间中为其 Argo CD 应用程序配置通知。 更多信息请参见通知 [基于名称空间的配置](notifications/index.md#namespace-based-configuration) 页面。

## 先决条件

### 集群范围的 Argo CD 安装

该功能只有在 Argo CD 作为集群范围实例安装时才能启用和使用，因此它拥有在集群范围内列出和操作资源的权限。 该功能无法在以 namespace-scoped 模式安装的 Argo CD 上使用。

### 切换资源跟踪方法

此外，虽然从技术上讲没有必要，但我们强烈建议您将应用程序跟踪方法从默认的 "标签 "设置切换为 "注解 "或 "注解+标签"。 这样做的原因是，应用程序名称将是名称空间名称和 "应用程序 "名称的合成，这很容易超出标签值的 63 个字符长度限制。 注释的长度限制明显更大。

要启用基于 Annotations 的资源跟踪，请参阅有关[资源跟踪方法]的文档（.../.../user-guide/resource_tracking/）。

## 实施细节

#### 概览

要在 Argo CD 控制平面名称空间之外管理和调节应用程序，必须满足两个先决条件：

1.对于 `argocd-application-controller` 和 `argocd-server` 工作负载，必须使用 `--application-namespaces` 参数显式启用 `Application` 的命名空间。该参数控制 Argo CD 允许从 global 中获取`应用程序`资源的 namespace 列表。任何未在此配置的 namespace 都不能被任何 `AppProject` 引用。
2.应用程序 "的".spec.project "字段引用的 "AppProject "必须在其".spec.sourceNamespaces "字段中列出名称空间。此设置将决定一个 `Application` 是否可以引用某个 `AppProject`。如果一个`应用程序`指定了一个不允许使用的`AppProject`，Argo CD 将拒绝处理该`应用程序`。如上所述，在 `.spec.sourceNamespaces` 字段中配置的任何 namespace 也必须在 global 中启用。

不同命名空间中的 "应用程序 "可以像之前 "argocd "命名空间中的其他 "应用程序 "一样，通过声明式或 Argo CD API（例如使用 CLI、Web UI、REST API 等）来创建和管理。

#### 重新配置 Argo CD 以允许使用某些 namespace

#### 更改工作负载启动参数

为了启用此功能，Argo CD 管理员必须重新配置 `argocd-server` 和 `argocd-application-controller` 工作负载，以便在容器的启动命令中添加 `--application-namespaces` 参数。

application-namespaces "参数是一个以逗号分隔的名称空间列表，其中的 "应用程序 "将被允许进入。 列表中的每个条目都支持 shell 风格的通配符，如 "*"，因此，例如，条目 "app-team-*"将匹配 "app-team-one "和 "app-team-two"。 要启用 Argo CD 所在集群上的所有名称空间，可以直接指定 "*"，即"--application-namespaces=*"。

argocd 服务器 "和 "argocd 应用程序控制器 "的启动参数也可以通过在 "argocd-cmd-params-cm "配置地图中指定 "application.namespaces "设置来方便地设置并保持同步，而不是更改各自工作负载的配置清单。 例如，在 "argocd-cmd-params-cm "配置地图中指定 "application.namespaces "设置：

```yaml
data:
  application.namespaces: app-team-one, app-team-two
```

将允许 `app-team-one` 和 `app-team-two` 命名空间管理 `Application` 资源。 在更改 `argocd-cmd-params-cm` 命名空间后，需要重新启动相应的工作负载：

```bash
kubectl rollout restart -n argocd deployment argocd-server
kubectl rollout restart -n argocd statefulset argocd-application-controller
```

#### 适配 Kubernetes RBAC

我们决定暂时不为 `argocd-server` 工作负载扩展 Kubernetes RBAC。 如果你想让其他 namespace 中的 `Applications` 由 Argo CD API（即 CLI 和 UI）管理，就需要为 `argocd-server` ServiceAccount 扩展 Kubernetes 权限。

我们在`examples/k8s-rbac/argocd-server-applications`目录中提供了适用于这一目的的`ClusterRole`和`ClusterRoleBinding`。 对于默认的 Argo CD 安装（即安装到`argocd`namespace），您可以直接应用它们：

```shell
kubectl apply -k examples/k8s-rbac/argocd-server-applications/
```

`argocd-notifications-controller-rbac-clusterrole.yaml` 和 `argocd-notifications-controller-rbac-clusterrolebinding.yaml` 被引用以支持通知控制器通知所有 namespace 中的应用程序。

注意 在以后的某个时间点，我们可能会将此集群角色作为默认安装配置清单的一部分。

### 允许在 AppProject 中添加 namespace

任何拥有 Kubernetes 访问 Argo CD 控制平面名称空间（"argocd"）权限的用户，尤其是拥有以声明式方式创建或更新 "应用程序 "权限的用户，都应被视为 Argo CD 管理员。

因此，Argo CD 的非特权用户过去无法声明式地创建或管理 "应用程序"。 这些用户只能使用应用程序接口（API），并受 Argo CD RBAC 的限制，该 RBAC 可确保只创建允许的 "应用程序项目 "中的 "应用程序"。

要在 `argocd` 名称空间之外创建一个 `Application`，该 `Application` 的 `.spec.project` 字段中引用的 `AppProject` 必须在其 `.spec.sourceNamespaces` 字段中包含该 `Application` 的名称空间。

例如，考虑以下两个（不完整的）"AppProject "规格：

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: project-one
  namespace: argocd
spec:
  sourceNamespaces:
  - namespace-one
```

和

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: project-two
  namespace: argocd
spec:
  sourceNamespaces:
  - namespace-two
```

如果应用程序要将 `.spec.project` 设置为 `project-one`，则必须在名称空间 `namespace-one` 或 `argocd` 中创建。 同样，如果应用程序要将 `.spec.project` 设置为 `project-two`，则必须在名称空间 `namespace-two` 或 `argocd` 中创建。

如果 "namespace-two "中的应用程序将其".spec.project "设置为 "project-one"，或者 "namespace-one "中的应用程序将其".spec.project "设置为 "project-two"，Argo CD 将认为这违反了权限，并拒绝对应用程序进行对账。

此外，无论 Argo CD RBAC 权限如何，Argo CD API 都将执行这些限制。

AppProject "的".spec.sourceNamespaces "字段是一个列表，可以包含任意数量的 namespace，每个条目都支持 shell 风格的通配符，因此可以允许使用 "team-one-*"等模式的 namespace。

!!! 警告 不要在任何有权限的 AppProject（如`default`项目）的`.spec.sourceNamespaces`字段中添加用户控制的命名空间。 始终确保 AppProject 遵循授予最少所需权限的原则。 切勿在 AppProject 中授予对`argocd`命名空间的访问权限。

注意 为了向后兼容，Argo CD 控制平面名称空间 (`argocd`)中的应用程序可以设置其`.spec.project`字段引用任何 AppProject，而不受 AppProject 的`.spec.sourceNamespaces`字段的限制。

### 应用名称

在 CLI 和用户界面中，应用程序现在以 `<namespace>/<name>` 的格式被提及和显示。

为实现向后兼容性，如果应用程序的命名空间是控制平面的命名空间（即 `argocd`），则在引用应用程序时可省略应用程序名称中的 `<namespace>`。例如，应用程序名称 `argocd/someapp` 和 `someapp` 在语义上是相同的，在 CLI 和 UI 中引用的是同一个应用程序。

### 应用 RBAC

应用程序对象的 RBAC 语法已从 `<project>/<application>` 改为 `<project>/<namespace>/<application>`，以适应根据要管理的应用程序的源名称空间限制访问的需要。

为了向后兼容，在 RBAC 策略规则中，"argocd "名称空间中的应用程序仍可称为 "<project>/<application>"。

通配符尚未区分项目和应用程序名称空间。 例如，以下 RBAC 规则将匹配属于项目 `foo` 的任何应用程序，而不管它是在哪个名称空间创建的：

```
p, somerole, applications, get, foo/*, allow
```

如果要限制只对名称空间 `bar` 中项目 `foo` 的 `Applications` 授予访问权限，则需要对规则作如下调整：

```
p, somerole, applications, get, foo/bar/*, allow
```

## 管理其他 namespace 中的应用程序

#### 声明式

对于声明式管理应用程序，只需在所需名称空间中通过 YAML 或 JSON 配置清单创建应用程序。 确保 `.spec.project` 字段指向允许此名称空间的 AppProject。 例如，以下（不完整的）应用程序配置清单在名称空间 `some-namespace` 中创建了一个应用程序：

```yaml
kind: Application
apiVersion: argoproj.io/v1alpha1
metadata:
  name: some-app
  namespace: some-namespace
spec:
  project: some-project
  # ...
```

然后，项目 `some-project` 需要在允许的源名称空间列表中指定 `some-namespace` ，例如

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
    name: some-project
    namespace: argocd
spec:
    sourceNamespaces:
    - some-namespace
```

### 使用 CLI

您可以使用所有现有的 Argo CD CLI 命令来管理其他 namespace 中的应用程序，就像使用 CLI 管理控制平面 namespace 中的应用程序一样。

例如，要检索名称空间 `bar` 中名为 `foo` 的 `Application` ，可以使用以下 CLI 命令：

```shell
argocd app get foo/bar
```

同样，要管理此应用程序，请继续将其称为 `foo/bar`：

```bash
# Create an application
argocd app create foo/bar ...
# Sync the application
argocd app sync foo/bar
# Delete the application
argocd app delete foo/bar
# Retrieve application's manifest
argocd app manifests foo/bar
```

如前所述，对于 Argo CD 控制平面名称空间中的应用程序，可以从应用程序名称中省略名称空间。

### 使用用户界面

与 CLI 类似，您可以在用户界面中将应用程序称为 `foo/bar`。

例如，要在 Web UI 的名称空间 "foo "中创建名为 "bar "的应用程序，则应将创建对话的 _Application Name_ 字段中的应用程序名称设置为 "foo/bar"。 如果省略名称空间，则会被引用控制平面的名称空间。

### 使用 REST API

如果使用的是 REST API，则不能将 `Application` 的 namespace 指定为应用程序名称，而需要使用可选的 `appNamespace` 查询参数指定资源。 例如，要处理命名空间 `bar` 中名为 `foo` 的 `Application` 资源，请求将如下所示：

```bash
GET /api/v1/applications/foo?appNamespace=bar
```

对于其他操作，如 `POST` 和 `PUT`，`appNamespace` 参数必须是请求有效载荷的一部分。

对于控制平面 namespace 中的 "Application "资源，此参数可以省略。