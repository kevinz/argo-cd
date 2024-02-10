<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 工具

## 生产

Argo CD 支持多种不同的 Kubernetes 清单定义方法： Argo CD

* [Kustomize](kustomize.md) 应用程序 * [helm](helm.md) 图表 * YAML/JSON/Jsonnet 清单目录，包括 [Jsonnet](jsonnet.md) 。 * 任何配置为配置管理插件的 [自定义配置管理工具](./operator-manual/config-management-plugins.md) * YAML/JSON/Jsonnet 清单目录，包括 [Jsonnet](jsonnet.md) 。

## 发展

Argo CD 还支持直接上传本地清单。 由于这是一种与 GitOps 模式相悖的做法，因此只能用于开发目的。`override`支持上述所有不同的 Kubernetes 部署工具。 要上传本地应用程序： Argo CD 还支持直接上传本地清单。

```bash
$ argocd app sync APPNAME --local /path/to/dir/
```