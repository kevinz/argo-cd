<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

对 Argo CD 应用程序事件的引用可通过以下方式定义`notifications.argoproj.io/subscribe.<trigger>.<service>:<recipient>`例如，下面的注解订阅了两个 Slack 频道，以接收 Argo CD 应用程序每次成功同步的通知： Argo CD 应用程序每次成功同步的通知

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel1;my-channel2
```

注释密钥由以下部分组成：

* `on-sync-succeeded` - 触发器名称 * `slack` - 通知服务名称 * `my-channel1;my-channel2` - 以分号分隔的收件人列表

通过在 AppProject CRD 中添加相同的注释，可以为 Argo CD 项目的所有应用程序创建订阅： AppProject CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel1;my-channel2
```

## 默认订阅

订阅可以在`argocd-notifications-cm`被引用的配置Map`subscriptions`默认订阅被引用到所有应用程序。 触发器和应用程序可以使用`triggers`和`selector`领域：

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

如果要在订阅中使用 webhook，则需要将自定义名称存储到收件人中。

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