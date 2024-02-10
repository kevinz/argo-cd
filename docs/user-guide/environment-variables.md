<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 环境变量

下列环境变量可被引用`argocd`CLI：

| Environment Variable | Description | |--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| |`ARGOCD_SERVER`| Argo CD 服务器的地址，而不用`https://`词头<br>(而不是指定`--server`为每条命令）<br>例如`ARGOCD_SERVER=argocd.example.com`如果通过带有 DNS | | | 的入口提供服务`ARGOCD_AUTH_TOKEN`| 阿尔戈光盘`apiKey`让您的 Argo CD 用户能够进行身份验证。`ARGOCD_OPTS`| 命令行选项传递给`argocd`CLI<br>例如`ARGOCD_OPTS="--grpc-web"`| |`ARGOCD_SERVER_NAME`| Argo CD API 服务器名称（默认为 "argocd-server"）。``ARGOCD_REPO_SERVER_NAME``ARGOCD_APPLICATION_CONTROLLER_NAME``| Argo CD 应用程序控制器名称（默认为 "argocd-application-controller"） | |`ARGOCD_REDIS_NAME`| Argo CD Redis