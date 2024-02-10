<!-- TRANSLATED by md-translate -->
# 集群管理

本指南适用于希望在 CLI 上管理集群的操作员。 如果你想为此使用 Kubernetes 资源，请查看 [声明式设置]（./declarative-setup.md#clusters）。

此处并未介绍所有命令，所有可用命令请参阅 [argocd 集群命令参考]（.../user-guide/commands/argocd_cluster.md）。

## 添加一个集群

运行 `argocd cluster add context-name`。

如果不确定上下文名称，可运行 `kubectl config get-contexts` 查看所有上下文。

这将连接到集群，并为 ArgoCD 安装必要的资源以连接到集群。 请注意，您需要对集群有权限访问。

## 删除集群

运行 `argocd 集群 rm context-name`。

删除指定名称的集群。

!!注意 "集群内不能移除" `集群内`集群不能用此移除。 如果要禁用`集群内`配置，需要更新`argocd-cm` ConfigMap。 设置 [`cluster.inClusterEnabled`](./argocd-cm-yaml.md) 为`"false"`。