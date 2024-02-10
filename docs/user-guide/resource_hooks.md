<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 资源挂钩

## 概览

可以使用资源钩子配置同步。 钩子是在同步操作之前、期间和之后运行脚本的方法。 如果同步操作在任何时候失败，也可以运行钩子。

* 使用 "PreSync "钩子，在部署新版本的应用程序之前执行数据库模式迁移。

Kubernetes 滚动更新策略

* 使用 `PostSync` 钩子在部署后运行集成和健康检查。 * 使用 `SyncFail` 钩子在同步操作失败时运行清理或定稿逻辑。 * 使用 `PostDelete` 钩子在删除所有应用程序资源后运行清理或定稿逻辑。 请注意，只有在删除策略与聚合删除钩子状态相匹配时，才会删除 `PostDelete` 钩子，而不会在删除应用程序后进行垃圾收集。

## 使用方法

钩子只是在 Argo CD 应用程序源代码库中跟踪的 Kubernetes 清单，并注有`argocd.argoproj.io/hook`例如

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: schema-migrate-
  annotations:
    argocd.argoproj.io/hook: PreSync
```

在同步操作中，Argo CD 会在部署的适当阶段应用该资源。 钩子可以是任何类型的 Kubernetes 资源种类，但倾向于 Pod、[工作](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)或[Argo 工作流程](https://github.com/argoproj/argo)可以用逗号分隔的列表指定多个钩子。

定义了以下钩子：

| | 说明 | |------|-------------| | |`PreSync`| 在应用清单之前执行。`Sync`| 终究还是执行了`PreSync`在应用清单的同时，钩子也完成并成功。`Skip`| 表示 Argo CD 跳过应用清单。`PostSync`| 终究还是执行了`Sync`勾子完成并成功，申请成功，并且所有资源都在一个`Healthy`状态。`SyncFail`| 同步操作失败时执行。`PostDelete`| 删除所有应用程序资源后执行。从 v2.10._ 开始提供。

#### 生成名称

命名挂钩（即带有`/metadata/name`如果希望每次都重新创建钩子，可以使用`BeforeHookCreation`政策（见下文）或`/metadata/generateName`。

## 选择性同步

在[选择性同步](selective_sync.md).

## 钩子删除政策

钩子可通过注释被自动删除：`argocd.argoproj.io/hooks-delete-policy`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: integration-test-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

可以用逗号分隔的列表指定多个钩子删除策略。

以下策略规定了何时删除挂钩。

| 政策 | 说明 | |--------|-------------| |`HookSucceeded`| 挂钩资源在挂钩成功（如作业/工作流成功完成）后被删除。`HookFailed`| 挂钩失败后，挂钩资源将被删除。`BeforeHookCreation`| 在创建新钩子之前，任何现有的钩子资源都会被删除（自 v1.3 起）。 它被引用与`/metadata/name`.

请注意，如果未指定删除策略，Argo CD 将自动假定`BeforeHookCreation`规则

### 使用生存时间（ttl）同步作业/工作流程的状态

工作岗位支持[ttlSecondsAfterFinished](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)字段，让各自的控制器在作业完成后删除作业。 Argo 工作流支持一个[ttlStrategy](https://argoproj.github.io/argo-workflows/fields/#ttlstrategy)属性，该属性还允许根据所选的 ttl 策略清理工作流。

使用上述任一属性都可能导致应用程序不同步（OutOfSync），这是因为 Argo CD 会检测到 git 仓库中定义的作业或工作流与集群上的作业或工作流之间存在差异，因为 ttl 属性会导致资源在完成后被删除。

不过，使用删除钩子而不是上述的 ttl 方法可以防止应用程序出现 OutOfSync 状态，即使作业或工作流在完成后已被删除。

## 使用钩子发送 Slack 消息

下面的示例使用 Slack API 在同步完成或失败时发送 Slack 消息：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: app-slack-notification-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: slack-notification
        image: curlimages/curl
        command:
          - "curl"
          - "-X"
          - "POST"
          - "--data-urlencode"
          - "payload={\"channel\": \"#somechannel\", \"username\": \"hello\", \"text\": \"App Sync succeeded\", \"icon_emoji\": \":ghost:\"}"
          - "https://hooks.slack.com/services/..."
      restartPolicy: Never
  backoffLimit: 2
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: app-slack-notification-fail-
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: slack-notification
        image: curlimages/curl
        command: 
          - "curl"
          - "-X"
          - "POST"
          - "--data-urlencode"
          - "payload={\"channel\": \"#somechannel\", \"username\": \"hello\", \"text\": \"App Sync failed\", \"icon_emoji\": \":ghost:\"}"
          - "https://hooks.slack.com/services/..."
      restartPolicy: Never
  backoffLimit: 2
```