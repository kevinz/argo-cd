<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 孤儿资源监测

Orphaned Kubernetes 资源是不属于任何 Argo CD 应用程序的顶级 namespaced 资源。 Orphaned 资源监控功能允许检测孤儿资源，使用 Argo CD UI 检查/删除资源并生成警告。

中启用了 "孤立资源 "监控。[项目](projects.md)设置，下面是使用 AppProject 自定义资源启用该功能的示例。

```yaml
kind: AppProject
metadata:
  ...
spec:
  ...
  orphanedResources:
    warn: true
...
```

启用该功能后，目标名称空间中存在孤儿资源的每个项目应用程序都会收到警告。

![无主资源](../assets/orphaned-resources.png)

启用该功能时，您可能需要考虑先禁用警告功能。

```yaml
spec:
  orphanedResources:
    warn: false # Disable warning
```

警告禁用后，应用程序用户仍可在用户界面中查看孤儿资源。

## 例外情况

并非 Kubernetes 集群中的每个资源都由最终用户控制。 以下资源永远不会被视为孤儿：

* 项目中拒绝使用的命名空间资源。 通常，此类资源由集群管理员管理，命名空间用户不得修改。 * 名称为 "default "的 "ServiceAccount"（以及相应的自动生成的 "ServiceAccountToken"）。 * 在 "default "命名空间中名称为 "kubernetes "的 "Service"。 * 在所有命名空间中名称为 "kube-root-ca.crt "的 "ConfigMap"。

此外，您还可以通过提供资源组、种类和名称列表来配置忽略资源。

```yaml
spec:
  orphanedResources:
    ignore:
    - kind: ConfigMap
      name: orphaned-but-ignored-configmap
```