<!-- TRANSLATED by md-translate -->
# 通知概述

Argo CD Notifications 可持续监控 Argo CD 应用程序，并提供灵活的方式通知用户应用程序状态的重要变化。 使用灵活的 [触发器](triggers.md) 和 [模板](templates.md) 机制，您可以配置何时发送通知以及通知内容。 Argo CD Notifications 包含有用的触发器和模板的 [目录](catalog.md)。 因此，您只需引用这些触发器和模板，而无需重新发明新的触发器和模板。

## 开始

* 从目录中安装触发器和模板
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications_catalog/install.yaml
    ```
* 在 `argocd-notifications-secret` secret 中添加电子邮件用户名和密码令牌
    ```bash
    EMAIL_USER=<your-username>
    密码=<your-password>
    
    kubectl apply -n argocd -f - &lt;&lt; EOF
    apiVersion: v1
    kind: Secret
    元数据：
      name: argocd-notifications-secret
    stringData：
      电子邮件用户名：$EMAIL_USER
      电子邮件密码：$PASSWORD
    类型：不透明
    EOF
    ```
* 注册电子邮件通知服务
    ```bash
    kubectl patch cm argocd-notifications-cm -n argocd --type merge -p '{"data"：{"service.email.gmail"："{ username: $email-username, password: $email-password, host: smtp.gmail.com, port：465, from: $email-username }"}}'
    ```
* 通过在 Argo CD 应用程序或项目中添加 `notations.argoproj.io/subscribe.on-sync-succeeded.slack` 注解来订阅通知：
    ``bash
    kubectl patch app<my-app> -n argocd -p '{"metadata"：{"Annotations"：{"notifications.argoproj.io/subscribe.on-sync-succeeded.slack":"<my-channel>"}}}' --合并类型
    ```

尝试同步应用程序，以便在同步完成后获得通知。

## 基于 namespace 的配置

Argo CD 通知的常见安装方法是将其安装在管理整个集群的专用命名空间中。 在这种情况下，一般只有管理员可以在该命名空间中配置通知。 但在某些情况下，需要允许最终用户为其 Argo CD 应用程序配置通知。 例如，最终用户可以在其可以访问且其 Argo CD 应用程序正在运行的命名空间中为其 Argo CD 应用程序配置通知。

该功能基于任意名称空间中的应用程序。 更多信息请参见 [任意名称空间中的应用程序](../app-any-namespace.md) 页面。

要启用此功能，Argo CD 管理员必须重新配置 argocd-notification-controller 工作负载，在容器的启动命令中添加 `--application-namespaces` 和 `-self-service-notification-enabled` 参数。

通过在 argocd-cmd-params-cm ConfigMap 中指定 "application.namespaces "和 "notificationscontroller.selfservice.enabled"，而不是更改相应工作负载的配置清单，也可以方便地设置两者的启动参数并保持同步。 例如：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  application.namespaces: app-team-one, app-team-two
  notificationscontroller.selfservice.enabled: "true"
```

要使用此功能，您可以在 Argo CD 应用程序所在的 namespace 中部署名为 `argocd-notifications-cm` 的 configmaps 和可能的 secret `argocd-notifications-secret` 。

这样配置后，控制器将使用控制器级配置（与控制器位于同一 namespace 中的 configmap）以及 Argo CD 应用程序所在的同一 namespace 中的配置发送通知。

示例：应用团队希望使用 PagerDutyV2 接收通知，而控制器级配置只支持 Slack。

以下两个资源部署在 Argo CD 应用程序所在的 namespace 中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.pagerdutyv2: |
    serviceKeys:
      my-service: $pagerduty-key-my-service
...
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argo-cd-notification-secret
type: Opaque
data:
  pagerduty-key-my-service: <pd-integration-key>
```

当 Argo CD 应用程序有以下订阅时，用户会收到呼叫器值班发送的应用程序同步失败消息。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.pagerdutyv2: "<serviceID for Pagerduty>"
```

注意 当控制器级配置和应用程序级配置中定义了相同的通知服务和触发器时，将根据各自的配置发送通知。

[在通知模板中定义和使用秘密](templates.md/#defining-and-using-secrets-within-notification-templates) 功能在启用"-self-service-notification-enable "标志时不可用。