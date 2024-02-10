<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 参数覆盖

Argo CD 提供了一种利用配置管理工具覆盖 Argo CD 应用程序参数的机制。 这为在 Git 中定义大部分应用程序清单提供了灵活性，同时也为有些这也是通过 Argo CD 更改应用程序参数，而不是在 Git 中更改清单来重新部署应用程序的另一种方式。

提示 许多人认为这种操作模式与 GitOps 模式相悖，因为真相的来源变成了 Git 仓库和应用程序重写的结合。 Argo CD 参数重写功能主要是为开发人员提供方便，旨在用于开发/测试环境，而非生产环境。

要被引用参数覆盖，请运行`argocd app set -p (COMPONENT=)PARAM=VALUE`指挥：

```bash
argocd app set guestbook -p image=example/guestbook:abcd123
argocd app sync guestbook
```

参数`预计是一个正常的 yaml 路径

```bash
argocd app set guestbook -p ingress.enabled=true
argocd app set guestbook -p ingress.hosts[0]=guestbook.myclusterurl
```

`argocd app set`[指挥部](./commands/argocd_app_set.md)支持更多针对特定工具的标记，例如`--kustomize-image`,`--jsonnet-ext-var-str`您也可以直接在应用程序规格的源字段中指定覆盖。 了解相应工具中支持选项的更多信息[文献资料](./application_sources.md)。

## When To Use Overrides?

以下是参数重载被引用的情况：

1.一个团队维护一个 "开发 "环境，该环境需要不断用最新的

为了解决这个用例，应用程序将暴露一个名为`image`中被引用的值。`dev`环境中包含一个占位符值（例如`example/guestbook:replaceme`占位符的值将由外部（Git 以外）确定，例如由构建系统确定。 然后，作为构建流水线的一部分，该占位符的参数值将由`image`将不断更新为新构建的映像（例如`argocd app set guestbook -p image=example/guestbook:abcd123`同步操作将导致应用程序使用新映像重新部署。

2.Helm 清单库已经公开（如 https://github.com/helm/charts）。

由于无法获得对版本库的提交访问权限，因此能够从公共版本库安装图表并使用不同的参数自定义部署，而无需分叉版本库来进行更改，是非常有用的。 例如，要从 helm 图表版本库安装 Redis 并自定义数据库密码，您可以运行

```bash
argocd app create redis --repo https://github.com/helm/charts.git --path stable/redis --dest-server https://kubernetes.default.svc --dest-namespace default -p password=abc123
```

## 将重写存储在 Git 中

配置管理工具的特定重载可在`.argocd-source.yaml`文件存储在 Git 仓库的源程序目录中。

.argocd-source.yaml`文件在清单生成过程中被引用，并覆盖应用程序源字段，如`kustomize`,`helm`等等。

例如

```yaml
kustomize:
  images:
    - gcr.io/heptio-images/ks-guestbook-demo:0.2
```

.argocd-source`正试图解决以下两个主要被引用的案例：

* 提供在 Git 中 "覆盖 "应用程序参数的统一方式，并启用 "回写 "功能

等项目[argocd-镜像-updater](https://github.com/argoproj-labs/argocd-image-updater)。

* 支持通过 [applicationset](https://github.com/argoproj/applicationset) 等项目在 Git 仓库中 "发现 "应用程序

(见[git 文件生成器](https://github.com/argoproj/argo-cd/blob/master/applicationset/examples/git-generator-files-discovery/git-generator-files.yaml))

如果要从资源库中的单一路径获取多个应用程序，也可以将参数重载存储在特定应用程序文件中。

应用程序专用文件必须命名为`.argocd-source-<appname>.yaml`其中`<appname>`是覆盖有效的应用程序名称。

如果存在一个非应用程序特定的`.argocd-source.yaml`首先合并该文件中包含的参数，然后合并应用程序特定参数，这些参数也可以包含对存储在非应用程序特定文件中的参数的覆盖。