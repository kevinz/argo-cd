<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 私人存储库

注意 有些 Git 托管程序（特别是 GitLab，也可能是内部部署的 GitLab 实例）要求您指定`.git`后缀的版本库 URL，否则它们将发送 HTTP 301 重定向到以`.git`Argo CD 将**不**遵循这些重定向，因此您必须调整版本库 URL，使其后缀为`.git`。

## 全权证书

Argo CD 支持 HTTPS 和 SSH Git 认证。

### https 用户名和密码证书

需要用户名和密码的私有存储库的 URL 开头通常为`https://`而不是`git@`或`ssh://`。

可使用 Argo CD CLI 配置凭据：

```bash
argocd repo add https://github.com/argoproj/argocd-example-apps --username <username> --password <password>
```

或用户界面：

1.导航至 "设置/仓库" ![connect repo overview](../assets/repo-add-overview.png) 2.点击 "Connect Repo using HTTPS "按钮并输入凭据 ![connect repo](../assets/repo-add-https.png)_ 注：截图中的用户名仅供参考，我们与此 GitHub 账户（如果存在）没有任何关系。

连接 repo](../assets/connect-repo.png)

#### 访问令牌

您可以使用访问令牌来代替用户名和密码，请按照 Git 托管服务的说明生成令牌：

* [GitHub](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line) * [GitLab](https://docs.gitlab.com/ee/user/project/deploy_tokens/) * [Bitbucket](https://confluence.atlassian.com/bitbucketserver/personal-access-tokens-939515499.html) * [Azure Repos](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&amp;tabs=preview-page)

然后，使用任意非空字符串作为用户名，使用访问令牌值作为密码，连接存储库。

注意 对于某些服务，您可能必须指定账户名作为用户名，而不是任何字符串。

### 用于 https 存储库的 tls 客户端证书

如果您的版本库服务器要求您使用 TLS 客户端证书进行身份验证，您可以配置 Argo CD 版本库以利用这些证书。 为此，您可以`--tls-client-cert-path`和`--tls-client-cert-key-path`切换到`argocd repo add`命令被用来引用本地系统中分别包含客户端证书和相应密钥的文件： Argo

```
argocd repo add https://repo.example.com/repo.git --tls-client-cert-path ~/mycert.crt --tls-client-cert-key-path ~/mycert.key
```

当然，您也可以将其与被引用的`--username`和`--password`开关，如果版本库服务器需要的话。 选项`--tls-client-cert-path`和`--tls-client-certkey-path`必须始终一起指定。

TLS 客户端证书和相应密钥也可通过用户界面配置，请参阅使用 HTTPS 添加 Git 仓库的说明。

注意 您的客户证书和密钥数据必须是 PEM 格式，其他格式（如 PKCS12）不被理解。 此外，请确保您的证书密钥未受密码保护，否则 Argo CD 无法引用。

注意 在网络用户界面的文本区域粘贴 tls 客户证书和密钥时，确保不包含意外换行或附加字符。

### SSH 私钥证书

需要 SSH 私钥的私有存储库的 URL 一般以`git@`或`ssh://`而不是`https://`。

你可以通过 CLI 或用户界面使用 SSH 配置 Git 仓库。

注意 Argo CD 2.4 已升级到 OpenSSH 8.9。[放弃了对](https://www.openssh.com/txt/release-8.8)见[2.3 至 2.4 升级指南](./operator-manual/upgrading/2.3-2.4.md)了解测试 SSH 服务器与 Argo CD 兼容性的详情，以及如何解决不支持更新算法的服务器的问题。

被引用 CLI：

```
argocd repo add git@github.com:argoproj/argocd-example-apps.git --ssh-private-key-path ~/.ssh/id_rsa
```

被引用的用户界面：

1.2.Click `Connect Repo using SSH` button, enter the URL and paste the SSH private key ![connect repo](../assets/repo-add-ssh.png) 3.Click `Connect` to test the connection and have the repository added

注意 在用户界面粘贴 ssh 私钥时，确保文本区域没有意外的换行或附加字符

注意 当 SSH 版本库从非标准端口提供服务时，必须使用`ssh://`-样式的 URL 来指定您的版本库。 scp 样式的`git@yourgit.com:yourrepo`URL 可以**不**支持端口指定，并将任何端口号视为版本库路径的一部分。

### GitHub 应用程序证书

在 GitHub.com 或 GitHub Enterprise 上托管的私有仓库可使用来自 GitHub 应用程序的引用进行访问。 请查阅[GitHub 文档](https://docs.github.com/en/developers/apps/about-apps#about-github-apps)如何创建应用程序。

注意 确保您的应用程序至少有`Read-only`权限`Contents`这是最低要求。

你可以通过 CLI 或用户界面，使用 GitHub App 方法配置对 GitHub.com 或 GitHub Enterprise 托管的 Git 仓库的访问。

被引用 CLI：

```
argocd repo add https://github.com/argoproj/argocd-example-apps.git --github-app-id 1 --github-app-installation-id 2 --github-app-private-key-path test.private-key.pem
```

被引用的用户界面：

1.导航至 "设置/版本库" ![connect repo overview](../assets/repo-add-overview.png) 2.点击 "Connect Repo using GitHub App "按钮，输入 URL、App Id、Installation Id 和应用程序的私钥。 ![connect repo](../assets/repo-add-github-app.png) 3.点击 "Connect "连接测试并添加版本库。

注意 在用户界面粘贴 GitHub 应用程序私钥时，确保文本区域没有意外的换行或附加字符

### 谷歌云源

可以使用 JSON 格式的 Google Cloud 服务账户密钥访问托管在 Google Cloud Source 上的私有存储库。 咨询[谷歌云文档](https://cloud.google.com/iam/docs/creating-managing-service-accounts)了解如何创建服务账户。

注意 确保您的应用程序至少有`Source Repository Reader`这是最低要求。

您可以使用 CLI 或用户界面配置对托管在 Google Cloud Source 上的 Git 仓库的访问权限。

被引用 CLI：

```
argocd repo add https://source.developers.google.com/p/my-google-cloud-project/r/my-repo --gcp-service-account-key-path service-account-key.json
```

被引用的用户界面：

1.导航至 `Settings/Repositories` ![connect repo overview](../assets/repo-add-overview.png) 2.单击 "Connect Repo using Google Cloud Source "按钮，以 JSON 格式输入 URL 和 Google Cloud 服务账户。 3.单击 "Connect "以测试连接并添加版本库。

## 证书模板

您还可以设置凭证作为连接存储库的模板，而无需重复凭证配置。`https://github.com/argoproj`，这些凭据将被引用到以该 url 为前缀的所有版本库（例如`https://github.com/argoproj/argocd-example-apps`）没有配置自己的证书。

要使用 Web UI 设置凭证模板，只需在***使用 SSH 连接软件仓库***或***使用 HTTPS 连接软件仓库***对话框（如上所述），但选择***保存为证书模板***而不是***连接***保存凭证模板。 请确保只输入前缀 URL（即`https://github.com/argoproj`），而不是完整的版本库 URL（即`https://github.com/argoproj/argocd-example-apps`）在实地***存储库 URL**。

要使用 CLI 管理凭证模板，请使用`repocreds`子命令，例如`argocd repocreds add https://github.com/argoproj --username youruser --password yourpass`会为 URL 前缀设置一个证书模板`https://github.com/argoproj`使用被引用的用户名/密码组合。`repo`子命令，还可以使用`argocd repocreds list`和`argocd repocreds rm`命令。

要让 Argo CD 使用任何给定资源库的凭证模板，必须满足以下条件： Argo CD

* 为凭证模板配置的 url（如 `https://github.com/argoproj` ）必须与版本库 url（如 `https://github.com/argoproj/argocd-example-apps` ）的前缀相匹配。

注意 只有在设置了匹配的版本库凭证后，才能使用 CLI 或 Web UI 添加需要验证的版本库，而无需指定凭证

注意，匹配凭证模板 URL 前缀是在一个最佳匹配与 v1.4 之前的配置相比，定义的顺序并不重要。

下面是一个 cli 会话示例，描述了存储库凭据的设置：

```bash
# Try to add a private repository without specifying credentials, will fail
$ argocd repo add https://docker-build/repos/argocd-example-apps
FATA[0000] rpc error: code = Unknown desc = authentication required 

# Setup a credential template for all repos under https://docker-build/repos
$ argocd repocreds add https://docker-build/repos --username test --password test
repository credentials for 'https://docker-build/repos' added

# Repeat first step, add repo without specifying credentials
# URL for template matches, will succeed
$ argocd repo add https://docker-build/repos/argocd-example-apps
repository 'https://docker-build/repos/argocd-example-apps' added

# Add another repo under https://docker-build/repos, specifying invalid creds
# Will fail, because it will not use the template (has own creds)
$ argocd repo add https://docker-build/repos/example-apps-part-two --username test --password invalid
FATA[0000] rpc error: code = Unknown desc = authentication required
```

### 自签名和不可信任的 TLS 证书

如果使用自签证书或 Argo CD 不知道的自定义证书颁发机构 (CA) 签发的证书连接 HTTPS 服务器上的版本库，由于安全原因，版本库将无法添加。 这将通过错误信息显示，例如`x509: certificate signed by unknown authority`。

1.你可以让 ArgoCD 以不安全的方式连接版本库，而无需验证服务器证书。这可以在使用 `argocd` CLI 工具添加版本库时被引用 `--insecure-skip-server-verification`（不安全地跳过服务器验证）"flag。不过，这只适用于非生产性设置，因为它可能会带来严重的中间人攻击安全问题。 2.您可以使用 `argocd` CLI 工具中的 `cert add-tls` 命令配置 ArgoCD 使用自定义证书来验证服务器证书。这是推荐的方法，适合在生产中使用。为此，您需要 PEM 格式的服务器证书或被引用签署服务器证书的 CA 的证书。

注意 对于无效的服务器证书，如不匹配服务器名称或过期的证书，添加 CA 证书将无济于事。 在这种情况下，您唯一的选择是使用`--insecure-skip-server-verification`强烈建议您在版本库服务器上被引用有效证书，或敦促服务器管理员用有效证书替换有问题的证书。

注意 tls 证书是按服务器配置的，而不是按存储库配置的。 如果从同一服务器连接多个存储库，则只需为该服务器配置一次证书。

注意，可能需要几分钟的时间才能完成更改。`argocd cert`命令会在集群中传播，具体取决于 Kubernetes 设置。

##使用 cli 管理 tls 证书

您可以通过被引用的`argocd cert list`命令，被引用为`--cert-type https`修改器：

```bash
$ argocd cert list --cert-type https
HOSTNAME TYPE SUBTYPE FINGERPRINT/SUBJECT
docker-build https rsa CN=ArgoCD Test CA
localhost https rsa CN=localhost
```

向 ArgoCD 添加 HTTPS 软件源而不验证服务器证书的示例 (**注意：**这是**不**被引用用于生产）：

```bash
argocd repo add --insecure-skip-server-verification https://git.example.com/test-repo
```

添加文件中 CA 证书的示例`~/myca-cert.pem`以正确验证版本库服务器：

```bash
argocd cert add-tls git.example.com --from ~/myca-cert.pem
argocd repo add https://git.example.com/test-repo
```

如果版本库服务器即将更换服务器证书，可能是更换由不同 CA 签发的证书，这可能会很有用。 这样，您就可以让旧（当前）和新（未来）证书同时存在。 如果您已经配置了旧证书，请使用`--upsert`标记，并在一次运行中添加新旧标记：。

```bash
cat cert1.pem cert2.pem | argocd cert add-tls git.example.com --upsert
```

注意 要替换服务器的现有证书，请使用`--upsert`标志到`cert add-tls`CLI 命令。

最后，可以通过被引用的`argocd cert rm`命令与`--cert-type https`修改器：

```bash
argocd cert rm --cert-type https localhost
```

### 使用 ArgoCD 网页用户界面管理 TLS 证书

可以使用 ArgoCD 网页用户界面添加和删除 TLS 证书：

1.在左侧导航窗格中点击 "设置"，然后从设置菜单中选择 "证书 "2。下面的页面列出了当前配置的所有证书，并提供了添加新 TLS 证书或 SSH 已知条目的选项： ![管理证书](../assets/cert-management-overview.png) 3. 点击 "添加 TLS 证书"，填写相关数据并点击 "创建"。注意只指定版本库服务器的 FQDN（而不是 URL），并将 TLS 证书的完整 PEM 复制到文本区域字段，包括 `----BEGIN CERTIFICATE----` 和 `----END CERTIFICATE----` 行： ！[添加 tls 证书](.../assets/cert-management-add-tls.png) 4.要删除证书，请点击证书条目旁边的小三点按钮，从弹出菜单中选择 "删除"，并在下面的对话框中确认删除。![删除证书](../assets/cert-management-remove.png)

### 使用声明式配置管理 tls 证书

您也可以在声明式、自我管理的 ArgoCD 设置中管理 TLS 证书。 所有 TLS 证书都存储在 ConfigMap 对象中`argocd-tls-certs-cm`请参阅[操作手册](.../.../operator-manual/declarative-setup/#repositories-using-self-signed-tls-certificates-or--are-signed-by-custom-ca)了解更多信息。

## 未知 SSH 主机

如果您通过 SSH 使用的是私有托管的 Git 服务，那么您有以下选项：

1.你可以让 ArgoCD 以不安全的方式连接版本库，而无需验证服务器的 SSH 主机密钥。这可以在使用 `argocd` CLI 工具添加版本库时被引用 `--insecure-skip-server-verification`（不安全地跳过服务器验证）"flag。不过，这只适用于非生产性设置，因为它可能通过中间人攻击带来严重的安全问题。 2.你可以使用 `argocd` CLI 工具中的 `cert add-ssh` 命令让 ArgoCD 知道服务器的 SSH 公钥。这是推荐的方法，适合在生产中使用。为此，你需要服务器的 SSH 公共主机密钥，格式为`ssh`理解的`known_hosts`。例如，你可以通过使用 `ssh-keyscan` 工具来引用服务器的 SSH 公共主机密钥。

注意，可能需要几分钟的时间才能完成更改。`argocd cert`命令会在集群中传播，具体取决于 Kubernetes 设置。

请注意，这将破坏 CLI 和 UI 证书管理，因此一般不建议使用。

### 使用 cli 管理 ssh 已知主机

您可以通过使用`argocd cert list`命令与`--cert-type ssh`修改器：

```bash
$ argocd cert list --cert-type ssh
HOSTNAME TYPE SUBTYPE FINGERPRINT/SUBJECT
bitbucket.org ssh ssh-rsa SHA256:46OSHA1Rmj8E8ERTC6xkNcmGOw9oFxYr0WF6zWW8l1E
github.com ssh ssh-rsa SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s
gitlab.com ssh ecdsa-sha2-nistp256 SHA256:HbW3g8zUjNSksFbqTiUWPWg2Bq1x8xdGUrliXFzSnUw
gitlab.com ssh ssh-ed25519 SHA256:eUXGGm1YGsMAS7vkcx6JOJdOGHPem5gQp4taiCfCLB8
gitlab.com ssh ssh-rsa SHA256:ROQFvPThGrW4RuWLoL9tq9I9zJ42fK4XywyRtbOz/EQ
ssh.dev.azure.com ssh ssh-rsa SHA256:ohD8VZEXGWo6Ez8GSEJQ9WpafgLFsOfLOtGGQCQo6Og
vs-ssh.visualstudio.com ssh ssh-rsa SHA256:ohD8VZEXGWo6Ez8GSEJQ9WpafgLFsOfLOtGGQCQo6Og
```

要添加 SSH 已知主机条目，可使用`argocd cert add-ssh`您可以从文件中添加（使用`--from<file>`修改器），或通过读取`stdin`当`-batch`在这两种情况下，输入必须以`known_hosts`的格式。

将服务器上所有可用的 SSH 公共主机密钥添加到 ArgoCD 的示例，这些密钥通过以下方式收集`ssh-keyscan`：

```bash
ssh-keyscan server.example.com | argocd cert add-ssh --batch
```

导入现有`known_hosts`文件到 ArgoCD：

```bash
argocd cert add-ssh --batch --from /etc/ssh/ssh_known_hosts
```

最后，可以使用被引用的`argocd cert rm`命令与`--cert-type ssh`修改器：

```bash
argocd cert rm bitbucket.org --cert-type ssh
```

如果给定主机有多个 SSH 已知主机条目，且密钥子类型不同（例如上例中的 gitlab.com，其密钥子类型为`ssh-rsa`,`ssh-ed25519`和`ecdsa-sha2-nistp256`），并且只想删除其中一个，您可以使用被引用的`-cert-sub-type`修改器：。

```bash
argocd cert rm gitlab.com --cert-type ssh --cert-sub-type ssh-ed25519
```

### 使用 ArgoCD 网页用户界面管理 SSH 已知主机数据

可以使用 ArgoCD 网页用户界面添加和删除 SSH 已知主机条目：

1.在左侧导航窗格中点击 "设置"，然后从设置菜单中选择 "证书 "2。以下页面列出了当前配置的所有证书，并提供了添加新 TLS 证书或 SSH 已知条目的选项： ![管理证书](../assets/cert-management-overview.png) 3. 点击 "添加 SSH 已知主机"，并在以下掩码中粘贴 SSH 已知主机数据。 **重要**：粘贴数据时，确保条目（密钥数据）中没有换行符。然后点击 "创建"。![管理 ssh 已知主机](../assets/cert-management-add-ssh.png) 4. 要删除证书，请点击证书条目旁边的小三点按钮，从弹出菜单中选择 "删除"，并在下面的对话框中确认删除。![删除证书](../assets/cert-management-remove.png)

### 使用声明式设置管理 ssh 已知主机数据

您还可以在声明式、自我管理的 ArgoCD 设置中管理 SSH 已知主机条目。 所有 SSH 公共主机密钥都存储在 ConfigMap 对象中`argocd-ssh-known-hosts-cm`更多详情，请参阅[操作手册](../operator-manual/declarative-setup.md#ssh-known-host-public-keys)。

## Git 子模块

支持子模块，并会自动拾取。 如果子模块版本库需要身份验证，则凭据需要与父版本库的凭据相匹配。 设置 ARGOCD_GIT_MODULES_ENABLED= false 可禁用子模块支持。

## 声明式配置

参见[声明式设置](.../operator-manual/declarative-setup.md#repositories)