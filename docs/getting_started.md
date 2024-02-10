<!-- TRANSLATED by md-translate -->
# 入门

提示 本指南假定您已掌握 Argo CD 所基于的工具。 请阅读[了解基础知识](understand_the_basics.md) 了解这些工具。

## 要求

* 安装了 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 命令行工具。
* 拥有 [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) 文件（默认位置为 `~/.kube/config`）。
* CoreDNS.可通过 `microk8s enable dns &amp;&amp; microk8s stop &amp;&amp; microk8s start` 为 microk8s 启用。

## 1. 安装 Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

这将创建一个新的命名空间 "argocd"，Argo CD 服务和应用程序资源将存放于此。

!!! 警告 安装配置清单中包含的 `ClusterRoleBinding` 资源引用了 `argocd` 名称空间。 如果您要将 Argo CD 安装到不同的名称空间，请确保更新名称空间引用。

提示 如果您对用户界面、SSO 和多集群功能不感兴趣，那么可以只安装 [core](operator-manual/core/#installing) Argo CD 组件。

该默认安装将使用自签名证书，如果不做一些额外的工作就无法访问。 请执行以下操作之一：

* 按照[配置证书的说明](operator-manual/tls.md)进行操作（并确保客户机操作系统信任它）。
* 配置客户端操作系统以信任自签名证书。
* 在本指南中，所有 Argo CD CLI 操作均被引用 --insecure 标志。

使用 `argocd login --core` [configure](./user-guide/commands/argocd_login.md) CLI 访问并跳过步骤 3-5。

## 2. 下载 Argo CD CLI

从 [https://github.com/argoproj/argo-cd/releases/latest](https://github.com/argoproj/argo-cd/releases/latest)下载最新的 Argo CD 版本。更详细的安装说明可通过 [CLI 安装文档](cli_installation.md)查找。

还提供 Mac、Linux 和 WSL Homebrew 版本：

```bash
brew install argocd
```

## 3. 访问 Argo CD API 服务器

默认情况下，Argo CD API 服务器不公开外部 IP。 要访问 API 服务器，请选择以下技术之一公开 Argo CD API 服务器：

#### 服务类型 负载平衡器

将 argocd-server 服务类型更改为 "LoadBalancer"：

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### ingress

请按照 [ingress 文档](operator-manual/ingress.md) 了解如何使用 ingress 配置 Argo CD。

### 端口转发

Kubectl 端口转发功能也可用于连接 API 服务器，而无需公开服务。

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

然后可通过 https://localhost:8080 被引用访问 API 服务器

## 4. 使用 CLI 登录

admin "账户的初始密码会自动生成，并以明文形式存储在 Argo CD 安装命名空间中名为 "argocd-initial-admin-secret "的秘密中的 "password "字段中。 您只需使用 "argocd "CLI 即可引用该密码：

```bash
argocd admin initial-password -n argocd
```

警告 更改密码后，应删除 Argo CD 名称空间中的 "argocd-initial-admin-secret"。 该秘密除了以明文存储最初生成的密码外，没有其他用途，可随时安全删除。 如果必须重新生成新的管理员密码，Argo CD 将根据要求重新创建该秘密。

被引用用户名 "admin "和密码，登录 Argo CD 的 IP 或主机名：

```bash
argocd login <ARGOCD_SERVER>
```

!!! 注意 CLI 环境必须能够与 Argo CD API 服务器通信。 如果不能如上文第 3 步所述直接访问它，可以通过以下机制之一告诉 CLI 使用端口转发访问它：1）在每条 CLI 命令中添加 `--port-forward-namespace argocd` 标志；或 2）设置 `ARGOCD_OPTS` 环境变量：`export ARGOCD_OPTS='--port-forward-namespace argocd'`。

使用该命令更改密码：

```bash
argocd account update-password
```

## 5. 注册要部署应用程序的集群（可选）

此步骤将集群的凭据注册到 Argo CD，只有在部署到外部集群时才有必要。 在内部部署时（部署到运行 Argo CD 的同一集群），https://kubernetes.default.svc 应被引用为应用程序的 k8s API 服务器地址。

首先列出当前 kubeconfig 中的所有集群上下文：

```bash
kubectl config get-contexts -o name
```

从列表中选择一个上下文名称，并将其提供给 `argocd cluster add CONTEXTNAME`。 例如，对于 docker-desktop 上下文，运行

```bash
argocd cluster add docker-desktop
```

上述命令将一个 ServiceAccount（"argocd-manager"）安装到该 kubectl 上下文的 kube-system namespace 中，并将该服务帐户绑定到管理员级别的 ClusterRole 上。 Argo CD 使用该服务帐户令牌来执行管理任务（即部署/监控）。

注意 `argocd-manager-role` 角色的规则可以修改，使其只对有限的一组 namespace、组、种类拥有 `create`、`update`、`patch`、`delete` 权限。 不过，Argo CD 需要在集群范围内拥有 `get`、`list`、`watch` 权限。

## 6. 从 Git 仓库创建应用程序

在 [https://github.com/argoproj/argocd-example-apps.git](https://github.com/argoproj/argocd-example-apps.git)上提供了一个包含留言簿应用程序的示例资源库，以演示 Argo CD 如何工作。

### 通过 CLI 创建应用程序

首先，我们需要运行以下命令将当前 namespace 设置为 argocd：

```bash
kubectl config set-context --current --namespace=argocd
```

使用以下命令创建留言簿应用程序示例：

```bash
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
```

### 通过用户界面创建应用程序

打开浏览器，进入 Argo CD 外部用户界面，在浏览器中访问 IP/主机名并使用步骤 4 中设置的凭据登录。

登录后，点击 **+ 新应用程序** 按钮，如下图所示：

![+ 新应用程序按钮](assets/new-app.png)

将应用程序命名为 "guestbook"，被引用的项目为 "default"，同步策略为 "Manual"：

应用程序信息](assets/app-ui-information.png)

将 [https://github.com/argoproj/argocd-example-apps.git](https://github.com/argoproj/argocd-example-apps.git) repo 连接到 Argo CD，方法是将 repository url 设置为 github repo url，将修订版本保留为 `HEAD`，并将路径设置为 `guestbook`：

[连接 repo]（assets/connect-repo.png）

对于 **目的地**，将集群 URL 设置为 `https://kubernetes.default.svc`（或将集群名称设置为 `in-cluster`），将 namespace 设置为 `default`：

目的地](assets/destination.png)

填写完上述信息后，点击用户界面顶部的**创建**，即可创建 "留言簿 "应用程序：

![目的地](assets/create-app.png)

## 7. 同步（部署）应用程序

### 通过 CLI 同步

创建留言簿应用程序后，您就可以查看其状态了：

```bash
$ argocd app get guestbook
Name:               guestbook
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://10.97.164.88/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
Sync Policy:        <none>
Sync Status:        OutOfSync from  (1ff8a67)
Health Status:      Missing

GROUP KIND NAMESPACE NAME STATUS HEALTH
apps Deployment default guestbook-ui OutOfSync Missing
       Service default guestbook-ui OutOfSync Missing
```

应用程序状态最初处于 "OutOfSync "状态，因为应用程序尚未部署，也未创建任何 Kubernetes 资源。 要同步（部署）应用程序，请运行

```bash
argocd app sync guestbook
```

该命令从资源库中获取配置清单，并对清单执行 "kubectl apply"。 guestbook 应用程序现已运行，你现在可以查看其资源组件、日志、事件和评估的健康状态。

### 通过用户界面进行同步

[留言簿应用程序](assets/guestbook-app.png) ![查看应用程序](assets/guestbook-tree.png)