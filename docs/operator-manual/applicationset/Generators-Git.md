<!-- TRANSLATED by md-translate -->
# Git 生成器

Git 生成器包含两个子类型：Git 目录生成器和 Git 文件生成器。

警告 Git 生成器通常用于方便（非管理员）开发人员创建应用程序。 如果 ApplicationSet 中的 `project` 字段是模板化的，开发人员可能会在权限过大的项目下创建应用程序。 对于带有模板化 `project` 字段的 ApplicationSet，[真相来源 _must_ 必须_由管理员控制](./Security.md#templated-project-field) - 在 git 生成器的情况下，PR 必须需要管理员批准。

## Git 生成器：目录

Git 目录生成器是 Git 生成器的两个子类型之一，它使用指定 Git 仓库的目录结构生成参数。

假设您有一个 Git 仓库，其目录结构如下：

```
├── argo-workflows
│   ├── kustomization.yaml
│   └── namespace-install.yaml
└── prometheus-operator
    ├── Chart.yaml
    ├── README.md
    ├── requirements.yaml
    └── values.yaml
```

该资源库包含两个目录，每个目录对应一个要部署的工作负载：

* Argo 工作流程控制器 kustomize YAML 文件
* 一个 Prometheus Operator Helm 图表

我们可以通过这个例子，部署这两种工作负载：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      directories:
      - path: applicationset/examples/git-generator-directory/cluster-addons/*
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/git-generator-directory)。_）

发电机参数为

* {{.path.path}}`：Git 仓库中与 `path` 通配符匹配的目录路径。
* `{{index .path.segments n}}`：Git 仓库中与`path`通配符匹配的目录路径，分成数组元素（`n` - 数组索引）
* {{.path.basename}}`：对于 Git 仓库中任何与`path`通配符匹配的目录路径，将提取最右边的路径名（例如，`/directory/directory2`将产生`directory2`）。
* {{.path.basenameNormalized}}`：该字段与 `path.basename`相同，但会用`-`替换不支持的字符（例如，`path`为`/directory/directory_2`，`path.basename`为`directory_2`，则此处会生成`directory-2`）。

**注意**：最右边的路径名总是变成"{{.path.basename}}"。 例如，对于"- path: /one/two/three/four"，"{{.path.basename}}"就是 "four"。

**注**：如果指定了`pathParamPrefix`选项，则上述所有与`path`相关的参数名将以指定值和点号分隔符作为前缀。 例如，如果`pathParamPrefix`为`myRepo`，则生成的参数名将为`.myRepo.path`，而不是`.path`。 在两个子生成器均为 Git 生成器的矩阵生成器中，有必要引用该选项（以避免合并子生成器的项目时发生冲突）。

每当在 Git 仓库中添加新的 helm chart/Kustomize YAML/Application/plain 子目录时，ApplicationSet 控制器就会检测到这一变化，并自动在新的 `Application` 资源中配置由此产生的清单。

与其他生成器一样，集群必须已在 Argo CD 中定义，才能为其生成应用程序。

### 排除目录

Git 目录生成器会自动排除以`.`开头的目录（如`.git`）。

Git 目录生成器还支持 "exclude"（排除）选项，以便将版本库中的目录排除在 ApplicationSet 控制器的扫描范围之外：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      directories:
      - path: applicationset/examples/git-generator-directory/excludes/cluster-addons/*
      - path: applicationset/examples/git-generator-directory/excludes/cluster-addons/exclude-helm-guestbook
        exclude: true
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/git-generator-directory/excludes)。_）

此示例从该 `ApplicationSet` 资源扫描的目录列表中排除了 `exclude- helm-guestbook` 目录。

注意 "排除规则的优先级高于包含规则"。

```
If a directory matches at least one `exclude` pattern, it will be excluded. Or, said another way, *exclude rules take precedence over include rules.*

As a corollary, which directories are included/excluded is not affected by the order of `path`s in the `directories` field list (because, as above, exclude rules always take precedence over include rules).
```

例如，这些目录

```
.
└── d
    ├── e
    ├── f
    └── g
```

假设您想包含 `/d/e`，但不包括 `/d/f` 和 `/d/g`：

```yaml
- path: /d/e
  exclude: false
- path: /d/*
  exclude: true
```

因为 `/d/*` 排除规则优先于 `/d/e` 包括规则。 当 Git 仓库中的 `/d/e` 路径被 ApplicationSet 控制器处理时，控制器会检测到至少有一条排除规则匹配，因此不应扫描该目录。

您需要做的是

```yaml
- path: /d/*
- path: /d/f
  exclude: true
- path: /d/g
  exclude: true
```

或者，更简短的方法（使用 [path.Match](https://golang.org/pkg/path/#Match) 语法）是：

```yaml
- path: /d/*
- path: /d/[fg]
  exclude: true
```

### Git 仓库根目录

可以通过提供 `'*'` 作为 `path` 来配置 Git 目录生成器，以便从 git 仓库的根目录进行部署。

要排除目录，只需输入不想部署的目录的名称/[path.Match](https://golang.org/pkg/path/#Match)。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/example/example-repo.git
      revision: HEAD
      directories:
      - path: '*'
      - path: donotdeploy
        exclude: true
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/example/example-repo.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'
```

### 通过 `values` 字段传递额外的键值对

您可以通过 git 目录生成器的 `values` 字段传递额外的任意字符串键值对。 通过 `values` 字段添加的值以 `values.(field)` 的形式添加。

在此示例中，传递了一个 `cluster` 参数值，它是从 `branch` 和 `path` 变量中插值出来的，然后被引用来确定目标名称空间。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/example/example-repo.git
      revision: HEAD
      directories:
      - path: '*'
      values:
        cluster: '{{.branch}}-{{.path.basename}}'
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: "my-project"
      source:
        repoURL: https://github.com/example/example-repo.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.values.cluster}}'
```

注意 `values.` 前缀总是被引用到通过 `generators.git.values` 字段提供的值中。 使用时请确保在 `template` 中的参数名称中包含此前缀。

在 `values` 中，我们还可以插值上述由 git 目录生成器设置的所有字段。

## Git 生成器：文件

Git 文件生成器是 Git 生成器的第二个子类型。 Git 文件生成器使用在指定版本库中找到的 JSON/YAML 文件内容生成参数。

假设您有一个 Git 仓库，其目录结构如下：

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

这些目录是

* guestbook "包含一个简单留言簿应用程序的 Kubernetes 资源
* `cluster-config` 包含描述各个工程集群的 JSON/YAML 文件：一个用于`dev`，一个用于`prod`。
* `git-generator-files.yaml`是将`guestbook`部署到指定集群的示例`ApplicationSet`资源。

`config.json` 文件包含描述集群的信息（以及额外的示例数据）：

```json
{
  "aws_account": "123456",
  "asset_id": "11223344",
  "cluster": {
    "owner": "cluster-admin@company.com",
    "name": "engineering-dev",
    "address": "https://1.2.3.4"
  }
}
```

Git 生成器会自动发现包含对 `config.json` 文件改动的 Git 提交，并解析这些文件的内容，将其转换为模板参数。 以下是为上述 JSON 生成的参数：

```text
aws_account: 123456
asset_id: 11223344
cluster.owner: cluster-admin@company.com
cluster.name: engineering-dev
cluster.address: https://1.2.3.4
```

所有发现的 `config.json` 文件的生成参数将被替换到 ApplicationSet 模板中：

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
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      files:
      - path: "applicationset/examples/git-generator-files-discovery/cluster-config/**/config.json"
  template:
    metadata:
      name: '{{.cluster.name}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: "applicationset/examples/git-generator-files-discovery/apps/guestbook"
      destination:
        server: '{{.cluster.address}}'
        namespace: guestbook
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/git-generator-files-discovery)。_）

在 `cluster-config` 目录下找到的任何 `config.json` 文件都将根据指定的 `path` 通配符模式进行参数化。 每个文件中的 JSON 字段都会被扁平化为键/值对，本 ApplicationSet 示例在模板中使用了 `cluster.address` 和 `cluster.name` 参数。

与其他生成器一样，集群必须已在 Argo CD 中定义，才能为其生成应用程序。

除了配置文件中的扁平化键/值对，还提供了以下生成器参数：

* {{.path.path}}`：Git 仓库中包含匹配配置文件的目录的路径。例如： `/clusters/clusterA`，如果配置文件是 `/clusters/clusterA/config.json
* {{索引 .路径 n}}`：Git 仓库中匹配配置文件的路径，分为数组元素（`n` - 数组索引）。例如：`index .path 0: 集群`, `index .path 1: 集群A`。
* `{{.path.basename}}`：包含配置文件的目录的路径基名（如上例中的`clusterA`）。
* {{.path.basenameNormalized}}`：该字段与 `.path.basename`相同，但会用`-`替换不支持的字符（例如，`path`为`/directory/directory_2`，`.path.basename`为`directory_2`，则此处会产生`directory-2`）。
* {{.path.filename}}`：匹配的文件名。例如，上例中的 `config.json`。
* {{.path.filenameNormalized}}`：匹配的文件名，其中不支持的字符用 `-` 替换。

**注意**：最右边的_目录名总是变成`{{.path.basename}}`。 例如，从`- path: /one/two/three/four/config.json`开始，`{{.path.basename}}`将变成`four`。 文件名总是可以被`{{.path.filename}}`引用。

**注**：如果指定了`pathParamPrefix`选项，则上述所有与`path`相关的参数名将以指定值和点号分隔符作为前缀。 例如，如果`pathParamPrefix`为`myRepo`，则生成的参数名将为`myRepo.path`，而不是`path`。 在两个子生成器均为 Git 生成器的矩阵生成器中，有必要引用该选项（以避免合并子生成器的项目时发生冲突）。

**注**：Git 文件生成器的默认行为非常贪婪，请参阅 [Git File Generator Globbing](./Generators-Git-File-Globbing.md)，了解更多信息。

### 通过 `values` 字段传递额外的键值对

您可以通过 git 文件生成器的 `values` 字段传递额外的任意字符串键值对。 通过 `values` 字段添加的值以 `values.(field)` 的形式添加。

在此示例中，传递了一个 `base_dir` 参数值，它是从 `path` 片段中插值出来的，然后用来确定源路径。

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
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      files:
      - path: "applicationset/examples/git-generator-files-discovery/cluster-config/**/config.json"
      values:
        base_dir: "{{index .path 0}}/{{index .path 1}}/{{index .path 2}}"
  template:
    metadata:
      name: '{{.cluster.name}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: "{{.values.base_dir}}/apps/guestbook"
      destination:
        server: '{{.cluster.address}}'
        namespace: guestbook
```

注意 `values.` 前缀总是被引用到通过 `generators.git.values` 字段提供的值中。 使用时请确保在 `template` 中的参数名称中包含此前缀。

在 `values` 中，我们还可以插值上述由 git 文件生成器设置的所有字段。

## Webhook 配置

使用 Git 生成器时，ApplicationSet 会每隔三分钟轮询一次 Git 仓库以检测更改。 为了消除轮询带来的延迟，可以配置 ApplicationSet webhook 服务器以接收 webhook 事件。 ApplicationSet 支持来自 GitHub 和 GitLab 的 Git webhook 通知。 下面将说明如何为 GitHub 配置 Git webhook，但相同的过程应适用于其他 Provider。

注意 ApplicationSet 控制器 webhook 使用的 webhook 与[此处](../webhook.md)定义的 API 服务器不同。 ApplicationSet 将 webhook 服务器作为 ClusterIP 类型的服务公开。 需要创建一个 ApplicationSet 专用 ingress 资源，以便将此服务公开给 webhook 源。

#### 1.在 Git Providers 中创建 webhook

在 Git Provider 中，导航至可配置 webhook 的设置页面。 在 Git Provider 中配置的有效载荷 URL 应使用 ApplicationSet 实例的 `/api/webhook` 端点（如 `https://applicationset.example.com/api/webhook`）。如果希望使用共享秘密，请在秘密中输入任意值。该值将在下一步配置 webhook 时被引用。

![添加 Webhook](.../.../assets/applicationset/webhook-config.png "Add Webhook")

注意 在 GitHub 中创建 webhook 时，需要将 "内容类型 "设置为 "application/json"。 用于处理钩子的库不支持默认值 "application/x-www-form-urlencoded"。

#### 2. 使用网络钩子秘密配置 ApplicationSet（可选）

配置 webhook 共享秘密是可选的，因为即使有未经验证的 webhook 事件，ApplicationSet 仍会刷新 Git 生成器生成的应用程序。 这样做是安全的，因为 webhook 有效载荷的内容被认为是不可信任的，只会导致应用程序的刷新（这个过程已经每隔三分钟发生一次）。 如果 ApplicationSet 可被公开访问，则建议配置 webhook 秘密，以防止 DDoS 攻击。

在 `argocd-secret` Kubernetes secret 中，包含步骤 1 中配置的 Git Provider 的 webhook secret。

编辑 Argo CD Kubernetes secret：

```bash
kubectl edit secret argocd-secret -n argocd
```

提示：为了方便输入秘密，Kubernetes 支持在 `stringData` 字段中输入秘密，这样就省去了对值进行 base64 编码并复制到 `data` 字段的麻烦。 只需将步骤 1 中创建的共享 webhook 秘密复制到 `stringData` 字段下相应的 GitHub/GitLab/BitBucket 密钥中即可：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...

stringData:
  # github webhook secret
  webhook.github.secret: shhhh! it's a github secret

  # gitlab webhook secret
  webhook.gitlab.secret: shhhh! it's a gitlab secret
```

保存后，请重新启动 ApplicationSet pod 使更改生效。