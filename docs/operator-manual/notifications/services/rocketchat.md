<!-- TRANSLATED by md-translate -->
# Rocket.Chat

## 参数

Rocket.Chat 通知服务配置包括以下设置：

* `email` - Rocker.Chat 用户的电子邮件地址
* `password` - Rocker.Chat 用户的密码
* `alias` - 用于发布信息的可选别名
* icon` - 可选的信息图标
* `avatar` - 可选的消息头像
* `serverUrl` - 可选的 Rocket.Chat 服务器网址

## 配置

1.登录您的 RocketChat 实例
2.进入用户管理

![2](https://user-images.githubusercontent.com/15252187/115824993-7ccad900-a411-11eb-89de-6a0c4438ffdf.png)

3.添加具有 `bot` 角色的新用户。还请注意，"要求更改密码 "复选框必须不勾选

![3](https://user-images.githubusercontent.com/15252187/115825174-b4d21c00-a411-11eb-8f20-cda48cea9fad.png)

4.复制为僵尸用户创建的用户名和密码
5.创建一个公共或私人频道，或一个团队，例如`my_channel`。
6.将机器人添加到该频道 **否则将无法运行**
7.在 argocd_notifications-secret Secret 中存储电子邮件和密码

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  rocketchat-email: <email>
  rocketchat-password: <password>
```

8.最后，在 `argocd-configmap` 配置映射中使用这些凭据配置 RocketChat 集成：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.rocketchat: |
    email: $rocketchat-email
    password: $rocketchat-password
```

9.为 Rocket.Chat 集成创建订阅：

注意：频道、团队或用户必须以 # 或 @ 为前缀，否则我们将把目的地解释为房间 ID_。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.rocketchat: #my_channel
```

## 模板

[通知模板](.../templates.md)可与 RocketChat [attachments](https://developer.rocket.chat/api/rest-api/methods/chat/postmessage#attachments-detail) 一起定制。

_注：Rocketchat 中的附件结构与 Slack 附件 [feature](https://api.slack.com/messaging/composing/layouts) 相同。_

<!-- TODO: @sergeyshevch Need to add screenshot with RocketChat attachments -->

邮件附件可在 `rocketchat` 字段下的 `attachments` 字符串字段中指定：

```yaml
template.app-sync-status: |
  message: |
    Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
    Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
  rocketchat:
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