<!-- TRANSLATED by md-translate -->
# PagerDuty

## 参数

PagerDuty 通知服务被引用用于创建 PagerDuty 事件，需要指定以下设置：

* `pagerdutyToken` - PagerDuty 验证令牌
* `from` - 与发出请求的账户相关联的有效用户的电子邮件地址。
* `serviceID` - 资源的 ID。

## 示例

以下代码段包含 PagerDuty 服务配置示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  pagerdutyToken: <pd-api-token>
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.pagerduty: |
    token: $pagerdutyToken
    from: <emailid>
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
    pagerduty:
      title: "Rollout {{.rollout.metadata.name}}"
      urgency: "high"
      body: "Rollout {{.rollout.metadata.name}} aborted "
      priorityID: "<priorityID of incident>"
```

注意： 优先级是一个标签，代表事件的重要性和影响。 只有标准计划和企业计划的 pagerduty 才有此功能。

##notations

pagerduty 通知的 Annotation 样本：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-rollout-aborted.pagerduty: "<serviceID for PagerDuty>"
```