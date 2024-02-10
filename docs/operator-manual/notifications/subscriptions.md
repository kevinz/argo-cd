<!-- TRANSLATED by md-translate -->
对 Argo CD 应用程序事件的订阅可以使用 "notifications.argoproj.io/subscribe.<trigger>.<service>:<recipient>"注解来定义。例如，下面的注解将订阅两个 Slack 频道关于 Argo CD 应用程序每次成功同步的通知：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel1;my-channel2
```

Annotations 密钥由以下部分组成：

* `on-sync-succeeded` - 触发器名称
* `slack` - 通知服务名称
* `my-channel1;my-channel2` - 分号分隔的收件人列表

通过在 AppProject 资源中添加相同的 Annotations，可以为 Argo CD 项目的所有应用程序创建订阅：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel1;my-channel2
```

## 默认订阅

订阅可在 `argocd-notifications-cm` ConfigMap 中使用 `subscriptions` 字段进行全局配置。 默认订阅会被引用到所有应用程序。 触发器和应用程序可使用 `triggers` 和 `selector` 字段进行配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  # Contains centrally managed global application subscriptions
  subscriptions: |
    # subscription for on-sync-status-unknown trigger notifications
    - recipients:
      - slack:test2
      - email:test@gmail.com
      triggers:
      - on-sync-status-unknown
    # subscription restricted to applications with matching labels only
    - recipients:
      - slack:test3
      selector: test=true
      triggers:
      - on-sync-status-unknown
```

如果要在订阅中使用 webhook，则需要在订阅的 "接收者 "字段中存储自定义 webhook 名称。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.webhook.<webhook-name>: |
    (snip)
  subscriptions: |
    - recipients:
      - <webhook-name>
      triggers:
      - on-sync-status-unknown
```