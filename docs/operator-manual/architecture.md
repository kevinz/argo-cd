<!-- TRANSLATED by md-translate -->
# 架构概览

![Argo CD 架构](../assets/argocd_architecture.png)

## 组件

### API 服务器

API 服务器是一个 gRPC/REST 服务器，用于公开 Web UI、CLI 和 CI/CD 系统使用的 API。 它的职责如下：

* 应用程序管理和状态报告
* 调用应用程序操作（如同步、回滚、用户自定义操作）
* 存储库和集群凭证管理（存储为 k8s secret）
* 外部身份 Provider 的认证和认证授权
* RBAC 执行
* Git webhook 事件的监听器/转发器

### 存储库服务器

配置清单服务器是一个内部服务，负责维护一个本地缓存，缓存中包含应用程序配置清单的 Git 仓库。 它负责在提供以下输入时生成并返回 Kubernetes 配置清单：

* 版本库 URL
* 修订（提交、标记、分支）
* 应用程序路径
* 模板特定设置：参数、helm values.yaml

### 应用控制器

应用程序控制器是一个 Kubernetes 控制器，它能持续监控运行中的应用程序，并将当前的实时状态与所需的目标状态（在软件仓库中指定）进行比较。 它能检测到 "OutOfSync "应用程序状态，并有选择地采取纠正措施。 它负责调用任何用户定义的生命周期事件钩子（PreSync、Sync、PostSync）。