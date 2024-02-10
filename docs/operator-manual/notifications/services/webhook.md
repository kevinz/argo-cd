<!-- TRANSLATED by md-translate -->
# Webhook

Webhook 通知服务允许使用模板化的请求正文和 URL 发送通用 HTTP 请求。 使用 Webhook，您可以触发 Jenkins 作业，更新 GitHub 提交状态。

## 参数

Webhook 通知服务配置包括以下设置：

* `url` - 将网络钩子发送到的网址
* `headers` - 可选，与 webhook 一起传递的头信息
* `basicAuth` - 可选项，与 webhook 一起传递的基本身份验证信息
* `insecureSkipVerify` - 可选的 bool，true 或 false
* `retryWaitMin` - 可选，重试之间的最短等待时间。默认值：1s。
* `retryWaitMax` - 可选，重试之间的最长等待时间。默认值：5 秒。
* `retryMax` - 可选项，重试的最大次数。默认值： 3.

## 重试行为

如果由于网络错误或服务器返回 5xx 状态代码导致请求失败，webhook 服务将自动重试请求。 重试次数和重试之间的等待时间可使用 `retryMax`、`retryWaitMin` 和 `retryWaitMax` 参数进行配置。

重试之间的等待时间介于 `retryWaitMin` 和 `retryWaitMax` 之间。 如果所有重试都失败，"发送 "方法将返回错误。

## 配置

使用以下步骤配置 webhook：

1 在 `argocd-notifications-cm` configMap 中注册 webhook：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.<webhook-name>: |
    url: https://<hostname>/<optional-path>
    headers: #optional headers
    - name: <header-name>
      value: <header-value>
    basicAuth: #optional username password
      username: <username>
      password: <api-key>
    insecureSkipVerify: true #optional bool
```

2 定义模板，自定义 webhook 请求方法、路径和正文：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  template.github-commit-status: |
    webhook:
      <webhook-name>:
        method: POST # one of: GET, POST, PUT, PATCH. Default value: GET 
        path: <optional-path-template>
        body: |
          <optional-body-template>
  trigger.<trigger-name>: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [github-commit-status]
```

3 为 webhook 集成创建订阅：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.<trigger-name>.<webhook-name>: ""
```

## 示例

#### 设置 GitHub 提交状态

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.github: |
    url: https://api.github.com
    headers: #optional headers
    - name: Authorization
      value: token $github-token
```

2 定义模板，自定义 webhook 请求方法、路径和正文：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.github: |
    url: https://api.github.com
    headers: #optional headers
    - name: Authorization
      value: token $github-token

  template.github-commit-status: |
    webhook:
      github:
        method: POST
        path: /repos/{{call .repo.FullNameByRepoURL .app.spec.source.repoURL}}/statuses/{{.app.status.operationState.operation.sync.revision}}
        body: |
          {
            {{if eq .app.status.operationState.phase "Running"}} "state": "pending"{{end}}
            {{if eq .app.status.operationState.phase "Succeeded"}} "state": "success"{{end}}
            {{if eq .app.status.operationState.phase "Error"}} "state": "error"{{end}}
            {{if eq .app.status.operationState.phase "Failed"}} "state": "error"{{end}},
            "description": "ArgoCD",
            "target_url": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
            "context": "continuous-delivery/{{.app.metadata.name}}"
          }
```

### 开始 Jenkins 工作

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.jenkins: |
    url: http://<jenkins-host>/job/<job-name>/build?token=<job-secret>
    basicAuth:
      username: <username>
      password: <api-key>

type: Opaque
```

### 发送表单数据

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.form: |
    url: https://form.example.com
    headers:
    - name: Content-Type
      value: application/x-www-form-urlencoded

  template.form-data: |
    webhook:
      form:
        method: POST
        body: key1=value1&key2=value2
```

### 发送 Slack

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.webhook.slack_webhook: |
    url: https://hooks.slack.com/services/xxxxx
    headers:
    - name: Content-Type
      value: application/json

  template.send-slack: |
    webhook:
      slack_webhook:
        method: POST
        body: |
          {
            "attachments": [{
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
          }
```