<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 目录

目录类型的应用程序会从`.yml`,`.yaml`和`.json`目录类型应用程序可以通过用户界面、CLI 或声明方式创建。 这是声明语法：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
```

无需明确添加`spec.source.directory`Argo CD 会自动检测源代码库/路径是否包含纯文件清单。

## 启用递归资源检测

默认情况下，目录应用程序只包含配置的存储库/路径根目录下的文件。

要启用递归资源检测，请设置`递归`选择。

```bash
argocd app set guestbook --directory-recurse
```

以声明方式做同样的事情，请使用以下语法：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    directory:
      recurse: true
```

!!! 警告 Directory 类型的应用程序仅适用于普通清单文件。 如果 Argo CD 在设置目录: 时遇到 Kustomize、 helm 或 Jsonnet 文件，它将无法渲染清单。

## 包括/排除文件

### 只包括某些文件

要在目录应用程序中只包含某些文件/目录，请设置`include`值是一个 glob 模式。

例如，如果您只想包含`.yaml`文件，就可以被引用这种模式：

```shell
argocd app set guestbook --directory-include "*.yaml"
```

注意，必须引用`*.yaml`这样，外壳在将图案发送到 Argo CD 之前就不会扩展图案。

也可以包含多个模式。`{}`要包括`.yml`和`.yaml`文件时，被引用这种模式：

```shell
argocd app set guestbook --directory-include "{*.yml,*.yaml}"
```

只包含某个目录，请使用类似这样的模式：

```shell
argocd app set guestbook --directory-include "some-directory/*"
```

以声明的方式完成同样的任务，请使用以下语法：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    directory:
      include: 'some-directory/*'
```

### 排除某些文件

例如，在包含一些清单和一个非清单 yaml 文件的版本库中，可以像这样排除配置文件：

```shell
argocd app set guestbook --directory-exclude "config.yaml"
```

可以排除多个模式，例如配置文件和无关目录：

```shell
argocd app set guestbook --directory-exclude "{config.yaml,env-use2/*}"
```

如果两个`include`和`exclude`则应用程序将包含符合`include`模式，并且不符合`exclude`例如，请看这个源码库：

```
config.json
deployment.yaml
env-use2/
  configmap.yaml
env-usw2/
  configmap.yaml
```

排除`config.json`和`env-usw2`目录，可以被引用这种组合模式：

```shell
argocd app set guestbook --directory-include "*.yaml" --directory-exclude "{config.json,env-usw2/*}"
```

这就是声明式语法：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    directory:
      exclude: '{config.json,env-usw2/*}'
      include: '*.yaml'
```