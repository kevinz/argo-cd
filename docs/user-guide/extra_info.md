<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 添加额外的应用程序信息

您可以在 Argo CD 面板上为应用程序添加其他信息。 如果您想添加可点击链接，请参阅[添加外部 URL](https://argo-cd.readthedocs.io/en/stable/user-guide/external-url/)。

这可以通过在应用程序清单中为 info "字段提供键值来实现。

例如

```yaml
project: argo-demo
source:
  repoURL: 'https://demo'
  path: argo-demo
destination:
  server: https://demo
  namespace: argo-demo
info:
  - name: Example:
    value: >-
      https://example.com
```

![外部链接](../assets/extra_info-1.png)

附加信息将在 Argo CD 申请详情页上显示。

![外部链接](../assets/extra_info.png)

![外部链接](../assets/extra_info-2.png)