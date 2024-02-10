<!-- TRANSLATED by md-translate -->
# 概览

<!-- markdownlint-disable MD026 -->

## Argo CD 是什么？

<!-- markdownlint-enable MD026 -->

Argo CD 是一款适用于 Kubernetes 的声明式 GitOps 持续交付工具。

Argo光盘用户界面](assets/argocd-ui.gif)

<!-- markdownlint-disable MD026 -->

## 为什么选择 Argo CD？

<!-- markdownlint-enable MD026 -->

应用程序的定义、配置和环境应该是声明式的，并受版本控制。 应用程序的部署和生命周期管理应该是自动化的、可审计的，并易于理解。

## 开始

#### 快速入门

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

请参考我们的[入门指南](getting_started.md)。 更多面向用户的[文档](user-guide/)可用于附加功能。 如果您希望升级 Argo CD，请参阅[升级指南](./operator-manual/upgrading/overview.md)。 面向开发者的[文档](developer-guide/)可用于对构建第三方集成感兴趣的人员。

## 工作原理

Argo CD 遵循**GitOps**模式，将 Git 仓库作为定义所需应用程序状态的真实来源。 Kubernetes 配置清单可以通过多种方式指定：

* [kustomize](https://kustomize.io) 应用程序
* [helm](https://helm.sh) 图表
* [jsonnet](https://jsonnet.org) 文件
* YAML/json 配置清单的普通目录
* 任何配置为配置管理插件的自定义配置管理工具

Argo CD 可在指定的目标环境中自动部署所需的应用程序状态。 应用程序部署可跟踪分支、标签的更新，或在 Git 提交时钉入特定版本的配置清单。 有关可用的不同跟踪策略的更多详情，请参阅 [tracking strategies](user-guide/tracking_strategies.md)。

有关 Argo CD 的 10 分钟快速概述，请查看在 Sig Apps 社区会议上展示的演示：

[![Argo CD 概览演示](https://img.youtube.com/vi/aWDIQMbp1cc/0.jpg)](https://youtu.be/aWDIQMbp1cc?t=1m4s)

## 架构

![Argo CD 架构](assets/argocd_architecture.png)

Argo CD 是作为一个 Kubernetes 控制器来实现的，它能持续监控运行中的应用程序，并将当前的实时状态与所需的目标状态（在 Git 仓库中指定）进行比较。 如果部署的应用程序的实时状态与目标状态有偏差，则会被视为 "不同步"。 Argo CD 会报告并可视化差异，同时提供自动或手动同步实时状态的功能，使其恢复到所需的目标状态。 对 Git 仓库中的所需目标状态所做的任何修改都会自动应用并反映在指定的目标环境中。

更多详情，请参阅 [架构概述](operator-manual/architecture.md)。

## 功能

* 将应用程序自动部署到指定的目标环境中
* 支持多种配置管理/模板工具（kustomize、helm、Jsonnet、plain-YAML）
* 管理和部署到多个集群的能力
* SSO 集成（OIDC、OAuth2、LDAP、SAML 2.0、GitHub、GitLab、Microsoft、LinkedIn）
* 多租户和 RBAC 授权策略
* 回滚/任意滚动到 Git 仓库中提交的任何应用程序配置
* 应用资源健康状态分析
* 自动配置漂移检测和可视化
* 自动或手动将应用程序同步到所需状态
* 提供应用程序活动实时视图的 Web UI
* 用于自动化和 CI 集成的 CLI
* Webhook 集成（GitHub、BitBucket、GitLab）
* 用于自动化的访问令牌
* 同步前、同步中、同步后钩子，以支持复杂的应用程序推出（如蓝色/绿色和金丝雀升级）
* 应用程序事件和 API 调用的审计跟踪
* Prometheus 指标
* 参数重写，用于重写 Git 中的 helm 参数

## 开发现状

Argo CD 正由社区积极开发。 我们的发布可在 [此处](https://github.com/argoproj/argo-cd/releases) 找到。

## Adoption

可在 [此处](https://github.com/argoproj/argo-cd/blob/master/USERS.md) 找到已正式采用 Argo CD 的组织。