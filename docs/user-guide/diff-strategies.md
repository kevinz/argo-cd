<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 差异策略

Argo CD 会计算期望状态和实时状态之间的差异，以确定应用程序是否不同步。 Argo CD UI 中也引用了同样的逻辑，以显示应用程序所属所有资源的实时状态和期望状态之间的差异。

Argo CD 目前有 3 种不同的差异计算策略： 1.

**Legacy** ：这是默认使用的主要差异策略。 它根据实时状态、期望状态和上次应用的配置（注释）应用 3 向差异。 **Structured-Merge Diff** ：启用服务器端应用同步选项时自动应用的策略。 **Server-Side Diff** ：在干运行模式下被引用服务器端应用的新策略，以生成预测的实时状态。

## 结构化合并差值

_当前状态：[测试版](https://github.com/argoproj/argoproj/blob/main/community/feature-status.md#beta) (自 v2.5.0 起)_

当启用服务器端应用同步选项时，会自动使用这种差异策略。 它使用的是[结构化合并差异](https://github.com/kubernetes-sigs/structured-merge-diff)Kubernetes 使用库根据字段所有权计算差异。 使用该策略计算定义了默认值的 CRD 的差异存在一些挑战。 在社区发现了不同的问题后，该策略被终止，转而使用服务器端 Diff。

## 服务器端差异

_当前状态：[测试版](https://github.com/argoproj/argoproj/blob/main/community/feature-status.md#beta) (自 v2.10.0 起)_

该差异策略将在干运行模式下对应用程序的每个资源执行服务器端应用。 然后将该操作的响应与实时状态进行比较，以提供差异结果。 差异结果将被缓存，只有在以下情况下才会触发向 Kube API 发送的新服务器端应用请求：。

* 请求刷新或硬刷新应用程序 * Argo CD 应用程序所针对的版本库中出现了新的修订 * Argo CD 应用程序规格发生了变化。

服务器差异端的一个优势是，Kubernetes Admission Controllers 会参与差异计算。 例如，如果验证 webhook 发现某个资源无效，那么 Argo CD 将在差异阶段而非同步阶段收到通知。

### 启用

服务器端 Diff 可在 Argo CD 控制器级别或按应用程序启用。

*为所有应用程序启用服务器端差分***

在 argocd-cmd-params-cm 配置表中添加以下条目：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  controller.diff.server.side: "true"
...
```

注意：有必要重新启动`argocd-application-controller`应用此配置后。

**为一个应用程序启用服务器端 Diff**

在 Argo CD 应用程序资源中添加以下注释：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/compare-options: ServerSideDiff=true
...
```

**禁用一个应用程序的服务器端 Diff**

如果在 Argo CD 实例中全局启用了服务器端 Diff，则可以在应用程序级别禁用它。 为此，请在应用程序资源中添加以下注解：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/compare-options: ServerSideDiff=false
...
```

_注意：请报告任何迫使您禁用服务器端 Diff 功能的问题_

### 突变网络钩子

默认情况下，服务器端差异不包含突变网络钩子所做的更改。 如果您想在 Argo CD 差异中包含突变网络钩子，请在 Argo CD 应用程序资源中添加以下注释： Argo CD 差异中包含突变网络钩子。

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/compare-options: IncludeMutationWebhook=true
...
```

注：此注释仅在启用服务器端 Diff 时有效。 要为特定应用程序启用这两个选项，请在 Argo CD 应用程序资源中添加以下注释： Argo CD

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/compare-options: ServerSideDiff=true,IncludeMutationWebhook=true
...
```