<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 构建环境

[定制工具](../operator-manual/config-management-plugins.md),[Helm](helm.md),[Jsonnet](jsonnet.md)和[kustomize](kustomize.md)支持以下构建环境变量

| 变量 | 说明 | |-------------------------------------|-------------------------------------------------------------------------| | |`ARGOCD_APP_NAME`| 应用程序的名称。`ARGOCD_APP_NAMESPACE`| 应用程序的目标名称空间。`ARGOCD_APP_REVISION`| 已解决的修订，例如`f913b6cbf58aa5ae5ca1f8a2b149477aebcbd9d8`. | |`ARGOCD_APP_REVISION_SHORT`| 已解决的简短修订，例如`f913b6c`. | |`ARGOCD_APP_SOURCE_PATH`| 源版本中应用程序的路径。`ARGOCD_APP_SOURCE_REPO_URL`| 源软件仓库 URL。`ARGOCD_APP_SOURCE_TARGET_REVISION`| 规范中的目标修订，例如`master`. | |`KUBE_VERSION`| Kubernetes 的版本。`KUBE_API_VERSIONS`| Kubernetes API 的版本。

如果您不希望对变量进行插值、`$`可以通过`$$`.

```
command:
  - sh
  - -c
  - |
    echo $$FOO
```