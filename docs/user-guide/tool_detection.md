<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 检测工具

用于构建应用程序的工具检测如下：

如果显式配置了特定工具，则会选择该工具来创建应用程序的清单。

可以在应用程序自定义资源中明确指定该工具，就像这样：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  ...
spec:
  ...
  source:
    ...

    # Tool -> plain directory
    directory:
      recurse: true
...
```

您也可以在网络用户界面的应用程序创建向导中选择工具。 默认为 "目录"。 如果要选择其他工具，请按工具名称下方的下拉按钮。

如果没有，则按以下方式隐式检测工具：

** helm** 如果有匹配`Chart.yaml`的文件。 **Kustomize** 如果有`kustomization.yaml`、`kustomization.yml`或`Kustomization`文件。

否则假定它是一个普通的**目录**申请。

## 禁用内置工具

可以通过在`argocd-cm`ConfigMap，以`false`:`kustomize.enable`,`helm.enable`或`jsonnet.enable`一旦禁用该工具，Argo CD 将假定应用程序目标目录包含纯 Kubernetes YAML 清单。

禁用未被引用的配置管理工具是一种有益的安全增强措施。 漏洞有时仅限于某些配置管理工具。 即使没有漏洞，攻击者也可能利用 Argo CD 实例中的错误配置使用某些工具。 禁用未被引用的配置管理工具限制了恶意行为者可用的工具。

## 参考文献

* [reposerver/repository/repository.go/GetAppSourceType](https://github.com/argoproj/argo-cd/blob/master/reposerver/repository/repository.go#L286) * [server/repository/repository.go/listAppType](https://github.com/argoproj/argo-cd/blob/master/server/repository/repository.go#L97)
