<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 最佳做法

## 区分配置库和源代码库

从以下原因出发，强烈建议使用单独的 Git 仓库来保存 Kubernetes 清单，将配置与应用程序源代码分开： Kubernetes 清单，将配置与应用程序源代码分开： Kubernetes 清单，将配置与应用程序源代码分开

1.Provider 提供了应用程序代码与应用程序配置的清晰分离。有时，您只想修改配置清单，而不想触发整个 CI 构建。例如，如果您只想增加部署规范中的副本数量，您可能不想触发构建。 2.更整洁的审计日志。出于审计目的，一个只保存配置的 repo 会有更清晰的 Git 历史记录，记录所做的更改，而不会出现因正常开发活动而产生的签入噪音。 3.您的应用程序可能由多个 Git 仓库构建的服务组成，但作为一个整体部署。通常情况下，微服务应用程序由具有不同版本方案和发布周期的服务组成（如 ELK、Kafka + ZooKeeper）。将配置清单存储在单个组件的某个源代码库中可能并不合理。 4.访问分离。开发应用程序的开发人员不一定是同一个人。

## 给不自信留有余地

可能需要为一些不确定性/自动化留有余地，而不是在 Git 清单中定义所有内容。[水平吊舱自动定标器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)那么您就不想跟踪`replicas`在 Git 中。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  # do not include replicas in the manifests if you want replicas to be controlled by HPA
  # replicas: 1
  template:
    spec:
      containers:
      - image: nginx:1.7.9
        name: nginx
        ports:
        - containerPort: 80
...
```

## 确保 Git 修订时的 Manifests 配置清单 真正不可变

使用模板工具时，如`helm`或`kustomize`这通常是由于上游 helm 资源库或 kustomize 库发生了变更。

例如，请看下面的 kustomization.yaml

```yaml
resources:
- github.com/argoproj/argo-cd//manifests/cluster-install
```

上述 kustomize 的远程基础是 Argo-cd repo 的 HEAD 修订版。 由于这不是一个稳定的目标，即使你自己的 Git 仓库没有任何改动，这个 kustomize 应用程序的清单也可能突然改变含义。

更好的版本是使用 Git 标签或提交 SHA，例如

```yaml
bases:
- github.com/argoproj/argo-cd//manifests/cluster-install?ref=v0.11.1
```