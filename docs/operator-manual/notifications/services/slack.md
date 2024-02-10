<!-- TRANSLATED by md-translate -->
# Slack

如果要使用传入的 webhook 发送消息，可以使用 [webhook](./webhook.md#send-slack)。

## 参数

Slack 通知服务配置包括以下设置：

| **Option**           | **Required** | **Type**       | **Description** | **Example** |
| -------------------- | ------------ | -------------- | --------------- | ----------- |
| `apiURL`             | False        | `string`       | The server URL. | `https://example.com/api` |
| `channels`           | False        | `list[string]` |                 | `["my-channel-1", "my-channel-2"]` |
| `icon`               | False        | `string`       | The app icon.   | `:robot_face:` or `https://example.com/image.png` |
| `insecureSkipVerify` | False        | `bool`         |                 | `true` |
| `signingSecret`       | False        | `string`       |                 | `8f742231b10e8888abcd99yyyzzz85a5` |
| `token`              | **True**     | `string`       | The app's OAuth access token. | `xoxb-1234567890-1234567890123-5n38u5ed63fgzqlvuyxvxcx6` |
| `username`           | False        | `string`       | The app username. | `argocd` |

## 配置

1.使用 https://api.slack.com/apps?new_app=1 创建 Slack 应用程序

![1](https://user-images.githubusercontent.com/426437/73604308-4cb0c500-4543-11ea-9092-6ca6bae21cbb.png)

1.创建应用程序后，导航至 "输入 OAuth 和权限

![2](https://user-images.githubusercontent.com/426437/73604309-4d495b80-4543-11ea-9908-4dea403d3399.png)

1.单击 "添加特性和功能 "部分下的 "权限 "并添加 "chat:write "作用域。要在 Slack 通知服务中使用可选的用户名和图标覆盖，还需添加 `chat:write.customize` 作用域。

![3](https://user-images.githubusercontent.com/426437/73604310-4d495b80-4543-11ea-8576-09cd91aea0e5.png)

1.返回顶部，点击 "将应用程序安装到工作区 "按钮并确认安装。

![4](https://user-images.githubusercontent.com/426437/73604311-4d495b80-4543-11ea-9155-9d216b20ec86.png)

1.安装完成后，复制 OAuth 令牌。

![5](https://user-images.githubusercontent.com/426437/73604312-4d495b80-4543-11ea-832b-a9d9d5e4bc29.png)

1.创建公共或私人频道，例如 `my_channel`
2.邀请你的松弛机器人进入该频道 **否则松弛机器人将无法向该频道发送通知**
3.在 `argocd-notifications-secret` secret 中存储 Oauth 访问令牌
    yaml
    apiVersion: v1
      kind: Secret
      元数据：
          name：<secret-name>
      stringData：
          slack-token：<Oauth-access-token>
    ```
4.在 `argocd-notifications-cm` configmaps 的数据部分定义服务类型 slack：
    ```yaml
    apiVersion: v1
      类型：configMap
      元数据：
        name：<config-map-name>
      data：
        service.slack：|
          token: $slack-token
    ```
5.在应用程序 yaml 文件中添加 Annotations，以启用针对特定 argocd 应用程序的通知。  以下示例被引用 [on-sync-succeeded trigger](../catalog.md#triggers)：
    ```yaml
    apiVersion: argoproj.io/v1alpha1
      种类：应用程序
      元数据：
        Annotations：
          notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my_channel
    ```
6.具有多个 [trigger](../catalog.md#triggers) 的 Annotations，具有多个目的地和接收者
    ```yaml
    apiVersion: argoproj.io/v1alpha1
      种类：应用程序
      元数据：
        Annotations：
          notifications.argoproj.io/subscriptions：|
            - trigger：[on-scaling-replica-set, on-rollout-updated, on-rollout-step-completed］
              destinations：
                - 服务： slack
                  收件人：[我的频道-1，我的频道-2］
                - 服务：电子邮件
                  收件人：[收件人-1、收件人-2、收件人-3 ]
            - 触发器：[on-rollout-aborted, on-analysis-run-failed, on-analysis-run-error] （分析运行失败时触发，分析运行错误时触发
              目的地：
                - 服务：松弛
                  收件人：[我的频道-21，我的频道-22］
    ```

## 模板

[通知模板](.../templates.md) 可进行自定义，以利用松弛消息块和附件[功能](https://api.slack.com/messaging/composing/layouts)。

![](https://user-images.githubusercontent.com/426437/72776856-6dcef880-3bc8-11ea-8e3b-c72df16ee8e6.png)

信息块和附件可在`slack`字段下的`blocks`和`attachments`字符串字段中指定：

```yaml
template.app-sync-status: |
  message: |
    Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
    Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
  slack:
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

信息可以通过分组键聚合到松弛线程中，分组键可以在`slack`字段下的`groupingKey`字符串字段中指定。 当多个应用程序同时或频繁更新时，松弛通道中的信息可以通过聚合 git 提交哈希、应用程序名称等信息轻松读取。此外，信息还可以通过`notifyBroadcast`字段广播到特定模板的通道中。

```yaml
template.app-sync-status: |
  message: |
    Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
    Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
  slack:
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
    # Aggregate the messages to the thread by git commit hash
    groupingKey: "{{.app.status.sync.revision}}"
    notifyBroadcast: false
template.app-sync-failed: |
  message: |
    Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
    Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
  slack:
    attachments: |
      [{
        "title": "{{.app.metadata.name}}",
        "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
        "color": "#ff0000",
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
    # Aggregate the messages to the thread by git commit hash
    groupingKey: "{{.app.status.sync.revision}}"
    notifyBroadcast: true
```

信息根据 `slack` 字段下的 `deliveryPolicy` 字符串字段发送。 可用的模式有 `Post`（默认）、`PostAndUpdate` 和 `Update`。 `PostAndUpdate`和 `Update`设置需要设置 `groupingKey`。