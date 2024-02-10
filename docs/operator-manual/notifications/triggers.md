<!-- TRANSLATED by md-translate -->
触发器定义了应发送通知的条件。 定义包括名称、条件和通知模板引用。 条件是一个谓词表达式，如果应发送通知，则返回 true。 触发器条件评估由 [antonmedv/expr](https://github.com/antonmedv/expr) 提供。条件语言语法在 [language-definition.md](https://github.com/antonmedv/expr/blob/master/docs/language-definition.md) 中描述。

触发器在 `argocd-notifications-cm` ConfigMap 中配置。 例如，以下触发器使用 `app-sync-status` 模板在应用程序同步状态变为 `Unknown` 时发送通知：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.on-sync-status-unknown: |
    - when: app.status.sync.status == 'Unknown'     # trigger condition
      send: [app-sync-status, github-commit-status] # template names
```

每个条件可能会使用多个模板。 通常，每个模板负责生成特定服务的通知部分。 在上面的示例中，"app-sync-status "模板 "知道 "如何创建电子邮件和 Slack 通知，而 "github-commit-status "则知道如何为 GitHub webhook 生成有效载荷。

## 条件捆绑

触发器通常由管理员管理，封装了有关何时以及应发送哪种通知的信息。 最终用户只需订阅触发器并指定通知目的地即可。 为改善用户体验，触发器可能包含多个条件，每个条件有一套不同的模板。 例如，以下触发器涵盖同步状态操作的所有阶段，并针对不同情况使用不同的模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.sync-operation-change: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [github-commit-status]
    - when: app.status.operationState.phase in ['Running']
      send: [github-commit-status]
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed, github-commit-status]
```

### 避免频繁发送相同的通知

在某些情况下，触发器条件可能会 "闪烁"。 下面的示例说明了这个问题。 触发器本应在 Argo CD 应用程序成功同步且健康时生成一次通知。 然而，应用程序的健康状态可能会间歇性地切换为 "进行中"，然后又切换回 "健康"，因此触发器可能会不必要地生成多个通知。 每一次 "字段可将触发器配置为仅在相应的应用程序字段发生变化时才生成通知。 下面示例中的 "部署时 "触发器仅在每次观察到部署仓库的 Git 修订版本时发送一次通知。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  # Optional 'oncePer' property ensure that notification is sent only once per specified field value
  # E.g. following is triggered once per sync revision
  trigger.on-deployed: |
    when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
    oncePer: app.status.sync.revision
    send: [app-sync-succeeded]
```

**Mono Repo Usage**

当一个 repo 被用于同步多个应用程序时，"oncePer: app.status.sync.revision" 字段将为每次提交触发通知。 对于单 repo，更好的方法是使用 "oncePer: app.status.operationState.syncResult.revision" 语句。 这样，通知将只针对特定应用程序的修订。

### 一次

oncePer "文件的支持方式如下。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    example.com/version: v0.1
```

```yaml
oncePer: app.metadata.annotations["example.com/version"]
```

## 默认触发器

您可以使用 `defaultTriggers` 字段来代替为 Annotations 指定单个触发器。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  # Holds list of triggers that are used by default if trigger is not specified explicitly in the subscription
  defaultTriggers: |
    - on-sync-status-unknown

  defaultTriggers.mattermost: |
    - on-sync-running
    - on-sync-succeeded
```

指定以下 Annotations 以使用 `defaultTriggers`. 在本例中，`slack` 在 `on-sync-status-unknown` 时发送，`mattermost` 在 `on-sync-running` 和 `on-sync-succeeded` 时发送。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.slack: my-channel
    notifications.argoproj.io/subscribe.mattermost: my-mattermost-channel
```

## 功能

触发器可以访问内置函数集。

例如

```yaml
when: time.Now().Sub(time.Parse(app.status.operationState.startedAt)).Minutes() >= 5
```

{! docs/operator-manual/notifications/functions.md! }