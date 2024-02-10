<!-- TRANSLATED by md-translate -->
# 集群引导

本指南适用于已安装 Argo CD 的操作员，他们拥有一个新的集群，并希望在该集群中安装许多应用程序。

要解决这个问题，并没有一个特定的模式，例如，你可以编写一个脚本来创建应用程序，甚至可以手动创建。 不过，Argo CD 的用户倾向于使用应用程序的***模式。

在任意[项目](./declarative-setup.md#projects)中创建应用程序的能力是管理员级别的能力。 只有管理员才有推送访问父应用程序源代码库的权限。 管理员应审查对该源代码库的拉取请求，特别注意每个应用程序中的 "项目 "字段。 可以访问安装 Argo CD 的命名空间的项目实际上拥有管理员级别的权限。

## Apps Of Apps Pattern

[声明式](declarative-setup.md) 指定一个 Argo CD 应用程序，该程序只由其他应用程序组成。

应用程序的应用](.../assets/application-of-applications.png)

### Helm 示例

本例展示了如何使用 helm 来实现这一目标。 当然，如果您愿意，也可以使用其他工具。

为此，Git 仓库的典型布局可以是

```
├── Chart.yaml
├── templates
│   ├── guestbook.yaml
│   ├── helm-dependency.yaml
│   ├── helm-guestbook.yaml
│   └── kustomize-guestbook.yaml
└── values.yaml
```

Chart.yaml` 是模板。

templates "大致为每个子应用程序包含一个文件：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: argocd
    server: {{ .Values.spec.destination.server }}
  project: default
  source:
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
```

同步策略为自动+剪枝，因此子应用会在配置清单发生变化时自动创建、同步和删除，但你可能希望禁用这一点。 我还添加了Finalizer，它可以确保你的应用被正确删除。

将修订版本固定为特定的 Git 提交 SHA，以确保即使子应用程序的 repo 发生变化，应用程序也只会在父应用程序更改该修订版本时才会发生变化。 或者，也可以将其设置为 HEAD 或分支名称。

由于您可能希望覆盖集群服务器，因此这是一个模板值。

`values.yaml` 包含默认值：

```yaml
spec:
  destination:
    server: https://kubernetes.default.svc
```

接下来，您需要创建并同步父应用程序，例如通过 CLI：

```bash
argocd app create apps \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/argoproj/argocd-example-apps.git \
    --path apps  
argocd app sync apps
```

父应用程序将显示为同步，但子应用程序将不同步：

新应用中的新应用](.../assets/new-app-of-apps.png)

&gt; 注意：您可能需要修改此行为，以便以波浪方式启动集群；有关修改的信息，请参阅 [v1.8 升级说明](upgrading/1.7-1.8.md)。

您可以通过用户界面进行同步，首先根据正确的标签进行筛选：

过滤器应用程序](../assets/filter-apps.png)

然后选择 "不同步 "应用程序并同步：

同步应用程序](.../assets/sync-apps.png)

或通过 CLI：

```bash
argocd app sync -l app.kubernetes.io/instance=apps
```

查看 [GitHub 上的示例](https://github.com/argoproj/argocd-example-apps/tree/master/apps)。

### 级联删除.

如果要确保在删除父应用程序时删除子应用程序及其所有资源，请确保在 "应用程序 "定义中添加适当的 [finalizer](../user-guide/app_deletion.md#about-the-deletion-finalizer)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
 ...
```