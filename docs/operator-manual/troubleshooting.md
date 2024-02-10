<!-- TRANSLATED by md-translate -->
# 故障排除工具

该文件介绍了如何使用 `argocd admin` 子命令来简化 Argo CD 设置的定制和连接问题的故障排除。

## 设置

Argo CD 提供了多种自定义系统行为的方法，并有大量设置。 在多个用户在生产中使用的 Argo CD 上修改设置可能很危险。 在应用设置之前，您可以使用 `argocd admin` 子命令来确保设置有效，并且 Argo CD 按预期运行。

argocd admin settings validate` 命令执行基本设置验证，并打印每个设置组的简短摘要。

**差异化定制**

[扩散自定义](.../user-guide/diffing.md) 允许从扩散过程中排除某些资源字段。 扩散自定义在 `argocd-cm` configmaps 的 `resource.customizations` 字段中配置。

下面的 `argocd admin` 命令将打印指定的 ConfigMap 中排除在 diffing 之外的字段信息。

```bash
argocd admin settings resource-overrides ignore-differences ./deploy.yaml --argocd-cm-path ./argocd-cm.yaml
```

**健康评估**

Argo CD 为多个 Kubernetes 资源提供了内置的 [health assessment]（./health.md），可通过在 [Lua](https://www.lua.org/) 中编写自己的健康检查来进一步定制。健康检查在 `argocd-cm` ConfigMap 的 `resource.customizations` 字段中配置。

下面的 `argocd admin` 命令使用在指定的 configmaps 中配置的 Lua 脚本评估资源健康状况。

```bash
argocd admin settings resource-overrides health ./deploy.yaml --argocd-cm-path ./argocd-cm.yaml
```

**资源行动**

资源操作允许配置命名的 Lua 脚本来执行资源修改。

下面的 `argocd admin` 命令使用指定 ConfigMap 中配置的 Lua 脚本执行操作，并打印被引用的修改。

```bash
argocd admin settings resource-overrides run-action /tmp/deploy.yaml restart --argocd-cm-path /private/tmp/argocd-cm.yaml
```

下面的 `argocd admin` 命令使用在指定的 ConfigMap 中配置的 Lua 脚本列出了指定资源的可用操作。

```bash
argocd admin settings resource-overrides list-actions /tmp/deploy.yaml --argocd-cm-path /private/tmp/argocd-cm.yaml
```

## 集群证书

如果您使用集群凭据手动创建了 Secret，并尝试需要排除连接问题，则 `argocd admin 集群 kubeconfig` 非常有用。 在这种情况下，建议使用以下步骤：

1 SSH 进入 [argocd-application-controller] pod。

```
kubectl exec -n argocd -it \
  $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller -o jsonpath='{.items[0].metadata.name}') bash
```

2 使用 `argocd admin cluster kubeconfig` 命令从配置的 Secret 导出 kubeconfig 文件：

```
argocd admin cluster kubeconfig https://<api-server-url> /tmp/kubeconfig --namespace argocd
```

3 使用 `kubectl` 获取有关连接问题的更多详细信息，修复问题并将更改应用回 secret：

```
export KUBECONFIG=/tmp/kubeconfig
kubectl get pods -v 9
```