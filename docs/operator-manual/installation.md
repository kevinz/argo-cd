<!-- TRANSLATED by md-translate -->
# 安装

Argo CD 有两种安装类型：多用户和核心。

## 多租户

多租户安装是 Argo CD 最常见的安装方式。 这种安装方式通常被引用来为企业中的多个应用开发团队提供服务，并由一个平台团队进行维护。

终端用户可以使用 Web UI 或 `argocd` CLI 通过 API 服务器访问 Argo CD。必须使用 `argocd login<server-host>` 命令配置 `argocd` CLI（了解更多 [此处](../user-guide/commands/argocd_login.md)）。

Provider 提供两种配置清单：

#### 非高可用性：

不建议用于生产。 这种安装方式通常在评估期间用于演示和测试。

* [install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/install.yaml) - 具有集群管理员权限的标准 Argo CD 安装。如果您计划在 Argo CD 运行的同一集群中使用 Argo CD 部署应用程序，请引用此
配置清单集。
即 kubernetes.svc.default）中部署应用程序时，请使用此清单集。它仍能通过输入的
凭据。
* [namespace-install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/namespace-install.yaml) - 安装 Argo CD 仅需要
namespace 级权限（不需要集群角色）。如果不需要在 Argo CD 中部署应用程序，请引用此配置清单集。
使用此清单集。
输入的集群凭据。使用此配置清单集的一个例子是，如果您为不同团队运行多个
Argo CD 实例，每个实例都将应用程序部署到外部集群。
外部集群。仍然可以部署到同一个集群（kubernetes.svc.default）
输入的凭据（即 `argocd cluster add<CONTEXT> --in-cluster --namespace<YOUR NAMESPACE>`）。
    &gt; 注意：Argo CD CRD 未包含在 [namespace-install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/namespace-install.yaml) 中。
    &gt; 必须单独安装。CRD 配置清单位于 [manifests/crds](https://github.com/argoproj/argo-cd/blob/master/manifests/crds) 目录中。
    &gt; 使用以下命令安装它们：
    &gt;
    &gt; ```
    &gt; kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
    &gt; ```

#### 高可用性：

高可用性安装建议在生产中使用。 该捆绑包包括相同的组件，但针对高可用性和弹性进行了调整。

* [ha/install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/ha/install.yaml) - 与 install.yaml 相同，但为支持的组件提供了多个副本。
支持的组件的多个副本。
* [ha/namespace-install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/ha/namespace-install.yaml) - 与namespace-install.yaml相同，但支持组件的
为支持的组件提供多个副本。

## 核心

Argo CD Core 安装主要用于在无头模式下部署 Argo CD。 这种类型的安装最适合独立使用 Argo CD 且不需要多租户功能的集群管理员。 这种安装包含的组件较少，更易于设置。 捆绑安装不包含 API 服务器或用户界面，安装的是每个组件的轻量级（非 HA）版本。

安装配置清单见 [core-install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/core-install.yaml)。

有关 Argo CD Core 的更多详情，请参阅[官方文档](./core.md)

## kustomize

Argo CD 配置清单也可使用 Kustomize 安装，建议将配置清单作为远程资源，并使用 Kustomize 补丁引用其他自定义配置。

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.2/manifests/install.yaml
```

有关示例，请参阅被引用用于部署 [Argoproj CI/CD 基础架构](https://github.com/argoproj/argoproj-deployments#argoproj-deployments) 的 [kustomization.yaml](https://github.com/argoproj/argoproj-deployments/blob/master/argocd/kustomization.yaml)。

## Helm

Argo CD 可通过 [Helm](https://helm.sh/) 安装。Helm 图表目前由社区维护，可通过 [argo-helm/charts/argo-cd](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd) 获取。

## 支持的版本

有关 Argo CD 版本支持政策的详细信息，请参阅[发布流程和 Cadence 文档](https://argo-cd.readthedocs.io/en/stable/developer-guide/release-process-and-cadence/)。

## 测试版本

下表显示了与 Argo CD 各个版本一起测试的 Kubernetes 版本。

{!docs/operator-manual/tested-kubernetes-versions.md!}