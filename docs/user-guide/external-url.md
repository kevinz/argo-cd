<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 添加外部 URL

您可以在 Argo CD 面板上添加其他外部链接，例如监控页面或文档链接，而不仅仅是入口主机或其他应用程序。

ArgoCD 可根据资源注释为资源生成可点击的外部页面链接。

例如

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-svc
  annotations:
    link.argocd.argoproj.io/external-link: http://my-grafana.example.com/pre-generated-link
```

外部链接](.../assets/external-link.png)

外部链接图标将在 ArgoCD 申请详情页面上显示相关资源。

外部链接](../assets/external-link-1.png)