<!-- TRANSLATED by md-translate -->
# 谷歌

将 Argo CD 登录与 Google Workspace 用户集成有三种不同的方法。 一般来说，OpenID Connect (_oidc_) 方法是推荐的集成方法（也更简单......），但根据您的需求，您可以选择不同的选项。

* [使用 Dex 的 OpenID 连接](#openid-connect-using-dex)  
如果不需要用户所属群组的信息，建议使用这种登录方法。Google 不会通过_oidc_公开 "群组 "声称，因此无法将 Google 群组成员信息用于 RBAC。
* [使用 Dex 的 SAML 应用程序认证](#saml-app-auth-using-dex)  
Dex [建议避免使用此方法](https://dexidp.io/docs/connectors/saml/#warning)。此外，通过这种方法也无法获得 Google Groups 成员信息。
* [使用 Dex 的 OpenID Connect 加 Google 群组](#openid-connect-plus-google-groups-using-dex)  
如果需要在 RBAC 配置中使用 Google Groups 成员资格，建议使用此方法。

一旦设置了上述集成之一，请务必编辑 `argo-rbac-cm` 以配置权限（如下面的示例所示）。 有关更详细的情况，请参阅 [RBAC Configurations](../rbac.md) 。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
```

## 使用 Dex 的 OpenID 连接

### 配置你的 OAuth 同意屏幕

如果您从未配置过此功能，则在尝试创建 OAuth 客户 ID 时会直接跳转到此页面

1.转到 [OAuth 同意](https://console.cloud.google.com/apis/credentials/consent) 配置。如果尚未创建，请选择 "内部 "或 "外部"，然后点击 "创建"。
2.转到[编辑 OAuth 同意屏幕](https://console.cloud.google.com/apis/credentials/consent/edit) 确认你在正确的项目中！
3.配置登录应用程序的名称和用户支持电子邮件地址
4.应用程序徽标和填写信息链接不是必须的，但它是登录页面的点睛之笔
5.在 "授权域名 "中添加允许登录 ArgoCD 的域名（例如，如果添加 "example.com"，则所有拥有"@example.com "地址的 Google Workspace 用户都能登录）
6.保存以继续 "范围 "部分
7.点击 "添加或删除作用域 "并添加".../auth/userinfo.profile "和 "openid "作用域
8.保存，查看更改摘要并完成

### 配置新的 OAuth 客户端 ID

1.访问 [Google API Credentials](https://console.cloud.google.com/apis/credentials) 控制台，确保您所在的项目正确无误。
2.点击 "+创建凭证"/"OAuth 客户 ID
3.在 "应用程序类型 "下拉菜单中选择 "Web 应用程序"，并输入应用程序的标识名称（例如 "Argo CD）
4.在 "Authorized JavaScript origins"（授权的 JavaScript 源）中填写 Argo CD 的 URL，例如：`https://argocd.example.com`。
5.在 "授权重定向 URI "中填写 Argo CD URL 和 `/api/dex/callback`，例如 `https://argocd.example.com/api/dex/callback`.
    ![](../../assets/google-admin-oidc-uris.png)
6.点击 "创建 "并保存您的 "客户 ID "和 "客户秘密 "以备后用

### 配置 Argo 以使用 OpenID Connect

编辑 `argocd-cm` 并将以下 `dex.config` 添加到数据部分，将 `clientID` 和 `clientSecret` 替换为之前保存的值：

```yaml
data:
  url: https://argocd.example.com
  dex.config: |
    connectors:
    - config:
        issuer: https://accounts.google.com
        clientID: XXXXXXXXXXXXX.apps.googleusercontent.com
        clientSecret: XXXXXXXXXXXXX
      type: oidc
      id: google
      name: Google
```

#### 参考资料

* [Dex oidc 连接器文档](https://dexidp.io/docs/connectors/oidc/)

## 使用 Dex 的 SAML 应用程序认证

### 配置新的 SAML 应用程序

---

！！！警告 "弃用警告"

```
Note that, according to [Dex documentation](https://dexidp.io/docs/connectors/saml/#warning), SAML is considered unsafe and they are planning to deprecate that module.
```

---

1.在 [Google 管理控制台](https://admin.google.com)，打开左侧菜单并选择 "应用程序"&gt;"SAML 应用程序
    谷歌管理应用程序菜单](.../../assets/google-admin-saml-apps-menu.png "选择应用程序/SAML 应用程序路径的谷歌管理菜单")
2.在 "添加应用程序 "下选择 "添加自定义 SAML 应用程序
    ![Google 管理添加自定义 SAML 应用程序](../../assets/google-admin-saml-add-app-menu.png "添加应用程序菜单，突出显示添加自定义 SAML 应用程序")
3.输入应用程序的 "名称"（例如 "Argo CD"），然后选择 "继续
    ![Google 管理应用程序菜单](../../assets/google-admin-saml-app-details.png "添加应用程序菜单，突出显示添加自定义 SAML 应用程序")
4.下载元数据或从身份 Provider 详情中复制 "SSO URL"、"证书 "和可选的 "实体 ID"，以便在下一部分中使用。选择 "继续"。
    - 对证书文件的内容进行 Base64 编码，例如
    - `$ cat ArgoCD.cer | base64` 。
    - 保留一份编码输出的副本，以便在下一节中被引用。
    - _Ensure that the certificate is in PEM format before base64 encoding_
    ![谷歌管理 IdP 元数据](../../assets/google-admin-idp-metadata.png "谷歌 IdP 元数据截图")
5.对于 "ACS URL "和 "实体 ID"，请使用您的 Argo Dex 回调 URL，例如："https://argocd.example.com/api/dex/callback`"。
    ![谷歌管理服务提供商详细信息](.../../assets/google-admin-service-provider-details.png "谷歌服务提供商详细信息截图")
6.添加 SAML 属性映射，将 "主要电子邮件 "映射到 "name"，将 "主要电子邮件 "映射到 "email"，然后单击 "添加映射 "按钮。
    ![谷歌管理员 SAML 属性映射详情]( .././assets/google-admin-saml-attribute-mapping-details.png "谷歌管理员 SAML 属性映射详情截图")
7.完成应用程序的创建。

### 配置 Argo 以使用新的 Google SAML 应用程序

编辑 `argocd-cm` 并将以下 `dex.config` 添加到数据部分，将 `caData`, `argocd.example.com`, `sso-url` 以及 `google-entity-id` 替换为 Google SAML 应用程序中的值：

```yaml
data:
  url: https://argocd.example.com
  dex.config: |
    connectors:
    - type: saml
      id: saml
      name: saml
      config:
        ssoURL: https://sso-url (e.g. https://accounts.google.com/o/saml2/idp?idpid=Abcde0)
        entityIssuer: https://argocd.example.com/api/dex/callback
        caData: |
          BASE64-ENCODED-CERTIFICATE-DATA
        redirectURI: https://argocd.example.com/api/dex/callback
        usernameAttr: name
        emailAttr: email
        # optional
        ssoIssuer: https://google-entity-id (e.g. https://accounts.google.com/o/saml2?idpid=Abcde0)
```

#### 参考资料

* [Dex SAML 连接器文档](https://dexidp.io/docs/connectors/saml/)
* [谷歌的 SAML 错误信息](https://support.google.com/a/answer/6301076?hl=en)

### 使用 Dex 的 OpenID Connect 加 Google 群组

---

警告 "组信息有限"

```
When using this feature you'll only receive the list of groups the user is a direct member.

So, lets say you have this hierarchy of groups and subgroups:  
`all@example.com --> tech@example.com --> devs@example.com --> you@example.com`  
The only group you would receive through Dex would be `devs@example.com`
```

---

我们将使用 Dex 的 `google` 连接器从用户那里获取更多 Google Groups 信息，从而可以在 RBAC 上使用组员资格，即赋予整个 `sysadmins@yourcompany.com` 组以 `admin` 角色。

该连接器被引用两种不同的凭证：

* oidc 客户端 ID 和 secret  
与配置[OpenID 连接](#openid-connect-using-dex)时一样，这将对用户进行身份验证
* 谷歌服务账户  
用于连接 Google 目录 API 并引用用户的 group 成员信息

此外，您还需要该域管理员用户的电子邮件地址。 Dex 将假冒该用户身份从 API 获取用户信息。

### 配置 OpenID Connect

除了配置 `argocd-cm` 之外，请执行与 [OpenID Connect using Dex](#openid-connect-using-dex) 相同的步骤。 我们稍后再做配置。

#### 设置目录 API 访问权限

1.按照[Google 说明创建全域授权服务账户](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)
    - 在为服务账户分配 API group 作用域时，***只能分配 `https://www.googleapis.com/auth/admin.directory.group.readonly` 作用域，不能分配其他任何作用域。如果分配了任何其他作用域，就无法从 API 中获取信息
    - 以 JSON 格式创建证书并将其存储在安全的地方，我们稍后会用到它们
2.启用 [Admin SDK](https://console.developers.google.com/apis/library/admin.googleapis.com/)

### 配置 Dex

1.用之前 json 文件的内容创建一个用 base64 编码的 secret，就像这样：
    ```
    apiVersion: v1
     kind: Secret
     元数据：
       name: argocd-google-groups-json
       namespace: argocd
     data：
       googleAuth.json：json_file_base64_encoded
    ```
2.编辑你的 `argocd-dex-server` 部署，将该秘密挂载为文件
    - 像这样在 `/spec/template/spec/containers/0/volumeMounts/` 中添加卷挂载。注意是编辑运行中的容器，而不是初始容器！
        ```
        volumeMounts：
          - mountPath：/shared
            名称： static-files
          - mountPath：/tmp
            名称： dexconfig
          - mountPath：/tmp/oidc
            名称： google-json
            只读：true
        ```
    - 像这样在 `/spec/template/spec/volumes/` 中添加一个卷：
        ```
        volumes：
          - emptyDir: {}
            name: static-files
          - emptyDir: {}
            名称: dexconfig
          - 名称： google-json
            secret：
              defaultMode：420
              secretName: argocd-google-groups-json
        ```
3.编辑 `argocd-cm` 并将以下 `dex.config` 添加到数据部分，将 `clientID` 和 `clientSecret` 替换为您之前保存的值，将 `adminEmail` 替换为您要冒充的管理员用户的地址，并将 `redirectURI` 替换为您的 Argo CD 域：
    ```
    dex.config：|
       connectors：
       - config：
           redirectURI: https://argocd.example.com/api/dex/callback
           clientID: XXXXXXXXXXXXX.apps.googleusercontent.com
           clientSecret: XXXXXXXXXXXX
           serviceAccountFilePath：/tmp/oidc/googleAuth.json
           adminEmail: admin-email@example.com
         类型：谷歌
         id: google
         名称： 谷歌
    ```
4.重启你的 `argocd-dex-server` 部署，确保它被引用了最新配置
5.登录 Argo CD，进入 "用户信息 "部分，在那里您应该可以看到您所在的组别

![用户信息](.../../assets/google-groups-membership.png) 6. 现在可以使用群组电子邮件地址授予 RBAC 权限了

#### 参考资料

* [Dex Google 连接器文档](https://dexidp.io/docs/connectors/google/)
