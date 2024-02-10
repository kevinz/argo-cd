<!-- TRANSLATED by md-translate -->
# GitHub

## 参数

GitHub 通知服务会被引用 [GitHub Apps](https://docs.github.com/en/developers/apps)来更改提交状态，需要指定以下设置：

* `appID` - 应用程序 ID
* `installationID` - 应用程序的安装 ID
* `privateKey` - 应用程序私钥
* `enterpriseBaseURL` - 可选 URL，例如 https://git.example.com/

## 配置

1.使用 https://github.com/settings/apps/new 创建 GitHub 应用程序
2.更改版本库权限，以启用写提交状态和/或部署和/或拉动请求注释功能

![2](https://user-images.githubusercontent.com/18019529/108397381-3ca57980-725b-11eb-8d17-5b8992dc009e.png) 3. 生成私钥，并自动下载 ![3](https://user-images.githubusercontent.com/18019529/108397926-d4a36300-725b-11eb-83fe-74795c8c3e03.png) 4. 将应用程序安装到账户 5. 将私钥存储在 `argocd-notifications-secret` Secret 中，并在 `argocd-notifications-cm` Configmaps 中配置 GitHub 集成

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.github: |
    appID: <app-id>
    installationID: <installation-id>
    privateKey: $github-privateKey
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  github-privateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    (snip)
    -----END RSA PRIVATE KEY-----
```

6.为 GitHub 集成创建订阅

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.<trigger-name>.github: ""
```

## 模板

![](https://user-images.githubusercontent.com/18019529/108520497-168ce180-730e-11eb-93cb-b0b91f99bdc5.png)

```yaml
template.app-deployed: |
  message: |
    Application {{.app.metadata.name}} is now running new version of deployments manifests.
  github:
    repoURLPath: "{{.app.spec.source.repoURL}}"
    revisionPath: "{{.app.status.operationState.syncResult.revision}}"
    status:
      state: success
      label: "continuous-delivery/{{.app.metadata.name}}"
      targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true"
    deployment:
      state: success
      environment: production
      environmentURL: "https://{{.app.metadata.name}}.example.com"
      logURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true"
      requiredContexts: []
      autoMerge: true
    pullRequestComment:
      content: |
        Application {{.app.metadata.name}} is now running new version of deployments manifests.
        See more here: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true
```

**注**：

* 如果信息设置为 140 个字符或更多，则会被截断。
* 如果 `github.repoURLPath` 和 `github.revisionPath` 与上述相同，则可以省略。
* 自动合并是可选选项，对于 github 部署，默认情况下为 `true`，以确保请求的 ref 与默认分支保持一致。如果想在默认分支中部署较早的 ref，则需要将此选项设置为 `false`。欲了解更多信息，请参阅 [GitHub 部署 API 文档](https://docs.github.com/en/rest/deployments/deployments?apiVersion=2022-11-28#create-a-deployment)。
* 如果 `github.pullRequestComment.content` 设置为 65536 个字符或更多，将被截断。
