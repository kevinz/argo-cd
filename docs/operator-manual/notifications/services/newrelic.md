<!-- TRANSLATED by md-translate -->
# NewRelic

## 参数

* `apiURL` - api 服务器网址，例如 https://api.newrelic.com
* `apiKey` - [NewRelic ApiKey](https://docs.newrelic.com/docs/apis/rest-api-v2/get-started/introduction-new-relic-rest-api-v2/#api_key)

## 配置

1.创建 NewRelic [Api Key](https://docs.newrelic.com/docs/apis/intro-apis/new-relic-api-keys/#user-api-key)
2.在 `argocd-notifications-secret` Secret 中存储 apiKey，并在 `argocd-notifications-cm` ConfigMap 中配置 NewRelic 集成

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.newrelic: |
    apiURL: <api-url>
    apiKey: $newrelic-apiKey
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  newrelic-apiKey: apiKey
```

3.复制 [申请 ID](https://docs.newrelic.com/docs/apis/rest-api-v2/get-started/get-app-other-ids-new-relic-one/#apm)
4.为您的 NewRelic 集成创建订阅

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.<trigger-name>.newrelic: <app-id>
```

## 模板

* `description` - **可选**，对此部署的高级描述，在 [Summary](https://docs.newrelic.com/docs/apm/applications-menu/monitoring/apm-overview-page) 页面和选择单个部署时在 [Deployments](https://docs.newrelic.com/docs/apm/applications-menu/events/deployments-page) 页面上可见。
    - 默认为 `message
* `changelog` - **可选**, 此部署中更改内容的摘要，当您选择（选定部署）&gt; Change logging 时，可在 [Deployments](https://docs.newrelic.com/docs/apm/applications-menu/events/deployments-page) 页面看到。
    - 默认为`{{（调用 .repo.GetCommitMetadata .app.status.sync.revision).Message}}`
* `user` - **可选**，与部署关联的用户名，在[摘要](https://docs.newrelic.com/docs/apm/applications-menu/events/deployments-page)和[部署](https://docs.newrelic.com/docs/apm/applications-menu/events/deployments-page)中可见。
    - 默认为 `{{(call .repo.GetCommitMetadata .app.status.sync.revision).Author}}}

```yaml
context: |
  argocdUrl: https://example.com/argocd

template.app-deployed: |
  message: Application {{.app.metadata.name}} has successfully deployed.
  newrelic:
    description: Application {{.app.metadata.name}} has successfully deployed
```