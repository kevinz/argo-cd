<!-- TRANSLATED by md-translate -->
# AWS SQS

## 参数

该通知服务能够向 AWS SQS 队列发送简单信息。

* queue` - 发送信息的队列名称。可通过目标目的地 Annotations 重载。
* `region` - sqs 队列的区域，可通过环境变量 AWS_DEFAULT_REGION Provider。
* `key` - 可选，AWS 访问密钥必须通过变量或环境变量 AWS_ACCESS_KEY_ID 从 secret 中引用
* `secret` - 可选，AWS 访问秘密必须通过变量或环境变量 AWS_SECRET_ACCESS_KEY 从秘密中引用
* `account` - 可选，队列的外部 accountId
* `endpointUrl` 可选，用于使用 localstack 开发

## 示例

#### 使用 Secret 进行证书引用：

资源 Annotations：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations:
    notifications.argoproj.io/subscribe.on-deployment-ready.awssqs: "overwrite-myqueue"
```

* 配置地图

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.awssqs: |
    region: "us-east-2"
    queue: "myqueue"
    account: "1234567"
    key: "$awsaccess_key"
    secret: "$awsaccess_secret"

  template.deployment-ready: |
    message: |
      Deployment {{.obj.metadata.name}} is ready!

  trigger.on-deployment-ready: |
    - when: any(obj.status.conditions, {.type == 'Available' && .status == 'True'})
      send: [deployment-ready]
    - oncePer: obj.metadata.annotations["generation"]
```

秘密

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  awsaccess_key: test
  awsaccess_secret: test
```

### 使用 AWS Env 变量进行最低配置

确保通过 OIDC 或其他方法注入以下环境变量列表，并假设 SQS 位于账户本地。 对于敏感数据，可以不使用 secret，也可以省略其他参数（通过 configmaps 设置参数优先）。

变量

```bash
export AWS_ACCESS_KEY_ID="test"
export AWS_SECRET_ACCESS_KEY="test"
export AWS_DEFAULT_REGION="us-east-1"
```

资源 Annotations：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations:
    notifications.argoproj.io/subscribe.on-deployment-ready.awssqs: ""
```

* 配置地图

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.awssqs: |
    queue: "myqueue"

  template.deployment-ready: |
    message: |
      Deployment {{.obj.metadata.name}} is ready!

  trigger.on-deployment-ready: |
    - when: any(obj.status.conditions, {.type == 'Available' && .status == 'True'})
      send: [deployment-ready]
    - oncePer: obj.metadata.annotations["generation"]
```