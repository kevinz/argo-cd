<!-- TRANSLATED by md-translate -->
# 集群生成器

在 Argo CD 中，受管理的集群[存储在 Argo CD 名称空间中的 Secrets](../../declarative-setup/#clusters) 中。 ApplicationSet 控制器使用这些相同的 Secrets 生成参数，以识别和定位可用的集群。

对于在 Argo CD 注册的每个集群，集群生成器会根据在集群秘密内发现的项目列表生成参数。

它会自动为每个集群的应用程序模板提供以下参数值：

* 姓名
* `nameNormalized` _("名称"，但规范化为只包含小写字母数字字符、'-'或'.')_
* 服务器
* `metadata.labels.<key>` _(秘密中的每个标签)_ `metadata.annotions.
* `metadata.annotations.<key>`_(为秘密中的每个注释)_

注意 如果集群名称包含对 Kubernetes 资源名称无效的字符（如下划线），请使用 `nameNormalized` 参数。 这样可以防止呈现名称为 `my_cluster-app1` 的无效 Kubernetes 资源，而将其转换为 `my-cluster-app1` 。

Argo CD 集群秘密]（.../.../声明式设置/#集群）内是描述集群的数据字段：

```yaml
kind: Secret
data:
  # Within Kubernetes these fields are actually encoded in Base64; they are decoded here for convenience.
  # (They are likewise decoded when passed as parameters by the Cluster generator)
  config: "{'tlsClientConfig':{'insecure':false}}"
  name: "in-cluster2"
  server: "https://kubernetes.default.svc"
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
# (...)
```

集群生成器将自动识别用 Argo CD 定义的集群，并提取集群数据作为参数：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters: {} # Automatically use all clusters defined within Argo CD
  template:
    metadata:
      name: '{{.name}}-guestbook' # 'name' field of the Secret
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{.server}}' # 'server' field of the secret
        namespace: guestbook
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/cluster)。_）

在此示例中，集群秘密的 `name` 和 `server` 字段被用来填充 `Application` 资源的 `name` 和 `server`（然后被用来引用同一集群）。

### 标签选择器

标签选择器可被用来将目标集群的范围缩小到仅与特定标签匹配的集群：

```yaml
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters:
      selector:
        matchLabels:
          staging: true
        # The cluster generator also supports matchExpressions.
        #matchExpressions:
        #  - key: staging
        #    operator: In
        #    values:
        #      - "true"
  template:
  # (...)
```

这将与 Argo CD 集群的秘密内容相匹配：

```yaml
kind: Secret
data:
  # (... fields as above ...)
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
    staging: "true"
# (...)
```

集群选择器还支持基于集合的要求，[几个核心 Kubernetes 资源](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements) 就引用了这种要求。

### 部署到本地集群

在 Argo CD 中，"本地集群 "是指安装了 Argo CD（和 ApplicationSet 控制器）的集群。 这是为了与 "远程集群 "区分开来，后者是指[声明式]（.../.../declarative-setup/#clusters）或通过[Argo CD CLI]（.../.../getting_started.md/#5-register-a-cluster-to-deploy-apps-to-optional）添加到 Argo CD 的集群。

对于每个与集群选择器相匹配的集群，集群生成器都会自动将本地集群和非本地集群作为目标。

如果您希望应用程序只针对远程集群（例如，您希望排除本地集群），则可使用带标签的集群选择器等：

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters:
      selector:
        matchLabels:
          argocd.argoproj.io/secret-type: cluster
        # The cluster generator also supports matchExpressions.
        #matchExpressions:
        #  - key: staging
        #    operator: In
        #    values:
        #      - "true"
```

该选择器不会匹配默认本地集群，因为默认本地集群没有 Secret（因此该 Secret 上没有 `argocd.argoproj.io/secret-type` 标签）。 任何在该标签上选择的集群选择器都会自动排除默认本地集群。

不过，如果您确实希望同时针对本地和非本地集群，同时也使用标签匹配，您可以在 Argo CD Web UI 中为本地集群创建一个秘密：

1.在 Argo CD Web UI 中，选择_Settings_（设置），然后选择_Clusters_（群集）。
2.选择本地集群，通常命名为 "in-cluster"。
3.单击 _Edit_ 按钮，将集群的 _NAME_ 更改为其他值，例如 `in-cluster-local`。其他任何值都可以。
4.其他字段保持不变。
5.单击_保存_。

这些步骤看似与直觉相反，但更改本地集群的某个默认值会导致 Argo CD Web UI 为该集群创建一个新的秘密。 在 Argo CD 名称空间中，您现在应该可以看到一个名为 "cluster-(集群后缀) "的秘密资源，其标签为 "argocd.argoproj.io/secret-type": "cluster"`。 您也可以通过声明式](./../declarative-setup/#clusters)或 CLI 使用 "argocd cluster add "(context name)" --in-cluster "创建本地[集群秘密]，而不是通过 Web UI。

### 通过 `values` 字段传递额外的键值对

您可以通过集群生成器的 "values "字段传递额外的任意字符串键值对。 通过 "values "字段添加的值以 "values.(field) "的形式添加。

在本例中，根据集群秘密上的匹配标签，传递了一个 `revision` 参数值：

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters:
      selector:
        matchLabels:
          type: 'staging'
      # A key-value map for arbitrary parameters
      values:
        revision: HEAD # staging clusters use HEAD branch
  - clusters:
      selector:
        matchLabels:
          type: 'production'
      values:
        # production uses a different revision value, for 'stable' branch
        revision: stable
  template:
    metadata:
      name: '{{.name}}-guestbook'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        # The cluster values field for each generator will be substituted here:
        targetRevision: '{{.values.revision}}'
        path: guestbook
      destination:
        server: '{{.server}}'
        namespace: guestbook
```

在本例中，"generators.集群 "字段中的 "Revisions "值以 "values.revision "的形式传入模板，其中包含 "HEAD "或 "stable"（基于哪个生成器生成了参数集）。

注意 "values. "前缀总是被引用到通过 "generators.clusters.values "字段提供的值中。 使用时请确保在 "template "中的参数名称中包含此前缀。

在 `values` 中，我们还可以对以下参数值进行插值（即与本页开头介绍的值相同）

* 姓名
* `nameNormalized` _("名称"，但规范化为只包含小写字母数字字符、'-'或'.')_
* 服务器
* `metadata.labels.<key>` _(秘密中的每个标签)_ `metadata.annotions.
* `metadata.annotations.<key>`_(为秘密中的每个注释)_

扩展自上面的例子，我们可以这样做：

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters:
      selector:
        matchLabels:
          type: 'staging'
      # A key-value map for arbitrary parameters
      values:
        # If `my-custom-annotation` is in your cluster secret, `revision` will be substituted with it.
        revision: '{{index .metadata.annotations "my-custom-annotation"}}' 
        clusterName: '{{.name}}'
  - clusters:
      selector:
        matchLabels:
          type: 'production'
      values:
        # production uses a different revision value, for 'stable' branch
        revision: stable
        clusterName: '{{.name}}'
  template:
    metadata:
      name: '{{.name}}-guestbook'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        # The cluster values field for each generator will be substituted here:
        targetRevision: '{{.values.revision}}'
        path: guestbook
      destination:
        # In this case this is equivalent to just using {{name}}
        server: '{{.values.clusterName}}'
        namespace: guestbook
```