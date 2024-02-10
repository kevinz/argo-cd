<!-- TRANSLATED by md-translate -->
通知模板用于生成通知内容，并在 `argocd-notifications-cm` ConfigMap 中进行配置。模板利用 [html/template](https://golang.org/pkg/html/template/) golang 包，允许自定义通知消息。 模板旨在可重用，并可被多个触发器引用。

以下模板被用来通知用户应用程序的同步状态。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  template.my-custom-template-slack-template: |
    message: |
      Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
      Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
```

每个模板都可以访问以下字段：

* `app` 保存应用程序对象。
* `context` 是用户定义的字符串映射，可能包括任何字符串键和值。
* `secrets` 提供对存储在 `argocd-notifications-secret` 中的敏感数据的访问权限
* `serviceType` 保存通知服务类型名称（如 "slack "或 "email"）。该字段可用于有条件地

呈现特定服务字段。

* `recipient` 保存收件人名称。

## 定义用户定义的 `语境

通过设置一个包含键值对的顶级 YAML 文档，可以在所有通知模板之间定义一些共享上下文，然后可以在模板内被引用，就像这样：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  context: |
    region: east
    environmentName: staging

  template.a-slack-template-with-context: |
    message: "Something happened in {{ .context.environmentName }} in the {{ .context.region }} data center!"
```

## 在通知模板中定义和被引用秘密

某些通知服务用例需要在模板中使用秘密，这可以通过使用模板中的 "secrets "数据变量来实现。

鉴于我们有以下 `argocd-notifications-secret`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
stringData:
  sampleWebhookToken: secret-token 
type: Opaque
```

我们可以在模板中这样引用已定义的 `sampleWebhookToken` ：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  template.trigger-webhook: |
      webhook:
        sample-webhook:
          method: POST
          path: 'webhook/endpoint/with/auth'
          body: 'token={{ .secrets.sampleWebhookToken }}&variables[APP_SOURCE_PATH]={{ .app.spec.source.path }}
```

## 通知服务特定字段

模板定义的 "消息 "字段允许为任何通知服务创建基本通知。 您可以利用通知服务的特定字段创建复杂的通知。 例如，使用特定服务，您可以为 Slack 添加块和附件，为电子邮件或 URL 路径添加主题，为 Webhook 添加正文。 更多信息请参阅相应的服务 [文档](services/overview.md)。

## 更改时区

您可以按以下方式更改通知中显示的时区。

1.调用时间功能。
    ```
    {{ (call .time.Parse .app.status.operationState.startedAt).Local.Format "2006-01-02T15:04:05Z07:00" }}
    ```
2.在 argocd-notifications-controller 容器上设置 `TZ` 环境变量。
    ```yaml
    apiVersion: apps/v1
    种类：部署
    元数据：
      name: argocd-notifications-controller
    spec：
      template：
        spec：
          containers：
          - name: argocd-notifications-controller
            env：
            - name: TZ
              Values：亚洲/东京
    ```

## 功能

模板可以访问内置函数集：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  template.my-custom-template-slack-template: |
    message: "Author: {{(call .repo.GetCommitMetadata .app.status.sync.revision).Author}}"
```

{! docs/operator-manual/notifications/functions.md! }