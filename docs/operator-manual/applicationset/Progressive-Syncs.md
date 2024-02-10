<!-- TRANSLATED by md-translate -->
# 渐进式同步

警告 "alpha 功能" 这是一项实验性的 alpha 质量功能，可让您控制 ApplicationSet 控制器创建或更新 ApplicationSet 资源所拥有的应用程序的顺序。 它可能会在未来的发布版本中删除，或以向后兼容的方式修改。

## 被引用的案例

渐进同步 "功能集旨在轻便灵活。 该功能仅与受管应用程序的健康状况交互，不支持与其他 Rollout 控制器（如本地 ReplicaSet 控制器或 Argo Rollouts）直接集成。

* 渐进式同步在进入下一阶段之前，会观察所管理的应用程序资源是否处于 "健康 "状态。
* 部署、DaemonSets、StatefulSets 和 [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)均受支持，因为应用程序在推出 pod 时会进入 "Progressing "状态。事实上，任何具有可报告 "进行中 "状态的健康检查的资源都受支持。
* 支持 [Argo CD Resource Hooks]（.../.../user-guide/resource_hooks.md）。我们建议需要高级功能的用户在无法引用 Argo Rollout 时使用这种方法，例如在 DaemonSet 更改后进行烟雾测试。

## 启用渐进式同步

作为一项试验性功能，必须通过以下方式之一明确启用渐进式同步。

1.将 `--enable-progressive-syncs` 传递给 ApplicationSet 控制器参数。
2.在 ApplicationSet 控制器环境变量中设置 `ARGOCD_APPLICATIONSET_CONTROLLER_ENABLE_PROGRESSIVE_SYNCS=true` 。
3.在 Argo CD `argocd-cmd-params-cm` 配置表中设置`applicationsetcontroller.enable.progressive.syncs:true`。

## 策略

* AllAtOnce（默认）
* 滚动同步

### AllAtOnce

这种默认的 Application 更新行为与最初的 ApplicationSet 实现相比没有变化。

更新 ApplicationSet 时，ApplicationSet 资源管理的所有应用程序都会同时更新。

### 滚动同步

此更新策略允许您按生成的 Application 资源上存在的标签对 Application 进行分组。 当 ApplicationSet 发生变化时，变化将按顺序应用到每组 Application 资源。

* 应用程序组被引用其标签和 `matchExpressions`进行选择。
* 要选择应用程序，所有 `matchExpressions` 必须为 true（多个表达式以 AND 行为匹配）。
* In "和 "NotIn "操作符必须至少匹配一个值才视为真（OR 行为）。
* 如果 `NotIn` 和 `In` 操作符都产生匹配，则 `NotIn` 操作符具有优先权。
* 在 ApplicationSet 控制器继续更新下一组应用程序之前，每组中的所有应用程序都必须变为 "健康"。
* 组内同时更新的应用程序数量不会超过其 `maxUpdate`参数（默认为 100%，无限制）。
* RollingSync 将捕获 ApplicationSet 资源外的外部更改，因为它依赖于观察管理应用程序的 OutOfSync 状态。
* RollingSync 会强制所有生成的应用程序禁用自动同步。对于启用了自动同步策略的任何应用程序规格，都会在应用程序集控制器日志中打印警告。
* 同步操作的触发方式与用户界面或 CLI 的触发方式相同（直接设置应用程序资源上的 `operation` 状态字段）。这意味着 RollingSync 会尊重同步窗口，就像用户点击 Argo UI 中的 "同步 "按钮一样。
* 当触发同步时，同步将使用为应用程序配置的相同同步策略执行。例如，这将保留应用程序的重试设置。
* 如果应用程序在 `applicationsetcontroller.default.application.progressing.timeout` 秒内被视为 "待处理"，则应用程序会自动转为 "健康 "状态（默认为 300）。

#### 示例

下面的示例说明了如何通过明确配置的环境标签在应用程序上进行渐进式同步。

推送更改后，将依次进行以下操作。

* 所有 `env-dev` 应用程序将同时更新。
* 更新将等待所有 `env-qa` 应用程序通过 `argocd` CLI 或用户界面中的同步按钮进行手动同步。
* 每次将更新所有 `env-prod` 应用程序的 10%，直至所有 `env-prod` 应用程序更新完毕。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://1.2.3.4
        env: env-dev
      - cluster: engineering-qa
        url: https://2.4.6.8
        env: env-qa
      - cluster: engineering-prod
        url: https://9.8.7.6/
        env: env-prod
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - env-dev
          #maxUpdate: 100%  # if undefined, all applications matched are updated together (default is 100%)
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - env-qa
          maxUpdate: 0      # if 0, no matched applications will be updated
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - env-prod
          maxUpdate: 10%    # maxUpdate supports both integer and percentage string values (rounds down, but floored at 1 Application for >0%)
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
      labels:
        envLabel: '{{.env}}'
    spec:
      project: my-project
      source:
        repoURL: https://github.com/infra-team/cluster-deployments.git
        targetRevision: HEAD
        path: guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```