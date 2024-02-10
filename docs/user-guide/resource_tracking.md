<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 资源跟踪

## 按标签跟踪 Kubernetes 资源

Argo CD 通过在所有被管理（即从 Git 对账）的资源上将应用程序实例标签设置为管理应用程序的名称，来识别其管理的资源。 使用的默认标签是众所周知的标签`app.kubernetes.io/instance`。

例如

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  labels:
    app.kubernetes.io/instance: some-application
```

这种方法在大多数情况下都很有效，因为标签的名称是标准化的，Kubernetes 生态系统中的其他工具也能理解。

但也存在一些局限性：

* 标签会被截断为 63 个字符。 根据标签的大小，您可能希望为应用程序存储更长的名称 * 其他外部工具可能会写入/附加此标签，从而与 Argo CD 产生冲突。 例如，一些 helm chart和操作员也会使用此标签生成清单，从而使 Argo CD 对应用程序的所有者产生混淆 * 您可能希望在同一集群上部署多个 Argo CD 实例（具有集群权限），并有一种简单的方法来识别哪个资源由哪个 Argo CD 实例管理。

### 使用自定义标签

而不是被引用默认的`app.kubernetes.io/instance`下面的示例将资源跟踪标签设置为`argocd.argoproj.io/instance`。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  application.instanceLabelKey: argocd.argoproj.io/instance
```

## 通过注释的其他跟踪方法

&gt; v2.2

为了提供更灵活的资源跟踪选项，并解决上一节概述的一些问题，可以指示 Argo CD 使用以下方法进行跟踪：

1.label`（默认）- Argo CD 被引用`app.kubernetes.io/instance`标签 2.`annotation+label` - Argo CD 使用`app.kubernetes.io/instance`标签，但仅供参考。该标签不被引用用于跟踪目的，如果值超过 63 个字符，仍会被截断。Annotations `argocd.argoproj.io/tracking-id` 被引用来跟踪应用程序资源。对于用 Argo CD 管理的资源，但仍需要与其他需要实例标签的工具兼容时，请使用此方法。 3. `annotation` - Argo CD 使用 `argocd.argoproj.io/tracking-id` 注解来跟踪应用程序资源。当不需要同时维护标签和 Annotations 时，请使用此方法。

下面是一个使用注释法跟踪资源的示例： 1.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  annotations:
    argocd.argoproj.io/tracking-id: my-app:apps/Deployment:default/nginx-deployment
```

使用跟踪 id 注释的好处是不会再与其他 Kubernetes 工具发生冲突，而且 Argo CD 也不会混淆资源的所有者。`annotation+label`如果想让其他工具了解 Argo CD 管理的资源，也可以引用该工具。

## 选择跟踪方法

要实际选择首选跟踪方法，请编辑`resourceTrackingMethod`内所包含的`argocd-cm`configmap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  application.resourceTrackingMethod: annotation
```

可能的值是`label`,`Annotations+label`和`Annotations`如上一节所述。

请注意，一旦更改了值，您需要再次同步应用程序（或等待同步机制启动），以便应用您的更改。

您可以重新更改配置图，恢复到之前的选择。