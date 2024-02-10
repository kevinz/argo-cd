<!-- TRANSLATED by md-translate -->
# 对账优化

默认情况下，每次属于 Argo CD 应用程序的资源发生变化时，都会刷新该应用程序。

Kubernetes 控制器经常会定期更新其监控的资源，从而导致应用程序上的持续对账操作以及 `argocd-application-controller` 的 CPU 占用率过高。 Argo CD 允许您选择忽略 [tracked resources] 的特定字段上的资源更新（.../user-guide/resource_tracking.md）。

当资源更新被忽略时，如果资源的[健康状况](./health.md)没有变化，则该资源所属的应用程序不会进行对账。

## 系统级配置

Argo CD 允许使用 [RFC6902 JSON patches](https://tools.ietf.org/html/rfc6902) 和 [JQ path expressions](https://stedolan.github.io/jq/manual/#path(path_expression)) 忽略特定 JSON 路径下的资源更新。可以在 `argocd-cm` ConfigMap 的 `resource.customizations` 键中为指定组和种类进行配置。

!!!重要 "启用该功能" 该功能在一个 flag 后面。 要启用该功能，请在 `argocd-cm` 配置表中将 `resource.ignoreResourceUpdatesEnabled` 设为 `"true"。

以下是忽略 [`外部秘密`](https://external-secrets.io/main/api/externalsecret/) 资源的`刷新时间`状态字段的自定义示例：

```yaml
data:
  resource.customizations.ignoreResourceUpdates.external-secrets.io_ExternalSecret: |
    jsonPointers:
    - /status/refreshTime
    # JQ equivalent of the above:
    # jqPathExpressions:
    # - .status.refreshTime
```

可以配置 "ignoreResourceUpdates"（忽略资源更新），使其应用于 Argo CD 实例管理的每个应用程序中的所有跟踪资源。 为此，可以像下面的示例一样配置资源自定义：

```yaml
data:
  resource.customizations.ignoreResourceUpdates.all: |
    jsonPointers:
    - /status
```

### 使用 ignoreDifferences 忽略对账

也可以使用现有的系统级 `ignoreDifferences` 自定义来忽略资源更新。 可以使用 `ignoreDifferencesOnResourceUpdates` 设置将所有忽略的差异添加为忽略的资源更新，而不是复制所有配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  resource.compareoptions: |
    ignoreDifferencesOnResourceUpdates: true
```

## 默认配置

默认情况下，所有资源的 "生成"、"资源版本 "和 "管理字段 "元数据字段总是被忽略。

## 寻找可忽略的资源

当资源变化触发刷新时，应用程序控制器会记录 logging。 你可以利用这些 logging 找到高流失率的资源种类，然后检查这些资源，找出要忽略的字段。

要查找这些日志，请搜索""由对象更新引起的应用程序刷新请求""。 日志包括 "api-版本 "和 "类型 "的结构化字段。 按 api-版本/类型统计触发的刷新次数，应能发现高消耗资源类型。

注意这些 logging 的级别为 "debug"，请将应用程序控制器的日志级别配置为 "debug"。

一旦确定了一些经常变化的资源，就可以尝试确定哪些字段在变化。 下面是一种方法：

```shell
kubectl get <resource> -o yaml > /tmp/before.yaml
# Wait a minute or two.
kubectl get <resource> -o yaml > /tmp/after.yaml
diff /tmp/before.yaml /tmp/after
```

通过 Diff 可以了解哪些字段正在发生变化，也许应该忽略。

## 检查资源更新是否被忽略

每当 Argo CD 因忽略资源更新而跳过刷新时，控制器都会记录以下一行："忽略对象的更改，因为所观察的资源字段都没有更改"。

搜索应用程序控制器日志中的这一行，以确认您的资源忽略规则已被应用。

注意，这些 logging 的级别为 "debug"，请将应用程序控制器的日志级别配置为 "debug"。

## 示例

### argoproj.io/Application

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  resource.customizations.ignoreResourceUpdates.argoproj.io_Application: |
    jsonPointers:
    # Ignore when ownerReferences change, for example when a parent ApplicationSet changes often.
    - /metadata/ownerReferences
    # Ignore reconciledAt, since by itself it doesn't indicate any important change.
    - /status/reconciledAt
    jqPathExpressions:
    # Ignore lastTransitionTime for conditions; helpful when SharedResourceWarnings are being regularly updated but not
    # actually changing in content.
    - .status.conditions[].lastTransitionTime
```