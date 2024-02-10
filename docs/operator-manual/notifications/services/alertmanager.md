<!-- TRANSLATED by md-translate -->
# 警报管理器

## 参数

通知服务被用来向 [Alertmanager](https://github.com/prometheus/alertmanager) 推送事件，需要指定以下设置：

* `targets` - 警报管理器服务地址，数组类型
* `scheme` - 可选，默认为 "http"，例如 http 或 https
* `apiPath` - 可选，默认为"/api/v2/alerts"。
* `insecureSkipVerify` - 可选，默认为 "false"，当方案为 https 时，是否跳过 ca 验证
* `basicAuth` - 可选，服务器验证
* bearerToken` - 可选，服务器验证
* `timeout` - 可选，发送警报时使用的超时（秒），默认为 "3 秒

如果同时设置了 "basicAuth "和 "bearerToken"，则 "basicAuth "优先于 "bearerToken"。

## 示例

### Prometheus Alertmanager 配置

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'
receivers:
- name: 'default'
  webhook_configs:
  - send_resolved: false
    url: 'http://10.5.39.39:10080/api/alerts/webhook'
```

应关闭 "send_resolved"，否则在 "resolve_timeout "后会收到不必要的恢复通知。

### Send one alertmanager without auth

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.alertmanager: |
    targets:
    - 10.5.39.39:9093
```

### 使用自定义 api 路径发送警报管理器集群

如果警报管理器更改了默认 api，则可以自定义 "apiPath"。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.alertmanager: |
    targets:
    - 10.5.39.39:443
    scheme: https
    apiPath: /api/events
    insecureSkipVerify: true
```

### 通过授权发送高可用性警报管理器

在 `argocd-notifications-secret` Secret 中存储认证令牌，并在 `argocd-notifications-cm` ConfigMap 中被引用。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  alertmanager-username: <username>
  alertmanager-password: <password>
  alertmanager-bearer-token: <token>
```

* 使用 basicAuth

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.alertmanager: |
    targets:
    - 10.5.39.39:19093
    - 10.5.39.39:29093
    - 10.5.39.39:39093
    scheme: https
    apiPath: /api/v2/alerts
    insecureSkipVerify: true
    basicAuth:
      username: $alertmanager-username
      password: $alertmanager-password
```

* 有 bearerToken

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.alertmanager: |
    targets:
    - 10.5.39.39:19093
    - 10.5.39.39:29093
    - 10.5.39.39:39093
    scheme: https
    apiPath: /api/v2/alerts
    insecureSkipVerify: true
    bearerToken: $alertmanager-bearer-token
```

## 模板

* `labels` - 至少需要一个标签对，根据警报管理器路由实现不同的通知策略
* `annotations` - 可选，指定一组信息标签，可用于存储更长的附加信息，但仅用于显示
* `generatorURL` - 可选，默认为"{{.app.spec.source.repoURL}}"，用于在客户端识别引起此警报的实体的反向链接

label "或 "Annotations "或 "generatorURL "值可以模板化。

```yaml
context: |
  argocdUrl: https://example.com/argocd

template.app-deployed: |
  message: Application {{.app.metadata.name}} has been healthy.
  alertmanager:
    labels:
      fault_priority: "P5"
      event_bucket: "deploy"
      event_status: "succeed"
      recipient: "{{.recipient}}"
    annotations:
      application: '<a href="{{.context.argocdUrl}}/applications/{{.app.metadata.name}}">{{.app.metadata.name}}</a>'
      author: "{{(call .repo.GetCommitMetadata .app.status.sync.revision).Author}}"
      message: "{{(call .repo.GetCommitMetadata .app.status.sync.revision).Message}}"
```

您可以根据标签在 [Alertmanager](https://github.com/prometheus/alertmanager) 上进行定向推送。

```yaml
template.app-deployed: |
  message: Application {{.app.metadata.name}} has been healthy.
  alertmanager:
    labels:
      alertname: app-deployed
      fault_priority: "P5"
      event_bucket: "deploy"
```

有一个特殊标签 `alertname` 如果不设置其值，默认情况下就等于模板名称。