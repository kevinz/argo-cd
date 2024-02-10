<!-- TRANSLATED by md-translate -->
# ApplicationSet 控制器支持的用例

通过生成器的概念，ApplicationSet 控制器提供了一套功能强大的工具，用于自动模板化和修改 Argo CD 应用程序。 生成器从 Argo CD 集群和 Git 资源库等各种来源生成模板参数数据，支持并启用新的用例。

虽然这些工具可用于任何需要的目的，但以下是 ApplicationSet 控制器旨在支持的一些特定用例。

## 用例：集群附加组件

ApplicationSet 控制器最初的设计重点是让基础架构团队的 Kubernetes 集群管理员能够在大量集群中自动创建大量不同的 Argo CD 应用程序集，并将这些应用程序作为一个单独的单元进行管理。 集群附加组件用例_就是一个说明为什么需要这样做的例子。

在_集群附加组件用例_中，管理员负责为一个或多个 Kubernetes 集群配置集群附加组件：集群附加组件是操作员，如[Prometheus 操作员](https://github.com/prometheus-operator/prometheus-operator)，或控制器，如[argo-workflows 控制器](https://argoproj.github.io/argo-workflows/)（[Argo 生态系统](https://argoproj.github.io/)的一部分）。

通常情况下，开发团队的应用需要这些附加组件（例如，作为多租户集群的租户，他们可能希望向 Provider 提供指标数据，或通过 Argo Workflows 协调工作流）。

由于安装这些附加组件需要集群级权限，而不是由单个开发团队掌握，因此安装工作由企业的基础设施/运营团队负责，而在大型企业中，该团队可能要负责数十、数百或数千个 Kubernetes 集群（新集群会定期添加/修改/删除）。

要在大量集群中进行扩展，并自动响应新集群的生命周期，就必须实现某种形式的自动化。 另一个要求是允许使用特定标准（如暂存集群与生产集群）将附加组件定位到集群子集。

![集群附加组件图](.../.../assets/applicationset/Use-Cases/Cluster-Add-Ons.png)

在本例中，基础架构团队维护着一个 Git 仓库，其中包含 Argo 工作流控制器和 Prometheus 操作员的应用配置清单。

基础架构团队希望使用 Argo CD 将这些附加组件部署到大量集群中，同样也希望轻松管理新集群的创建/删除。

在此用例中，我们可以使用 ApplicationSet 控制器的 List、集群或 Git 生成器来提供所需的行为：

* 列表生成器_：管理员维护两个 `ApplicationSet` 资源，每个应用程序（Workflows 和 Prometheus）一个，并在每个资源的列表生成器元素中包含他们希望针对的集群列表。
    - 使用此生成器，添加/删除集群需要手动更新 `ApplicationSet` 资源的列表元素。
* 集群生成器管理员维护两个 `ApplicationSet` 资源，每个应用程序（Workflows 和 Prometheus）一个，并确保所有新集群都在 Argo CD 中定义。
    - 由于集群生成器会自动检测并锁定 Argo CD 中定义的集群，因此[从 Argo CD 中添加/删除集群](.../.../声明式设置/#clusters)将自动导致 Argo CD 应用程序资源（针对每个应用程序）由 ApplicationSet 控制器创建。
* _Git生成器_：Git 生成器是生成器中最灵活/功能最强大的一个，因此有许多不同的方法来处理这种用例。以下是几种：
    - 使用 Git 生成器的 `files` 字段：集群列表以 JSON 文件的形式保存在 Git 仓库中。通过 Git 提交更新 JSON 文件，可添加/删除新的集群。
    - 被引用的 Git 生成器 "目录 "字段：对于每个目标集群，Git 仓库中都有一个对应的目录。通过 Git 提交添加/修改目录，将触发共享该目录名的集群的更新。

有关每个生成器的详细信息，请参阅[生成器部分]（Generators.md）。

## 用例：单回路设计

在_monorepo用例_中，Kubernetes集群管理员从单个Git仓库管理单个Kubernetes集群的整个状态。

合并到 Git 仓库的配置清单变更应自动部署到集群。

![Monorepo图](.../../assets/applicationset/Use-Cases/Monorepos.png)

在此示例中，基础架构团队维护着一个 Git 仓库，其中包含 Argo Workflows 控制器和 Prometheus 操作员的应用配置清单。 独立开发团队还添加了他们希望部署到集群的其他服务。

对 Git 仓库所做的更改（例如，更新已部署工件的版本）会自动导致 Argo CD 将更新应用到相应的 Kubernetes 集群。

Git 生成器可被引用来支持这种用例：

* Git 生成器的 "目录 "字段可用于指定包含要部署的各个应用程序的特定子目录（使用通配符）。
* Git 生成器的 "文件 "字段可引用包含 JSON 元数据的 Git 仓库文件，该元数据描述要部署的各个应用程序。
* 更多详情，请参阅 Git 生成器文档。

## 使用案例：多租户集群上 Argo CD 应用程序的自助服务

自助服务用例旨在让开发人员（作为 Kubernetes 多租户集群的最终用户）在以下方面拥有更大的灵活性：

* 使用 Argo CD，以自动化方式向单个集群部署多个应用程序
* 使用 Argo CD 以自动化方式向多个集群部署应用程序
* 但是，在这两种情况下，都要让这些开发人员有能力这样做，而无需集群管理员参与（代表他们创建必要的 Argo CD 应用程序/应用程序项目资源）

针对这种用例的一个潜在解决方案是，开发团队在一个 Git 仓库（包含他们希望部署的配置清单）中定义 Argo CD "应用程序 "资源，采用 [app-of-apps 模式]（.../.../cluster-bootstrapping/#app-of-apps-pattern），然后集群管理员通过合并请求审核/接受对该仓库的更改。

这听起来似乎是一个有效的解决方案，但它的一个主要缺点是需要高度的信任/审查才能接受包含 Argo CD "应用程序 "规范更改的提交。 这是因为 "应用程序 "规范中有许多敏感字段，包括 "项目"、"集群 "和 "名称空间"。 粗心大意的合并可能会让应用程序访问不属于它们的名称空间/集群。

因此，在自助服务用例中，管理员希望只允许开发人员控制 "应用程序 "规范的某些字段（如 Git 源代码库），而不允许其他字段（如目标名称空间或目标集群应受到限制）。

幸运的是，ApplicationSet 控制器为这种用例提供了另一种解决方案：集群管理员可以安全地创建一个包含 Git 生成器的 `ApplicationSet` 资源，该生成器通过 `template` 字段将应用程序资源的部署限制为固定值，同时允许开发人员随意定制 "安全 "字段。

`config.json` 文件包含描述应用程序的信息。

```json
{
  (...)
  "app": {
    "source": "https://github.com/argoproj/argo-cd",
    "revision": "HEAD",
    "path": "applicationset/examples/git-generator-files-discovery/apps/guestbook"
  }
  (...)
}
```

```yaml
kind: ApplicationSet
# (...)
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      files:
      - path: "apps/**/config.json"
  template:
    spec:
      project: dev-team-one # project is restricted
      source:
        # developers may customize app details using JSON files from above repo URL
        repoURL: {{.app.source}}
        targetRevision: {{.app.revision}}
        path: {{.app.path}}
      destination:
        name: production-cluster # cluster is restricted
        namespace: dev-team-one # namespace is restricted
```

详见 [Git 生成器](Generators-Git.md)。