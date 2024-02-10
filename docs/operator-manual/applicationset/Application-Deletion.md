<!-- TRANSLATED by md-translate -->
# 应用程序修剪和资源删除

ApplicationSet 控制器（从 ApplicationSet）创建的所有 `Application` 资源都将包含：

* 返回到_parent_ `ApplicationSet`资源的`.metadata.ownerReferences`引用
* 如果 `.syncPolicy.preserveResourcesOnDeletion` 设置为 false，应用程序的 `.metadata.finalizers` 中会出现 Argo CD `resources-finalizer.argocd.argoproj.io` finalizer。

最终结果是，当删除一个 ApplicationSet 时，会发生以下情况（按大致顺序）：

* 删除 `ApplicationSet` 资源本身
* 从该 `ApplicationSet` 创建的任何 `Application` 资源（通过 Owners 引用识别）
* 将删除（Argo CD）从该 "应用程序 "资源创建的受管集群上的任何已部署资源（"部署"、"服务"、"配置地图 "等）。
    - Argo CD 负责通过[the deletion finalizer]（.../.../.../user-guide/app_deletion/#about-the-deletion-finalizer）处理该删除。
    - 要保留已部署的资源，请在 ApplicationSet 中将 `.syncPolicy.preserveResourcesOnDeletion` 设为 true。

因此，"ApplicationSet"、"Application "和 "Application "资源的生命周期是等同的。

注意 另请参阅 [控制资源修改](Controlling-Resource-Modification.md) 页面，了解有关如何防止 ApplicationSet 控制器删除或修改应用程序资源的更多信息。

使用非级联删除，仍有可能删除 `ApplicationSet` 资源，同时防止 `Application` （及其部署的资源）也被删除：

```
kubectl delete ApplicationSet (NAME) --cascade=orphan
```

!!! 警告 即使使用非级联删除，"resources-finalizer.argocd.argoproj.io "仍会在 "Application "上指定。 因此，当 "Application "被删除时，其部署的所有资源也将被删除（"Application "及其_child_对象的生命周期仍是等价的）。

```
To prevent the deletion of the resources of the Application, such as Services, Deployments, etc, set `.syncPolicy.preserveResourcesOnDeletion` to true in the ApplicationSet. This syncPolicy parameter prevents the finalizer from being added to the Application.
```