<!-- TRANSLATED by md-translate -->
# Opsgenie

要使用 argocd-notifications 发送通知，您必须在 [Opsgenie Team](https://docs.opsgenie.com/docs/teams) 内创建一个 [API Integration](https://docs.opsgenie.com/docs/integrations-overview)。

1.登录 Opsgenie https://app.opsgenie.com 或 https://app.eu.opsgenie.com（如果您的账户在欧盟）。
2.确保您已经有一个团队，如果没有，请遵循本指南 https://docs.opsgenie.com/docs/teams
3.点击左侧菜单中的 "团队
4.选择要通知的团队
5.在团队配置菜单中选择 "集成
6. 单击右上角的 "添加集成
7.选择 "API "集成
8.为集成命名，复制 "API 密钥 "并将其保存在某个地方以备不时之需
9.确保选中 "创建和更新访问 "和 "启用 "复选框，禁用其他复选框以删除不必要的权限
10.点击底部的 "安全集成
11.检查浏览器中的服务器 apiURL 是否正确。如果是 "app.opsgenie.com"，则在下一步中使用美国/国际 api 网址 "api.opsgenie.com"，否则使用 "api.eu.opsgenie.com"（欧洲 API）。
12.配置 opsgenie 完成。现在需要配置 argocd-notifications。使用 apiUrl、团队名称和 apiKey 在 `argocd-notifications-secret` secrets 中配置 Opsgenie 集成。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.opsgenie: |
    apiUrl: <api-url>
    apiKeys:
      <your-team>: <integration-api-key>
```