<!-- TRANSLATED by md-translate -->
# 入门

本指南假定您熟悉 Argo CD 及其基本概念。 更多信息请参阅 [Argo CD 文档](./../core_concepts.md)。

## 要求

* 已安装 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 命令行工具
* 拥有 [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) 文件（默认位置为 `~/.kube/config`）。

## 安装

安装 ApplicationSet 控制器有几种选择。

### A) 安装作为 Argo CD 一部分的 ApplicationSet

从 Argo CD v2.3 版开始，ApplicationSet 控制器与 Argo CD 捆绑，不再需要从 Argo CD 单独安装 ApplicationSet 控制器。

请按照 [Argo CD 入门](../../getting_started.md) 说明获取更多信息。

### B) 将 ApplicationSet 安装到现有的 Argo CD 安装中（Argo CD v2.3 之前的版本）

**注意**：这些说明仅适用于 Argo CD v2.3.0 之前的版本。

ApplicationSet 控制器必须与 Argo CD 安装在同一个 namespace 中。

假设 Argo CD 已安装到 `argocd` 名称空间，请运行以下命令：

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/applicationset/v0.4.0/manifests/install.yaml
```

安装后，ApplicationSet 控制器无需额外设置。

manifests/install.yaml "文件包含安装 ApplicationSet 控制器所需的 Kubernetes 配置清单：

* ApplicationSet` 资源的自定义资源定义
* 用于 `argocd-applicationset-controller` 的部署
* 被应用集控制器引用的服务帐号，用于访问 Argo CD 资源
* 为 ServiceAccount 授予 RBAC 访问所需资源权限的角色
* 角色绑定用于绑定服务账户和角色

<!-- ### C) Install development builds of ApplicationSet controller for access to the latest features

Development builds of the ApplicationSet controller can be installed by running the following command:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/applicationset/master/manifests/install.yaml
```

With this option you will need to ensure that Argo CD is already installed into the `argocd` namespace.

How it works:

- After each successful commit to *argoproj/applicationset* `master` branch, a GitHub action will run that performs a container build/push to [`argoproj/argocd-applicationset:latest`](https://quay.io/repository/argoproj/argocd-applicationset?tab=tags )
- [Documentation for the `master`-branch-based developer builds](https://argocd-applicationset.readthedocs.io/en/master/)  is available from Read the Docs.

!!! warning
    Development builds contain newer features and bug fixes, but are more likely to be unstable, as compared to release builds.

See the `master` branch [Read the Docs](https://argocd-applicationset.readthedocs.io/en/master/) page for documentation on post-release features. -->

<!-- ## Upgrading to a Newer Release

To upgrade from an older release (eg 0.1.0, 0.2.0) to a newer release (eg 0.3.0), you only need to `kubectl apply` the `install.yaml` for the new release, as described under *Installation* above.

There are no manual upgrade steps required between any release of ApplicationSet controller, (including 0.1.0, 0.2.0, and 0.3.0) as of this writing, however, see the behaviour changes in ApplicationSet controller v0.3.0, below.

### Behaviour changes in ApplicationSet controller v0.3.0

There are no breaking changes, however, a couple of behaviours have changed from v0.2.0 to v0.3.0. See the [v0.3.0 upgrade page](upgrading/v0.2.0-to-v0.3.0.md) for details. -->

## 启用高可用性模式

要启用高可用性，必须在 argocd-applicationset-controller 容器中设置"--enable-leader-election=true "命令，并增加副本。

对配置清单/install.yaml 做以下修改

```bash
spec:
      containers:
      - command:
        - entrypoint.sh
        - argocd-applicationset-controller
        - --enable-leader-election=true
```

### 可选：额外的升级后保障措施

请参阅[控制资源修改](Controlling-Resource-Modification.md) 页面，了解您可能希望添加到 `install.yaml` 中 Providers 资源的其他参数，以提供额外的安全性，防止任何初始的、意外的升级后行为。

例如，要暂时阻止升级后的 ApplicationSet 控制器进行任何更改，可以这样做：

* 启用干运行
* 被引用仅创建策略
* 在应用程序集上启用 "preserveResourcesOnDeletion
* 暂时禁用 ApplicationSet 模板中的自动同步功能

这些参数可让您观察/控制新版 ApplicationSet 控制器在环境中的行为，确保您对结果感到满意（详情请查看 ApplicationSet logging 文件）。 测试完成后，别忘了删除任何临时更改即可！

不过，如上所述，这些步骤并非绝对必要：ApplicationSet 控制器的升级应该是一个微创过程，这些步骤只是建议作为额外安全的可选预防措施。

## 接下来的步骤

ApplicationSet 控制器启动并运行后，请访问 [Use Cases](Use-Cases.md)，了解更多有关支持场景的信息，或直接访问 [Generators](Generators.md) 查看示例 `ApplicationSet` 资源。