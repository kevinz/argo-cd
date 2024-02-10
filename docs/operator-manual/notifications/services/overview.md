<!-- TRANSLATED by md-translate -->
通知服务代表了与服务（如 slack、电子邮件或自定义 webhook）的集成。服务在 `argocd-notifications-cm` ConfigMap 中使用 `service.<type>.(<custom-name>)` 键进行配置，并可能引用 `argocd-notifications-secret` Secret 中的敏感数据。 以下示例演示了 slack 服务配置：

```yaml
service.slack: |
    token: $slack-token
```

slack "表示服务发送松弛通知；名称缺失，默认为 "slack"。

## 敏感数据

身份验证令牌等敏感数据应存储在 `<secret-name>` Secret 中，并可在服务配置中使用 `$<secret-key>` 格式引用。例如，"$slack-token "引用了 `<secret-name>` Secret 中的密钥 "slack-token "的值。

## 自定义名称

服务自定义名称允许配置相同服务类型的两个实例。

```yaml
service.slack.workspace1: |
    token: $slack-token-workspace1
  service.slack.workspace2: |
    token: $slack-token-workspace2
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.workspace1: my-channel
    notifications.argoproj.io/subscribe.on-sync-succeeded.workspace2: my-channel
```

## 服务类型

* [AwsSqs](./awssqs.md)
* [电子邮件](./email.md)
* [GitHub](./github.md)
* [Slack](./slack.md)
* [Mattermost](./mattermost.md)
* [Opsgenie](./opsgenie.md)
* [Grafana](./grafana.md)
* [Webhook](./webhook.md)
* [Telegram](./telegram.md)
* [团队](./teams.md)
* [谷歌聊天](./googlechat.md)
* [Rocket.Chat](./rocketchat.md)
* [Pushover](./pushover.md)
* [Alertmanager](./alertmanager.md)
