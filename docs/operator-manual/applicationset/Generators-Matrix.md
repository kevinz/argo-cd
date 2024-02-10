<!-- TRANSLATED by md-translate -->
# 矩阵生成器

矩阵生成器将两个子生成器生成的参数组合在一起，迭代每个生成器生成的参数组合。

通过组合两个发生器的参数，产生每一种可能的组合，这样就可以获得两个发生器的固有特性。 例如，许多可能的用例中的一小部分包括：

* _SCM Provider Generator + Cluster Generator_：扫描 GitHub 组织的资源库中的应用资源，并将这些资源定向到所有可用的集群。
* _Git 文件生成器 + 列表生成器_：通过配置文件提供要部署的应用程序列表，其中包含可选配置选项，并将其部署到固定的集群列表中。
* _Git目录生成器+集群决定资源生成器_：定位 Git 仓库文件夹中包含的应用资源，并将其部署到通过外部自定义资源 Providers 的集群列表中。
* 等等...

可以引用任意一组生成器，并像往常一样将这些生成器的组合值插入 "模板 "参数。

**注**：如果两个子生成器都是 Git 生成器，其中一个或两个生成器必须使用 `pathParamPrefix` 选项，以避免合并子生成器的项目时发生冲突。

## 示例：Git 目录生成器 + 集群生成器

举个例子，设想我们有两个集群：

* 暂存 "集群（位于 `https://1.2.3.4`)
* 生产 "集群（位于 `https://2.4.6.8`)

我们的应用程序 YAML 定义在 Git 仓库中：

* Argo 工作流控制器（examples/git-generator-directory/cluster-addons/argo-workflows）
* Prometheus 操作员 (/examples/git-generator-directory/cluster-addons/prometheus-operator)

我们的目标是将这两个应用程序部署到这两个集群上，而且更广泛地说，在未来自动部署 Git 仓库中的新应用程序，以及 Argo CD 中定义的新集群。

为此，我们将引用矩阵生成器，并将 Git 和集群作为子生成器：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-git
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    # matrix 'parent' generator
    - matrix:
        generators:
          # git generator, 'child' #1
          - git:
              repoURL: https://github.com/argoproj/argo-cd.git
              revision: HEAD
              directories:
                - path: applicationset/examples/matrix/cluster-addons/*
          # cluster generator, 'child' #2
          - clusters:
              selector:
                matchLabels:
                  argocd.argoproj.io/secret-type: cluster
  template:
    metadata:
      name: '{{.path.basename}}-{{.name}}'
    spec:
      project: '{{index .metadata.labels "environment"}}'
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: '{{.server}}'
        namespace: '{{.path.basename}}'
```

首先，Git 目录生成器会扫描 Git 仓库，发现指定路径下的目录。 它还会发现 Argo-workflows 和 prometheus-operator 应用程序，并生成两组相应的参数：

```yaml
- path: /examples/git-generator-directory/cluster-addons/argo-workflows
  path.basename: argo-workflows

- path: /examples/git-generator-directory/cluster-addons/prometheus-operator
  path.basename: prometheus-operator
```

接下来，集群生成器会扫描[Argo CD 中定义的集群集](Generators-Cluster.md)，找到暂存集群和生产集群的秘密，并生成两组相应的参数：

```yaml
- name: staging
  server: https://1.2.3.4

- name: production
  server: https://2.4.6.8
```

最后，矩阵发生器将合并两组输出，并产生结果：

```yaml
- name: staging
  server: https://1.2.3.4
  path: /examples/git-generator-directory/cluster-addons/argo-workflows
  path.basename: argo-workflows

- name: staging
  server: https://1.2.3.4
  path: /examples/git-generator-directory/cluster-addons/prometheus-operator
  path.basename: prometheus-operator

- name: production
  server: https://2.4.6.8
  path: /examples/git-generator-directory/cluster-addons/argo-workflows
  path.basename: argo-workflows

- name: production
  server: https://2.4.6.8
  path: /examples/git-generator-directory/cluster-addons/prometheus-operator
  path.basename: prometheus-operator
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/matrix)。_）

## 在另一个子生成器中被引用一个子生成器中的参数

矩阵生成器允许在另一个子生成器中使用一个子生成器生成的参数。 下面是一个将 git-files 生成器与集群生成器结合使用的示例。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-git
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    # matrix 'parent' generator
    - matrix:
        generators:
          # git generator, 'child' #1
          - git:
              repoURL: https://github.com/argoproj/applicationset.git
              revision: HEAD
              files:
                - path: "examples/git-generator-files-discovery/cluster-config/**/config.json"
          # cluster generator, 'child' #2
          - clusters:
              selector:
                matchLabels:
                  argocd.argoproj.io/secret-type: cluster
                  kubernetes.io/environment: '{{.path.basename}}'
  template:
    metadata:
      name: '{{.name}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/applicationset.git
        targetRevision: HEAD
        path: "examples/git-generator-files-discovery/apps/guestbook"
      destination:
        server: '{{.server}}'
        namespace: guestbook
```

以下是 git-files 生成器被引用的 git 仓库的相应文件夹结构：

```
├── apps
│   └── guestbook
│       ├── guestbook-ui-deployment.yaml
│       ├── guestbook-ui-svc.yaml
│       └── kustomization.yaml
├── cluster-config
│   └── engineering
│       ├── dev
│       │   └── config.json
│       └── prod
│           └── config.json
└── git-generator-files.yaml
```

在上例中，git-files 生成器生成的"{{.path.basename}}"参数将解析为 "dev "和 "prod"。在第二个子生成器中，标签为 "kubernetes.io/environment: {{.path.basename}}"的标签选择器将解析为第一个子生成器的参数值（"kubernetes.io/environment: prod "和 "kubernetes.io/environment: dev"）。

因此，在上面的示例中，标签为 `kubernetes.io/environment: prod` 的集群将只应用针对 prod 的配置（即 `prod/config.json`），而标签为 `kubernetes.io/environment: dev` 的集群将只应用针对 dev 的配置（即 `dev/config.json`)

## 在另一个子生成器中覆盖一个子生成器的参数

矩阵生成器允许在多个子生成器中定义具有相同名称的参数。 例如，在一个生成器中为所有阶段定义默认值，而在另一个生成器中用特定阶段的值覆盖它们，这非常有用。 下面的示例使用矩阵生成器生成了一个基于 Helm 的应用程序，其中有两个 git 生成器：第一个提供特定阶段的值（每个阶段一个目录），第二个为所有阶段提供全局值。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: parameter-override-example
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/example/values.git
              revision: HEAD
              files:
                - path: "**/stage.values.yaml"
          - git:
               repoURL: https://github.com/example/values.git
               revision: HEAD
               files:
                  - path: "global.values.yaml"
  goTemplate: true
  template:
    metadata:
      name: example
    spec:
      project: default
      source:
        repoURL: https://github.com/example/example-app.git
        targetRevision: HEAD
        path: .
        helm:
          values: |
            {{ `{{ . | mustToPrettyJson }}` }}
      destination:
        server: in-cluster
        namespace: default
```

鉴于示例/ Values 资源库的结构/内容如下：

```
├── test
│   └── stage.values.yaml
│         stageName: test
│         cpuRequest: 100m
│         debugEnabled: true
├── staging
│   └── stage.values.yaml
│         stageName: staging
├── production
│   └── stage.values.yaml
│         stageName: production
│         memoryLimit: 512Mi
│         debugEnabled: false
└── global.values.yaml
      cpuRequest: 200m
      memoryLimit: 256Mi
      debugEnabled: true
```

上述矩阵生成器会产生以下结果：

```yaml
- stageName: test
  cpuRequest: 100m
  memoryLimit: 256Mi
  debugEnabled: true

- stageName: staging
  cpuRequest: 200m
  memoryLimit: 256Mi
  debugEnabled: true

- stageName: production
  cpuRequest: 200m
  memoryLimit: 512Mi
  debugEnabled: false
```

## 示例：两个使用 `pathParamPrefix` 的 Git 生成器

如果矩阵生成器的子生成器产生的结果包含不同值的相同键，则生成器会失败。 这给两个子生成器都是 Git 生成器的矩阵生成器带来了问题，因为它们会在输出中自动填充与 "路径 "相关的参数。 为避免这个问题，请在一个或两个子生成器上指定 "pathParamPrefix"，以避免输出中出现冲突的参数键。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: two-gits-with-path-param-prefix
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          # git file generator referencing files containing details about each
          # app to be deployed (e.g., `appName`).
          - git:
              repoURL: https://github.com/some-org/some-repo.git
              revision: HEAD
              files:
                - path: "apps/*.json"
              pathParamPrefix: app
          # git file generator referencing files containing details about
          # locations to which each app should deploy (e.g., `region` and
          # `clusterName`).
          - git:
              repoURL: https://github.com/some-org/some-repo.git
              revision: HEAD
              files:
                - path: "targets/{{.appName}}/*.json"
              pathParamPrefix: target
  template: {} # ...
```

然后，给出以下文件结构/内容：

```
├── apps
│   ├── app-one.json
│   │   { "appName": "app-one" }
│   └── app-two.json
│       { "appName": "app-two" }
└── targets
    ├── app-one
    │   ├── east-cluster-one.json
    │   │   { "region": "east", "clusterName": "cluster-one" }
    │   └── east-cluster-two.json
    │       { "region": "east", "clusterName": "cluster-two" }
    └── app-two
        ├── east-cluster-one.json
        │   { "region": "east", "clusterName": "cluster-one" }
        └── west-cluster-three.json
            { "region": "west", "clusterName": "cluster-three" }
```

......上述矩阵生成器会产生以下结果：

```yaml
- appName: app-one
  app.path: /apps
  app.path.filename: app-one.json
  # plus additional path-related parameters from the first child generator, all
  # prefixed with "app".
  region: east
  clusterName: cluster-one
  target.path: /targets/app-one
  target.path.filename: east-cluster-one.json
  # plus additional path-related parameters from the second child generator, all
  # prefixed with "target".

- appName: app-one
  app.path: /apps
  app.path.filename: app-one.json
  region: east
  clusterName: cluster-two
  target.path: /targets/app-one
  target.path.filename: east-cluster-two.json

- appName: app-two
  app.path: /apps
  app.path.filename: app-two.json
  region: east
  clusterName: cluster-one
  target.path: /targets/app-two
  target.path.filename: east-cluster-one.json

- appName: app-two
  app.path: /apps
  app.path.filename: app-two.json
  region: west
  clusterName: cluster-three
  target.path: /targets/app-two
  target.path.filename: west-cluster-three.json
```

## 限制

1.矩阵生成器目前只支持合并两个子生成器的 Output（例如不支持生成 3 个或更多的组合）。
2.每个数组条目只能指定一个生成器，例如，这种情况是无效的：
    ```
    - matrix：
         generators：
         - list：# (...)
           git：# (...)
    ```- 虽然 Kubernetes API 验证会接受这种方法，但控制器在生成时会报错。每个生成器都应在单独的数组元素中指定，如上面的示例。
3.矩阵生成器目前不支持在子生成器上指定 [`模板`覆盖](Template.md#generator-templates)，例如，该`模板`将不会被处理：
    ```
    - 矩阵：
         generators：
           - list：
               elements：
                 - # (...)
               模板：{ }# 未处理
    ```
4.组合型生成器（矩阵或合并）只能嵌套一次。例如，这样做是不行的：
    ```
    - matrix：
         generators：
           - matrix：
               generators：
                 - matrix：  # 第三层无效。
                     生成器：
                       - list：
                           elements：
                             - # (...)
    ```
5.当一个子生成器的参数被引用到另一个子生成器时，_消耗_参数的子生成器必须在_产生_参数的子生成器之后。

例如，下面的示例将是无效的（集群生成器必须在 git-files 生成器之后）：

```
- matrix:
        generators:
          # cluster generator, 'child' #1
          - clusters:
              selector:
                matchLabels:
                  argocd.argoproj.io/secret-type: cluster
                  kubernetes.io/environment: '{{.path.basename}}' # {{.path.basename}} is produced by git-files generator
          # git generator, 'child' #2
          - git:
              repoURL: https://github.com/argoproj/applicationset.git
              revision: HEAD
              files:
                - path: "examples/git-generator-files-discovery/cluster-config/**/config.json"
```

1.两个子生成器不能同时使用对方的参数。在下面的示例中，集群生成器消耗的是 git-files 生成器生成的"{{.path.basename}}"参数，而 git-files 生成器消耗的是集群生成器生成的"{{.name}}"参数。这将导致循环依赖，是无效的。
    ```
    - 矩阵：
         生成器：
           # 集群生成器, 'child' #1
           - 集群：
               selector：
                 matchLabels：
                   argocd.argoproj.io/secret-type: cluster
                   kubernetes.io/environment：'{{.path.basename}}'。# {{.path.basename}} 由 git-files 生成器生成
           # git 生成器, 'child' #2
           - git：
               repoURL: https://github.com/argoproj/applicationset.git
               修订：HEAD
               files：
                 - path："examples/git-generator-files-discovery/cluster-config/engineering/{{.name}}**/config.json" # {{.name}} 由集群生成器生成
    ```
2.使用嵌套在另一个矩阵或合并生成器中的矩阵生成器时，只有通过 `spec.applyNestedSelectors` 启用后，才会引用该嵌套生成器的生成器的 [Post Selectors](Generators-Post-Selector.md)。即使您的 "后置选择器 "不在嵌套矩阵或合并生成器中，而是嵌套矩阵或合并生成器的同级生成器，也可能需要启用此功能。
    ```
    - 矩阵
         generators：
           - matrix：
               generators：
                 - 列表
                     elements：
                       - # (...)
                   选择器{ }# 仅在 applyNestedSelectors 为 true 时应用
    ```
