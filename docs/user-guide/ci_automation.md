<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 从 CI 管道实现自动化

Argo CD 遵循 GitOps 部署模式，首先将所需的配置更改推送到 Git，然后集群状态同步到 git 中的所需状态。 这与传统上不使用 Git 仓库来保存应用程序配置的命令式管道不同。

要将新的容器映像推送到 Argo CD 管理的集群，可以使用以下工作流程（或各种变体）： Argo

## 构建并发布新的容器映像

```bash
docker build -t mycompany/guestbook:v2.0 .
docker push mycompany/guestbook:v2.0
```

## 使用你首选的模板工具更新本地清单，并将更改推送至 Git

强烈建议使用不同的 Git 仓库来保存 Kubernetes 清单（与应用程序源代码分开）。 请参见[最佳做法](best_practices.md)以进一步说明理由。

```bash
git clone https://github.com/mycompany/guestbook-config.git
cd guestbook-config

# kustomize
kustomize edit set image mycompany/guestbook:v2.0

# plain yaml
kubectl patch --local -f config-deployment.yaml -p '{"spec":{"template":{"spec":{"containers":[{"name":"guestbook","image":"mycompany/guestbook:v2.0"}]}}}}' -o yaml

git add . -m "Update guestbook to v2.0"
git push
```

## 同步应用程序（可选）

为了方便起见，可以直接从 API 服务器下载 argocd CLI，这样就可以使 CI 管道中引用的 CLI 始终保持同步，并且使用的 argocd 二进制文件始终与 Argo CD API 服务器兼容。

```bash
export ARGOCD_SERVER=argocd.example.com
export ARGOCD_AUTH_TOKEN=<JWT token generated from project>
curl -sSL -o /usr/local/bin/argocd https://${ARGOCD_SERVER}/download/argocd-linux-amd64
argocd app sync guestbook
argocd app wait guestbook
```

如果[自动同步](auto_sync.md)控制器将自动检测到新配置（使用一个[网络钩子](../operator-manual/webhook.md)或每 3 分钟轮询一次），并自动同步新清单。