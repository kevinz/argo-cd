<!-- TRANSLATED by md-translate -->
# 拉取请求生成器

拉取请求生成器使用 SCMaaS 提供商（GitHub、Gitea 或 Bitbucket Server）的 API 自动发现版本库中的开放拉取请求。 这非常符合在创建拉取请求时构建测试环境的风格。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      # When using a Pull Request generator, the ApplicationSet controller polls every `requeueAfterSeconds` interval (defaulting to every 30 minutes) to detect changes.
      requeueAfterSeconds: 1800
      # See below for provider specific options.
      github:
        # ...
```

注意 了解 ApplicationSet 中 PR 生成器的安全影响。[只有管理员可以创建 ApplicationSet](./Security.md#only-admins-may-createupdatedelete-applicationsets)以避免 Secrets 外泄，如果带有 PR 生成器的 ApplicationSet 的 `project` 字段是模板化的，则[只有管理员可以创建 PR](./Security.md#templated-project-field)以避免授权管理越界资源。

## GitHub

指定从哪个版本库获取 GitHub 拉取请求。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      github:
        # The GitHub organization or user.
        owner: myorg
        # The Github repository
        repo: myrepository
        # For GitHub Enterprise (optional)
        api: https://git.example.com/
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: github-token
          key: token
        # (optional) use a GitHub App to access the API instead of a PAT.
        appSecretName: github-app-repo-creds
        # Labels is used to filter the PRs that you want to target. (optional)
        labels:
        - preview
      requeueAfterSeconds: 1800
  template:
  # ...
```

* Owner`：GitHub 组织或用户的必填名称。
* `repo`：GitHub 仓库的必填名称。
* `api`：如果被引用到 GitHub 企业版，则是访问该版本库的 URL。可选
* `tokenRef`：包含 GitHub 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将使用匿名请求，其速率限制较低，且只能查看公共仓库。可选
* 标签将 PR 筛选为包含***列出的所有标签的 PR。可选
* `appSecretName`：包含[repo-creds 格式]（.../declarative-setup.md#repository-credentials）的 GitHub 应用秘密的`秘密`名称。

## GitLab

指定从中获取 GitLab 合并请求的项目。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      gitlab:
        # The GitLab project.
        project: myproject
        # For self-hosted GitLab (optional)
        api: https://git.example.com/
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: gitlab-token
          key: token
        # Labels is used to filter the MRs that you want to target. (optional)
        labels:
        - preview
        # MR state is used to filter MRs only with a certain state. (optional)
        pullRequestState: opened
        # If true, skips validating the SCM provider's TLS certificate - useful for self-signed certificates.
        insecure: false
      requeueAfterSeconds: 1800
  template:
  # ...
```

* project`：GitLab 项目的必填名称。
* `api`：如果被引用的是自托管的 GitLab，访问它的 URL。可选
* `tokenRef`：包含被引用的 GitLab 访问令牌的 `Secret` 名称和密钥。如果未指定，将使用匿名请求，其速率限制较低，且只能查看公共仓库。可选
* 标签Labels 用于过滤您要针对的 MR。可选
* `pullRequestState`：PullRequestState 是一个额外的 MR 筛选器，用于只获取具有特定状态的 MR。默认：""（所有状态）
* `insecure`：被引用（false）- 跳过检查单片机证书的有效性- 对自签名 TLS 证书有用。

除了将 "不安全 "设置为 "true"，还可以通过[将自签名证书挂载到 ApplicationSet 控制器]（./Generators-SCM-Provider.md#self-signed-tls-certificates）为 Gitlab 配置自签名 TLS 证书。

## Gitea

指定从哪个版本库获取 Gitea 拉取请求。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      gitea:
        # The Gitea organization or user.
        owner: myorg
        # The Gitea repository
        repo: myrepository
        # The Gitea url to use
        api: https://gitea.mydomain.com/
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: gitea-token
          key: token
        # many gitea deployments use TLS, but many are self-hosted and self-signed certificates
        insecure: true
      requeueAfterSeconds: 1800
  template:
  # ...
```

* Owner`：Gitea 组织或用户的必填名称。
* `repo`：Gitea 仓库的必填名称。
* `api`：Gitea 实例的 URL。
* `tokenRef`：包含 Gitea 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将发出匿名请求，其速率限制较低，且只能查看公共存储库。可选
* insecure`：`允许使用自签名证书，主要用于测试。

## Bitbucket 服务器

从 Bitbucket Server（与 Bitbucket Cloud 不同）上托管的版本库中获取拉取请求。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      bitbucketServer:
        project: myproject
        repo: myrepository
        # URL of the Bitbucket Server. Required.
        api: https://mycompany.bitbucket.org
        # Credentials for Basic authentication. Required for private repositories.
        basicAuth:
          # The username to authenticate with
          username: myuser
          # Reference to a Secret containing the password or personal access token.
          passwordRef:
            secretName: mypassword
            key: password
      # Labels are not supported by Bitbucket Server, so filtering by label is not possible.
      # Filter PRs using the source branch name. (optional)
      filters:
      - branchMatch: ".*-argocd"
  template:
  # ...
```

* 项目必填的 Bitbucket 项目名称
* `repo`：Bitbucket 仓库的必填名称。
* `api`：访问 Bitbucket REST API 所需的 URL。在上面的例子中，API 请求的地址是 `https://mycompany.bitbucket.org/rest/api/1.0/projects/myproject/repos/myrepository/pull-requests`.
* `branchMatch`：可选的 regexp 过滤器，用于匹配源分支名称。这是 Bitbucket 服务器不支持的标签的替代方法。

如果要访问私有资源库，还必须提供 Basic auth 的凭据（这是目前唯一支持的 auth）：

* 用户名：要验证的用户名。它只需要相关版本库的读取权限。
* `passwordRef`：包含密码或个人访问令牌的 `Secret` 名称和密钥，用于请求。

## Bitbucket 云

从 Bitbucket 云上托管的版本库中获取拉取请求。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - pullRequest:
        bitbucket:
          # Workspace name where the repoistory is stored under. Required.
          owner: myproject
          # Repository slug. Required.
          repo: myrepository
          # URL of the Bitbucket Server. (optional) Will default to 'https://api.bitbucket.org/2.0'.
          api: https://api.bitbucket.org/2.0
          # Credentials for Basic authentication (App Password). Either basicAuth or bearerToken
          # authentication is required to access private repositories
          basicAuth:
            # The username to authenticate with
            username: myuser
            # Reference to a Secret containing the password or personal access token.
            passwordRef:
              secretName: mypassword
              key: password
          # Credentials for Bearer Token (App Token) authentication. Either basicAuth or bearerToken
          # authentication is required to access private repositories
          bearerToken:
            tokenRef:
              secretName: repotoken
              key: token
        # Labels are not supported by Bitbucket Cloud, so filtering by label is not possible.
        # Filter PRs using the source branch name. (optional)
        filters:
          - branchMatch: ".*-argocd"
  template:
  # ...
```

* `owner`：Bitbucket 工作区的必填名称
* `repo`：Bitbucket 仓库的必填名称。
* `api`：用于访问 Bitbucket REST API 的可选 URL。在上面的例子中，API 请求将发送到 `https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests`。如果未设置，默认为`https://api.bitbucket.org/2.0`。
* branchMatch`：可选的 regexp 过滤器，用于匹配源分支名称。这是 Bitbucket 服务器不支持的标签的替代方案。

如果要访问私有版本库，Argo CD 需要访问 Bitbucket 云版本库的凭证。 可以使用 Bitbucket App Password（为每个用户生成，可访问整个工作区），或 Bitbucket App Token（为每个版本库生成，仅可访问版本库范围）。 如果同时定义了 App Password 和 App Token，则将使用 App Token。

要使用 Bitbucket 应用程序密码，请使用 "basicAuth "部分。

* 用户名：要验证的用户名。它只需要相关版本库的读取权限。
* `passwordRef`：包含密码或个人访问令牌的 `Secret` 名称和密钥，用于请求。

如果使用 Bitbucket 应用程序令牌，请使用 "bearerToken "部分。

* tokenRef`：秘密 "名称和密钥，其中包含用于请求的应用程序令牌。

## Azure DevOps

指定要从中获取拉取请求的组织、项目和版本库。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      azuredevops:
        # Azure DevOps org to scan. Required.
        organization: myorg
        # Azure DevOps project name to scan. Required.
        project: myproject
        # Azure DevOps repo name to scan. Required.
        repo: myrepository
        # The Azure DevOps API URL to talk to. If blank, use https://dev.azure.com/.
        api: https://dev.azure.com/
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: azure-devops-token
          key: token
        # Labels is used to filter the PRs that you want to target. (optional)
        labels:
        - preview
      requeueAfterSeconds: 1800
  template:
  # ...
```

* 组织"：Azure DevOps 组织的必填名称。
* 项目`：Azure DevOps 项目的必填名称。
* `repo`：Azure DevOps 存储库的必填名称。
* `api`：如果使用自行托管的 Azure DevOps Repos，则是访问它的 URL。可选
* tokenRef`：包含 Azure DevOps 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将发出匿名请求，其速率限制较低，且只能查看公共版本库。可选
* 标签将 PR 筛选为包含所列***全部标签的 PR。可选

## 过滤器

过滤器允许选择要生成哪些拉取请求。 每个过滤器可以声明一个或多个条件，所有条件都必须通过。 如果存在多个过滤器，则任何一个都可以匹配以包含一个版本库。 如果未指定过滤器，则将处理所有拉取请求。 目前，与 [SCM provider](Generators-SCM-Provider.md)过滤器比较时，只有过滤器的子集可用。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      # ...
      # Include any pull request ending with "argocd". (optional)
      filters:
      - branchMatch: ".*-argocd"
  template:
  # ...
```

* 分支匹配与源分支名称匹配的 regexp。
* `targetBranchMatch`：与目标分支名称匹配的 regexp。

[GitHub](#github) 和 [GitLab](#gitlab) 也支持 "标签 "过滤器。

## 模板

与所有生成器一样，在生成的应用程序中有多个密钥可供替换。

下面是一个全面的 Helm 应用程序示例；

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
    # ...
  template:
    metadata:
      name: 'myapp-{{.branch}}-{{.number}}'
    spec:
      source:
        repoURL: 'https://github.com/myorg/myrepo.git'
        targetRevision: '{{.head_sha}}'
        path: kubernetes/
        helm:
          parameters:
          - name: "image.tag"
            value: "pull-{{.head_sha}}"
      project: "my-project"
      destination:
        server: https://kubernetes.default.svc
        namespace: default
```

下面是一个强大的 kustomize 示例；

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
    # ...
  template:
    metadata:
      name: 'myapp-{{.branch}}-{{.number}}'
    spec:
      source:
        repoURL: 'https://github.com/myorg/myrepo.git'
        targetRevision: '{{.head_sha}}'
        path: kubernetes/
        kustomize:
          nameSuffix: '{{.branch}}'
          commonLabels:
            app.kubernetes.io/instance: '{{.branch}}-{{.number}}'
          images:
          - 'ghcr.io/myorg/myrepo:{{.head_sha}}'
      project: "my-project"
      destination:
        server: https://kubernetes.default.svc
        namespace: default
```

* number`：拉取请求的 ID 编号。
* 分支拉取请求头部分支的名称。
* `branch_slug`：分支名称将被清理以符合 [RFC 1123](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) 中定义的 DNS 标签标准，并被截断为 50 个字符，以便有空间再追加/后缀 13 个字符。
* 目标分支"：拉取请求的目标分支名称。
* target_branch_slug`：目标分支名称将被清理为符合 [RFC 1123](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) 中定义的 DNS 标签标准，并被截断为 50 个字符，以便在附加/后缀时多加 13 个字符。
* head_sha`：这是拉取请求头部的 SHA。
* head_short_sha`：这是拉取请求头部的 SHA 短值（8 个字符，或头部 SHA 的长度（如果更短））。
* head_short_sha_7`：这是拉取请求头部的 SHA 短标识（7 个字符长，或头部 SHA 的长度（如果更短））。
* 标签拉取请求标签数组。(仅支持 Go Template ApplicationSet 配置清单）。

## Webhook 配置

使用拉取请求生成器时，ApplicationSet 控制器会每隔 "requeueAfterSeconds"（默认为每 30 分钟）轮询一次，以检测更改。 为消除轮询延迟，可将 ApplicationSet webhook 服务器配置为接收 webhook 事件，这将触发拉取请求生成器生成应用程序。

配置与[Git 生成器](Generators-Git.md)中描述的几乎相同，但有一点不同：如果还想使用拉取请求生成器，需要额外配置以下设置。

注意 ApplicationSet 控制器 webhook 使用的 webhook 与[此处](../webhook.md)定义的 API 服务器不同。 ApplicationSet 将 webhook 服务器作为 ClusterIP 类型的服务公开。 需要创建一个 ApplicationSet 专用 ingress 资源，以便将此服务公开给 webhook 源。

### Github webhook 配置

在第 1 节_"在 Git Provider 中创建 webhook"_中，添加一个事件，以便在拉取请求创建、关闭或标签更改时发送 webhook 请求。

使用 uri`/api/webhook` 添加 Webhook URL，并选择内容类型为 json ![Add Webhook URL](././assets/applicationset/webhook-config-pullrequest-generator.png "Add Webhook URL")

选择 "让我选择单个事件 "并启用 "拉取请求 "复选框。

![添加网络钩子](.../.../assets/applicationset/webhook-config-pull-request.png "Add Webhook Pull Request")

当下一个操作发生时，拉动请求生成器将重新请求。

* 打开
* 关闭
* 重新打开
* 已标记
* 未标记
* 同步

有关每个事件的详细信息，请参阅 [官方文档](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads)。

### Gitlab webhook 配置

在触发器列表中启用 "合并请求事件 "复选框。

![添加 Gitlab Webhook](../../assets/applicationset/webhook-config-merge-request-gitlab.png "添加 Gitlab 合并请求 Webhook")

当下一个操作发生时，拉动请求生成器将重新请求。

* 打开
* 关闭
* 重新打开
* 更新
* 合并

有关每个事件的详细信息，请参阅 [官方文档](https://docs.gitlab.com/ee/user/project/integrations/webhook_events.html#merge-request-events)。

## 生命周期

当发现符合配置标准的拉取请求时，即对于 GitHub，当拉取请求符合指定的 "标签 "和/或 "拉取请求状态 "时，将生成应用程序。 当拉取请求不再符合指定标准时，应用程序将被移除。