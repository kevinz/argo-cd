<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 状态徽章

Argo CD 可以为任何应用程序显示包含健康状况和同步状态的徽章。 由于徽章图像可供任何未经身份验证的用户使用，因此该功能默认为禁用。 该功能可通过以下方式启用`statusbadge.enabled`的关键`argocd-cm`ConfigMap（见[argocd-cm.yaml](../operator-manual/argocd-cm.yaml))。

健康并同步](.../assets/status-badge-healthy-synced.png)

要显示此徽章，请使用以下 URL 格式`${argoCdBaseUrl}/api/badge?name=${appName}`例如，http://localhost:8080/api/badge?name=guestbook。状态图像的 URL 可在应用程序页面详细信息获取：

2.向下滚动到 "状态徽章 "部分。 3. 选择所需的模板，如 URL、Markdown 等。

为状态图像 URL 提供了 markdown、html 等格式。

复制文本并粘贴到您的 readme 或网站上。