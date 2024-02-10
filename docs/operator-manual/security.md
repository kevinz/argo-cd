<!-- TRANSLATED by md-translate -->
# 安全

Argo CD 经过了严格的内部安全审查和渗透测试，以满足[PCI 合规性](https://www.pcisecuritystandards.org)要求。以下是 Argo CD 的一些安全主题和实施细节。

## 验证

Argo CD API 服务器的身份验证完全使用[JSON Web 标记](https://jwt.io) (JWT)。用户名/密码承载标记不用于身份验证。 JWT 通过以下方式之一获取/管理：

1.对于本地 `admin` 用户，用户名/密码会通过 `/api/v1/session` 端点与 JWT 交换。
端点交换一个 JWT。该令牌由 Argo CD API 服务器本身签名和签发，24 小时后过期（该令牌过去不会过期，但现在会过期）。 
(该令牌过去不会过期，参见 [CVE-2021-26921](https://github.com/argoproj/argo-cd/security/advisories/GHSA-9h6w-j7w4-jr52)）。
当管理员密码更新时，所有现有的管理员 JWT 标记都会立即失效。
密码以 bcrypt 哈希值的形式存储在 [`argocd-secret`](https://github.com/argoproj/argo-cd/blob/master/manifests/base/config/argocd-secret.yaml) Secret 中。
2.对于单点登录用户，用户完成 OAuth2 登录流后，将登录到配置的 OIDC 身份提供商（可通过捆绑包委托）。
提供商（通过捆绑的 Dex 提供商授权，或直接向自我管理的 OIDC
提供商）。该 JWT 由 IDP 签发，过期和撤销由提供商处理。
Providers 处理。Dex 令牌在 24 小时后过期。
3.自动化令牌是使用 `/api/v1/projects/{project}/roles/{role}/token` 为项目生成的。
端点为项目生成，并由 Argo CD 签发。这些令牌的范围和权限有限、
只能用于管理其所属项目中的应用程序资源。项目
JWT 有可配置的过期时间，可通过删除项目角色中的 JWT 引用
ID 即可立即撤销。

## 授权

授权是通过遍历用户 JWT group claims 中的 group 成员列表，并将每个 group 与 [RBAC](../rbac) 策略中的角色/规则进行比较来完成的。 任何匹配的规则都允许访问 API 请求。

## TLS

所有网络通信均通过 TLS 进行，包括三个组件（argocd-server、argocd-repo-server、argocd-application-controller）之间的服务对服务通信。 Argo CD API 服务器可使用 flag: `--tlsminversion 1.2` 强制使用 TLS 1.2。 与 Redis 的通信默认通过普通 HTTP 进行。 TLS 可通过命令行参数设置。

## Git 和 Helm 仓库

Git 和 helm 资源库由名为 repo-server 的独立服务管理。 repo-server 不具有任何 Kubernetes 权限，也不存储任何服务（包括 git）的凭据。 repo-server 负责克隆 Argo CD 操作员允许和信任的资源库，并在资源库的给定路径上生成 Kubernetes 配置清单。 为了提高性能和带宽效率，repo-server 会维护这些资源库的本地克隆，以便高效下载资源库的后续提交。

在配置允许 Argo CD 部署的 git 仓库时，有一些安全方面的注意事项。 简而言之，如果未经授权就写入 Argo CD 信任的 git 仓库，将会产生如下所述的严重安全问题。

### 未经授权的部署

由于 Argo CD 部署的是在 git 中定义的 Kubernetes 资源，因此攻击者若能访问受信任的 git repo，就能影响已部署的 Kubernetes 资源。 例如，攻击者可以更新部署配置清单，将恶意容器镜像部署到环境中，或删除 git 中的资源，导致它们在实时环境中被剪切。

### 工具命令调用

In addition to raw YAML, Argo CD natively supports two popular Kubernetes config management tools,
helm and kustomize. When rendering manifests, Argo CD executes these config management tools
(i.e. `helm template`, `kustomize build`) to generate the manifests. It is possible that an attacker
with write access to a trusted git repository may construct malicious helm charts or kustomizations
that attempt to read files out-of-tree. This includes adjacent git repos, as well as files on the
repo-server itself. Whether or not this is a risk to your organization depends on if the contents
in the git repos are sensitive in nature. By default, the repo-server itself does not contain
sensitive information, but might be configured with Config Management Plugins which do
(e.g. decryption keys). If such plugins are used, extreme care must be taken to ensure the
repository contents can be trusted at all times.

可选择单独禁用内置配置管理工具。 如果知道用户不需要某个配置管理工具，建议禁用该工具。 更多信息请参阅 [Tool Detection]（.../user-guide/tool_detection.md）。

### 远程基地和 helm chart 依赖关系

Argo CD 的版本库允许列表只限制克隆的初始版本库。 然而，kustomize 和 helm 都包含引用和跟踪附加版本库（如 kustomize 远程基地、helm 图表依赖）的功能，而这些版本库可能不在版本库允许列表中。 Argo CD 操作员必须明白，拥有写权限访问可信 git 版本库的用户可能会引用其他远程 git 版本库，其中包含在配置的 git 版本库中不易搜索或审计的 Kubernetes 资源。

## 敏感信息

#### 秘密

Argo CD 从不从其 API 返回敏感数据，并对 API 有效载荷和 logging 中的所有敏感数据进行编辑。 这包括：

* 集群证书
* Git 认证
* OAuth2 客户端秘密
* Kubernetes 秘密值

#### 外部集群证书

为了管理外部集群，Argo CD 会将外部集群的凭证存储为 argocd 名称空间中的 Kubernetes Secret。 该 Secret 包含与在 `argocd cluster add` 期间创建的 `argocd-manager` ServiceAccount 相关联的 K8s API 承载令牌，以及与该 API 服务器的连接选项（TLS 配置/证书、AWS 角色arn 等......）。 这些信息被用于向 Argo CD 服务所引用的集群重构 REST config 和 kubeconfig。

要轮换 Argo CD 使用的承载令牌，可以删除该令牌（例如使用 kubectl），这将导致 Kubernetes 用新的承载令牌生成新的 secret。 可以通过重新运行 `argocd cluster add` 将新令牌重新输入 Argo CD。 针对 __managed__ 集群运行以下命令：

```bash
# run using a kubeconfig for the externally managed cluster
kubectl delete secret argocd-manager-token-XXXXXX -n kube-system
argocd cluster add CONTEXTNAME
```

!!! 注意 Kubernetes 1.24 [停止自动为服务账户创建令牌](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md#no-really-you-must-read-this-before-you-upgrade)。[从 Argo CD 2.4 开始](https://github.com/argoproj/argo-cd/pull/9546)，在添加 1.24 集群时，`argocd cluster add` 会创建一个 ServiceAccount _and_ 一个非过期的服务账户令牌 Secret。未来，Argo CD 将 [添加对 Kubernetes TokenRequest API 的支持](https://github.com/argoproj/argo-cd/issues/9610)，以避免被引用过期令牌。

要取消 Argo CD 对托管集群的访问权限，请删除针对 __managed__ 集群的 RBAC 工具，并从 Argo CD 中删除集群条目：

```bash
# run using a kubeconfig for the externally managed cluster
kubectl delete sa argocd-manager -n kube-system
kubectl delete clusterrole argocd-manager-role
kubectl delete clusterrolebinding argocd-manager-role-binding
argocd cluster rm https://your-kubernetes-cluster-addr
```

<!-- markdownlint-disable MD027 -->

&gt; 注意：对于 AWS EKS 集群，[get-token](https://docs.aws.amazon.com/cli/latest/reference/eks/get-token.html) 命令

被用于对外部集群进行身份验证，该集群使用 IAM 角色代替本地存储的令牌，因此无需轮换令牌，也可通过 IAM 进行撤销。

<!-- markdownlint-enable MD027 -->

## 集群 RBAC

默认情况下，Argo CD 被引用[clusteradmin 级角色](https://github.com/argoproj/argo-cd/blob/master/manifests/base/application-controller/argocd-application-controller-role.yaml)，以便：

1. 观察和操作集群状态
2. 向集群部署资源

尽管 Argo CD 需要整个集群的**_read_**权限才能正常运行，但它并不一定需要集群的全部**_write_**权限。 可以修改 argocd-server 和 argocd-application-controller 被引用的 ClusterRole，使写入权限仅限于希望 Argo CD 管理的 namespace 和资源。

要微调外部管理集群的权限，请编辑 `argocd-manager-role` 的 ClusterRole

```bash
# run using a kubeconfig for the externally managed cluster
kubectl edit clusterrole argocd-manager-role
```

要微调 Argo CD 在自己的集群（即 "https://kubernetes.default.svc"）中的权限，请编辑 Argo CD 运行所在的下列集群角色：

```bash
# run using a kubeconfig to the cluster Argo CD is running in
kubectl edit clusterrole argocd-server
kubectl edit clusterrole argocd-application-controller
```

提示 如果您想拒绝 Argo CD 访问某类资源，请将其添加为 [excluded resource]（声明式设置.md#resource-exclusion）。

## 审计

作为 GitOps 部署工具，Git 提交历史记录提供了一个天然的审计日志，记录了对应用程序配置所做的更改、更改时间和更改人。 不过，该审计日志只适用于 Git 中发生的情况，并不一定与集群中发生的事件一一对应。 例如，用户 A 可能对应用程序配置清单做了多次提交，但用户 B 可能只是在之后的某个时间才将这些更改同步到集群。

作为对 Git 修订历史的补充，Argo CD 还会发布有关应用程序活动的 Kubernetes 事件，并在适当情况下指出负责的行为者。 例如，Argo CD 会在 Kubernetes 事件的基础上进行修订：

```bash
$ kubectl get events
LAST SEEN FIRST SEEN COUNT NAME KIND SUBOBJECT TYPE REASON SOURCE MESSAGE
1m 1m 1 guestbook.157f7c5edd33aeac Application Normal ResourceCreated argocd-server admin created application
1m 1m 1 guestbook.157f7c5f0f747acf Application Normal ResourceUpdated argocd-application-controller Updated sync status:  -> OutOfSync
1m 1m 1 guestbook.157f7c5f0fbebbff Application Normal ResourceUpdated argocd-application-controller Updated health status:  -> Missing
1m 1m 1 guestbook.157f7c6069e14f4d Application Normal OperationStarted argocd-server admin initiated sync to HEAD (8a1cb4a02d3538e54907c827352f66f20c3d7b0d)
1m 1m 1 guestbook.157f7c60a55a81a8 Application Normal OperationCompleted argocd-application-controller Sync operation to 8a1cb4a02d3538e54907c827352f66f20c3d7b0d succeeded
1m 1m 1 guestbook.157f7c60af1ccae2 Application Normal ResourceUpdated argocd-application-controller Updated sync status: OutOfSync -> Synced
1m 1m 1 guestbook.157f7c60af5bc4f0 Application Normal ResourceUpdated argocd-application-controller Updated health status: Missing -> Progressing
1m 1m 1 guestbook.157f7c651990e848 Application Normal ResourceUpdated argocd-application-controller Updated health status: Progressing -> Healthy
```

然后，可以使用其他工具（如 [Event Exporter](https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/event-exporter) 或 [Event Router](https://github.com/heptiolabs/eventrouter)）将这些事件长期保存。

## Webhook 有效载荷

来自 webhook 事件的有效载荷被认为是不可信任的。 Argo CD 仅检查有效载荷，以推断 webhook 事件涉及的应用程序（例如哪个软件仓库被修改），然后刷新相关应用程序进行核对。 这种刷新与每隔三分钟定期进行的刷新相同，只是被 webhook 事件快速跟踪。

## Logging

### 安全领域

与安全相关的 logging 都标有 "安全 "字段，以便于查找、分析和报告。

| 级别 | 友好级别 | 描述 | 示例 | |-------|----------------|---------------------------------------------------------------------------------------------------|---------------------------------------------| | 1 | 低 | 非感知、非恶意事件 | 成功访问 | 2 | 中等 | 可能表示恶意事件，但很有可能是用户/系统错误 | 拒绝访问 | 3 | 高 | 可能是恶意事件，但其中一个没有副作用或被阻止 | repo 中的越界符号链接 | 4 | 严重 | 任何有副作用的恶意或可被利用的事件 | 秘密被遗弃在

在适用情况下，还会添加一个 `CWE` 字段，指定[通用弱点枚举](https://cwe.mitre.org/index.html) 编号。

!!!警告 请注意，目前还没有对所有安全日志进行全面标记，这些示例也不一定能实现。

### API 日志

Argo CD 会记录大多数 API 请求的有效载荷，但被认为敏感的请求除外，如 `/cluster.ClusterService/Create`、`/session.SessionService/Create` 等。方法的完整列表可参见 [server/server.go](https://github.com/argoproj/argo-cd/blob/abba8dddce8cd897ba23320e3715690f465b4a95/server/server.go#L516)。

Argo CD 不会记录请求 API 端点的客户端的 IP 地址，因为 API 服务器通常位于代理服务器之后。 相反，建议在位于 API 服务器前面的代理服务器中配置 IP 地址记录。

## ApplicationSet

Argo CD 的 ApplicationSet 功能有自己的 [安全注意事项](./applicationset/Security.md)。 在使用 ApplicationSets 之前，请注意这些问题。

## 限制目录应用程序内存 Usage

&gt;&gt; 2.2.10, 2.1.16, &gt;2.3.5

目录型应用程序（源文件为原始 JSON 或 YAML 文件的应用程序）会消耗大量 [repo-server](architecture.md#repository-server) 内存，具体取决于 YAML 文件的大小和架构。

为避免在 repo-server 中过度使用内存（可能导致崩溃和拒绝服务），请在 [argocd-cmd-params-cm](argocd-cmd-params-cm.yaml) 中设置 `reposerver.max.combined.directory.manifests.size` 配置选项。

该选项限制了单个应用程序中所有 JSON 或 YAML 文件的总和大小。 请注意，清单的内存配置可能是磁盘上清单大小的 300 倍。 还请注意，该限制是针对每个应用程序的。 如果同时为多个应用程序生成清单，内存使用量会更高。

**示例：**

假设您的版本服务器有 10G 内存限制，而您有 10 个使用原始 JSON 或 YAML 文件的应用程序。 要计算每个应用程序的最大安全组合文件大小，请用 10G 除以 300 * 10 个应用程序（300 是配置清单最坏情况下的内存增长系数）。

```
10G / 300 * 10 = 3M
```

因此，这种设置的合理安全配置是每个应用程序限制为 3M。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  reposerver.max.combined.directory.manifests.size: '3M'
```

300 倍的比率是假定配置清单文件被恶意伪造。 如果只想防止意外使用过多内存，使用较小的比率可能比较安全。

请记住，如果恶意用户可以创建更多应用程序，就会增加总内存使用量。 请谨慎授予 [应用程序创建权限](rbac.md)。