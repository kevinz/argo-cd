<!-- TRANSLATED by md-translate -->
# Mattermost

## 参数

* `apiURL` - 服务器网址，例如 https://mattermost.example.com
* token` - 机器人令牌
* `insecureSkipVerify` - 可选的 bool，true 或 false

## 配置

1.创建机器人账户，并在创建后复制令牌

![1](https://user-images.githubusercontent.com/18019529/111499520-62ed0500-8786-11eb-88b0-d0aade61fed4.png) 2. 邀请团队 ![2](https://user-images.githubusercontent.com/18019529/111500197-1229dc00-8787-11eb-98e5-587ee36c94a9.png) 3. 在 `argocd-notifications-secret` Secret 中存储令牌，并在 `argocd-notifications-cm` ConfigMap 中配置 Mattermost 集成

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.mattermost: |
    apiURL: <api-url>
    token: $mattermost-token
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  mattermost-token: token
```

4.复制频道 ID

![4](https://user-images.githubusercontent.com/18019529/111501289-333efc80-8788-11eb-9731-8353170cd73a.png)

5.为您的 Mattermost 整合创建订阅

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.<trigger-name>.mattermost: <channel-id>
```

## 模板

![](https://user-images.githubusercontent.com/18019529/111502636-5fa74880-8789-11eb-97c5-5eac22c00a37.png)

您可以重复使用 Slack 的模板。 Mattermost 与 Slack 的附件兼容。 参见 [Mattermost 集成指南](https://docs.mattermost.com/developer/message-attachments.html)。

```yaml
template.app-deployed: |
  message: |
    Application {{.app.metadata.name}} is now running new version of deployments manifests.
  mattermost:
    attachments: |
      [{
        "title": "{{.app.metadata.name}}",
        "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
        "color": "#18be52",
        "fields": [{
          "title": "Sync Status",
          "value": "{{.app.status.sync.status}}",
          "short": true
        }, {
          "title": "Repository",
          "value": "{{.app.spec.source.repoURL}}",
          "short": true
        }]
      }]
```