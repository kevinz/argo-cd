<!-- TRANSLATED by md-translate -->
# Grafana

要使用 argocd-notifications 创建 Grafana 注释，必须在 [Grafana](https://grafana.com) 中创建一个 [API Key](https://grafana.com/docs/grafana/latest/http_api/auth/#create-api-key)。

![sample](https://user-images.githubusercontent.com/18019529/112024976-0f106080-8b78-11eb-9658-7663305899be.png)

可用参数 ：

* `apiURL` - 服务器网址，例如 https://grafana.example.com
* `apiKey` - 服务账户的 API 密钥
* `insecureSkipVerify` - 可选 bool，true 或 false

1.以 `admin` 的身份登录 Grafana 实例
2.在左侧菜单中，转到配置/API 密钥
3.单击 "添加 API 密钥
4.在密钥中填入名称 "ArgoCD 通知"、角色 "编辑 "和存活时间 "10 年"（例如）
5.点击 "添加 "按钮
6.将 apiKey 存储在 `argocd-notifications-secret` Secret 中，复制您的 API Key 并将其定义在 `argocd-notifications-cm` configmaps 中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.grafana: |
    apiUrl: https://grafana.example.com/api
    apiKey: $grafana-api-key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  grafana-api-key: api-key
```

7.为您的 Grafana 集成创建订阅

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.<trigger-name>.grafana: tag1|tag2 # list of tags separated with |
```

8.更改 Annotations 设置

![8](https://user-images.githubusercontent.com/18019529/112022083-47fb0600-8b75-11eb-849b-d25d41925909.png)