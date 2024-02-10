<!-- TRANSLATED by md-translate -->
# Argo CD 核心

## 简介

Argo CD Core 是以无头模式运行 Argo CD 的另一种安装方式。 有了这种安装方式，你将拥有一个功能齐全的 GitOps 引擎，能够从 Git 仓库获取所需的状态，并将其应用于 Kubernetes。

以下功能组在此安装中不可用：

* Argo CD RBAC 模型
* Argo CD 应用程序接口
* 基于 OIDC 的身份验证

以下功能将部分被引用（详见下面的 [usage](#using) 部分）：

* Argo CD Web UI
* Argo CD CLI
* 多租户（严格基于 git 推送权限的 GitOps）

以下是一些被引用为运行 Argo CD Core 的案例：

* 作为集群管理员，我只想依赖 Kubernetes RBAC。
* 作为一名开发工程师，我不想学习新的 API 或依赖其他 CLI 来实现自动化部署。我只想依赖 Kubernetes API。
* 作为集群管理员，我不想向开发人员提供 Argo CD UI 或 Argo CD CLI。

## 架构

由于 Argo CD 在设计时考虑到了基于组件的架构，因此可以进行更简约的安装。 在这种情况下，安装的组件较少，但 GitOps 的主要功能仍可运行。

下图中，"核心 "框显示了选择 Argo CD Core 时将安装的组件：

![Argo CD Core](../assets/argocd-core-components.png)

请注意，即使 Argo CD 控制器可以在不引用 Redis 的情况下运行，也不推荐使用 Redis。 Argo CD 控制器使用 Redis 作为重要的缓存机制，以减少 Kube API 和 Git 的负载。 因此，本安装方法中也包含了 Redis。

## 安装

Argo CD Core 只需配置一个包含所有所需资源的清单文件即可安装。

例如

```
export ARGOCD_VERSION=<desired argo cd release version (e.g. v2.7.0)>
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/core-install.yaml
```

## 被引用

Argo CD Core 安装完成后，用户就可以依靠 GitOps 与之交互。 可用的 Kubernetes 资源将是 "Application "和 "ApplicationSet "CRD。 通过使用这些资源，用户可以在 Kubernetes 中部署和管理应用程序。

即使在运行 Argo CD Core 时，也可以使用 Argo CD CLI。 在这种情况下，CLI 将生成一个本地 API 服务器进程，用于处理 CLI 命令。 命令结束后，本地 API 服务器进程也将终止。 这对用户来说是透明的，无需额外的命令。 请注意，Argo CD Core 将仅依赖 Kubernetes RBAC，调用 CLI 的用户（或进程）需要有访问 Argo CD namespace 的权限，并在 `Application` 和 `ApplicationSet` 资源中拥有执行给定命令的适当权限。

要在核心模式下使用 Argo CD CLI，需要在 `login` 子命令中被引用 `--core` flag。

例如

```bash
kubectl config set-context --current --namespace=argocd # change current kube context to argocd namespace
argocd login --core
```

同样，如果用户希望使用此方法与 Argo CD 进行交互，也可以在本地运行 Web UI。 可以通过运行以下命令在本地启动 Web UI：

```
argocd admin dashboard -n argocd
```

Argo CD Web UI 将在 `http://localhost:8080` 上提供。