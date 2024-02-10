<!-- TRANSLATED by md-translate -->
# 配置管理插件

Argo CD 的 "原生 "配置管理工具有 Helm、Jsonnet 和 Kustomize。 如果您想引用其他配置管理工具，或者 Argo CD 的原生工具支持不包括您需要的功能，您可能需要使用配置管理插件（CMP）。

Argo CD 的 "repo 服务器 "组件负责根据 Helm、OCI 或 git 仓库中的源文件构建 Kubernetes 清单。 当配置管理插件配置正确时，repo 服务器可将构建清单的任务委托给插件。

以下各节将介绍如何创建、安装和被引用插件。请查阅 [示例插件](https://github.com/argoproj/argo-cd/tree/master/examples/plugins) 获取更多指导。

!!! 警告 插件在 Argo CD 系统中被赋予了一定程度的信任，因此安全地实施插件非常重要。 Argo CD 管理员只应安装来自可信来源的插件，而且应审核插件以权衡其特定风险和收益。

## 安装配置管理插件

### Sidecar 插件

操作员可以通过 repo-server 的侧挂配置插件工具。 配置新插件需要进行以下更改：

#### 编写插件配置文件

插件将通过位于插件容器内的 ConfigManagementPlugin 配置清单进行配置。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  # The name of the plugin must be unique within a given Argo CD instance.
  name: my-plugin
spec:
  # The version of your plugin. Optional. If specified, the Application's spec.source.plugin.name field
  # must be <plugin name>-<plugin version>.
  version: v1.0
  # The init command runs in the Application source directory at the beginning of each manifest generation. The init
  # command can output anything. A non-zero status code will fail manifest generation.
  init:
    # Init always happens immediately before generate, but its output is not treated as manifests.
    # This is a good place to, for example, download chart dependencies.
    command: [sh]
    args: [-c, 'echo "Initializing..."']
  # The generate command runs in the Application source directory each time manifests are generated. Standard output
  # must be ONLY valid Kubernetes Objects in either YAML or JSON. A non-zero exit code will fail manifest generation.
  # To write log messages from the command, write them to stderr, it will always be displayed.
  # Error output will be sent to the UI, so avoid printing sensitive information (such as secrets).
  generate:
    command: [sh, -c]
    args:
      - |
        echo "{\"kind\": \"ConfigMap\", \"apiVersion\": \"v1\", \"metadata\": { \"name\": \"$ARGOCD_APP_NAME\", \"namespace\": \"$ARGOCD_APP_NAMESPACE\", \"annotations\": {\"Foo\": \"$ARGOCD_ENV_FOO\", \"KubeVersion\": \"$KUBE_VERSION\", \"KubeApiVersion\": \"$KUBE_API_VERSIONS\",\"Bar\": \"baz\"}}}"
  # The discovery config is applied to a repository. If every configured discovery tool matches, then the plugin may be
  # used to generate manifests for Applications using the repository. If the discovery config is omitted then the plugin 
  # will not match any application but can still be invoked explicitly by specifying the plugin name in the app spec. 
  # Only one of fileName, find.glob, or find.command should be specified. If multiple are specified then only the 
  # first (in that order) is evaluated.
  discover:
    # fileName is a glob pattern (https://pkg.go.dev/path/filepath#Glob) that is applied to the Application's source 
    # directory. If there is a match, this plugin may be used for the Application.
    fileName: "./subdir/s*.yaml"
    find:
      # This does the same thing as fileName, but it supports double-start (nested directory) glob patterns.
      glob: "**/Chart.yaml"
      # The find command runs in the repository's root directory. To match, it must exit with status code 0 _and_ 
      # produce non-empty output to standard out.
      command: [sh, -c, find . -name env.yaml]
  # The parameters config describes what parameters the UI should display for an Application. It is up to the user to
  # actually set parameters in the Application manifest (in spec.source.plugin.parameters). The announcements _only_
  # inform the "Parameters" tab in the App Details page of the UI.
  parameters:
    # Static parameter announcements are sent to the UI for _all_ Applications handled by this plugin.
    # Think of the `string`, `array`, and `map` values set here as "defaults". It is up to the plugin author to make 
    # sure that these default values actually reflect the plugin's behavior if the user doesn't explicitly set different
    # values for those parameters.
    static:
      - name: string-param
        title: Description of the string param
        tooltip: Tooltip shown when the user hovers the
        # If this field is set, the UI will indicate to the user that they must set the value.
        required: false
        # itemType tells the UI how to present the parameter's value (or, for arrays and maps, values). Default is
        # "string". Examples of other types which may be supported in the future are "boolean" or "number".
        # Even if the itemType is not "string", the parameter value from the Application spec will be sent to the plugin
        # as a string. It's up to the plugin to do the appropriate conversion.
        itemType: ""
        # collectionType describes what type of value this parameter accepts (string, array, or map) and allows the UI
        # to present a form to match that type. Default is "string". This field must be present for non-string types.
        # It will not be inferred from the presence of an `array` or `map` field.
        collectionType: ""
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        string: default-string-value
      # All the fields above besides "string" apply to both the array and map type parameter announcements.
      - name: array-param
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        array: [default, items]
        collectionType: array
      - name: map-param
        # This field communicates the parameter's default value to the UI. Setting this field is optional.
        map:
          some: value
        collectionType: map
    # Dynamic parameter announcements are announcements specific to an Application handled by this plugin. For example,
    # the values for a Helm chart's values.yaml file could be sent as parameter announcements.
    dynamic:
      # The command is run in an Application's source directory. Standard output must be JSON matching the schema of the
      # static parameter announcements list.
      command: [echo, '[{"name": "example-param", "string": "default-string-value"}]']

  # If set to `true` then the plugin receives repository files with original file mode. Dangerous since the repository
  # might have executable files. Set to true only if you trust the CMP plugin authors.
  preserveFileMode: false
```

注意 虽然 ConfigManagementPlugin 看起来像一个 Kubernetes 对象，但它实际上并不是一个自定义资源。 它只是遵循了 kubernetes 风格的规范约定。

生成 "命令必须将有效的 Kubernetes YAML 或 JSON 对象流打印到 stdout。 启动 "和 "生成 "命令都在应用程序源代码目录内执行。

discover.fileName "被引用为 [glob](https://pkg.go.dev/path/filepath#Glob) 模式，用于确定插件是否支持应用程序存储库。

```yaml
discover:
    find:
      command: [sh, -c, find . -name env.yaml]
```

如果未提供 `discover.fileName`，则会执行 `discover.find.command`，以确定插件是否支持应用程序资源库。 如果支持应用程序源类型，则 `find` 命令应返回非错误退出代码，并产生输出到 stdout。

#### 将插件配置文件放在侧卡中

Argo CD 希望插件配置文件位于 sidecar 中的 `/home/argocd/cmp-server/config/plugin.yaml`。

如果侧边栏被引用了自定义镜像，则可以直接将文件添加到该镜像中。

```dockerfile
WORKDIR /home/argocd/cmp-server/config/
COPY plugin.yaml ./
```

如果您使用库存镜像作为边卡，或更愿意在 ConfigMap 中维护插件配置，只需将插件配置文件嵌套到`plugin.yaml`键下的 ConfigMap 中，并将 ConfigMap 挂载到边卡中（见下一节）。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-plugin-config
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: my-plugin
    spec:
      version: v1.0
      init:
        command: [sh, -c, 'echo "Initializing..."']
      generate:
        command: [sh, -c, 'echo "{\"kind\": \"ConfigMap\", \"apiVersion\": \"v1\", \"metadata\": { \"name\": \"$ARGOCD_APP_NAME\", \"namespace\": \"$ARGOCD_APP_NAMESPACE\", \"annotations\": {\"Foo\": \"$ARGOCD_ENV_FOO\", \"KubeVersion\": \"$KUBE_VERSION\", \"KubeApiVersion\": \"$KUBE_API_VERSIONS\",\"Bar\": \"baz\"}}}"']
      discover:
        fileName: "./subdir/s*.yaml"
```

#### 注册插件侧卡

要安装插件，请打上 argocd-repo-server 补丁，将插件容器作为侧卡运行，argocd-cmp-server 作为其入口点。 您可以使用现成或定制的插件镜像作为侧卡镜像，例如

```yaml
containers:
- name: my-plugin
  command: [/var/run/argocd/argocd-cmp-server] # Entrypoint should be Argo CD lightweight CMP server i.e. argocd-cmp-server
  image: busybox # This can be off-the-shelf or custom-built image
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
  volumeMounts:
    - mountPath: /var/run/argocd
      name: var-files
    - mountPath: /home/argocd/cmp-server/plugins
      name: plugins
    # Remove this volumeMount if you've chosen to bake the config file into the sidecar image.
    - mountPath: /home/argocd/cmp-server/config/plugin.yaml
      subPath: plugin.yaml
      name: my-plugin-config
    # Starting with v2.4, do NOT mount the same tmp volume as the repo-server container. The filesystem separation helps 
    # mitigate path traversal attacks.
    - mountPath: /tmp
      name: cmp-tmp
volumes:
- configMap:
    name: my-plugin-config
  name: my-plugin-config
- emptyDir: {}
  name: cmp-tmp
```

确保使用 `/var/run/argocd/argocd-cmp-server` 作为入口点。 `argocd-cmp-server` 是一个轻量级 GRPC 服务，允许 Argo CD 与插件进行交互。 2. 确保 sidecar 容器以用户 999 的身份运行。 3. 确保插件配置文件位于 `/home/argocd/cmp-server/config/plugin.yml`。 可以通过 configmaps 进行卷映射，也可以将其嵌入镜像中。

### 在插件中使用环境变量

插件命令可以访问

1.系统环境变量
2.[标准构建环境变量](.../user-guide/build-environment.md)
3.应用程序规范中的变量（对系统变量和构建变量的引用将在变量值中插值）：
    ```
    apiVersion: argoproj.io/v1alpha1
     种类应用程序
     spec：
       source：
         plugin：
           env：
             - name: FOO
               values: bar
             - name：REV
               value: test-$ARGOCD_APP_REVISION
    在执行 `init.command`、`generate.command` 和 `discover.find.command` 命令之前，Argo CD 会在所有用户提供的环境变量（如上文 #3 所述）前加上 `ARGOCD_ENVISION` 前缀。 
     ARGOCD_ENV_` 前缀。这可以防止用户直接设置 
     潜在敏感的环境变量。
4.应用程序规范中的参数：
    ```
    apiVersion: argoproj.io/v1alpha1
     种类应用程序
     spec：
      source：
        plugin：
          parameters：
            - name: Values-files
              array：[Values-dev.yaml] [值-dev.yaml]。
            - name: helm-parameters
              map：
                image.tag: v1.2.3
    参数以 JSON 格式存在于 `ARGOCD_APP_PARAMETERS` 环境变量中。上面的示例将
     生成以下 JSON：````
    {"name"："values-files"，"array"：["values-dev.yaml"]}, {"name"："helm-parameters", "map"：{"镜像.标签"："v1.2.3"}}]
    注意
         在 `ARGOCD_APP_PARAMETERS` 中，参数公告（即使它们指定了默认值）不会被发送到插件。
         只有在应用程序规范中明确设置的参数才会发送给插件。插件可自行应用
         同样的参数也可以作为单独的环境变量使用。环境变量的名称
     环境变量的名称遵循以下约定
    - 名称：some-string-param
          string: some-string-values
        # PARAM_SOME_STRING_PARAM=some-string-values
    
        - 名称： 某数组参数
          Values：[item1, item2］
        # PARAM_SOME_ARRAY_PARAM_0=item1
        # PARAM_SOME_ARRAY_PARAM_1=item2
    
        - 名称： some-map-param
          map：
            image.tag: v1.2.3
        # PARAM_SOME_MAP_PARAM_IMAGE_TAG=v1.2.3
    ```

作为 Argo CD 配置清单生成系统的一部分，配置管理插件受到一定程度的信任。 请务必在插件中转义用户输入，以防止恶意输入导致不必要的行为。

## 在应用程序中使用配置管理插件

您可以在 "插件 "部分留空 "名称 "字段，以便插件根据其发现规则自动与应用程序匹配。 如果您提及名称，请确保在 "配置管理插件 "规范中提及版本的情况下，名称为 "<metadata.name>-<spec.version>"，否则仅为 "<metadata.name>"。如果明确指定了名称，只有在其发现模式/命令与 Provider 提供的应用程序 repo 匹配时，才会引用该特定插件。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
    plugin:
      env:
        - name: FOO
          value: bar
```

如果不需要设置任何环境变量，可以设置一个空的插件部分。

```yaml
plugin: {}
```

如果 CMP 命令运行时间过长，命令将被杀死，用户界面也会显示错误。 CMP 服务器会尊重`argocd-cm`中`server.repo.server.timeout.seconds`和`controller.repo.server.timeout.seconds`项所设置的超时时间。 在默认值 60s 的基础上增加它们的值。

```
Each CMP command will also independently timeout on the `ARGOCD_EXEC_TIMEOUT` set for the CMP sidecar. The default
is 90s. So if you increase the repo server timeout greater than 90s, be sure to set `ARGOCD_EXEC_TIMEOUT` on the
sidecar.
```

!!! note
    Each Application can only have one config management plugin configured at a time. If you're converting an existing
    plugin configured through the `argocd-cm` ConfigMap to a sidecar, make sure to update the plugin name to either `<metadata.name>-<spec.version>` 
    if version was mentioned in the `ConfigManagementPlugin` spec or else just use `<metadata.name>`. You can also remove the name altogether 
    and let the automatic discovery to identify the plugin.
!!! note
    If a CMP renders blank manfiests, and `prune` is set to `true`, Argo CD will automatically remove resources. CMP plugin authors should ensure errors are part of the exit code. Commonly something like `kustomize build . | cat` won't pass errors because of the pipe. Consider setting `set -o pipefail` so anything piped will pass errors on failure.

## 调试 CMP

如果您正在积极开发安装了挎斗的 CMP，请注意以下几点：

1.如果从 configmaps 挂载 plugin.yaml，则必须重启 repo-server Pod，这样插件才能接收更改。
2.如果将 plugin.yaml 添加到镜像中，则必须在 repo-server Pod 上构建、推送并强制重新拉取该镜像，这样插件才能接收更改。如果您使用的是 `:latest`，Pod 将始终提取新镜像。如果您使用的是不同的静态标签，请设置 `imagePullPolicy：始终"。
3.CMP 错误由 Redis 中的版本服务器缓存。重启版本服务器 Pod 不会清除缓存。在积极开发 CMP 时，请始终进行 "硬刷新"，以便获得最新输出。
4.通过查看 Pod 验证您的 sidecar 是否已正常启动，并查看两个容器是否正在运行`kubectl get pod -l app.kubernetes.io/component=repo-server -n argocd`
5.将日志信息写入 stderr，并在 sidecar 中设置 `--loglevel=info` 标志。这将打印所有写入 stderr 的内容，即使命令执行成功也是如此。

### 其他常见错误

错误信息 | | 原因 | | -- | | -- | | | `在版本 "argoproj.io/v1alpha1 "中没有匹配的 "ConfigManagementPlugin "类型 | | `ConfigManagementPlugin` CRD 在 Argo CD 2.4 中被弃用，并在 2.8 中被移除。 这个错误意味着您试图将插件的配置作为 CRD 直接放入 Kubernetes。 请参阅本[部分文档]（#write-the-plugin-configuration-file），了解如何编写插件配置文件并将其正确放入侧载。

## 插件焦油流排除

为了提高配置清单的生成速度，可以将某些文件和文件夹排除在外，使其不被发送到您的插件。 如果没有必要，我们建议排除您的 `.git` 文件夹。请使用 Go 的 [filepatch.Match](https://pkg.go.dev/path/filepath#Match) 语法。例如，使用 `.git/*` 来排除 `.git` 文件夹。

您可以通过三种方式之一进行设置：

1.版本服务器上的 `--plugin-tar-exclude` 参数。
2.如果被引用`argocd-cmd-params-cm`，则使用`reposerver.plugin.tar.exclusions`键
3.在 repo 服务器上直接设置 `ARGOCD_REPO_SERVER_PLUGIN_TAR_EXCLUSIONS` 环境变量。

对于选项 1，可以多次重复使用 flag；对于选项 2 和 3，可以用分号分隔指定多个 globs。

## 迁移自 argocd-cm 插件

通过修改 argocd-cm ConfigMap 安装插件的做法从 v2.4 版起已被弃用，并从 v2.8 版起被完全删除。

CMP 插件的工作方式是在 `argocd-repo-server` 中添加一个侧卡，并在位于 `/home/argocd/cmp-server/config/plugin.yaml`的侧卡中添加配置。 argocd-cm 插件可以通过以下步骤轻松转换。

### 将 ConfigMap 条目转换为配置文件

首先，将插件的配置复制到自己的 YAML 文件中。 以下面的 ConfigMap 条目为例：

```yaml
data:
  configManagementPlugins: |
    - name: pluginName
      init:                          # Optional command to initialize application source directory
        command: ["sample command"]
        args: ["sample args"]
      generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
        command: ["sample command"]
        args: ["sample args"]
      lockRepo: true                 # Defaults to false. See below.
```

`pluginName` 项将被转换为类似这样的配置文件：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: pluginName
spec:
  init:                          # Optional command to initialize application source directory
    command: ["sample command"]
    args: ["sample args"]
  generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
    command: ["sample command"]
    args: ["sample args"]
```

注意 `lockRepo` 密钥与 sidecar 插件无关，因为 sidecar 插件在生成配置清单时不会共享单个源软件仓库目录。

接下来，我们需要决定如何将 yaml 添加到 sidecar 中。 我们可以将 yaml 直接烘焙到镜像中，也可以从 ConfigMap 中加载它。

如果被引用 configmaps，我们的示例将如下所示：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pluginName
  namespace: argocd
data:
  pluginName.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: pluginName
    spec:
      init:                          # Optional command to initialize application source directory
        command: ["sample command"]
        args: ["sample args"]
      generate:                      # Command to generate Kubernetes Objects in either YAML or JSON
        command: ["sample command"]
        args: ["sample args"]
```

然后将其安装在我们的插件侧架上。

#### 为插件编写发现规则

Sidecar 插件可使用发现规则或插件名称将应用程序与插件匹配。 如果省略了发现规则，则必须在应用程序规范中明确指定插件名称，否则该特定插件将无法与任何应用程序匹配。

如果您想使用 "发现 "而不是插件名称来将应用程序与您的插件相匹配，请[使用上述说明]编写适用于您的插件的规则（#1-write-the-plugin-configuration-file），并将其添加到您的配置文件中。

要使用名称而不是发现，请将应用程序配置清单中的名称更新为 `<metadata.name>-<spec.version>`（如果 `ConfigManagementPlugin` 规范中提到了版本），否则只需使用 `<metadata.name>`。 例如：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  source:
    plugin:
      name: pluginName  # Delete this for auto-discovery (and set `plugin: {}` if `name` was the only value) or use proper sidecar plugin name
```

### 确保插件能够访问所需的工具

使用 argocd-cm 配置的插件在 Argo CD 镜像上运行。 这使它可以访问默认安装在该镜像上的所有工具（有关基础镜像和安装的工具，请参阅 [Dockerfile](https://github.com/argoproj/argo-cd/blob/master/Dockerfile)）。

您可以使用库存镜像（如 busybox 或 alpine/k8s），也可以使用插件所需的工具设计自己的基础镜像。 为了安全起见，请避免使用安装的二进制文件超过插件实际需要的镜像。

### 测试插件

根据上述说明]（#installing-a-config-management-plugin）将插件安装为侧挂件后，先在几个应用程序上进行测试，然后再将所有应用程序迁移到侧挂件插件。

测试完成后，从 argocd-cm ConfigMap 中删除插件条目。

### 其他设置

#### 保存版本库文件模式

默认情况下，配置管理插件会以重置文件模式接收源代码库文件。 这是出于安全考虑。 如果要保留原始文件模式，可以在插件规范中将 `preserveFileMode` 设置为 `true`：

警告 请确保您信任正在使用的插件。 如果将 `preserveFileMode` 设置为 `true`，那么插件可能会接收具有可执行权限的文件，这可能会带来安全风险。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: pluginName
spec:
  init:
    command: ["sample command"]
    args: ["sample args"]
  generate:
    command: ["sample command"]
    args: ["sample args"]
  preserveFileMode: true
```