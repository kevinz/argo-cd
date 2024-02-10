<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

#### 利用应用程序集控制器自动生成 Argo CD 应用程序

[应用程序集控制器](./operator-manual/applicationset/index.md)是 Argo CD 增加应用程序自动化的一部分，旨在改进 Argo CD 中的多集群支持和集群多租户支持。Argo CD 应用程序可从多个不同来源（包括 Git 或 Argo CD 自己定义的集群列表）进行模板化。

ApplicationSet 控制器提供的工具集还可用于允许开发人员（无法访问 Argo CD 名称空间）在没有集群管理员干预的情况下独立创建应用程序。

警告 请注意[安全影响](../operator-manual/applicationset/Security.md)在允许开发人员通过 ApplicationSets 创建应用程序之前。

ApplicationSet 控制器与 Argo CD 同时安装（在同一 namespace 中），控制器会根据新的`ApplicationSet`自定义资源 (CR)。

下面是一个例子`ApplicationSet`资源，可用于将 Argo CD 应用程序引用到多个集群：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
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
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: applicationset/examples/list-generator/guestbook/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
```

列表生成器会将`url`和`cluster`字段作为`{{param}}`-样式的参数，然后将其呈现为三个相应的 Argo CD 应用程序（每个已定义的集群一个）。 瞄准新集群（或移除现有集群）只需更改`ApplicationSet`资源，并自动创建相应的 Argo CD 应用程序。

同样，对 ApplicationSet`template`因此，管理一组多个 Argo CD 应用程序就像管理单个应用程序一样简单。

在 ApplicationSet 中，除了列表生成器外，还有其他功能更强大的生成器，包括集群生成器（自动使用 Argo CD 定义的集群来制作应用程序模板）和 Git 生成器（使用 Git 仓库的文件/目录来制作应用程序模板）。

要了解有关 ApplicationSet 控制器的更多信息，请查阅[应用程序集文档](../operator-manual/applicationset/index.md)与 Argo CD 一起安装 ApplicationSet 控制器。

*注：***开始`v2.3`在 Argo CD 中，我们不需要单独安装 ApplicationSet Controller，而是将其作为 Argo CD 安装的一部分。