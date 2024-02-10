<!-- TRANSLATED by md-translate -->
# Microsoft

注意 "" Entra ID 的前身是 Azure AD。

* [使用 Dex 的 Entra ID SAML 企业应用程序认证](#entra-id-saml-enterprise-app-auth-using-dex)
* [Entra ID 应用程序注册认证使用 OIDC](#entra-id-app-registration-auth-using-oidc)
* [Entra ID 应用程序注册认证使用 Dex](#entra-id-app-registration-auth-using-dex)

## Entra ID SAML Enterprise App Auth 被引用 Dex

### 配置新的 Entra ID 企业应用程序

1.从 "Microsoft Entra ID"&gt;"企业应用程序 "菜单中选择 "+ 新应用程序
2.选择 `非图库应用程序
3.输入应用程序的`名称`（例如`Argo CD`），然后选择`添加
4.创建应用程序后，从 "企业应用程序 "菜单中打开它。
5.在应用程序的 "用户和组 "菜单中，添加任何需要访问服务的用户或组。![Azure 企业 SAML 用户](../../assets/azure-enterprise-users.png "Azure 企业 SAML 用户")
6.从 "单点登录 "菜单，编辑 "基本 SAML 配置 "部分如下（用 Argo URL 替换 "my-argo-cd-url"）：
    - **标识符（实体 ID）：** https://`<my-argo-cd-url>`/api/dex/callback
    - **回复 URL（断言消费者服务 URL）：** https://`<my-argo-cd-url>`/api/dex/callback
    - **登录 URL:** https://`<my-argo-cd-url>`/auth/login
    - **中继状态：** `<empty>`
    - **注销 URL:** `<empty>`
    ![Azure 企业 SAML URLs](../../assets/azure-enterprise-saml-urls.png "Azure 企业 SAML URLs")
7.从 "单点登录 "菜单，编辑 "用户属性和声称 "部分，创建以下声称：
    - `+ Add new claim` | **Name:** email | **Source:** Attribute | **Source attribute:** user.mail
    - `+ Add group claim` | **Which groups:** All groups | **Source attribute:** Group ID | **Customize:** True | **Name:** Group | **Namespace:**`<empty>`|*Emit groups as role claims:** False
    - 注意："Unique User Identifier"（唯一用户标识符）所需的声称可以保留为默认的 "user.userprincipalname"_
    ![Azure Enterprise SAML Claims](../../assets/azure-enterprise-claims.png "Azure Enterprise SAML Claims")
8.从 "单点登录 "菜单下载 SAML 签名证书 (Base64)
    - 对下载的证书文件内容进行 Base64 编码，例如
    - $ cat ArgoCD.cer | base64
    - 保留一份编码输出的副本，以便在下一节中引用。
9.从 "单点登录 "菜单中复制 "登录 URL "参数，以便在下一节中使用。

### 配置 Argo 以使用新的 Entra ID 企业应用程序

1.编辑 `argocd-cm` 并将以下 `dex.config` 添加到数据部分，将 `caData`, `my-argo-cd-url` 和 `my-login-url` 替换为 Entra ID 应用程序中的值：
    ```
    data：
           url: https://my-argo-cd-url
           dex.config：|
             logger：
               level: debug
               格式： json
             connectors：
             - type: saml
               id: saml
               name: saml
               config：
                 entityIssuer: https://my-argo-cd-url/api/dex/callback
                 ssoURL: https://my-login-url （如 https://login.microsoftonline.com/xxxxx/a/saml2）
                 caData：|
                    my-base64-encoded-certificate-data
                 redirectURI: https://my-argo-cd-url/api/dex/callback
                 usernameAttr: 电子邮件
                 emailAttr: 电子邮件
                 groupsAttr：组
    ```
2.编辑 `argocd-rbac-cm` 以配置权限，类似于下面的示例。
    - 被引用 Entra ID`Group ID` 分配角色。
    - 请参阅 [RBAC 配置](../rbac.md) 了解更多详细情况。
        ```
        # 策略示例
        policy.default: 角色：只读
        policy.csv：|
           p, role:org-admin, applications, *, */*, allow
           p，角色：org-admin，集群，get，*，allow
           p，角色：org-admin，资源库，获取，*，允许
           p，角色：org-admin，资源库，创建，*，允许
           p，角色：org-admin，版本库，更新，*，允许
           p，角色：org-admin，版本库，删除，*，允许
           g, "84ce98d1-e359-4f3b-85af-985b458de3c6", role:org-admin # （分配给角色的 azure 组）
        ```

## Entra ID 应用程序注册认证被引用 OIDC

### 配置新的 Entra ID 应用程序注册

#### 添加新的 Entra ID 应用程序注册

1.从 "Microsoft Entra ID"&gt;"应用程序注册 "菜单中选择 "+ 新注册
2.输入应用程序的 "名称"（例如，"Argo CD"）。
3.指定谁可以使用该应用程序（例如，`仅限本组织目录中的帐户`）。
4.输入重定向 URI（可选）如下（用 Argo URL 替换`my-argo-cd-url`），然后选择`添加`。
    - **平台：** `web
    - **重定向 URI:** https://`<my-argo-cd-url>`/auth/callback
5.注册完成后，Azure 门户会显示应用程序注册的 "概览 "窗格。您会看到应用程序（客户端）ID。    ![Azure 应用程序注册的概述](../../assets/azure-app-registration-overview.png "Azure 应用程序注册的概述")

#### 为 ArgoCD CLI 配置其他平台设置

1.在 Azure 门户的应用程序注册中，选择您的应用程序。
2.在 "管理 "下，选择 "身份验证"。
3.在平台配置下，选择添加平台。
4.在 "配置平台 "下，选择 "移动和桌面应用程序 "磁贴。使用下面的 Values。不应更改。
    - **重定向 URI:** `http://localhost:8085/auth/callback`!
    ![Azure 应用程序注册的身份验证](../../assets/azure-app-registration-authentication.png "Azure 应用程序注册的身份验证")

#### 添加证书一个新的 Entra ID 应用程序注册

1.从 "证书和秘密 "菜单中选择 "+ 新客户秘密
2.输入秘密的 `Name` 名称（例如 `ArgoCD-SSO`）。
    - 确保复制并保存生成的值。这是 `client_secret` 的值。
    ![Azure 应用程序注册的 Secret](../../assets/azure-app-registration-secret.png "Azure 应用程序注册的 Secret")

#### Entra ID 应用程序的设置权限

1.从 "API 权限 "菜单中选择 "+ 添加一个权限
2.找到`用户.读取`权限（在`Microsoft Graph`下）并将其授予创建的应用程序： ![Entra ID API permissions](../../assets/azure-api-permissions.png "Entra ID API permissions")
3.从 "令牌配置 "菜单中选择 "+ 添加组索赔" ![Entra ID 令牌配置]( ./../assets/azure-token-configuration.png "Entra ID 令牌配置")

### 将 Entra ID 组关联到你的 Entra ID 应用程序注册

1.从 "Microsoft Entra ID"&gt;"企业应用程序 "菜单，搜索您创建的应用程序（如 "Argo CD"）。
    - 添加新 Entra ID App 注册时，会创建与 Entra ID App 注册名称相同的企业应用程序。
2.从应用程序的 "用户和组 "菜单，添加任何需要访问服务的用户或组。![Azure 企业 SAML 用户](../../assets/azure-enterprise-users.png "Azure 企业 SAML 用户")

### 配置 Argo 以使用新的 Entra ID App 注册

1.编辑 `argocd-cm` 并配置 `data.oidc.config` 和 `data.url` 部分：
    ```
    configmaps -&gt; argocd-cm
    
         data：
            url: https://argocd.example.com/ # 替换为 Argo CD 的外部基本 URL
            oidc.config：|
                  名称： Azure
                  issuer: https://login.microsoftonline.com/{directory_tenant_id}/v2.0
                  客户端 ID： {azure_ad_application_client_id}
                  客户端密码：$oidc.azure.clientSecret
                  requestedIDTokenClaims：
                     组：
                        essential: true
                  requestedScopes：
                     - openid
                     - 个人资料
                     - 电子邮件
    ```
2.编辑 `argocd-secret` 并配置 `data.oidc.azure.clientSecret` 部分：
    ```
    Secret -&gt; argocd-secret
    
         data.oidc.azure.clientSecret
            oidc.azure.clientSecret: {client_secret | base64_encoded}
    ```
3.编辑 `argocd-rbac-cm` 以配置权限。使用 Azure 中的组 ID 分配角色
   [RBAC 配置](../rbac.md)
    ```
    configmaps -&gt; argocd-rbac-cm
    
         policy.default: 角色：只读
         policy.csv：|
            p，角色：org-admin，应用程序，*，*/*，允许
            p，角色：org-admin，集群，get，*，allow
            p，角色：org-admin，资源库，获取，*，允许
            p，角色：org-admin，资源库，创建，*，允许
            p，角色：org-admin，版本库，更新，*，允许
            p，角色：org-admin，版本库，删除，*，允许
            g，"84ce98d1-e359-4f3b-85af-985b458de3c6"，角色：org-admin
    ```
4.将角色从 jwt 令牌映射到 Argo
如果要将 jwt 令牌中的角色映射到默认角色（readonly 和 admin），则必须更改 rbac-configmap 中的 scope 变量。
    ```
    policy.default: 角色：只读
         policy.csv：|
            p, role:org-admin, applications, *, */*, allow
            p、role:org-admin、集群、get、*、allow
            p，角色：org-admin，资源库，获取，*，允许
            p，角色：org-admin，资源库，创建，*，允许
            p，角色：org-admin，版本库，更新，*，允许
            p，角色：org-admin，版本库，删除，*，允许
            g，"84ce98d1-e359-4f3b-85af-985b458de3c6"，角色：org-admin
         范围: '[组, 电子邮件]'
    ```请参阅 [operator-manual/argocd-rbac-cm.yaml](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-rbac-cm.yaml) 了解所有可用变量。

## Entra ID 应用程序注册认证被引用 Dex

如上所述，配置新的 AD 应用程序注册。 然后，将 `dex.config` 添加到 `argocd-cm` 中：

```yaml
ConfigMap -> argocd-cm

data:
    dex.config: |
      connectors:
      - type: microsoft
        id: microsoft
        name: Your Company GmbH
        config:
          clientID: $MICROSOFT_APPLICATION_ID
          clientSecret: $MICROSOFT_CLIENT_SECRET
          redirectURI: http://localhost:8080/api/dex/callback
          tenant: ffffffff-ffff-ffff-ffff-ffffffffffff
          groups:
            - DevOps
```

## 验证

### 使用 SSO 登录 ArgoCD 用户界面

1.打开新的浏览器标签页并输入 ArgoCD URI： https://`<my-argo-cd-url>` ![Azure SSO Web 登录](././assets/azure-sso-web-log-in-via-azure.png "Azure SSO Web 登录")
2.单击 "LOGIN VIA AZURE "按钮，使用 Microsoft Entra ID 帐户登录。您将看到 ArgoCD 应用程序屏幕。Azure SSO Web 应用程序](.../.../assets/azure-sso-web-application.png "Azure SSO Web 应用程序")
3.导航至用户信息并验证组 ID。组将具有您在 "为 Entra ID Application 设置权限 "步骤中添加的组对象 ID。Azure SSO Web 用户信息](.../../assets/azure-sso-web-user-info.png "Azure SSO Web 用户信息")

### 使用 CLI 被引用登录 ArgoCD

1.打开终端，执行以下命令
    ```
    argocd login<my-argo-cd-url> --grpc-web-root-path / --sso
    ```
2.从浏览器输入凭据后，你会看到下面的信息。
![Azure SSO CLI 登录](././assets/azure-sso-cli-log-in-success.png "Azure SSO CLI 登录")
3.您的终端输出将如下所示。
    ```
    警告：服务器证书出错：x509: 证书对 ingress.local 有效，对 my-argo-cd-url 无效。不安全地继续（y/n）？
         打开浏览器进行身份验证
         INFO[0003] RequestedClaims: map[groups:essential:true ]。
         Performing authorization_code flow login: https://login.microsoftonline.com/XXXXXXXXXXXXX/oauth2/v2.0/authorize?access_type=offline&amp;claims=%7B%22id_token%22%3A%7B%22groups%22%3A%7B%22essential%22%3Atrue%7D%7D%7D&amp;client_id=XXXXXXXXXXXXX&amp;code_challenge=XXXXXXXXXXXXX&amp;code_challenge_method=S256&amp;redirect_uri=http%3A%2F%2Flocalhost%3A8085%2Fauth%2Fcallback&amp;response_type=code&amp;scope=openid+profile+email+offline_access&amp;state=XXXXXXX
         验证成功
         yourid@example.com' 登录成功
         更新了'my-argo-cd-url'上下文
    ```如果您没有使用正确签名的证书，可能会收到警告。请参阅 [Why Am I Getting x509: certificate signed by unknown authority When Using The CLI?] (https://argo-cd.readthedocs.io/en/stable/faq/#why-am-i-getting-x509-certificate-signed-by-unknown-authority-when-using-the-cli)。
