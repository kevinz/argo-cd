<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 同步选项

Argo CD 允许用户自定义其在目标集群中同步所需状态的某些方面。 某些同步选项可定义为特定资源中的注释。 大多数同步选项在应用程序资源中配置`spec.syncPolicy.syncOptions`属性配置的多个同步选项。`argocd.argoproj.io/sync-options`注释可以与`,`中的注释值；空白处将被修剪。

下面是每个同步可用选项的详细信息：

## 没有修剪资源

&gt; v1.1

您可能希望阻止修剪对象：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

在用户界面中，pod 会显示为不同步：

[同步选项无剪枝](.../assets/sync-option-no-prune.png)

同步状态面板会显示剪枝是否被跳过以及跳过的原因：

[同步选项无剪枝](.../assets/sync-option-no-prune-sync-status.png)

如果 Argo CD 希望修剪资源，应用程序将无法同步。[比较选项](compare-options.md).

## 禁用 Kubectl 验证

对于某一类对象，有必要`kubectl apply`它们被引用到`--validate=false`标记，例如 Kubernetes 类型被引用为`RawExtension`例如[服务目录](https://github.com/kubernetes-incubator/service-catalog/blob/master/pkg/apis/servicecatalog/v1beta1/types.go#L497)您可以使用该注释：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Validate=false
```

如果想全局性地排除整类对象，可以考虑设置`resource.customizations`于[系统级配置](../user-guide/diffing.md#system-level-configuration)。

## 跳过新自定义资源类型的模拟运行

同步尚未为集群所知的自定义资源时，通常有两种选择：

1.CRD 清单是同一同步的一部分。 然后 Argo CD 将自动跳过干运行，应用 CRD 并创建资源。 2. 在某些情况下，CRD 不是同步的一部分，但可以通过其他方式创建，例如由集群中的控制器创建。 [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) 就是一个例子、 2.

根据用户定义的要求创建 CRD`ConstraintTemplates`Argo CD 无法在同步中找到 CRD，因此会出现以下错误`the server could not find the requested resource`。

要跳过缺失资源类型的模拟运行，请使用以下注释：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

如果集群中已存在 crd，则仍将执行模拟运行。

## 不删除资源

对于某些资源，例如持久卷索赔，您可能希望在应用程序删除后仍保留这些资源。 在这种情况下，您可以使用以下注释阻止在删除应用程序时清理这些资源： 1.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Delete=false
```

## 选择性同步

目前，当使用自动同步时，Argo CD 会引用应用程序中的每个对象。 对于包含数千个对象的应用程序来说，这需要花费很长的时间，并对 api 服务器造成过大的压力。 打开选择性同步选项，将只同步不同步的资源。

您可以通过以下方式添加该选项

在清单中添加 `ApplyOutOfSyncOnly=true

例如

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - ApplyOutOfSyncOnly=true
```

2.通过 argocd cli 设置同步选项

例如

```bash
$ argocd app set guestbook --sync-option ApplyOutOfSyncOnly=true
```

## 资源 删除 修剪 传播策略

默认情况下，无关资源会被引用前台删除策略进行修剪。 传播策略可通过以下方式控制`PrunePropagationPolicy`支持的策略有背景策略、前台策略和孤儿策略。有关这些策略的更多信息，请参见[这里](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#controlling-how-the-garbage-collector-deletes-dependents)。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - PrunePropagationPolicy=foreground
```

## 最后修剪

该功能的目的是在其他资源部署完毕、变得健康并成功完成所有其他操作后，将资源修剪作为同步操作的最后一个隐式波进行。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - PruneLast=true
```

这也可以在单个资源层面进行配置。

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: PruneLast=true
```

## 替换资源，而不是应用更改

默认情况下，Argo CD 会执行`kubectl apply`操作来应用存储在 Git 中的配置。`kubectl apply`例如，资源规格可能太大，无法放入`kubectl.kubernetes.io/last-applied-configuration`添加的`kubectl apply`在这种情况下，您可以被引用`Replace=true`同步选项：.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - Replace=true
```

如果`Replace=true`同步选项被引用时，Argo CD 将使用`kubectl replace`或`kubectl create`命令来应用更改。

警告 在同步过程中，将使用 "kubectl replace/create "命令同步资源。 该同步选项有可能具有破坏性，可能导致资源必须重新创建，从而造成应用程序中断。

这也可以在单个资源层面进行配置。

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Replace=true
```

## 服务器端应用

该选项使 Kubernetes[服务器端应用](https://kubernetes.io/docs/reference/using-api/server-side-apply/)。

默认情况下，Argo CD 会执行`kubectl apply`操作来应用存储在 Git 中的配置。 这是一个客户端操作，依赖于`kubectl.kubernetes.io/last-app-configuration`注解来存储之前的资源状态。

不过，在某些情况下，您会希望被引用`kubectl apply --serverside`越过`kubectl apply`：

* 资源太大，无法容纳在 262144 字节允许的注释大小中。 在这种情况下，可以使用服务器端引用来避免这个问题，因为在这种情况下不使用注释。 * 对集群上未完全由 Argo CD 管理的现有资源进行修补。 * 使用更具声明性的方法，即跟踪用户的字段管理，而不是跟踪用户的最后应用状态。

如果`ServerSideApply=true`同步选项被引用时，Argo CD 将使用`kubectl apply --server-side`命令来应用更改。

可以像下面的示例一样，在应用程序级别启用该功能：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - ServerSideApply=true
```

若要仅为单个资源启用 ServerSideApply，可使用 sync-option 注解：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: ServerSideApply=true
```

ProviderApply 还可以通过提供部分 yaml 来修补现有资源。 例如，如果只需要更新给定部署中的副本数量，可以向 Argo CD 提供以下 yaml：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
```

请注意，根据部署模式规范，这并不是一个有效的清单。 在这种情况下，需要增加一个同步选项必须下面的示例显示了如何配置应用程序以启用两个必要的同步选项： - 在这种情况下，需要增加一个有效的清单。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - ServerSideApply=true
    - Validate=false
```

在这种情况下，Argo CD 将被引用`kubectl apply --server-side --validate=false`命令来应用更改。

请注意：[替换=true](#replace-resource-instead-of-applying-changes)优先于`ServerSideApply=true`。

## 如果发现共享资源，同步失败

默认情况下，Argo CD 将应用在应用程序中配置的 git 路径中找到的所有清单，而不管 yamls 中定义的资源是否已被其他应用程序应用。 如果在应用程序的`FailOnSharedResource`如果设置了同步选项，只要 Argo CD 发现当前应用程序中的资源已被其他应用程序应用到集群中，就会导致同步失败。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - FailOnSharedResource=true
```

## 尊重忽略差异配置

该同步选项用于使 Argo CD 能够考虑在`spec.ignoreDifferences`默认情况下，Argo CD 在同步阶段也会被引用`ignoreDifferences`配置只是用来计算实时状态和期望状态之间的差异，这决定了应用程序是否同步。 但在同步阶段，期望状态会被引用。 补丁是通过实时状态、期望状态以及`last-applied-configuration`这有时会导致不想要的结果。 这种行为可以通过设置`RespectIgnoreDifferences=true`同步选项，如下面的示例： Argo CD 在同步阶段也会被引用`ignoreDifferences`配置只是用来计算实时状态和期望状态之间的差异，这决定了应用程序是否同步。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:

  ignoreDifferences:
  - group: "apps"
    kind: "Deployment"
    jsonPointers:
    - /spec/replicas

  syncPolicy:
    syncOptions:
    - RespectIgnoreDifferences=true
```

上面的示例展示了如何配置 Argo CD 应用程序，使其忽略`spec.replicas`字段，这是通过在集群中应用之前计算和预修补所需的状态来实现的。`RespectIgnoreDifferences`同步选项仅在资源已在集群中创建时有效。 如果应用程序正在创建，且不存在实时状态，则会按原样应用所需的状态。

## 创建 namespace

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: some-namespace
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
```

上面的示例展示了如何配置 Argo CD 应用程序，使其能够创建在以下文件中指定的名称空间`spec.destination.namespace`如果没有在应用程序清单中声明或在 CLI 中通过`-sync-option CreateNamespace=true`如果名称空间不存在，应用程序将无法同步。

注意，要创建的 namespace 必须在`spec.destination.namespace`字段。`metadata.namespace`字段必须与此值匹配，也可以省略，以便在适当的目标中创建资源。

### 名称空间元数据

我们还可以通过`managedNamespaceMetadata`如果我们扩展自上面的例子，就有可能实现下面的功能：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: test
spec:
  syncPolicy:
    managedNamespaceMetadata:
      labels: # The labels to set on the application namespace
        any: label
        you: like
      annotations: # The annotations to set on the application namespace
        the: same
        applies: for
        annotations: on-the-namespace
    syncOptions:
    - CreateNamespace=true
```

为了让 Argo CD 管理 namespace 上的标签和注释、`CreateNamespace=true`需要被设置为同步选项，否则什么也不会发生。 如果名称空间还不存在，或者已经存在但还没有设置标签和/或注释，就可以使用了。 使用`managedNamespaceMetadata`还会在名称空间上设置资源跟踪标签（或注释），这样就可以轻松跟踪哪些名称空间由 Argo CD 管理。

如果您没有任何自定义注释或标签，但仍希望在名称空间中设置资源跟踪，可以通过设置`managedNamespaceMetadata`用一个空的`labels`和/或`Annotations`地图，如下图所示： Annotations

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: test
spec:
  syncPolicy:
    managedNamespaceMetadata:
      labels: # The labels to set on the application namespace
      annotations: # The annotations to set on the application namespace
    syncOptions:
    - CreateNamespace=true
```

如果 Argo CD 要 "采用 "一个已经设置了元数据的现有名称空间，则应首先[将资源升级为服务器端应用](https://kubernetes.io/docs/reference/using-api/server-side-apply/#upgrading-from-client-side-apply-to-server-side-apply)在启用`managedNamespaceMetadata`Argo CD 依赖于`kubectl`如果不将资源升级为服务器端应用，Argo CD 可能会删除现有标签/注释，这可能也是可能不是所希望的行为。

另一个需要注意的问题是，如果您在 Argo CD 应用程序中为相同的 namespace 设置了 k8s 清单，那么该清单将优先于 Argo CD 应用程序。_覆盖在 `managedNamespaceMetadata` 中设置的任何值_____________________________。换句话说，如果您的应用程序设置了`managedNamespaceMetadata`，那么该清单将优先于 Argo CD 应用程序。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    managedNamespaceMetadata:
      annotations:
        abc: 123
    syncOptions:
      - CreateNamespace=true
```

但你也有一个名称匹配的 k8s 清单

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: foobar
  annotations:
    foo: bar
    something: completely-different
```

生成的 namespace 的注释将设置为

```yaml
annotations:
    foo: bar
    something: completely-different
```