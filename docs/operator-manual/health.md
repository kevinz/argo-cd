<!-- TRANSLATED by md-translate -->
# 资源健康

## 概览

Argo CD 为几种标准的 Kubernetes 类型提供内置的健康评估，然后将其浮现为整个应用程序的整体健康状态。 以下是针对特定类型的 Kubernetes 资源进行的检查：

### 部署、复制集、状态集、守护进程集

* 观测生成量等于期望生成量。
* 更新**副本的数量等于期望副本的数量。

### 服务

* 如果服务类型为 "LoadBalancer"，则 "status.loadBalancer.ingress "列表为非空、

至少包含一个 `hostname` 或 `IP` 值。

### ingress

* status.loadBalancer.ingress "列表非空，至少包含一个 "hostname "或 "IP "值。

### 工作

* 如果作业 `.spec.suspended` 设置为 "true"，则作业和应用程序健康状况将被标记为暂停。

### PersistentVolumeClaim

* status.phase "为 "Bound"。

### Argocd 应用程序

`argoproj.io/Application` CRD 的健康评估已在 argocd 1.8 中移除（更多信息请参阅 [#3781](https://github.com/argoproj/argo-cd/issues/3781)）。如果您正在使用 app-of-apps 模式并使用 sync waves 协调同步，则可能需要恢复它。 在 `argocd-cm` ConfigMap 中添加以下资源定制：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  resource.customizations: |
    argoproj.io/Application:
      health.lua: |
        hs = {}
        hs.status = "Progressing"
        hs.message = ""
        if obj.status ~= nil then
          if obj.status.health ~= nil then
            hs.status = obj.status.health.status
            if obj.status.health.message ~= nil then
              hs.message = obj.status.health.message
            end
          end
        end
        return hs
```

## 自定义健康检查

Argo CD 支持被引用到 [Lua](https://www.lua.org/)中的自定义健康检查。这对您非常有用：

* 受到已知问题的影响，即由于资源控制器中的错误，您的 `Ingress` 或 `StatefulSet` 资源被卡在 `Progressing` 状态。
* Argo CD 没有内置健康检查的自定义资源。

配置自定义健康检查有两种方法，下面两节将分别介绍。

#### 方法 1.在 `argocd-cm` 配置地图中定义自定义健康检查

可在

```yaml
resource.customizations: |
    <group/kind>:
      health.lua: |
```

字段。如果使用 argocd-operator，则会被 [argocd-operator resourceCustomizations](https://argocd-operator.readthedocs.io/en/latest/reference/argocd/#resource-customizations) 覆盖。

下面的示例演示了对 `cert-manager.io/Certificate`进行健康检查。

```yaml
data:
  resource.customizations: |
    cert-manager.io/Certificate:
      health.lua: |
        hs = {}
        if obj.status ~= nil then
          if obj.status.conditions ~= nil then
            for i, condition in ipairs(obj.status.conditions) do
              if condition.type == "Ready" and condition.status == "False" then
                hs.status = "Degraded"
                hs.message = condition.message
                return hs
              end
              if condition.type == "Ready" and condition.status == "True" then
                hs.status = "Healthy"
                hs.message = condition.message
                return hs
              end
            end
          end
        end

        hs.status = "Progressing"
        hs.message = "Waiting for certificate"
        return hs
```

为了防止对潜在的多个资源进行重复的自定义健康检查，也可以在资源种类和资源组中的任何地方指定通配符，如下面这样：

```yaml
resource.customizations: |
    ec2.aws.crossplane.io/*:
      health.lua: |
        ...
```

```yaml
resource.customizations: |
    "*.aws.crossplane.io/*":
      health.lua: | 
        ...
```

如果通配符以 `*` 开头，请注意资源自定义健康状况部分所需的引号。

`obj` 是一个包含资源的 global 变量。 脚本必须返回一个包含状态和可选消息字段的对象。 自定义健康检查可能会返回以下健康状态之一：

* 健康"--资源处于健康状态
* 进展中"--资源尚不健康，但仍在进展中，可能很快就会健康
* 降级` - 资源已降级
* Suspended`（暂停）- 资源已暂停，正在等待某些外部事件来恢复（例如暂停的 CronJob 或暂停的部署

默认情况下，健康状况通常返回 "进行中 "状态。

注意：作为一项安全措施，访问标准 Lua 库的权限默认将被禁用。管理员可通过设置 `resource.customizations.useOpenLibs.<group_kind>` 来控制访问权限。在下面的示例中，标准库已为 `cert-manager.io/Certificate` 的健康检查启用。

```yaml
data:
  resource.customizations: |
    cert-manager.io/Certificate:
      health.lua.useOpenLibs: true
      health.lua: |
        # Lua standard libraries are enabled for this script
```

#### 方式 2.

可以在 Argo CD 中捆绑健康检查。自定义健康检查脚本位于 [https://github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd)的 `resource_customizations` 目录中。该目录必须具有以下目录结构：

```
argo-cd
|-- resource_customizations
|    |-- your.crd.group.io               # CRD group
|    |    |-- MyKind                     # Resource kind
|    |    |    |-- health.lua            # Health check
|    |    |    |-- health_test.yaml      # Test inputs and expected results
|    |    |    +-- testdata              # Directory with test resource YAML definitions
```

每个健康检查都必须在 `health_test.yaml` 文件中定义测试。 `health_test.yaml` 是一个 YAML 文件，其结构如下：

```yaml
tests:
- healthStatus:
    status: ExpectedStatus
    message: Expected message
  inputPath: testdata/test-resource-definition.yaml
```

要测试已实施的自定义健康检查，请运行 `go test -v ./util/lua/`。

PR#1139](https://github.com/argoproj/argo-cd/pull/1139) 是 Cert Manager CRD 自定义健康检查的一个示例。

请注意，不支持带通配符的捆绑健康检查。

## 健康检查

Argo CD 应用程序的健康状况是根据其直接子资源（源控制中表示的资源）的健康状况推断出来的。

但是，资源的健康状况不会从子资源继承--它只使用资源本身的信息进行计算。 资源的状态字段可能包含也可能不包含子资源的健康状况信息，资源的健康检查可能考虑也可能不考虑该信息。

资源的健康状况不能从其子资源推断，因为子资源的健康状况可能与父资源的健康状况无关。 例如，部署的健康状况不一定受其 Pod 健康状况的影响。

```
App (healthy)
└── Deployment (healthy)
    └── ReplicaSet (healthy)
        └── Pod (healthy)
    └── ReplicaSet (unhealthy)
        └── Pod (unhealthy)
```

如果希望子资源的健康状况影响父资源的健康状况，则需要配置父资源的健康检查，以便将子资源的健康状况考虑在内。 由于只有父资源的状态可供健康检查使用，因此父资源的控制器需要在父资源的状态字段中提供子资源的健康状况。

```
App (healthy)
└── CustomResource (healthy) <- This resource's health check needs to be fixed to mark the App as unhealthy
    └── CustomChildResource (unhealthy)
```