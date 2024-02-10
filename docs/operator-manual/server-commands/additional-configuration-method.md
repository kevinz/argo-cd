<!-- TRANSLATED by md-translate -->
## 其他配置方法

用于配置命令 `argocd-server`, `argocd-repo-server` 和 `argocd-application-controller` 的附加配置方法。

#### 简介

也可以通过在 `argocd-cmd-params-cm.yaml` 中设置可用选项的相应 flag 来配置命令。 每个组件都有与之相关的特定前缀。

```
argocd-server                 --> server
argocd-repo-server            --> reposerver
argocd-application-controller --> controller
```

没有前缀的标志在多个组件中共享。 其中一个标志是 `repo.server` 可用标志的列表可以在 [argocd-cmd-params-cm.yaml](../argocd-cmd-params-cm.yaml) 中找到。

### 示例

要设置 `argocd-application-controller` 的 `logformat` ，请在配置 maps `argocd-cmd-params-cm.yaml` 中添加以下条目。

```
data:
    controller.log.format: "text"
```