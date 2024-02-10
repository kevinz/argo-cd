<!-- TRANSLATED by md-translate -->
# PagerDuty V2

## 参数

PagerDuty 通知服务被用来引用 PagerDuty 事件，需要指定以下设置：

* `serviceKeys` - 具有以下结构的字典：
    - service-name：$pagerduty-key-service-name"，其中，"service-name "是您要用于制作事件的服务名称，而"$pagerduty-key-service-name "是对包含实际 PagerDuty 集成密钥（事件 API v2 集成）的 secret 的引用。

如果希望多个 Argo 应用程序向各自的 PagerDuty 服务触发事件，请在要设置警报的每个服务中创建一个集成密钥。

要创建 PagerDuty 集成密钥，[请按照以下说明](https://support.pagerduty.com/docs/services-and-integrations#create-a-generic-events-api-integration) 向您选择的服务添加事件 API v2 集成。

## 配置

以下代码段包含 PagerDuty 服务配置示例，假定要发出警报的服务名为 "my-service"。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  pagerduty-key-my-service: <pd-integration-key>
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.pagerdutyv2: |
    serviceKeys:
      my-service: $pagerduty-key-my-service
```

## 模板

[通知模板]（.../templates.md）支持为 PagerDuty 通知指定主题：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  template.rollout-aborted: |
    message: Rollout {{.rollout.metadata.name}} is aborted.
    pagerdutyv2:
      summary: "Rollout {{.rollout.metadata.name}} is aborted."
      severity: "critical"
      source: "{{.rollout.metadata.name}}"
```

模板中的 PagerDuty 配置参数一般与 Events API v2 端点的有效载荷相匹配。 所有参数均为字符串。

* `summary` - （必填）事件的简要文本摘要，用于生成任何相关警报的摘要/标题.
* `severity` -（必填）事件描述的受影响系统状态的严重程度。允许的 Values 值：危急"、"警告"、"错误"、"信息
* source` - （必填）受影响系统的唯一位置，最好是主机名或 FQDN。
* `component` - 源计算机中对事件负责的组件。
* `group` - 服务组件的逻辑分组。
* `class` - 事件的类别/类型。
* `url` - PagerDuty 中 "在 ArgoCD 中查看 "链接应引用的 URL。

目前不支持 `timestamp` 和 `custom_details` 参数。

##notations

PagerDuty 通知的 Annotations 示例：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-rollout-aborted.pagerdutyv2: "<serviceID for PagerDuty>"
```