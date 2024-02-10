<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 一个应用程序的多个来源

警告 "测试版功能" 为应用程序指定多个源是一项测试版功能。 ui 和 cli 的行为通常仍与只指定第一个源相同。 将在未来发布的版本中添加对 ui/cli 的全面支持。 在标记为稳定版之前，该功能可能会以向后不兼容的方式进行更改。

Argo CD 可对所有资源进行编译，并对合并资源进行调节。

您可以通过使用`sources`当您指定`sources`字段，Argo CD 将忽略`source`（单数）字段。

请参阅下面的示例，了解如何指定多个数据源：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  sources:
    - chart: elasticsearch
      repoURL: https://helm.elastic.co
      targetRevision: 8.5.1
    - repoURL: https://github.com/argoproj/argocd-example-apps.git
      path: guestbook
      targetRevision: HEAD
```

上述示例指定了两个来源，Argo CD 将分别为每个来源生成的清单，然后将生成的清单合并。

如果多个来源生产相同的资源（相同的`group`,`kind`,`name`和`namespace`Argo CD 会生成一个`RepeatedResourceWarning`在这种情况下，Providers 会同步资源，这就提供了一种便捷的方法，用 Git 仓库中的资源覆盖图表中的资源。

## Helm 从外部 Git 仓库获取值文件

Helm 源可以引用来自 git 源的值文件。 这样，您就可以使用第三方 Helm 图表与自定义的、由 git 托管的值。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  sources:
  - repoURL: 'https://prometheus-community.github.io/helm-charts'
    chart: prometheus
    targetRevision: 15.7.1
    helm:
      valueFiles:
      - $values/charts/prometheus/values.yaml
  - repoURL: 'https://git.example.com/org/value-files.git'
    targetRevision: dev
    ref: values
```

在上述示例中`prometheus`图表将被引用来自`git.example.gom/org/value-files.git`.`$values`解析为`value-files`存放处。`$values`变量只能在值文件路径的开头指定。

如果`path`字段设置在`$values`源，Argo CD 将尝试从该 URL 上的 git 仓库生成资源。`path`字段未设置时，Argo CD 将只把存储库作为值文件的来源。

注意 来源与`ref`字段集不得同时指定`chart`Argo CD 目前不支持被引用。另一个 Helm 图表作为值文件的来源。