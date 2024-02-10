<!-- TRANSLATED by md-translate -->
# ApplicationSet 控制器简介

## 简介

ApplicationSet 控制器是一个[Kubernetes 控制器](https://kubernetes.io/docs/concepts/architecture/controller/)，它增加了对 "ApplicationSet"[CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) (CRD) 的支持。该控制器/CRD 可实现自动化和更大的灵活性，在大量集群和单核中管理[Argo CD](./../index.md)应用程序，还可在多租户 Kubernetes 集群上实现自服务使用。

ApplicationSet 控制器与现有的 [Argo CD 安装]（.../.../index.md）一起工作。 Argo CD 是一个声明式的 GitOps 持续交付工具，它允许开发人员在现有的 Git 工作流中定义和控制 Kubernetes 应用程序资源的部署。

从 Argo CD v2.3 版开始，ApplicationSet 控制器与 Argo CD 捆绑。

ApplicationSet 控制器是 Argo CD 的补充，增加了更多支持集群管理员的功能。 ApplicationSet 控制器提供以下功能：

* 通过 Argo CD，使用单个 Kubernetes 配置清单就能瞄准多个 Kubernetes 集群
* 通过 Argo CD，可使用单个 Kubernetes 配置清单从一个或多个 Git 仓库部署多个应用程序
* 改进了对 monorepos 的支持：在 Argo CD 中，monorepo 是指在单个 Git 仓库中定义的多个 Argo CD 应用程序资源。
* 在多租户集群中，提高了单个集群租户使用 Argo CD 部署应用程序的能力（无需集群管理员启用目标集群/名称空间）。

注意 在被引用之前，请了解 ApplicationSet 的 [安全影响](./Security.md)。

## ApplicationSet 资源

本例定义了一个新的 `guestbook` 资源，其类型为 `ApplicationSet`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://1.2.3.4
      - cluster: engineering-prod
        url: https://2.4.6.8
      - cluster: finance-preprod
        url: https://9.8.7.6
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: my-project
      source:
        repoURL: https://github.com/infra-team/cluster-deployments.git
        targetRevision: HEAD
        path: guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

在这个示例中，我们要将我们的 `guestbook` 应用程序（该应用程序的 Kubernetes 资源来自 Git，因为这是 GitOps）部署到 Kubernetes 集群列表（目标集群列表定义在 `ApplicationSet` 资源的 List items 元素中）。

虽然有多种类型的_生成器可与 `ApplicationSet` 资源一起使用，但本示例使用的是 List 生成器，它只包含一个固定的、字面意义上的目标集群列表。 一旦 ApplicationSet 控制器处理了 `ApplicationSet` 资源，该集群列表将成为 Argo CD 部署 `guestbook` 应用程序资源的集群。

生成器（如 List 生成器）负责生成_参数_。 参数是键值对，在模板渲染过程中被替换到 ApplicationSet 资源的 "template: "部分。

ApplicationSet 控制器目前支持多种生成器：

* **列表生成器**：根据集群名称/URL 值的固定列表生成参数，如上例所示。
* **集群生成器**：集群生成器不会像列表生成器那样生成一个字面意义上的集群列表，而是根据 Argo CD 中定义的集群自动生成集群参数。
* **Git 生成器**：Git 生成器根据生成器资源中定义的 Git 仓库中包含的文件或文件夹生成参数。
    - 包含 JSON 值的文件将被解析并转换为模板参数。
    - Git 仓库中的单个目录路径也可被引用为参数值。
* **矩阵生成器**：矩阵生成器结合了其他两个生成器生成的参数。

有关单个生成器和其他未列出的生成器的更多信息，请参阅[生成器部分]（Generators.md）。

## 将参数替换为模板

无论使用哪种生成器，生成器生成的参数都会被替换到 "ApplicationSet "资源的 "template: "部分中的"{{参数名}}"值中。 在本例中，List 生成器定义了 "集群 "和 "url "参数，然后将其分别替换到模板的"{{集群}}"和"{{url}}"值中。

替换后，这个 `guestbook``ApplicationSet` 资源就会应用到 Kubernetes 集群：

1.ApplicationSet 控制器会处理生成器条目，生成一组模板参数。
2.这些参数被替换到模板中，每组参数替换一次。
3.每个呈现的模板都会转换成 Argo CD`Application` 资源，然后在 Argo CD namespace 中创建（或更新）。
4.最后，Argo CD 控制器会收到这些 "应用 "资源的通知，并负责处理它们。

在我们的示例中定义了三个不同的集群--"engineering-dev"、"engineering-prod "和 "finance-preprod"--这将产生三个新的 Argo CD "Application "资源：每个集群一个。

下面是为位于 `1.2.3.4` 的 `engineering-dev` 集群创建的其中一个 `Application` 资源的示例：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: engineering-dev-guestbook
spec:
  source:
    repoURL: https://github.com/infra-team/cluster-deployments.git
    targetRevision: HEAD
    path: guestbook/engineering-dev
  destination:
    server: https://1.2.3.4
    namespace: guestbook
```

我们可以看到，生成的值已被替换到模板的 `server` 和 `path` 字段中，模板已被渲染成一个完整的 Argo CD 应用程序。

现在在 Argo CD UI 中也能看到应用程序：

![Argo CD Web UI 中的列表生成器示例]( ./../assets/applicationset/Introduction/List-Example-In-Argo-CD-Web-UI.png)

ApplicationSet 控制器将确保对 "ApplicationSet "资源所做的任何更改、更新或删除都会自动应用到相应的 "Application"。

例如，如果在列表生成器中添加了一个新的集群/URL 列表条目，就会相应地为这个新的集群创建一个新的 Argo CD `Application` 资源。 对`guestbook```ApplicationSet`资源所做的任何编辑都会影响到该资源实例化的所有 Argo CD 应用程序，包括新的应用程序。

虽然列表生成器的集群字面列表相当简单，但 ApplicationSet 控制器中的其他可用生成器可以支持更复杂的情况。