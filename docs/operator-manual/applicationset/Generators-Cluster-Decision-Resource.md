<!-- TRANSLATED by md-translate -->
# 集群决策资源生成器

集群决策资源会生成 Argo CD 集群列表。这是用 [duck-typing](https://pkg.go.dev/knative.dev/pkg/apis/duck)完成的，不需要了解引用的 Kubernetes 资源的完整形态。 以下是基于集群决策资源的 ApplicationSet 生成器示例：

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
 - clusterDecisionResource:
    # ConfigMap with GVK information for the duck type resource
    configMapRef: my-configmap  
    name: quak           # Choose either "name" of the resource or "labelSelector"
    labelSelector:
      matchLabels:       # OPTIONAL
        duck: spotted
      matchExpressions:  # OPTIONAL
      - key: duck
        operator: In
        values:
        - "spotted"
        - "canvasback"   
    # OPTIONAL: Checks for changes every 60sec (default 3min)
    requeueAfterSeconds: 60
 template:
   metadata:
     name: '{{.name}}-guestbook'
   spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{.clusterName}}' # 'server' field of the secret
        namespace: guestbook
```

由 ApplicationSet `clusterDecisionResource` 生成器引用的 `quak` 资源：

```yaml
apiVersion: mallard.io/v1beta1
kind: Duck
metadata:
  name: quak
spec: {}
status:
  # Duck-typing ignores all other aspects of the resource except 
  # the "decisions" list
  decisions:
  - clusterName: cluster-01
  - clusterName: cluster-02
```

ApplicationSet "资源引用了一个 "ConfigMap"，该 "ConfigMap "定义了将在此 duck-typing 中使用的资源。 每个 "ArgoCD "实例只需要一个 "ConfigMap "来标识资源。 您可以通过为每种资源创建一个 "ConfigMap "来支持多种资源类型。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  # apiVersion of the target resource
  apiVersion: mallard.io/v1beta1  
  # kind of the target resource
  kind: ducks
  # status key name that holds the list of Argo CD clusters
  statusListKey: decisions
  # The key in the status list whose value is the cluster name found in Argo CD
  matchKey: clusterName
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/clusterDecisionResource)。_）

本示例利用了 [open-cluster-management.io community](https://open-cluster-management.io/) 的集群管理功能。通过为 `open-cluster-management.io` 安置规则创建一个带有 GVK 的 `ConfigMap` ，您的 ApplicationSet 可以通过多种新颖的方式为不同的集群进行配置。 其中一个示例是让 ApplicationSet 在 3 个或更多的集群上仅维护两个 Argo CD 应用程序。然后，当维护或中断发生时，ApplicationSet 将始终维护两个应用程序，并根据安置规则的指示将应用程序移动到可用的集群上。

## 工作原理

ApplicationSet 需要在 Argo CD 名称空间中创建，将 `ConfigMap` 放在相同的名称空间中可让 ClusterDecisionResource 生成器读取它。 `ConfigMap` 存储 GVK 信息和状态键定义。 在 open-cluster-management 示例中，ApplicationSet 生成器将读取 apiVersion 为 `apps.open-cluster-management.io/v1` 的 `placementrules` 类型。 它将尝试从键 `decisions` 中提取集群的 **list** ，然后根据列表中每个元素的键 `clusterName` 中的 **value** 验证 Argo CD 中定义的实际集群名称。

ClusterDecisionResource 生成器会将鸭型资源状态列表中的 "名称"、"服务器 "和任何其他键/值作为参数传递到 ApplicationSet 模板中。 在本例中，决策数组包含一个额外的键 `clusterName`，现在 ApplicationSet 模板可以使用该键。

注意 "列为 `Status.Decisions` 的集群必须在 Argo CD 中预定义"。 `Status.Decisions` 中列出的集群名称 _must_ 必须在 Argo CD 中定义，以便为这些 Values 生成应用程序。 ApplicationSet 控制器不会在 Argo CD 中创建集群。

```
The Default Cluster list key is `clusters`.
```