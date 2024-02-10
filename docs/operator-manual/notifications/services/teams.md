<!-- TRANSLATED by md-translate -->
# 团队

## 参数

团队通知服务使用团队机器人发送消息通知，需要指定以下设置：

* `recipientUrls` - 网络钩子网址映射，例如 `channelName: https://example.com`

## 配置

1.打开 "团队"，转到 "应用程序"。
2.找到 `Incoming Webhook` microsoft 应用程序并点击它
3.按 "添加到团队 "键 -&gt; 选择团队和频道 -&gt; 按 "设置连接器 "键
4.输入 webhook 名称并上传镜像（可选）
5.按 "创建"，然后复制 webhook 网址并将其存储在 "argocd-notifications-secret "中，并在 "argocd-notifications-cm "中对其进行定义

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.teams: |
    recipientUrls:
      channelName: $channel-teams-url
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  channel-teams-url: https://example.com
```

6.为团队集成创建订阅：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.teams: channelName
```

## 模板

![](https://user-images.githubusercontent.com/18019529/114271500-9d2b8880-9a4c-11eb-85c1-f6935f0431d5.png)

[通知模板](.../templates.md) 可以自定义，以利用团队消息部分、事实、themeColor、摘要和 potentialAction [功能](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using)。

```yaml
template.app-sync-succeeded: |
  teams:
    themeColor: "#000080"
    sections: |
      [{
        "facts": [
          {
            "name": "Sync Status",
            "value": "{{.app.status.sync.status}}"
          },
          {
            "name": "Repository",
            "value": "{{.app.spec.source.repoURL}}"
          }
        ]
      }]
    potentialAction: |-
      [{
        "@type":"OpenUri",
        "name":"Operation Details",
        "targets":[{
          "os":"default",
          "uri":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true"
        }]
      }]
    title: Application {{.app.metadata.name}} has been successfully synced
    text: Application {{.app.metadata.name}} has been successfully synced at {{.app.status.operationState.finishedAt}}.
    summary: "{{.app.metadata.name}} sync succeeded"
```

###事实领域

您可以使用 "facts "字段代替 "sections "字段。

```yaml
template.app-sync-succeeded: |
  teams:
    facts: |
      [{
        "name": "Sync Status",
        "value": "{{.app.status.sync.status}}"
      },
      {
        "name": "Repository",
        "value": "{{.app.spec.source.repoURL}}"
      }]
```

### 主题颜色区域

您可以将主题颜色设置为信息的十六进制字符串。

![](https://user-images.githubusercontent.com/1164159/114864810-0718a900-9e24-11eb-8127-8d95da9544c1.png)

```yaml
template.app-sync-succeeded: |
  teams:
    themeColor: "#000080"
```

#### 摘要栏

您可以设置将显示在通知和活动反馈上的信息摘要

![](https://user-images.githubusercontent.com/6957724/116587921-84c4d480-a94d-11eb-9da4-f365151a12e7.jpg)

![](https://user-images.githubusercontent.com/6957724/116588002-99a16800-a94d-11eb-807f-8626eb53b980.jpg)

```yaml
template.app-sync-succeeded: |
  teams:
    summary: "Sync Succeeded"
```