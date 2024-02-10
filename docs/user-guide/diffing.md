<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# Diffing 定制

应用程序有可能`OutOfSync`造成这种情况的一些原因可能是

* 清单中存在一个 Bug，其中包含来自实际 k8s 规范的额外/未知字段。 在查询 Kubernetes 的实时状态时，这些额外字段会被丢弃、

导致`OutOfSync`状态，表明检测到缺少一个字段。

已同步执行（已禁用剪枝功能），但有资源需要删除。 * 控制器或[突变网络钩子](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)在对象被删除后对其进行了更改。 * 控制器或[突变网络钩子](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)在对象被删除后对其进行了更改。

提交给 Kubernetes 的方式与 Git 相矛盾。

* helm chart 被引用模板函数，如 [`randAlphaNum`](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml#L16) 、 [`andAlphaNum`](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml#L16) 、 [`andAlphaNum`](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml#L16) 、 [`andAlphaNum`](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml#L16) 。

每次都产生不同的数据`helm template`被调用。

* 对于水平 Pod Autoscaling (HPA) 对象，已知 HPA 控制器会按照特定顺序重新排列 `spec.metrics` 。请参阅 [kubernetes issue #74099](https://github.com/kubernetes/kubernetes/issues/74099) 。要解决这个问题，可以在 Git 中按照控制器偏好的顺序排列 `spec.metrics` 。

如果无法修复上游问题，Argo CD 允许您选择忽略有问题资源的差异。 可为单个或多个应用程序资源或在系统级别配置差异定制。

## 应用程序级配置

Argo CD 允许忽略特定 JSON 路径上的差异，使用被引用的[RFC6902 JSON 补丁](https://tools.ietf.org/html/rfc6902)和[JQ 路径表达式](https://stedolan.github.io/jq/manual/#path(path_expression))也可以忽略与在`metadata.managedFields`活资源中。

以下示例应用程序被配置为忽略`spec.replicas`用于所有部署：

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

请注意`group`字段涉及[Kubernetes API 小组](https://kubernetes.io/docs/reference/using-api/#api-groups)上述定制可以缩小到具有指定名称和可选名称空间的资源：

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    name: guestbook
    namespace: default
    jsonPointers:
    - /spec/replicas
```

要忽略列表中的元素，可以使用 JQ 路径表达式根据项目内容识别列表项目： - 要忽略列表中的元素，可以使用 JQ 路径表达式根据项目内容识别列表项目

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jqPathExpressions:
    - .spec.template.spec.initContainers[] | select(.name == "injected-init-container")
```

忽略实时资源中定义的特定 managed 所拥有的字段：

```yaml
spec:
  ignoreDifferences:
  - group: "*"
    kind: "*"
    managedFieldsManagers:
    - kube-controller-manager
```

上述配置将忽略由`kube-controller-manager`的所有资源。

如果有斜线`/`在你的指针路径中，你可以使用被引用的`~1`例如

```yaml
spec:
  ignoreDifferences:
  - kind: Node
    jsonPointers: /metadata/labels/node-role.kubernetes.io~1worker
```

## 系统级配置

可在系统级别上自定义与众所周知的问题进行资源比较。`resource.customizations`的关键`argocd-cm`下面是一个自定义的例子，它忽略了`caBundle`的字段`MutatingWebhookConfiguration`网络钩子：.

```yaml
data:
  resource.customizations.ignoreDifferences.admissionregistration.k8s.io_MutatingWebhookConfiguration: |
    jqPathExpressions:
    - '.webhooks[]?.clientConfig.caBundle'
```

资源定制也可以配置为忽略由 managedField.manager 在系统级别上做出的所有差异。 下面的示例显示了如何配置 Argo CD 忽略由`kube-controller-manager`于`Deployment`资源

```yaml
data:
  resource.customizations.ignoreDifferences.apps_Deployment: |
    managedFieldsManagers:
    - kube-controller-manager
```

可以将 ignoreDifferences 配置为应用于 Argo CD 实例管理的每个应用程序中的所有资源。 为此，可以像下面的示例一样配置资源自定义： Argo CD 实例管理

```yaml
data:
  resource.customizations.ignoreDifferences.all: |
    managedFieldsManagers:
    - kube-controller-manager
    jsonPointers:
    - /spec/replicas
```

status`领域`CustomResourceDefinitions`通常存储在 Git/Helm 清单中，在差异化过程中应忽略。

```yaml
data:
  resource.compareoptions: |
    # disables status field diffing in specified resource types
    # 'crd' - CustomResourceDefinitions (default)
    # 'all' - all resources
    # 'none' - disabled
    ignoreResourceStatusField: crd
```

默认情况下`status`字段在为`CustomResourceDefinition`该行为可扩展自所有被引用的资源。`all`值或被引用禁用。`none`.

### 忽略 AggregateRoles 所做的 RBAC 更改

如果您被引用的是[聚合群集角色](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)并且不希望 Argo CD 检测到`rules`您可以设置`resource.compareoptions.ignoreAggregatedRoles: true`然后，Argo CD 将不再把这些更改检测为需要同步的事件。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  resource.compareoptions: |
    # disables status field diffing in specified resource types
    ignoreAggregatedRoles: true
```

## CRD 中已知的 Kubernetes 类型（资源限制、卷挂载等）

一些 CRD 正在重复使用 Kubernetes 源代码库中定义的数据结构，因此继承了自定义 JSON/YAML 编译。 自定义编译器可能会以略有不同的格式对 CRD 进行序列化，从而在漂移检测过程中造成误报。

一个典型的例子是`argoproj.io/Rollout`被引用的 CRD`core/v1/PodSpec`Pod 资源请求可能会被"...... "的自定义 marshaller 重新格式化。`IntOrString`数据类型：

来自

```yaml
resources:
  requests:
    cpu: 100m
```

到：

```yaml
resources:
  requests:
    cpu: 0.1
```

解决方法是在`resource.customizations`部分`argocd-cm`配置地图：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  resource.customizations.knownTypeFields.argoproj.io_Rollout: |
    - field: spec.template.spec
      type: core/v1/PodSpec
```

支持的 Kubernetes 类型列表可在[diffing_known_types.txt](https://raw.githubusercontent.com/argoproj/argo-cd/master/util/argo/normalizers/diffing_known_types.txt)此外

* 核心/数量
