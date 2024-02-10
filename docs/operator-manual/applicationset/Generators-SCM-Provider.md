<!-- TRANSLATED by md-translate -->
# SCM Provider 生成器

SCM Provider 生成器被引用 SCMaaS 提供商（如 GitHub）的 API 来自动发现组织内的资源库。 这非常适合 GitOps 的布局模式，即在许多资源库中分割微服务。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      # Which protocol to clone using.
      cloneProtocol: ssh
      # See below for provider specific options.
      github:
        # ...
```

* `cloneProtocol`：被引用到 SCM URL 的协议。默认为特定于 Provider 的协议，但可能的话使用 ssh。并非所有 Provider 都支持所有协议，有关可用选项，请参阅下面的 Provider 文档。

只有管理员可以创建 ApplicationSet](./Security.md#only-admins-may-createupdatedelete-applicationsets)，以避免泄露秘密；如果使用 SCM 生成器的 ApplicationSet 的 `project` 字段是模板化的，则[只有管理员可以创建 repos/分支](./Security.md#templated-project-field)，以避免授权管理越界资源。

## GitHub

GitHub 模式被引用 GitHub API 扫描 github.com 或 GitHub Enterprise 中的组织。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      github:
        # The GitHub organization to scan.
        organization: myorg
        # For GitHub Enterprise:
        api: https://git.example.com/
        # If true, scan every branch of every repository. If false, scan only the default branch. Defaults to false.
        allBranches: true
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: github-token
          key: token
        # (optional) use a GitHub App to access the API instead of a PAT.
        appSecretName: gh-app-repo-creds
  template:
  # ...
```

* `organization`：需要扫描的 GitHub 组织名称。如果有多个组织，请被引用多个生成器。
* `api`：如果被引用为 GitHub 企业版，则是访问它的 URL。
* `allBranches`：默认情况下（false），模板只对每个 repo 的默认分支进行评估。如果为 "true"，则每个版本库的每个分支都会传递给过滤器。如果被引用此 flag，则可能需要使用 `branchMatch` 过滤器。
* tokenRef`：一个包含 GitHub 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将发出匿名请求，其速率限制较低，且只能查看公共仓库。
* 应用程序密名包含[repo-creds 格式]（.../declarative-setup.md#repository-credentials）的 GitHub 应用秘密的 `Secret` 名称。

在标签过滤方面，存储库主题被引用。

可用的克隆协议有`ssh`和`https`。

## Gitlab

GitLab 模式使用 GitLab API 在 gitlab.com 或自托管的 GitLab 中进行扫描和组织。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      gitlab:
        # The base GitLab group to scan. You can either use the group id or the full namespaced path.
        group: "8675309"
        # For self-hosted GitLab:
        api: https://gitlab.example.com/
        # If true, scan every branch of every repository. If false, scan only the default branch. Defaults to false.
        allBranches: true
        # If true, recurses through subgroups. If false, it searches only in the base group. Defaults to false.
        includeSubgroups: true
        # If true and includeSubgroups is also true, include Shared Projects, which is gitlab API default.
        # If false only search Projects under the same path. Defaults to true.
        includeSharedProjects: false
        # filter projects by topic. A single topic is supported by Gitlab API. Defaults to "" (all topics).
        topic: "my-topic"
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: gitlab-token
          key: token
        # If true, skips validating the SCM provider's TLS certificate - useful for self-signed certificates.
        insecure: false
  template:
  # ...
```

* group`：需要扫描的 GitLab 基本组名称。如果有多个基础组，请被引用多个生成器。
* `api`：被引用的 GitLab 的 URL。
* `allBranches`：默认情况下（false），模板只对每个 repo 的默认分支进行评估。如果为 "true"，则每个版本库的每个分支都会传递给过滤器。如果被引用此 flag，则可能需要使用 `branchMatch` 过滤器。
* includeSubgroups`（包含子组）：默认情况下（false），控制器只会直接搜索基本组中的 repos。如果为 "true"，它将遍历所有子组，搜索要扫描的软件版本。
* includeSharedProjects`：如果为 true 且 includeSubgroups 也为 true，则包括共享项目，这是 gitlab API 的默认设置。false 则只搜索相应路径下的项目。一般来说，大多数人都希望设置为 false 时的行为。默认为 true。
* topic`：按主题过滤项目。Gitlab API 支持单个主题。默认为""（所有主题）。
* `tokenRef`：一个包含 GitLab 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将发出匿名请求，其速率限制较低，且只能查看公共仓库。
* 不安全被引用（false）--跳过检查 SCM 证书的有效性--对自签 TLS 证书有用。

在标签过滤方面，存储库主题被引用。

可用的克隆协议有`ssh`和`https`。

### 自签名 TLS 证书

除了将 `insecure` 设为 true 之外，还可以为 Gitlab 配置自签名 TLS 证书。

要让应用程序集的 SCM/PR Gitlab 生成器使用自签名 TLS 证书，必须在应用程序集控制器上加载该证书。 加载证书的路径必须使用环境变量 "ARGOCD_APPLICATIONSET_CONTROLLER_SCM_ROOT_CA_PATH "或参数"--scm-root-ca-path "明确设置。 应用程序集控制器将读取加载的证书，为 SCM/PR Providers 创建 Gitlab 客户端。

通过在 argocd-cmd-params-cm ConfigMap 中设置 "applicationsetcontroller.scm.root.ca.path"，可以方便地实现这一点。 设置此值后，请务必重启 ApplicationSet 控制器。

## Gitea

Gitea 模式使用 Gitea API 扫描实例中的组织

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      gitea:
        # The Gitea owner to scan.
        owner: myorg
        # The Gitea instance url
        api: https://gitea.mydomain.com/
        # If true, scan every branch of every repository. If false, scan only the default branch. Defaults to false.
        allBranches: true
        # Reference to a Secret containing an access token. (optional)
        tokenRef:
          secretName: gitea-token
          key: token
  template:
  # ...
```

* Owner`：需要扫描的 Gitea 组织名称。如果有多个组织，请被引用多个生成器。
* `api`：您正在使用的 Gitea 实例的 URL。
* `allBranches`：默认情况下（false），模板只对每个 repo 的默认分支进行评估。如果为 "true"，则每个版本库的每个分支都会传递给过滤器。如果被引用此 flag，则可能需要使用 `branchMatch` 过滤器。
* tokenRef`：一个包含 Gitea 访问令牌的 `Secret` 名称和密钥，用于请求。如果未指定，将发出匿名请求，其速率限制较低，且只能查看公共版本库。
* 不安全允许自签名 TLS 证书。

该 SCM Provider 尚不支持标签过滤功能

可用的克隆协议有`ssh`和`https`。

## Bitbucket 服务器

使用 Bitbucket Server API（1.0）扫描项目中的软件仓库。 请注意，Bitbucket Server 与 Bitbucket Cloud（API 2.0）不同。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      bitbucketServer:
        project: myproject
        # URL of the Bitbucket Server. Required.
        api: https://mycompany.bitbucket.org
        # If true, scan every branch of every repository. If false, scan only the default branch. Defaults to false.
        allBranches: true
        # Credentials for Basic authentication. Required for private repositories.
        basicAuth:
          # The username to authenticate with
          username: myuser
          # Reference to a Secret containing the password or personal access token.
          passwordRef:
            secretName: mypassword
            key: password
        # Support for filtering by labels is TODO. Bitbucket server labels are not supported for PRs, but they are for repos
  template:
  # ...
```

* 项目必填的 Bitbucket 项目名称
* `api`：访问 Bitbucket REST api 所需的 URL。
* `allBranches`：默认情况下（false），模板只对每个版本库的默认分支进行评估。如果为 "true"，则每个版本库的每个分支都会传递给过滤器。如果被引用此 flag，则可能需要使用 `branchMatch` 过滤器。

如果要访问私有资源库，还必须提供 Basic auth 的凭据（这是目前唯一支持的 auth）：

* 用户名：要验证的用户名。它只需要相关版本库的读取权限。
* `passwordRef`：包含密码或个人访问令牌的 `Secret` 名称和密钥，用于请求。

可用的克隆协议有`ssh`和`https`。

## Azure DevOps

被引用的 Azure DevOps API 可根据 Azure DevOps 组织内的团队项目查找符合条件的存储库。默认 Azure DevOps URL 为 `https://dev.azure.com`，但可通过字段 `azureDevOps.api` 改写。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      azureDevOps:
        # The Azure DevOps organization.
        organization: myorg
        # URL to Azure DevOps. Optional. Defaults to https://dev.azure.com.
        api: https://dev.azure.com
        # If true, scan every branch of eligible repositories. If false, check only the default branch of the eligible repositories. Defaults to false.
        allBranches: true
        # The team project within the specified Azure DevOps organization.
        teamProject: myProject
        # Reference to a Secret containing the Azure DevOps Personal Access Token (PAT) used for accessing Azure DevOps.
        accessTokenRef:
          secretName: azure-devops-scm
          key: accesstoken
  template:
  # ...
```

* 组织"：必填。Azure DevOps 组织的名称。
* 团队项目必填。指定 `organization` 中团队项目的名称。
* `accessTokenRef`：必填。包含要用于请求的 Azure DevOps 个人访问令牌 (PAT) 的 `Secret` 名称和密钥。
* `api`：可选。Azure DevOps 的 URL。如果未设置，则被引用 `https://dev.azure.com`。
* `allBranches`：可选，默认为 `false`。如果 `true`，则扫描符合条件的版本库的每个分支。如果 `false`，则只检查符合条件的软件源的默认分支。

## Bitbucket 云

Bitbucket 模式被引用 Bitbucket API V2 来扫描 bitbucket.org 中的工作区。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      bitbucket:
        # The workspace id (slug).  
        owner: "example-owner"
        # The user to use for basic authentication with an app password.
        user: "example-user"
        # If true, scan every branch of every repository. If false, scan only the main branch. Defaults to false.
        allBranches: true
        # Reference to a Secret containing an app password.
        appPasswordRef:
          secretName: appPassword
          key: password
  template:
  # ...
```

* Owner`：查找软件源时被引用的工作区 ID（slug）。
* `user`：被引用到 bitbucket.org 的 Bitbucket API V2 验证的用户。
* `allBranches`：默认情况下（false），模板只对每个版本库的主分支进行评估。如果为 "true"，则每个版本库的每个分支都会传给过滤器。如果被引用此 flag，则可能需要使用 `branchMatch` 过滤器。
* `appPasswordRef`：包含 bitbucket 应用程序密码的 `Secret` 名称和密钥，用于请求。

该 SCM Provider 尚不支持标签过滤功能

可用的克隆协议有`ssh`和`https`。

### AWS CodeCommit (Alpha)

被引用 AWS ResourceGroupsTagging 和 AWS CodeCommit API 跨 AWS 账户和地区扫描 repos。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
    - scmProvider:
        awsCodeCommit:
          # AWS region to scan repos.
          # default to the environmental region from ApplicationSet controller.
          region: us-east-1
          # AWS role to assume to scan repos.
          # default to the environmental role from ApplicationSet controller.
          role: arn:aws:iam::111111111111:role/argocd-application-set-discovery
          # If true, scan every branch of every repository. If false, scan only the main branch. Defaults to false.
          allBranches: true
          # AWS resource tags to filter repos with.
          # see https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/API_GetResources.html#resourcegrouptagging-GetResources-request-TagFilters for details
          # default to no tagFilters, to include all repos in the region.
          tagFilters:
            - key: organization
              value: platform-engineering
            - key: argo-ready
  template:
  # ...
```

* region`：（可选）扫描 repos 的 AWS 区域。默认情况下，被引用的是 ApplicationSet 控制器的当前区域。
* `role`：（可选）要扫描 repos 的 AWS 角色。默认情况下，使用 ApplicationSet 控制器的当前角色。
* allBranches`：（可选）如果 `true`，则扫描符合条件的版本库的每个分支。如果 `false`，则只检查符合条件的软件源的默认分支。默认为 `false`。
* tagFilters：（可选）用于过滤 AWS CodeCommit 仓库的 tagFilters 列表。详情请参阅 [AWS ResourceGroupsTagging API](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/API_GetResources.html#resourcegrouptagging-GetResources-request-TagFilters)。默认情况下，不包含任何过滤器。

该 SCM Provider 不支持以下功能

* 标签过滤
* sha"、"short_sha "和 "short_sha_7 "模板参数

可用的克隆协议有`ssh`、`https`和`https-fips`。

### AWS IAM 权限注意事项

为了调用 AWS API 发现 AWS CodeCommit repos，ApplicationSet 控制器必须配置有效的 AWS 环境配置，如当前 AWS 区域和 AWS 凭据。 AWS 配置可通过所有标准选项提供，如实例元数据服务（IMDS）、配置文件、环境变量或服务账户的 IAM 角色（IRSA）。

根据是否在 `awsCodeCommit` 属性中提供 `role` ，AWS IAM 权限要求有所不同。

#### 在与 ApplicationSet Controller 相同的 AWS 账户中发现 AWS CodeCommit 存储库

如果不指定 "角色"，ApplicationSet 控制器将使用自己的 AWS 身份扫描 AWS CodeCommit repos。 这适用于所有 AWS CodeCommit repos 与 Argo CD 位于同一 AWS 账户的简单设置。

由于 ApplicationSet 控制器 AWS 身份被直接引用用于发现 repo，因此必须授予 AWS 以下权限。

* 标签:获取资源
* codecommit:ListRepositories`目录
* 代码提交:获取资源库
* 代码提交:获取文件夹
* codecommit:ListBranches`目录

#### 跨 AWS 账户和地区发现 AWS CodeCommit 资源库

通过指定 "角色"，ApplicationSet 控制器将首先假定 "角色"，并使用它来发现软件仓库。 这样就能在更复杂的用例中发现来自不同 AWS 账户和地区的软件仓库。

应授予 ApplicationSet 控制器 AWS 身份承担目标 AWS 角色的权限。

* sts:AssumeRole`（承担角色

所有 AWS 角色都必须拥有与 repo 发现相关的权限。

* 标签:获取资源
* codecommit:ListRepositories`目录
* 代码提交:获取资源库
* 代码提交:获取文件夹
* codecommit:ListBranches`目录

## 过滤器

过滤器允许选择要生成的版本库。 每个过滤器可以声明一个或多个条件，所有条件都必须通过。 如果存在多个过滤器，则任何一个过滤器都可以匹配以包含一个版本库。 如果没有指定过滤器，则将处理所有版本库。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  generators:
  - scmProvider:
      filters:
      # Include any repository starting with "myapp" AND including a Kustomize config AND labeled with "deploy-ok" ...
      - repositoryMatch: ^myapp
        pathsExist: [kubernetes/kustomization.yaml]
        labelMatch: deploy-ok
      # ... OR include any repository starting with "otherapp" AND a Helm folder and doesn't have file disabledrepo.txt.
      - repositoryMatch: ^otherapp
        pathsExist: [helm]
        pathsDoNotExist: [disabledrepo.txt]
  template:
  # ...
```

* repositoryMatch`：与版本库名称匹配的 regexp。
* `pathsExist`：版本库中必须存在的路径数组。可以是文件或目录。
* `pathsDoNotExist`：版本库中必须不存在的路径数组。可以是文件或目录。
* `labelMatch`：与版本库标签匹配的 regexp。如果有标签匹配，则包含该版本库。
* 分支匹配与分支名称匹配的 regexp。

## 模板

与所有生成器一样，会生成几个参数供 `ApplicationSet` 资源模板使用。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - scmProvider:
    # ...
  template:
    metadata:
      name: '{{ .repository }}'
    spec:
      source:
        repoURL: '{{ .url }}'
        targetRevision: '{{ .branch }}'
        path: kubernetes/
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: default
```

* `organization`：版本库所在组织的名称。
* `repository`：版本库的名称。
* `url`：版本库的克隆 URL。
* 分支版本库的默认分支。
* `sha`：分支的 Git 提交 SHA。
* `short_sha`：该分支的 Git 提交 SHA 缩写（8 个字符或 `sha` 的长度（如果更短））。
* `short_sha_7`：分支的 Git 提交 SHA 缩写（7 个字符，如果更短，则与 `sha` 长度相同）。
* 标签以逗号分隔的仓库标签列表（如果是 Gitea），以及仓库主题列表（如果是 Gitlab 和 Github）。Bitbucket Cloud、Bitbucket Server 或 Azure DevOps 不支持。
* 分支规范化分支 "的值，规范化后只包含小写字母数字字符、"-"或"."。

## 通过 `values` 字段传递额外的键值对

您可以通过任何单片机生成器的 "values "字段传递额外的任意字符串键值对。 通过 "values "字段添加的值以 "values.(field) "的形式添加。

在此示例中，传递了一个 `name` 参数值，它从 `organization` 和 `repository` 插值生成一个不同的模板名称。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapps
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - scmProvider:
      bitbucketServer:
        project: myproject
        api: https://mycompany.bitbucket.org
        allBranches: true
        basicAuth:
          username: myuser
          passwordRef:
            secretName: mypassword
            key: password
      values:
        name: "{{.organization}}-{{.repository}}"

  template:
    metadata:
      name: '{{ .values.name }}'
    spec:
      source:
        repoURL: '{{ .url }}'
        targetRevision: '{{ .branch }}'
        path: kubernetes/
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: default
```

注意 `values.` 前缀总是被引用到通过 `generators.scmProvider.values` 字段提供的值中。 使用时请确保在 `template` 中的参数名称中包含此前缀。

在 `values` 中，我们还可以对单片机生成器设置的所有字段进行上述插值。