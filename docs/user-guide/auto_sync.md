<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 自动同步政策

当 Argo CD 检测到 Git 中的预期清单与集群中的实时状态存在差异时，它能够自动同步应用程序。 自动同步的一个好处是，CI/CD 管道不再需要直接访问 Argo CD API 服务器来执行部署。 相反，管道会进行提交，并将跟踪 Git 仓库中清单的变更推送到 Git 仓库。

配置自动同步运行：

```bash
argocd app set <APPNAME> --sync-policy automated
```

或者，如果创建应用程序清单，可指定一个同步策略，该策略应带有`automated`政策。

```yaml
spec:
  syncPolicy:
    automated: {}
```

## 自动修剪

默认情况下（作为一种安全机制），当 Argo CD 检测到 Git 中不再定义资源时，自动同步将不会删除该资源。 要修剪资源，可始终执行手动同步（选中修剪）。 也可通过运行启用修剪，使其作为自动同步的一部分自动执行： Argo CD。

```bash
argocd app set <APPNAME> --auto-prune
```

或者在自动同步策略中将剪枝选项设置为 true： true

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
```

## 使用 Allow-Empty 自动剪枝（v1.8）

默认情况下（作为一种安全机制），当没有目标资源时，使用剪枝的自动同步具有防止任何自动/人为错误的功能。 它可以防止应用程序出现空资源。 要允许应用程序出现空资源，请运行

```bash
argocd app set <APPNAME> --allow-empty
```

或者在自动同步策略中将允许清空选项设置为 true： true

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      allowEmpty: true
```

## 自动自愈

默认情况下，对实时集群所做的更改不会触发自动同步。 要在实时集群的状态偏离 Git 中定义的状态时启用自动同步，请运行

```bash
argocd app set <APPNAME> --self-heal
```

或者在自动同步策略中将自愈选项设置为 true： true

```yaml
spec:
  syncPolicy:
    automated:
      selfHeal: true
```

## 自动同步语义

* 只有当应用程序处于 "不同步"（OutOfSync）状态时，才会执行自动同步。 处于 "已同步"（Synced）或错误状态的应用程序不会尝试自动同步。 如果历史记录中最近一次成功的同步已针对相同的提交-SHA 和参数执行，则不会尝试第二次同步，除非 "自愈"（selfHeal）标志设置为 true。

由`--selfheal-timeout-seconds`国旗`argocd-application-controller`部署。

* 自动同步将不会重新尝试同步，如果之前针对相同提交-SHA 和参数的同步尝试失败的话。 自动同步的时间间隔由[`argocd-cm` 配置表中的 `timeout.reconciliation` 值](../faq.md#how-often-does-argo-cd-check-for-changes-to-my-git-or-helm-repository)决定，默认值为 `180s`（3 分钟）。
