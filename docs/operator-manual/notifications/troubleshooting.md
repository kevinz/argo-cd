<!-- TRANSLATED by md-translate -->
`argocd admin notifications` 是一个 CLI 命令组，可帮助配置控制器设置和排除故障。 命令详情请参阅 [command reference](../../user-guide/commands/argocd_admin_notifications.md).

## global flags

以下全局 flag 适用于所有子命令：

* `--config-map` - 包含 `argocd-notifications-cm` 配置地图的文件路径。如果未指定

则命令会使用本地 Kubernetes 配置文件加载 `argocd-notification-cm` ConfigMap。

* `--secret` - 包含 `argocd-notifications-secret` ConfigMap 的文件路径。如果没有

此外，还可以指定 `:empty`，以使用没有通知服务设置的空 secret。

**示例：**

* 获取本地 config maps 中配置的触发器列表：
    ```bash
    argocd admin notifications trigger get\
      --config-map ./argocd admin notifications-cm.yaml --secret :empty
    ```
* 使用集群内配置映射和 secret 触发通知：
    ```bash
    argocd admin notifications template notify \
      app-sync-succeeded guestbook --recipient slack:argocd admin notifications
    ```

## kustomize

如果使用 Kustomize 管理 "argocd-notifications "配置，则可使用"--config-map - "标记将整个 "kustomize build "输出导入 stdin：

```bash
kustomize build ./argocd-notifications | \
  argocd-notifications \
  template notify app-sync-succeeded guestbook --recipient grafana:argocd \
  --config-map -
```

## 如何获得

### 在您的笔记本电脑上

您可以从 GitHub [release](https://github.com/argoproj/argo-cd/releases) 附件中下载 `argocd` CLI。

二进制文件可在 `quay.io/argoproj/argocd` 镜像中找到。 使用 `docker run` 和 volume mount 可以在任何平台上执行二进制文件。

**示例：**

```bash
docker run --rm -it -w /src -v $(pwd):/src \
  quay.io/argoproj/argocd:<version> \
  /app/argocd admin notifications trigger get \
  --config-map ./argocd-notifications-cm.yaml --secret :empty
```

#### 在你的集群中

SSH 进入正在运行的 `argocd-notifications-controller` pod，并使用 `kubectl exec` 命令验证集群内配置。

**示例**

```bash
kubectl exec -it argocd-notifications-controller-<pod-hash> \
  /app/argocd admin notifications trigger get
```

## 命令

以下命令可能有助于调试通知问题：

* [`argocd 管理通知模板 get`](../../user-guide/commands/argocd_admin_notifications_template_get.md)
* [`argocd 管理通知模板 notify`](../../user-guide/commands/argocd_admin_notifications_template_notify.md)
* [`argocd 管理通知触发器获取`]( ./../user-guide/commands/argocd_admin_notifications_trigger_get.md)
* [`argocd admin notifications trigger run`](../../user-guide/commands/argocd_admin_notifications_trigger_run.md)

## 错误

{！docs/operator-manual/notifications/troubleshooting-errors.md！} 。