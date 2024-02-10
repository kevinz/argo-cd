<!-- TRANSLATED by md-translate -->
# Git Webhook 配置

## 概览

Argo CD 每三分钟轮询一次 Git 仓库，以检测清单的变更。 为消除轮询带来的延迟，可将 API 服务器配置为接收 webhook 事件。 Argo CD 支持来自 GitHub、GitLab、Bitbucket、Bitbucket Server、Azure DevOps 和 Gogs 的 Git webhook 通知。 下面将介绍如何为 GitHub 配置 Git webhook，但同样的流程应适用于其他 Providers。

注意 webhook 处理程序不会区分分支事件和标签事件（分支和标签名称相同）。 推送到分支 `x` 的钩子事件将触发指向同一版本库的应用程序刷新，该版本库的 `targetRevision: refs/tags/x`。

## 1.在 Git Provider 中创建 Webhook

在 Git Provider 中，导航至可配置 webhook 的设置页面。 在 Git Provider 中配置的有效载荷 URL 应使用 Argo CD 实例的 `/api/webhook` 端点（如 `https://argocd.example.com/api/webhook`）。如果希望使用共享秘密，请在秘密中输入任意值。该值将在下一步配置 webhook 时被引用。

## Github

![添加 Webhook]（.../assets/webhook-config.png "添加 Webhook）

注意 在 GitHub 中创建 webhook 时，需要将 "内容类型 "设置为 "application/json"。 用于处理钩子的库不支持默认值 "application/x-www-form-urlencoded"。

## Azure DevOps

![Add Webhook]（.../assets/azure-devops-webhook-config.png "Add Webhook")

Azure DevOps 可选支持使用基本身份验证保护 webhook 的安全。 要使用它，请在 webhook 配置中指定用户名和密码，并在 `argocd-secret` Kubernetes secret 中的 `webhook.azuredevops.username` 和 `webhook.azuredevops.password` 密钥中配置相同的用户名/密码。

## 2. 使用 Webhook Secret 配置 Argo CD（可选）

配置 webhook 共享秘密是可选的，因为 Argo CD 仍会刷新与 Git 仓库相关的应用程序，即使是未经身份验证的 webhook 事件。 这样做是安全的，因为 webhook 有效载荷的内容被认为是不可信任的，只会导致应用程序的刷新（这一过程已经每隔三分钟发生一次）。 如果 Argo CD 可被公开访问，则建议配置 webhook 秘密，以防止 DDoS 攻击。

在 `argocd-secret` Kubernetes secret 中，用步骤 1 中配置的 Git Providers 的 webhook secret 配置以下密钥之一。

| Provider | k8s 密钥 | |-----------------|----------------------------------| | GitHub | `webhook.github.secret` | | GitLab | `webhook.gitlab.secret` | | BitBucket | `webhook.bitbucket.uuid` | | BitBucketServer | `webhook.bitbucketserver.secret` | | Gogs | `webhook.gogs.secret` | | Azure DevOps | `webhook.azuredevops.username` | | | `webhook.azuredevops.password` | | | GitLab | `webhook.gitLab.secret` | | BitBucket

编辑 Argo CD Kubernetes secret：

```bash
kubectl edit secret argocd-secret -n argocd
```

提示：为了方便输入秘密，Kubernetes 支持在 `stringData` 字段中输入秘密，这样就省去了对值进行 base64 编码并复制到 `data` 字段的麻烦。 只需将步骤 1 中创建的共享 webhook 秘密复制到 `stringData` 字段下相应的 GitHub/GitLab/BitBucket 密钥中即可：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...

stringData:
  # github webhook secret
  webhook.github.secret: shhhh! it's a GitHub secret

  # gitlab webhook secret
  webhook.gitlab.secret: shhhh! it's a GitLab secret

  # bitbucket webhook secret
  webhook.bitbucket.uuid: your-bitbucket-uuid

  # bitbucket server webhook secret
  webhook.bitbucketserver.secret: shhhh! it's a Bitbucket server secret

  # gogs server webhook secret
  webhook.gogs.secret: shhhh! it's a gogs server secret

  # azuredevops username and password
  webhook.azuredevops.username: admin
  webhook.azuredevops.password: secret-password
```

保存后，更改将自动生效。