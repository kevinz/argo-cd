<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 使用 Kubectl 同步应用程序

您可以使用 "kubectl "来要求 Argo CD 同步应用程序，就像使用 CLI 或 UI 一样。 许多配置，如 "强制"、"剪枝"、"应用"，甚至同步特定资源列表，都同样支持。 具体做法是通过定义 "操作 "的文档来引用或修补 Argo CD 应用程序。

该 "操作 "定义了同步的方式和同步的资源。

有许多配置选项可以添加到 "操作 "中。 接下来将对其中几个选项进行说明。 更多详情，请参阅 CRD[applications.argoproj.io](https://github.com/argoproj/argo-cd/blob/master/manifests/crds/application-crd.yaml)其中有些是必需的，有些则是可选的。

我们可以要求 Argo CD 同步某个应用程序的所有资源： Argo

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: <namespace>
spec:
  ...
operation:
  initiatedBy:
    username: <username>
  sync:
    syncStrategy:
      hook: {}
```

```bash
$ kubectl apply -f <apply-file>
```

最重要的部分是 "操作 "字段中的 "同步 "定义。 您可以传递 "信息 "或 "由谁发起 "等可选信息。"信息 "允许您以列表形式添加有关操作的信息。

或者，如果您愿意，也可以进行修补：

```yaml
operation:
  initiatedBy:
    username: <username>
  sync:
    syncStrategy:
    hook: {}
```

```bash
$ kubectl patch -n <namespace> app <app-name> --patch-file <patch-file> --type merge
```

请注意，补丁（特别是使用合并策略的补丁）可能无法按照您的预期运行，尤其是在您更改同步策略或选项时。 在这种情况下，"kubectl apply "会带来更好的效果。

无论是使用 "kubectl patch "还是 "kubectl apply"，同步状态都会在应用程序对象的 "operationState" 字段中报告。

```bash
$ kubectl get -n <namespace> get app <app-name> -o yaml
...
status:
  operationState:
    finishedAt: "2023-08-03T11:16:17Z"
    message: successfully synced (all tasks run)
    phase: Succeeded
```

# 应用和挂钩同步策略

同步策略有两种："挂钩"（默认值）和 "应用"。

apply "同步策略会告诉 Argo CD "kubectl apply"，而 "hook "同步策略则会通知 Argo CD 提交操作中引用的任何资源。 这样，这些资源的同步就会考虑到资源已注释的任何钩子。

```yaml
operation:
  sync:
    syncStrategy:
      apply: {}
```

```yaml
operation:
  sync:
    syncStrategy:
      hook: {}
```

这两种策略都支持 "强制"，但需要注意的是，强制操作会在补丁重试 5 次后遇到冲突时删除资源。

```yaml
operation:
  sync:
    syncStrategy:
      apply:
        force: true
```

```yaml
operation:
  sync:
    syncStrategy:
      hook:
        force: true
```

# Prune

如果您想在应用前修剪资源，可以指示 Argo CD 这样做：

```yaml
operation:
  sync:
    prune: true
```

#清单资源

您可以传递一个资源列表，该列表可以是应用程序管理的所有资源，也可以是一个子集，例如因某种原因不同步的资源。

在引用资源时，只有 "种类 "和 "名称 "是必填字段，但也可以定义 "组 "和 "名称空间 "字段：

```yaml
operation:
  sync:
    resources:
      - kind: Namespace
        name: namespace-name
      - kind: ServiceAccount
        name: service-account-name
        namespace: namespace-name
      - group: networking.k8s.io
        kind: NetworkPolicy
        name: network-policy-name
        namespace: namespace-name
```

# 同步选项

在操作中，还可以传递同步选项。 每个选项都以 "name=values "对的形式传递，例如

```yaml
operations:
  sync:
    syncOptions:
      - Validate=false
      - Prune=false
```

有关同步选项的更多信息，请参阅[同步选项](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)