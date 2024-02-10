<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# kustomize

kustomize 有以下配置选项：

* `namePrefix` 是附加到 kustomize 应用程序资源的前缀 * `nameSuffix` 是附加到 kustomize 应用程序资源的后缀 * `images` 是 Kustomize 镜像覆盖列表 * `replicas` 是 Kustomize 复制覆盖列表 * `commonLabels` 是附加标签的字符串映射 * `forceCommonLabels` 是一个布尔值，用于定义是否允许覆盖现有标签 * `commonAnnotations` 是附加注释的字符串映射 * `namespace` 是一个 Kubernetes 资源名称空间 * `forceCommonAnnotations` 是一个布尔值，用于定义是否允许覆盖现有注释 * `namespace` 是一个 Kubernetes 资源名称空间 * `namespace` 是一个 Kubernetes 资源名称空间。namespace`是一个 Kubernetes 资源命名空间 * `forceCommonAnnotations` 是一个布尔值，用于定义是否允许覆盖现有注释 * `commonAnnotationsEnvsubst` 是一个布尔值，用于在注释值中启用环境变量替换 * `patches` 是一个支持内联更新的 Kustomize 补丁列表 * `components` 是一个 Kustomize 组件列表

要将 kustomize 与覆盖层一起使用，请将路径指向覆盖层。

提示 如果您正在生成资源，您应该阅读如何使用[忽略无关内容](compare-options.md)。

## 补丁

补丁是在 Argo CD 应用程序中使用内联配置对资源进行自定义的一种方式。`patches`任何针对现有 kustomization 文件的补丁都将被合并。

这个 kustomize 示例从`/kustomize-guestbook`文件夹中的`argoproj/argocd-example-apps`资源库，并修补`Deployment`被引用到端口`443`容器上。

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomize-inline-example
namespace: test1
resources:
  - https://raw.githubusercontent.com/argoproj/argocd-example-apps/master/kustomize-guestbook/
patches:
  - target:
      kind: Deployment
      name: guestbook-ui
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/ports/0/containerPort
        value: 443
```

这`Application`被引用时，会产生等效的`kustomize.patches`配置。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-inline-guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: test1
    server: https://kubernetes.default.svc
  project: default
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: master
    kustomize:
      patches:
        - target:
            kind: Deployment
            name: guestbook-ui
          patch: |-
            - op: replace
              path: /spec/template/spec/containers/0/ports/0/containerPort
              value: 443
```

内嵌式 kustomize 补丁可以很好地与｀ApplicationSets＇现在，无需为每个集群维护一个修补程序或覆盖程序，而可以在｀Application＇例如，使用[外部-dns](https://github.com/kubernetes-sigs/external-dns/)以设置[txt-Owner-id](https://github.com/kubernetes-sigs/external-dns/blob/e1adc9079b12774cccac051966b2c6a3f18f7872/docs/registry/registry.md?plain=1#L6)到集群名称。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: external-dns
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters: {}
  template:
    metadata:
      name: 'external-dns'
    spec:
      project: default
      source:
        repoURL: https://github.com/kubernetes-sigs/external-dns/
        targetRevision: v0.14.0
        path: kustomize
        kustomize:
          patches:
          - target:
              kind: Deployment
              name: external-dns
            patch: |-
              - op: add
                path: /spec/template/spec/containers/0/args/3
                value: --txt-owner-id={{.name}}   # patch using attribute from generator
      destination:
        name: 'in-cluster'
        namespace: default
```

## 组件

kustomize[部件](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/components.md)它们提供了在 Kubernetes 应用程序中模块化和重复使用配置的强大方法。

在 Argo CD 之外，要使用组件，必须将以下内容添加到`kustomization.yaml`例如

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
...
components:
- ../component
```

增加了对`v2.10.0`现在，您可以在应用程序中直接引用组件：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application-kustomize-components
spec:
  ...
  source:
    path: examples/application-kustomize-components/base
    repoURL: https://github.com/my-user/my-repo
    targetRevision: main

    # This!
    kustomize:
      components:
        - ../component  # relative to the kustomization.yaml (`source.path`).
```

## 私人远程基地

如果你的远程基地是 (a) HTTPS，需要用户名/密码 (b) SSH，需要 SSH 私钥，那么它们将从应用程序的 repo 中继承用户名/密码。

如果远程基地使用相同的凭据/私钥，这将起作用。 如果它们使用不同的凭据/私钥，这将不起作用。 出于安全原因，您的应用程序只知道自己的 repo（不包括其他团队或用户的 repo），因此即使 Argo CD 知道其他私有 repo，您也无法访问它们。

了解更多[私有库](private-repositories.md)。

## `kustomize build` 选项/参数

为以下人员提供选项构建`kustomize build`的默认 kustomize 版本，使用`kustomize.buildOptions`领域`argocd-cm`配置地图。 被引用`kustomize.buildOptions.<version>`来注册特定版本的构建选项。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
    kustomize.buildOptions: --load-restrictor LoadRestrictionsNone
    kustomize.buildOptions.v4.4.0: --output /tmp
```

修改后`kustomize.buildOptions`您可能需要重新启动 ArgoCD 才能使更改生效。

## kustomize 定制版本

Argo CD 支持同时使用多个 kustomize 版本，并为每个应用程序指定了所需版本。 要添加其他版本，请确保所需的版本是[捆绑](./operator-manual/custom_tools.md)然后被引用`kustomize.path.<version>`领域`argocd-cm`ConfigMap 用于注册捆绑的附加版本。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
    kustomize.path.v3.5.1: /custom-tools/kustomize_3_5_1
    kustomize.path.v3.5.4: /custom-tools/kustomize_3_5_4
```

一旦配置了新版本，就可以在应用程序规范中引用它，如下所示：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: kustomize-guestbook

    kustomize:
      version: v3.5.4
```

此外，还可使用 "应用程序 "详细信息 "页面的 "参数 "选项卡或以下 CLI 命令配置应用程序的 kustomize 版本： kustomize

```bash
argocd app set <appName> --kustomize-version v3.5.4
```

## 构建环境

kustomize 应用程序可以访问[标准构建环境](build-environment.md)可与被引用的[配置管理插件](../operator-manual/config-management-plugins.md)来改变已呈现的清单。

您可以在 Argo CD 应用程序清单中被引用这些构建环境变量。 您可以通过设置`.spec.source.kustomize.commonAnnotationsEnvsubst`至`true`在应用程序清单中。

例如，以下应用程序清单将设置`app-source`注释的应用程序名称：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-app
  namespace: argocd
spec:
  project: default
  destination:
    namespace: demo
    server: https://kubernetes.default.svc
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
    kustomize:
      commonAnnotationsEnvsubst: true
      commonAnnotations:
        app-source: ${ARGOCD_APP_NAME}
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

## kustomize helm charts

可以[使用 Kustomize 渲染 Helm 图表](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/chart.md)这样做需要通过`--enable-helm`标志到`kustomize build`该标记不属于 Argo CD 中的 Kustomize 选项。 如果您想在 Argo CD 应用程序中通过 Kustomize 渲染 Helm 图表，您有两个选择：您可以创建一个[自定义插件](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/)或修改`argocd-cm`ConfigMap 包括`--enable-helm`标志，适用于所有 kustomize 应用程序： kustomize

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  kustomize.buildOptions: --enable-helm
```

## 设置清单的 namespace

`spec.destination.namespace`字段只在 Kustomize 生成的清单中缺少名称空间时才会添加该名称空间。 它还会引用`kubectl`来设置名称空间，有时会漏掉某些资源（例如自定义资源）中的 namespace 字段。 在这种情况下，可能会出现类似下面的错误：`ClusterRoleBinding.rbac.authorization.k8s.io "example" is invalid: subjects[0].namespace: Required value.`

直接被引用来设置缺失的 namespace 可以解决这个问题。 设置`spec.source.kustomize.namespace`指示 kustomize 将 namespace 字段设置为给定值。

如果`spec.destination.namespace`和`spec.source.kustomize.namespace`同时设置，Argo CD 将遵从后者，即 Kustomize 设置的名称空间值。