<!-- TRANSLATED by md-translate -->
# 控制 ApplicationSet 控制器是否/何时修改 `Application` 资源

ApplicationSet 控制器支持一系列限制控制器对生成的应用程序进行更改的设置，例如，防止控制器删除子应用程序。

通过这些设置，您可以控制对应用程序及其相应集群资源（"部署"、"服务 "等）进行更改的时间和方式。

以下是一些控制器设置，可以通过修改这些设置来改变 ApplicationSet 控制器的资源处理行为。

### 干运行：防止 ApplicationSet 创建、修改或删除所有应用程序

为防止 ApplicationSet 控制器创建、修改或删除任何 "应用程序 "资源，可以启用 "干运行 "模式。 这基本上是将控制器切换为 "只读 "模式，在该模式下，控制器的 "重构 "循环将运行，但不会修改任何资源。

要启用干运行，请在 ApplicationSet 部署的容器启动参数中添加 `--dryrun true"。

有关如何将此参数添加到控制器的详细步骤，请参阅下面的 "如何修改 ApplicationSet 容器参数"。

### 管理应用程序修改政策

ApplicationSet 控制器支持参数"--政策"，该参数在启动时（在控制器部署容器内）指定，用于限制对 Argo CD "应用程序 "资源进行哪些类型的修改。

策略 "参数有四个 Values 值："同步"、"仅创建"、"创建-删除 "和 "创建-更新"（"同步 "为默认值，如果未指定"--策略 "参数，则使用该值；其他策略将在下文介绍）。

也可以按 ApplicationSet 设置该策略。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  # (...)
  syncPolicy:
    applicationsSync: create-only # create-update, create-delete sync
```

* 仅限创建 "策略：防止 ApplicationSet 控制器修改或删除应用程序。防止应用程序控制器根据 [ownerReferences](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) 的来源自删除应用程序。
* 策略 `create-update`：防止 ApplicationSet 控制器删除应用程序。允许更新。防止应用程序控制器根据 [OwnerReferences](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) 删除应用程序。
* 策略 `创建-删除`：防止 ApplicationSet 控制器修改应用程序。允许删除。
* 策略 `sync`：允许更新和删除。

如果控制器参数 `--policy`被设置，它将优先于字段 `applicationsSync`. 可以通过将变量 `ARGOCD_APPLICATIONSET_CONTROLLER_ENABLE_POLICY_OVERRIDE` 设置为 argocd-cmd-params-cm `applicationsetcontroller.enable.policy.override` 或直接使用控制器参数 `--enable-policy-override`（默认为 `false`）来允许每个 ApplicationSet 的同步策略。

### 控制器参数

要允许 ApplicationSet 控制器_创建_"应用程序 "资源，但阻止任何进一步的修改，如删除或修改应用程序字段，请在 ApplicationSet 控制器中添加此参数：

```
--policy create-only
```

在 ApplicationSet 层级

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  # (...)
  syncPolicy:
    applicationsSync: create-only
```

## 策略 - `create-update`：防止 ApplicationSet 控制器删除应用程序

要允许 ApplicationSet 控制器创建或修改 "应用程序 "资源，但阻止删除应用程序，可在 ApplicationSet 控制器 "部署 "中添加以下参数：

```
--policy create-update
```

这可能对寻求额外保护以防止控制器生成的应用程序被删除的用户有用。

在 ApplicationSet 层级

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  # (...)
  syncPolicy:
    applicationsSync: create-update
```

## 忽略应用程序的某些更改

ApplicationSet 规范包含一个 "ignoreApplicationDifferences "字段，可用于指定在比较应用程序时应忽略 ApplicationSet 的哪些字段。

该字段支持多条忽略规则。 每条忽略规则可指定要忽略的 `jsonPointers` 或 `jqPathExpressions` 列表。

也可选择指定 "名称"，将忽略规则应用于特定应用程序；或省略 "名称"，将忽略规则应用于所有应用程序。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/source/targetRevision
    - name: some-app
      jqPathExpressions:
        - .spec.source.helm.values
```

###允许临时切换自动同步

忽略差异的最常见用例之一是允许临时切换应用程序的自动同步。

例如，如果您有一个配置为自动同步应用程序的 ApplicationSet，您可能希望暂时禁用特定应用程序的自动同步。 您可以通过为 `spec.syncPolicy.automated` 字段添加忽略规则来实现这一点。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/syncPolicy
```

###`ignoreApplicationDifferences` 的限制

调节 ApplicationSet 时，控制器会将 ApplicationSet 规范与它所管理的每个应用程序的规范进行比较。 如果存在任何差异，控制器会生成一个补丁来更新应用程序，使其与 ApplicationSet 规范相匹配。

根据 MergePatch 文档，当列表发生变化时，"现有列表将被新列表完全替换"。

当被忽略的字段位于一个列表中时，这就限制了 `ignoreApplicationDifferences` 的有效性。 例如，如果您的应用程序有多个源，而您想忽略其中一个源的 `targetRevision` 的更改，那么其他字段或其他源的更改将导致整个 `sources` 列表被替换，而 `targetRevision` 字段将重置为 ApplicationSet 中定义的值。

例如，请看这个 ApplicationSet：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  ignoreApplicationDifferences:
    - jqPathExpressions:
        - .spec.sources[] | select(.repoURL == "https://git.example.com/org/repo1").targetRevision
  template:
    spec:
      sources:
      - repoURL: https://git.example.com/org/repo1
        targetRevision: main
      - repoURL: https://git.example.com/org/repo2
        targetRevision: main
```

您可以随意更改 `repo1` 源代码的 `targetRevision` 版本，ApplicationSet 控制器不会覆盖您的更改。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  sources:
  - repoURL: https://git.example.com/org/repo1
    targetRevision: fix/bug-123
  - repoURL: https://git.example.com/org/repo2
    targetRevision: main
```

但是，如果更改了 `repo2` 源的 `targetRevision` ，ApplicationSet 控制器将覆盖整个 `sources` 字段。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  sources:
  - repoURL: https://git.example.com/org/repo1
    targetRevision: main
  - repoURL: https://git.example.com/org/repo2
    targetRevision: main
```

注意[未来的改进](https://github.com/argoproj/argo-cd/issues/15975) ApplicationSet 控制器可能会消除这个问题。例如，可以将 `ref` 字段设为合并键，允许 ApplicationSet 控制器生成并使用 StrategicMergePatch 而不是 MergePatch。这样，您就可以通过 `ref` 针对特定源，忽略对该源中字段的更改，而对其他源的更改不会导致被忽略的字段被覆盖。

## 当父应用程序被删除时，防止 `Application` 的子资源被删除

默认情况下，当 ApplicationSet 控制器删除 "应用程序 "资源时，该应用程序的所有子资源也将被删除（例如，该应用程序的所有 "部署"、"服务 "等）。

要防止在删除父应用程序时删除应用程序的子资源，请在 ApplicationSet 的 `syncPolicy` 中添加 `preserveResourcesOnDeletion: true` 字段：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  # (...)
  syncPolicy:
    preserveResourcesOnDeletion: true
```

有关 "preserveResourcesOnDeletion "的具体行为以及 ApplicationSet 控制器和 Argo CD 中一般删除行为的更多信息，请参阅 [Application Deletion](Application-Deletion.md)页面。

## 防止应用程序的子资源被修改

对 ApplicationSet 所做的更改会传播到 ApplicationSet 所管理的应用程序，然后 Argo CD 会将应用程序的更改传播到相应的集群资源（如 [Argo CD Integration](Argo-CD-Integration.md)）。

应用程序更改向集群的传播由[自动同步设置](.../.../user-guide/auto_sync.md)管理，[自动同步设置]在 ApplicationSet `template` 字段中被引用：

* spec.template.syncPolicy.automated`：如果启用，应用程序的更改将自动传播到集群的集群资源。
    - 在 ApplicationSet 模板中取消设置此项，可 "暂停 "对 "应用程序 "资源所管理集群资源的更新。
* `spec.template.syncPolicy.automated.prune`：默认情况下，当 Argo CD 检测到 Git 中不再定义资源时，自动同步不会删除该资源。
    - 为更加安全起见，请将此设置为 false，以防止后备 Git 仓库的意外更改影响集群资源。

### 如何修改 ApplicationSet 容器启动参数

有几种方法可以修改 ApplicationSet 容器参数，从而启用上述设置。

#### A) 使用 `kubectl edit` 修改集群上的部署情况

编辑集群上的 ApplicationSet-controller `Deployment` 资源：

```
kubectl edit deployment/argocd-applicationset-controller -n argocd
```

找到 `.spec.template.spec.containers[0].command` 字段，并添加所需的参数：

```yaml
spec:
    # (...)
  template:
    # (...)
    spec:
      containers:
      - command:
        - entrypoint.sh
        - argocd-applicationset-controller
        # Insert new parameters here, for example:
        # --policy create-only
    # (...)
```

保存并退出编辑器，等待包含更新参数的新 "Pod "启动。

#### 或者，B) 编辑 ApplicationSet 安装的 `install.yaml` 配置清单

与其直接编辑集群资源，不如选择修改被引用用于安装 ApplicationSet 控制器的安装 YAML：

适用于版本小于 0.4.0 的应用程序集。

```bash
# Clone the repository

git clone https://github.com/argoproj/applicationset

# Checkout the version that corresponds to the one you have installed.
git checkout "(version of applicationset)"
# example: git checkout "0.1.0"

cd applicationset/manifests

# open 'install.yaml' in a text editor, make the same modifications to Deployment 
# as described in the previous section.

# Apply the change to the cluster
kubectl apply -n argocd -f install.yaml
```

## 保存对应用程序 Annotations 和标签所做的更改

注意：使用上述 [`ignoreApplicationDifferences`](#ignore-certain-changes-to-applications)功能，可以在每个应用程序上实现相同的行为。 不过，保留字段可以进行全局配置，"ignoreApplicationDifferences`"尚不具备这一功能。

在 Kubernetes 中，将状态存储在 Annotations 中是一种常见做法，操作员通常会利用这一点。 为了实现这一点，可以配置一个 Annotations 列表，以便 ApplicationSet 在对账时保留这些注释。

例如，假设我们从 ApplicationSet 创建了一个应用程序，但后来（在应用程序中）添加了一个自定义 Annotations 和标签，而该标签并不存在于`ApplicationSet`资源中：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  # This annotation and label exists only on this Application, and not in 
  # the parent ApplicationSet template:
  annotations: 
    my-custom-annotation: some-value
  labels:
    my-custom-label: some-value
spec:
  # (...)
```

要保留此 Annotations 和标签，我们可以像这样使用 `ApplicationSet` 的 `preservedFields` 属性：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  # (...)
  preservedFields:
    annotations: ["my-custom-annotation"]
    labels: ["my-custom-label"]
```

尽管 ApplicationSet 本身的元数据中没有定义该注解和标签，但在对账时，ApplicationSet 控制器仍会保留该注解和标签。

默认情况下，Argo CD 通知和 Argo CD 刷新类型 Annotations 也会保留。

注意 我们还可以通过向 `ARGOCD_APPLICATIONSET_CONTROLLER_GLOBAL_PRESERVED_ANNOTATIONS` 和 `ARGOCD_APPLICATIONSET_CONTROLLER_GLOBAL_PRESERVED_LABELS` 分别传递逗号分隔的 Annotations 和标签列表，为控制器设置全局保留字段。

## 调试应用程序的意外更改

ApplicationSet 控制器更改应用程序时，会在调试级别记录补丁。 要查看这些日志，请在 `argocd` 名称空间的 `argocd-cmd-params-cm` ConfigMap 中将日志级别设置为调试：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  applicationsetcontroller.log.level: debug
```