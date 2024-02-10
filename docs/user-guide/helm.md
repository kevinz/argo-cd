<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# Helm

## 式 申明

你可以通过用户界面安装 Helm chart，也可以用声明式 GitOps 方式安装。Helm 是[只被引用来为图表充气。](.../.../faq#after-deploying-my-helm-application-with-argo-cd-i-cannot-see-it-with-helm-ls-and-other-helm-commands)应用程序的生命周期由 Argo CD 代替 Helm 处理。 下面是一个示例： Helm chart 是[只被引用来为图表充气。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  source:
    chart: sealed-secrets
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 1.16.1
    helm:
      releaseName: sealed-secrets
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubeseal
```

另一个被引用的例子是公开的 OCI helm chart： 舵图

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
spec:
  project: default
  source:
    chart: nginx
    repoURL: registry-1.docker.io/bitnamicharts  # note: the oci:// syntax is not included.
    targetRevision: 15.9.0
  destination:
    name: "in-cluster"
    namespace: nginx
```

注意 "当使用多种方式 Providers 值时"，优先顺序为`parameters &gt; valuesObject &gt; values &gt; valueFiles &gt; helm repository values.yaml`（见[这里](./helm.md#helm-value-precedence)更详细的例子）。

## 值文件

Helm 可以使用不同的，甚至多个 "values.yaml "文件来获取参数。 可以使用`--values`该标志可以重复使用，以支持多个值文件： Values.yaml "文件来获取参数。

```bash
argocd app set helm-guestbook --values values-production.yaml
```

注意之前`v2.6`在 Argo CD 中，值文件必须与 Helm 图表位于同一个 git 仓库中。 文件可以位于不同的位置，在这种情况下，可以使用相对于 Helm 图表根目录的相对路径来访问它。 截至`v2.6`通过利用 Helm chart，可以从与 Helm chart 不同的存储库中获取值文件。[应用程序的多个来源](./multiple_sources.md#helm-value-files-from-external-git-repository)。

在声明式语法中

```yaml
source:
  helm:
    valueFiles:
    - values-production.yaml
```

## 数值

Argo CD 支持在应用程序清单中直接使用等效的值文件，即使用`source.helm.valuesObject`键。

```yaml
source:
  helm:
    valuesObject:
      ingress:
        enabled: true
        path: /
        hosts:
          - mydomain.example.com
        annotations:
          kubernetes.io/ingress.class: nginx
          kubernetes.io/tls-acme: "true"
        labels: {}
        tls:
          - secretName: mydomain-tls
            hosts:
              - mydomain.example.com
```

或者，可以使用`source.helm.values`键。

```yaml
source:
  helm:
    values: |
      ingress:
        enabled: true
        path: /
        hosts:
          - mydomain.example.com
        annotations:
          kubernetes.io/ingress.class: nginx
          kubernetes.io/tls-acme: "true"
        labels: {}
        tls:
          - secretName: mydomain-tls
            hosts:
              - mydomain.example.com
```

## Helm 参数

Helm 具有设置参数值的功能，这些参数值可以覆盖在`values.yaml`例如`service.type`是 Helm 图表中常见的参数： Values.

```bash
helm template . --set service.type=LoadBalancer
```

同样，Argo CD 可以覆盖`values.yaml`参数被引用`argocd app set`命令，格式为`-p PARAM=VALUE`例如

```bash
argocd app set helm-guestbook -p service.type=LoadBalancer
```

在声明式语法中

```yaml
source:
  helm:
    parameters:
    - name: "service.type"
      value: LoadBalancer
```

## Helm 值优先级

值注入的优先顺序如下`parameters &gt; valuesObject &gt; values &gt; valueFiles &gt; helm repository values.yaml` 或者说

```
lowest  -> valueFiles
            -> values
            -> valuesObject
    highest -> parameters
```

因此，values/valuesObject 优先于 valueFiles，而参数优先于两者。

valueFiles 本身的优先级是它们在以下文件中定义的顺序

```
if we have

valuesFile:
  - values-file-2.yaml
  - values-file-1.yaml

the last values-file i.e. values-file-1.yaml will trump the first
```

当找到多个相同密钥时，最后一个获胜，即

```
e.g. if we only have values-file-1.yaml and it contains

param1: value1
param1: value3000

we get param1=value3000
```

```
parameters:
  - name: "param1"
    value: value2
  - name: "param1"
    value: value1

the result will be param1=value1
```

```
values: |
  param1: value2
  param1: value5

the result will be param1=value5
```

注意 "当使用 valuesFiles 或 values 时"，在ui 中看到的参数列表不是被引用的资源，而是与参数合并的 values/valuesObject（请参阅[本期](https://github.com/argoproj/argo-cd/issues/9213)作为一种变通方法，使用参数而不是值/valuesObject 可以更好地概括资源将被引用的内容。

## Helm 发布名称

在默认情况下，Helm 的发布名称等于其所属的应用程序名称。 有时，尤其是在集中式 Argo CD 上，您可能希望覆盖该名称，这可以通过使用`release-name`标志： "......"。

```bash
argocd app set helm-guestbook --release-name myRelease
```

或被引用为 yaml 的 releaseName：

```yaml
source:
    helm:
      releaseName: myRelease
```

!!! 警告 "关于覆盖发布名称的重要通知" 请注意，覆盖 Helm 发布名称可能会在您部署的图表被引用了`app.kubernetes.io/instance`Argo CD 会将应用程序名称的值注入此标签以进行跟踪。 因此，当覆盖发布名称时，应用程序名称将不再等于发布名称。 由于 Argo CD 会用应用程序名称覆盖标签，因此可能会导致资源上的某些选择器停止工作。 为了避免这种情况，我们可以在[ArgoCD 配置表 argocd-cm.yaml](../operator-manual/argocd-cm.yaml)- 检查描述`application.instanceLabelKey`。

## Helm 挂钩

舵钩类似于[阿尔戈 CD 钩子](resource_hooks.md)在 Helm 中，钩子是任何正常的 Kubernetes 资源，并用`helm.sh/hook`注释。

Argo CD 通过将 Helm 注释映射到 Argo CD 自身的钩子注释上，支持许多（大部分） Helm 钩子：

| Helm Annotation | Notes | | ------------------------------- |-----------------------------------------------------------------------------------------------| | |`helm.sh/hook: crd-install`| 支持相当于`argocd.argoproj.io/hook: PreSync`. | |`helm.sh/hook: pre-delete`| 不支持。 在 Helm 稳定版中，有 3 种情况被引用来清理 CRD，3 种情况被引用来清理作业。`helm.sh/hook: pre-rollback`| 不支持，从未在 Helm 稳定版中被引用。`helm.sh/hook: pre-install`| 支持相当于`argocd.argoproj.io/hook: PreSync`. | |`helm.sh/hook: pre-upgrade`| 支持相当于`argocd.argoproj.io/hook: PreSync`. | |`helm.sh/hook: post-upgrade`| 支持相当于`argocd.argoproj.io/hook: PostSync`. | |`helm.sh/hook: post-install`| 支持相当于`argocd.argoproj.io/hook: PostSync`. | |`helm.sh/hook: post-delete`| 支持相当于`argocd.argoproj.io/hook: PostDelete`. | |`helm.sh/hook: post-rollback`| 不支持，从未在 Helm 稳定版中被引用。`helm.sh/hook: test-success`| 不支持，Argo CD

不支持的钩子会被忽略。 在 Argo CD 中，钩子是通过引用`kubectl apply`而不是`kubectl create`这意味着，如果钩子已被命名并且已经存在，它将不会更改，除非您用`before-hook-creation`。

!!! 警告 "Helm 钩子 + ArgoCD 钩子" 如果您定义了任何 Argo CD 钩子、全部_舵钩将被忽略。

!!! 警告"'安装' vs '升级' vs '同步'" Argo CD 无法知道它是在执行首次 "安装 "还是 "升级"--每次操作都是 "同步"。 这意味着，默认情况下，已经安装过 "安装 "或 "升级 "的应用程序将不会再运行。`pre-install`和`pre-upgrade`将同时运行这些钩子。

#### 吊钩技巧

* 在`pre-install`和`post-install`中注释`hook-weight: "-1"` 以确保它在任何安装或升级钩子之前成功运行。 在`pre-upgrade`和`post-upgrade`中注释`hook-delete-policy: before-hook-creation`以确保它在每次同步时运行。

了解更多[阿尔戈钩](resource_hooks.md)和[Helm 挂钩](https://helm.sh/docs/topics/charts_hooks/)。

## 随机数据

helm 模板能够在图表渲染过程中通过`randAlphaNum`功能。[图表库](https://github.com/helm/charts)例如，以下是被引用的 Secret[redis helm chart](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml)：

```yaml
data:
  {{- if .Values.password }}
  redis-password: {{ .Values.password | b64enc | quote }}
  {{- else }}
  redis-password: {{ randAlphaNum 10 | b64enc | quote }}
  {{- end }}
```

Argo CD 应用程序控制器会定期比较 Git 状态和实时状态，运行`helm template<CHART>`命令来生成 helm 清单。 由于每次比较时都会重新生成随机值，因此任何被引用了`randAlphaNum`函数将始终在`OutOfSync`这可以通过在 values.yaml 中明确设置一个值或被引用为`argocd app set`命令来覆盖数值，使每次比较之间的数值保持稳定。 例如

```bash
argocd app set redis -p password=abc123
```

## 构建环境

Helm 应用程序可以访问[标准构建环境](build-environment.md)通过替换作为参数。

例如，通过 CLI：

```bash
argocd app create APPNAME \
  --helm-set-string 'app=${ARGOCD_APP_NAME}'
```

或者通过声明式语法：

```yaml
spec:
    source:
      helm:
        parameters:
        - name: app
          value: $ARGOCD_APP_NAME
```

也可以使用构建环境变量来设置 Helm 值文件路径： Helm

```yaml
spec:
    source:
      helm:
        valueFiles:
        - values.yaml
        - myprotocol://somepath/$ARGOCD_APP_NAME/$ARGOCD_APP_REVISION
```

## Helm 插件

Argo CD 对您使用的云提供商和 Helm 插件没有任何意见，这也是 ArgoCD 镜像没有被引用插件的原因。

但有时您想使用自定义插件。 也许您想使用 Google 云存储或 Amazon S3 存储来保存 Helm 图表，例如：https://github.com/hayorov/helm-gcs，在这里您可以使用`gs://`有两种方法可以安装自定义插件；可以修改 ArgoCD 容器镜像，也可以使用 Kubernetes 的`initContainer`。

### 修改 ArgoCD 容器映像

使用该插件的一种方法是准备自己的 ArgoCD 图像，其中包含该插件。

示例`Dockerfile`：

```dockerfile
FROM argoproj/argocd:v1.5.7

USER root
RUN apt-get update && \
    apt-get install -y \
        curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER argocd

ARG GCS_PLUGIN_VERSION="0.3.5"
ARG GCS_PLUGIN_REPO="https://github.com/hayorov/helm-gcs.git"

RUN helm plugin install ${GCS_PLUGIN_REPO} --version ${GCS_PLUGIN_VERSION}

ENV HELM_PLUGINS="/home/argocd/.local/share/helm/plugins/"
```

您必须记住`helm_plugins`环境属性 - 这是插件正常工作所必需的。

之后，您必须使用自定义镜像来安装 ArgoCD。

### 使用 `initContainers

另一种方法是通过 Kubernetes 安装 Helm 插件`initContainers`一些用户发现这种模式比维护他们自己版本的 ArgoCD 容器映像更好。

下面是一个示例，说明如何在安装 ArgoCD 时添加 Helm 插件。[ArgoCD 官方舵图](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd)：

```yaml
repoServer:
  volumes:
    - name: gcp-credentials
      secret:
        secretName: my-gcp-credentials
  volumeMounts:
    - name: gcp-credentials
      mountPath: /gcp
  env:
    - name: HELM_CACHE_HOME
      value: /helm-working-dir
    - name: HELM_CONFIG_HOME
      value: /helm-working-dir
    - name: HELM_DATA_HOME
      value: /helm-working-dir
  initContainers:
    - name: helm-gcp-authentication
      image: alpine/helm:3.8.1
      volumeMounts:
        - name: helm-working-dir
          mountPath: /helm-working-dir
        - name: gcp-credentials
          mountPath: /gcp
      env:
        - name: HELM_CACHE_HOME
          value: /helm-working-dir
        - name: HELM_CONFIG_HOME
          value: /helm-working-dir
        - name: HELM_DATA_HOME
          value: /helm-working-dir
      command: [ "/bin/sh", "-c" ]
      args:
        - apk --no-cache add curl;
          helm plugin install https://github.com/hayorov/helm-gcs.git;
          helm repo add my-gcs-repo gs://my-private-helm-gcs-repository;
          chmod -R 777 $HELM_DATA_HOME;
```

## Helm 版本

Argo CD 将假定 Helm 图表是 v3 版本（即使图表中的 apiVersion 字段是 Helm v2），除非在 Argo CD 应用程序中明确指定了 v2 版本（见下文）。

如果需要，可以通过设置`helm-version`标志（v2 或 v3）：

```bash
argocd app set helm-guestbook --helm-version v3
```

或被引用声明式语法：

```yaml
spec:
  source:
    helm:
      version: v3
```

## Helm `--pass-credentials` `-pass-credentials

Helm、[从 v3.6.1 开始](https://github.com/helm/helm/releases/tag/v3.6.1)该选项可防止发送版本库凭据，以下载从与版本库不同的域提供的图表。

如果需要，可以通过设置`helm-pass-credentials`标志：

```bash
argocd app set helm-guestbook --helm-pass-credentials
```

或被引用声明式语法：

```yaml
spec:
  source:
    helm:
      passCredentials: true
```

## helm `--skip-crds`

Helm 将自定义资源定义安装在`crds`如果它们不存在，默认情况下会将它们放在文件夹中。 请参见[CRD 最佳做法](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)了解详情。

如果需要，可以使用`helm-skip-crds`标志：

```bash
argocd app set helm-guestbook --helm-skip-crds
```

或被引用声明式语法：

```yaml
spec:
  source:
    helm:
      skipCrds: true
```