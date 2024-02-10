<!-- TRANSLATED by md-translate -->
# 身份中心（AWS SSO）

!!! 注 "您正在使用吗？ 请贡献！" 如果您正在使用此 IdP，请考虑[贡献]（.../.../developer-guide/site.md）到此文档。

使用身份中心（AWS SSO）的单点登录配置已通过以下方法实现：

* [SAML（带 Dex）](#saml-with-dex)

## SAML（使用 Dex）

1.在身份中心创建新的 SAML 应用程序并下载证书。
    - 身份中心 SAML 应用程序 1](.../.../assets/identity-center-1.png)
    - 身份中心 SAML 应用程序 2](./../assets/identity-center-2.png)
2.在身份中心创建应用程序后点击 "分配用户"，选择您希望授予访问该应用程序权限的用户或用户组。
    - ![身份中心 SAML 应用程序 3](../../assets/identity-center-3.png)
3.将 Argo CD URL 复制到 ``argocd-cm` ConfigMap 中的 `data.url` 字段。
    ```
    数据：
       url: https://argocd.example.com
    ```
4.配置属性映射。
    注意："组属性映射并非官方支持！"
         在AWS文档中，组属性映射不被官方支持，但目前该变通方法是可行的。
    - ![Identity Center SAML App 5](../../assets/identity-center-5.png)

<!-- markdownlint-enable MD046 -->

5.下载要在 `argocd-cm` 配置中使用的 CA 证书。
    - 如果被引用 `caData` 字段，则需要对整个证书进行 base64 编码，包括 `-----BEGIN CERTIFICATE-----` 和 `-----END CERTIFICATE-----` 节（例如，`base64 my_cert.pem`）。
    - 如果使用`ca`字段并将CA证书作为秘密单独存储，则需要将该秘密挂载到`argocd-dex-server`部署中的`dex`容器上。
    - ![Identity Center SAML App 6](../../assets/identity-center-6.png)
6.编辑 `argocd-cm` 并配置 `data.dex.config` 部分：

<!-- markdownlint-disable MD046 -->

```yaml
dex.config: |
  logger:
    level: debug
    format: json
  connectors:
  - type: saml
    id: aws
    name: "AWS IAM Identity Center"
    config:
      # You need value of Identity Center APP SAML (IAM Identity Center sign-in URL)
      ssoURL: https://portal.sso.yourregion.amazonaws.com/saml/assertion/id
      # You need `caData` _OR_ `ca`, but not both.
      caData: <CA cert (IAM Identity Center Certificate of Identity Center APP SAML) passed through base64 encoding>
      # Path to mount the secret to the dex container
      entityIssuer: https://external.path.to.argocd.io/api/dex/callback
      redirectURI: https://external.path.to.argocd.io/api/dex/callback
      usernameAttr: email
      emailAttr: email
      groupsAttr: groups
```

<!-- markdownlint-enable MD046 -->

### 将身份中心组连接到 Argo CD 角色

Argo CD 可识别符合**组属性声明** regex的身份中心组中的用户成员身份。

在上面的示例中，被引用的 regex `argocd-*` 使 Argo CD 意识到一个名为 `argocd-admins` 的组。

修改 `argocd-rbac-cm` configmaps，将 `ArgoCD-administrators` Identity Center 组连接到内置的 Argo CD `admin` 角色。

<!-- markdownlint-disable MD046 -->

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data:
  policy.csv: |
    g, <Identity Center Group ID>, role:admin
  scopes: '[groups, email]'
```

<!-- markdownlint-enable MD046 -->