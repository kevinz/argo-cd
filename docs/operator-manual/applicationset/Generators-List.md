<!-- TRANSLATED by md-translate -->
# 列表生成器

List 生成器根据任意键/值对列表生成参数（只要值是字符串值即可）。 在本例中，我们的目标是名为 `engineering-dev` 的本地集群：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://kubernetes.default.svc
      - cluster: engineering-prod
        url: https://kubernetes.default.svc
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: applicationset/examples/list-generator/guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/list-generator)。_）

在本例中，List 生成器会将 `url` 和 `cluster` 字段作为参数传递到模板中。 如果我们想添加第二个环境，可以取消对第二个元素的注释，ApplicationSet 控制器就会自动将其作为已定义应用程序的目标。

在 ApplicationSet v0.1.0 发布时，人们只能_指定`url`和`集群`元素字段（加上任意`值`）。 从 ApplicationSet v0.2.0 开始，支持任何键/值`元素`对（这也完全向后兼容 v0.1.0 表单）：

```yaml
spec:
  generators:
  - list:
      elements:
        # v0.1.0 form - requires cluster/url keys:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
          values:
            additional: value
        # v0.2.0+ form - does not require cluster/URL keys
        # (but they are still supported).
        - staging: "true"
          gitRepo: https://kubernetes.default.svc   
# (...)
```

注意 "集群必须在 Argo CD 中预定义"，这些集群_必须_已经在 Argo CD 中定义，以便为这些 Values 生成应用程序。 ApplicationSet 控制器不会在 Argo CD 中创建集群（例如，它没有这样做的凭证）。

## 动态生成元素

列表生成器还可以根据从上一个生成器（如 git）获取的 yaml/json 动态生成元素，方法是将两者与矩阵生成器结合使用。 在本例中，我们将矩阵生成器与 git 结合使用，然后再与列表生成器结合使用，并将 git 中的文件内容作为输入传递给列表生成器的 `elementsYaml` 字段：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: elementsYaml
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - matrix:
      generators:
      - git:
          repoURL: https://github.com/argoproj/argo-cd.git
          revision: HEAD
          files:
          - path: applicationset/examples/list-generator/list-elementsYaml-example.yaml
      - list:
          elementsYaml: "{{ .key.components | toJson }}"
  template:
    metadata:
      name: '{{.name}}'
    spec:
      project: default
      syncPolicy:
        automated:
          selfHeal: true    
        syncOptions:
        - CreateNamespace=true        
      sources:
        - chart: '{{.chart}}'
          repoURL: '{{.repoUrl}}'
          targetRevision: '{{.version}}'
          helm:
            releaseName: '{{.releaseName}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.namespace}}'
```

list-elementsYaml-example.yaml "内容：

```yaml
key:
  components:
    - name: component1
      chart: podinfo
      version: "6.3.2"
      releaseName: component1
      repoUrl: "https://stefanprodan.github.io/podinfo"
      namespace: component1
    - name: component2
      chart: podinfo
      version: "6.3.3"
      releaseName: component2
      repoUrl: "ghcr.io/stefanprodan/charts"
      namespace: component2
```