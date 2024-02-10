<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 跳过申请对账

!!! 警告 "Alpha 功能" 这是一个实验性的、alpha 质量的功能。 主要用例是提供与第三方项目的集成。 此功能可能会在未来的发布中被移除或以向后不兼容的方式进行修改。

Argo CD 允许用户停止应用程序的对账。 跳过对账选项是通过`argocd.argoproj.io/skip-reconcile: "true"`当应用程序被配置为跳过对账时，应用程序的所有处理都会停止。 在应用程序不处理期间，应用程序`status`字段将不会更新。 如果新创建的应用程序带有跳过对账注释，那么应用程序的`status`要恢复对账或处理申请，请删除注释或将其设为`"false"`。

请参阅以下示例，了解如何使应用程序跳过对账：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/skip-reconcile: "true"
```

请参阅以下示例，了解启用跳过对账功能后新创建的应用程序：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd.argoproj.io/skip-reconcile: "true"
  name: guestbook
  namespace: argocd
spec:
  destination:
    namespace: guestbook
    server: https://kubernetes.default.svc
  project: default
  source:
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
```

status`字段不存在。

## 主要被引用情况

跳过对账选项适用于希望更新应用程序状态而不被应用程序控制器覆盖的第三方项目。[开放式集群管理（OCM）](https://github.com/open-cluster-management-io/)被引用的项目[拉动-整合](https://github.com/open-cluster-management-io/argocd-pull-integration)在该示例中，集群应用程序不是由 Argo CD 应用程序控制器调节，而是由 OCM 拉动集成控制器使用从远程/支点/托管集群收集的应用程序状态来填充主集群/集群应用程序状态。

## 备选用例

这个跳过对账选项还有其他的使用情况。 需要注意的是，这是一个实验性的、alpha 质量的功能，一般不推荐以下使用情况。

* 在灾难恢复过程中，暂停和恢复应用程序的对账。 通过不允许应用程序立即开始对账，提供另一种可供选择的审批流程。
