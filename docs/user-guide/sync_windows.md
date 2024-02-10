<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 同步窗口

这些窗口会影响手动和自动同步的运行，但允许覆盖手动同步，如果您只对防止自动同步感兴趣，或者需要临时覆盖一个窗口来执行同步，这就非常有用了。

窗口的工作方式如下： 如果没有与应用程序匹配的窗口，则允许所有同步。 如果有任何窗口，则允许所有同步。`allow`窗口与应用程序相匹配，那么只有当有一个活动的`allow`窗口。`deny`窗口与应用程序相匹配时，所有同步将被拒绝。`deny`窗口处于活动状态。`allow`和主动匹配`deny`则会拒绝同步，因为`deny`窗口覆盖`allow`用户界面和 CLI 都会显示同步窗口的状态。 用户界面有一个面板，会根据状态显示不同的颜色。 颜色如下。`Red: sync denied`,`Orange:manual allowed`和`Green: sync allowed`。

要使用 CLI 显示同步状态：

```bash
argocd app get APP
```

将返回同步状态和任何匹配窗口。

```
Name:               guestbook
Project:            default
Server:             in-cluster
Namespace:          default
URL:                http://localhost:8080/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Denied
Assigned Windows:   deny:0 2 * * *:1h,allow:0 2 3 3 3:1h
Sync Policy:        Automated
Sync Status:        Synced to  (5c2d89b)
Health Status:      Healthy
```

可使用 CLI 创建窗口：

```bash
argocd proj windows add PROJECT \
    --kind allow \
    --schedule "0 22 * * *" \
    --duration 1h \
    --applications "*"
```

或者，也可以直接在`AppProject`体现：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
spec:
  syncWindows:
  - kind: allow
    schedule: '10 1 * * *'
    duration: 1h
    applications:
    - '*-prod'
    manualSync: true
  - kind: deny
    schedule: '0 22 * * *'
    timeZone: "Europe/Amsterdam"
    duration: 1h
    namespaces:
    - default
  - kind: allow
    schedule: '0 23 * * *'
    duration: 1h
    clusters:
    - in-cluster
    - cluster1
```

为了在窗口阻止同步时执行同步，可以使用 CLI、用户界面或直接在窗口中配置窗口允许手动同步。

```bash
argocd proj windows enable-manual-sync PROJECT ID
```

禁用

```bash
argocd proj windows disable-manual-sync PROJECT ID
```

可以使用 cli 列出窗口，也可以在用户界面中查看：

```bash
argocd proj windows list PROJECT
```

```bash
ID STATUS KIND SCHEDULE DURATION APPLICATIONS NAMESPACES CLUSTERS MANUALSYNC
0 Active allow  * * * * *   1h        -             -           prod1 Disabled
1 Inactive deny   * * * * 1 3h        -             default     -         Disabled
2 Inactive allow 1 2 * * *   1h prod-*        -           -         Enabled
3 Active deny   * * * * *   1h        -             default     -         Disabled
```

窗口的所有字段都可以通过 CLI 或 UI 进行更新。`applications`,`namespaces`和`clusters`字段要求更新包含所有必填值。 例如，如果更新`namespaces`字段，且其中已包含 default 和 kube-system，那么新值就必须包括这些字段。

```bash
argocd proj windows update PROJECT ID --namespaces default,kube-system,prod1
```