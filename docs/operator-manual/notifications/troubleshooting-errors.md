<!-- TRANSLATED by md-translate -->
## 解析新设置失败

将 YAML 转换为 JSON 的 #### 错误

YAML 语法不正确。

**不正确：**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
    icon: :rocket:
```

**正确：**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
    icon: ":rocket:"
```

#### 服务类型 'xxxx' 不支持

您需要检查您的 argocd-notifications 控制器版本。 例如，团队集成支持 `v1.1.0` 及更高版本。

## 通知收件人失败

### 不支持 "XXXX "通知服务

您未在 `argocd-notifications-cm` 中定义 `xxxx` 或无法解析设置。