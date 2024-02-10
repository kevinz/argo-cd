<!-- TRANSLATED by md-translate -->
# 谷歌聊天

## 参数

谷歌聊天通知服务向谷歌聊天 webhook 发送消息通知。 该服务被引用的设置如下：

* `webhooks` - 格式为 `webhookName: webhookUrl` 的映射表

## 配置

1.打开 "谷歌聊天"，进入要发送信息的空间
2.从页面顶部的菜单中选择 **配置 Webhook**。
3.在**传入网络钩子**下，点击**添加网络钩子**
4.为网络钩子命名，可选择添加镜像，然后单击**保存**
5.复制网络钩子旁边的 URL
6.将 URL 存储在 `argocd-notification-secret` 中，并在 `argocd-notifications-cm` 中声明方式

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.googlechat: |
    webhooks:
      spaceName: $space-webhook-url
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  space-webhook-url: https://chat.googleapis.com/v1/spaces/<space_id>/messages?key=<key>&token=<token>
```

6.为您的空间创建订阅

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.googlechat: spaceName
```

## 模板

您可以向 Google Chat 空间发送[简单文本](https://developers.google.com/chat/reference/message-formats/basic) 或[卡片信息](https://developers.google.com/chat/reference/message-formats/cards)。简单文本信息模板可定义如下：

```yaml
template.app-sync-succeeded: |
  message: The app {{ .app.metadata.name }} has successfully synced!
```

贺卡信息的定义如下

```yaml
template.app-sync-succeeded: |
  googlechat:
    cardsV2: |
      - header:
          title: ArgoCD Bot Notification
        sections:
          - widgets:
              - decoratedText:
                  text: The app {{ .app.metadata.name }} has successfully synced!
          - widgets:
              - decoratedText:
                  topLabel: Repository
                  text: {{ call .repo.RepoURLToHTTPS .app.spec.source.repoURL }}
              - decoratedText:
                  topLabel: Revision
                  text: {{ .app.spec.source.targetRevision }}
              - decoratedText:
                  topLabel: Author
                  text: {{ (call .repo.GetCommitMetadata .app.status.sync.revision).Author }}
```

支持所有[卡片字段](https://developers.google.com/chat/api/reference/rest/v1/cards#Card_1)，并可在通知中使用。也可以使用以前的（现已废弃）"cards "键来使用传统的卡片字段，但不建议这样做，因为 Google 已废弃了该字段，建议使用较新的 "cardsV2"。

卡片信息也可以用 JSON 格式编写。

## 聊天主题

通过为线程指定唯一密钥，可以在聊天线程中发送简单文本和卡片信息。 线程密钥定义如下：

```yaml
template.app-sync-succeeded: |
  message: The app {{ .app.metadata.name }} has successfully synced!
  googlechat:
    threadKey: {{ .app.metadata.name }}
```