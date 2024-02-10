<!-- TRANSLATED by md-translate -->
# 电子邮件

## 参数

电子邮件通知服务使用 SMTP 协议发送电子邮件通知，需要指定以下设置：

* host` - SMTP 服务器主机名
* `port` - SMTP 服务器端口
* `username` - 用户名
* `password` - 密码
* `from` - 发件人电子邮件地址
* `html` - 可选的 bool，true 或 false
* `insecure_skip_verify` - 可选 bool，true 或 false

## 示例

以下代码段包含 Gmail 服务配置示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.email.gmail: |
    username: $email-username
    password: $email-password
    host: smtp.gmail.com
    port: 465
    from: $email-username
```

无需验证：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.email.example: |
    host: smtp.example.com
    port: 587
    from: $email-username
```

## 模板

[通知模板]（.../templates.md）支持为电子邮件通知指定主题：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  template.app-sync-succeeded: |
    email:
      subject: Application {{.app.metadata.name}} has been successfully synced.
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} has been successfully synced at {{.app.status.operationState.finishedAt}}.
      Sync operation details are available at: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true .
```