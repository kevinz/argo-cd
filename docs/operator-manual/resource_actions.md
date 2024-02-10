<!-- TRANSLATED by md-translate -->
# 资源行动

## 概览

Argo CD 允许操作员定义用户可在特定资源类型上执行的自定义操作。 这在内部被引用来提供诸如 `DaemonSet` 的 `restart` 或 Argo Rollout 的 `retry` 等操作。

操作员可以以 Lua 脚本的形式为自定义资源添加操作，并扩展这些功能。

## 自定义资源操作

Argo CD 支持用 [Lua](https://www.lua.org/)编写的自定义资源操作。这对您非常有用：

* 拥有 Argo CD 未提供任何内置操作的自定义资源。
* 用户通过 `kubectl` 执行的常见手动任务可能容易出错

资源操作作用于单个对象。

您可以在 `argocd-cm` ConfigMap 中定义自己的自定义资源操作。

#### 自定义资源操作类型

#### 修改源资源的操作

该操作修改并返回源资源。 2.8 版之前只有这种操作，现在仍受支持。

#### 生成新资源或已修改资源列表的操作

**alpha功能，在2.8.**中引入

该操作会返回一个受影响资源的列表，每个受影响资源都有一个 k8s 资源和一个要执行的操作。 目前支持的操作有 "创建 "和 "修补"，"修补 "仅支持源资源。 可以创建新资源，方法是为返回列表中的每个此类资源指定一个 "创建 "操作。 返回的资源之一可以是修改后的源对象，如有需要，还可以执行 "修补 "操作。 请参阅下面的定义示例。

### 在 `argocd-cm` configmaps 中定义自定义资源动作

自定义资源操作可在 `argocd-cm` 的 `resource.customizations.actions.<group_kind>` 字段中定义。以下示例演示了一组针对 `CronJob` 资源的自定义操作，每个操作都会返回修改后的 CronJob。自定义键的格式为 `resource.customizations.actions.<apiGroup_Kind>`。

```yaml
resource.customizations.actions.batch_CronJob: |
  discovery.lua: |
    actions = {}
    actions["suspend"] = {["disabled"] = true}
    actions["resume"] = {["disabled"] = true}

    local suspend = false
    if obj.spec.suspend ~= nil then
        suspend = obj.spec.suspend
    end
    if suspend then
        actions["resume"]["disabled"] = false
    else
        actions["suspend"]["disabled"] = false
    end
    return actions
  definitions:
  - name: suspend
    action.lua: |
      obj.spec.suspend = true
      return obj
  - name: resume
    action.lua: |
      if obj.spec.suspend ~= nil and obj.spec.suspend then
          obj.spec.suspend = false
      end
      return obj
```

discovery.lua "脚本必须返回一个表，其中的键名代表动作名称。 您可以根据当前对象的状态，有选择地加入启用或禁用某些动作的逻辑。

每个动作名称都必须在`definitions`列表中表示，并附带`action.lua`脚本，以控制资源修改。 `obj` 是一个包含资源的全局变量。 每个动作脚本都会返回资源的可选修改版本。 在本例中，我们只是将`.spec.suspend`设置为`true`或`false`。

#### 使用自定义操作创建新资源

重要 通过 Argo CD UI 创建资源是有意从战略上偏离 GitOps 原则的行为。 我们建议您尽量少用此功能，而且只用于不属于应用程序理想状态的资源。

调用该操作的资源将被称为 "源资源"。 新资源和由此隐式创建的所有资源必须在 AppProject 层面上得到允许，否则创建将失败。

##### 使用自定义操作创建源资源子资源

如果新资源代表源资源的 k8s 子资源，则必须在新资源上设置源资源的 ownerReference。 下面是一个 Lua 代码段示例，用于构建一个作为源 CronJob 资源子资源的 Job 资源--`obj` 是一个全局变量，其中包含源资源：

```lua
-- ...
ownerRef = {}
ownerRef.apiVersion = obj.apiVersion
ownerRef.kind = obj.kind
ownerRef.name = obj.metadata.name
ownerRef.uid = obj.metadata.uid
job = {}
job.metadata = {}
job.metadata.ownerReferences = {}
job.metadata.ownerReferences[1] = ownerRef
-- ...
```

##### 使用自定义操作创建独立的子资源

如果新资源独立于源资源，这种新资源的默认行为是，源资源的应用程序不知道它（因为它不是所需状态的一部分，也没有 `ownerReference`）。 要让应用程序知道新资源，必须在资源上设置 `app.kubernetes.io/instance` 标签（或其他 ArgoCD 跟踪标签，如果已配置）。 可以像这样从源资源中复制：

```lua
-- ...
newObj = {}
newObj.metadata = {}
newObj.metadata.labels = {}
newObj.metadata.labels["app.kubernetes.io/instance"] = obj.metadata.labels["app.kubernetes.io/instance"]
-- ...
```

要保留资源，可使用此 Lua 代码段在资源上设置 `Prune=false` 注释：

```lua
-- ...
newObj.metadata.annotations = {}
newObj.metadata.annotations["argocd.argoproj.io/sync-options"] = "Prune=false"
-- ...
```

(如果设置 `Prune=false` 行为，则删除 App 时不会删除资源，需要手动清理）。

现在，资源和应用程序会出现不同步，这是 ArgoCD 在创建不属于所需状态的资源时的预期行为。

如果希望将此类应用程序视为同步应用程序，请在 Lua 代码中添加以下资源 Annotations：

```lua
-- ...
newObj.metadata.annotations["argocd.argoproj.io/compare-options"] = "IgnoreExtraneous"
-- ...
```

#### 生成资源列表的操作 - 一个完整的示例：

```yaml
resource.customizations.actions.ConfigMap: |
  discovery.lua: |
    actions = {}
    actions["do-things"] = {}
    return actions
  definitions:
  - name: do-things
    action.lua: |
      -- Create a new ConfigMap
      cm1 = {}
      cm1.apiVersion = "v1"
      cm1.kind = "ConfigMap"
      cm1.metadata = {}
      cm1.metadata.name = "cm1"
      cm1.metadata.namespace = obj.metadata.namespace
      cm1.metadata.labels = {}
      -- Copy ArgoCD tracking label so that the resource is recognized by the App
      cm1.metadata.labels["app.kubernetes.io/instance"] = obj.metadata.labels["app.kubernetes.io/instance"]
      cm1.metadata.annotations = {}
      -- For Apps with auto-prune, set the prune false on the resource, so it does not get deleted
      cm1.metadata.annotations["argocd.argoproj.io/sync-options"] = "Prune=false"	  
      -- Keep the App synced even though it has a resource that is not in Git
      cm1.metadata.annotations["argocd.argoproj.io/compare-options"] = "IgnoreExtraneous"		  
      cm1.data = {}
      cm1.data.myKey1 = "myValue1"
      impactedResource1 = {}
      impactedResource1.operation = "create"
      impactedResource1.resource = cm1

      -- Patch the original cm
      obj.metadata.labels["aKey"] = "aValue"
      impactedResource2 = {}
      impactedResource2.operation = "patch"
      impactedResource2.resource = obj

      result = {}
      result[1] = impactedResource1
      result[2] = impactedResource2
      return result
```