<!-- TRANSLATED by md-translate -->
# Post Selector all generators

选择器允许使用 Kubernetes 常用的 labelSelector 格式，根据生成的值进行后过滤。 在示例中，列表生成器生成了一组两个应用程序，然后根据键值进行过滤，只选择值为`staging`的`env`：

## 示例：列表生成器 + Post Selector

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
          url: https://kubernetes.default.svc
          env: staging
        - cluster: engineering-prod
          url: https://kubernetes.default.svc
          env: prod
    selector:
      matchLabels:
        env: staging
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: examples/list-generator/guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

列表生成器 + Post Selector 只生成一组参数：

```yaml
- cluster: engineering-dev
  url: https://kubernetes.default.svc
  env: staging
```

也可以使用 `matchExpressions` 来获得更强大的选择器。

```yaml
spec:
  generators:
    - clusters: {}
      selector:
        matchExpressions:
          - key: server
            operator: In
            values:
              - https://kubernetes.default.svc
              - https://some-other-cluster
```