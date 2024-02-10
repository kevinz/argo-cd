<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 同步相位和波形

&gt; v1.1

<iframe width="560" height="315" src="https://www.youtube.com/embed/zIHe3EVp528" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Argo CD 分几个步骤执行同步操作。 概括地说，有三个阶段同步前,同步和后同步。

在每个阶段中，您可以有一个或多个波段，这样就可以在同步后续资源之前确保某些资源是健康的。

## 如何配置阶段？

前同步和后同步只能包含钩子。 应用钩子注释：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

[了解更多关于挂钩的信息]（resource_hooks.md）。

## 如何配置波浪？

使用以下注释指定波形：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

钩子和资源默认分配给 0 波。 波可以是负数，因此您可以创建一个在所有其他资源之前运行的波。

## How Does It Work?

当 Argo CD 开始同步时，它会按照以下优先顺序排列资源： 1.

* 按类型（例如，[先是 namespace，然后是其他 Kubernetes 资源，最后是自定义资源]() * 按名称是 namespace，然后是其他 Kubernetes 资源，最后是自定义资源](https://github.com/argoproj/gitops-engine/blob/bc9ce5764fa306f58cf59199a94f6c968c775a2d/pkg/sync/sync_tasks.go#L27-L66) ) * 按名称

然后，它会确定要应用的下一个波的编号。 这是任何资源不同步或不健康的第一个编号。

它应用了这一波段的资源。

它会重复这一过程，直到所有相位和波都同步和健康。

由于应用程序的资源在第一波可能是不健康的，因此应用程序可能永远无法达到健康状态。

在修剪资源的过程中，先处理较高波段的资源，然后再处理较低波段的资源。 如果由于任何原因，某波段的资源未被移除/修剪，则下一波段的资源也不会被处理。

请注意，目前每个同步波之间的延迟都有 2 秒，可通过环境变量进行配置`ARGOCD_SYNC_WAVE_DELAY`。