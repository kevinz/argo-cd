<!-- TRANSLATED by md-translate -->
# 概览

Argo CD 安装后有一个内置的 "admin "用户，可以完全访问系统。 建议仅在初始配置时使用 "admin "用户，然后切换到本地用户或配置 SSO 集成。

## 本地用户/账户

本地用户/账户功能有两个主要用途：

* 用于 Argo CD 管理自动化的认证令牌。可以配置具有有限权限的 API 账户，并生成认证令牌。

这种令牌可被用来自动创建应用程序、项目等。

* 为规模很小的团队提供额外用户，在这种情况下，使用 SSO 集成可能会被认为是矫枉过正。本地用户不提供群组等高级功能、

因此，如果需要这些功能，强烈建议使用 SSO。

注意 在创建本地用户时，每个用户都需要设置额外的 [RBAC 规则](../rbac.md)，否则他们将退回到 `argocd-rbac-cm` 配置地图的 `policy.default` 字段指定的默认策略。

本地账户用户名的最大长度为 32。

#### 创建新用户

新用户应在 `argocd-cm` configMap 中定义：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # add an additional local user with apiKey and login capabilities
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.alice: apiKey, login
  # disables user. User is enabled by default
  accounts.alice.enabled: "false"
```

每个用户可能有两种能力：

* apiKey - 允许为访问 API 生成身份验证令牌
* 登录 - 允许使用用户界面登录

### 删除用户

要删除用户，必须删除在 `argocd-cm` configMap 中定义的相应条目：

例如

```bash
kubectl patch -n argocd cm argocd-cm --type='json' -p='[{"op": "remove", "path": "/data/accounts.alice"}]'
```

建议同时删除 `argocd-secret` Secret 中的密码条目：

例如

```bash
kubectl patch -n argocd secrets argocd-secret --type='json' -p='[{"op": "remove", "path": "/data/accounts.alice.password"}]'
```

### 禁用管理员用户

一旦创建了其他用户，建议立即禁用 `admin` 用户：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  admin.enabled: "false"
```

### 管理用户

Argo CD CLI 提供了一套设置用户密码和生成令牌的命令。

* 获取完整的用户列表

```bash
argocd account list
```

* 获取特定用户的详细信息

```bash
argocd account get --account <username>
```

* 设置用户密码

```bash
# if you are managing users as the admin user, <current-user-password> should be the current admin password.
argocd account update-password \
  --account <name> \
  --current-password <current-user-password> \
  --new-password <new-user-password>
```

* 生成授权令牌

```bash
# if flag --account is omitted then Argo CD generates token for current user
argocd account generate-token --account <username>
```

### 登录失败率限制

Argo CD 会在登录失败次数过多时拒绝登录尝试，以防止密码暴力破解。 以下环境变量可用于控制节流设置：

* argocd_session_failure_max_fail_count`：Argo CD 启动前登录失败的最大次数

拒绝登录尝试。 默认值：5。

* argocd_session_failure_window_seconds`：故障窗口的秒数。

默认值：300（5 分钟）。 如果设置为 0，则会禁用失败窗口，登录尝试会在连续 10 次登录失败后被拒绝，无论失败发生在什么时间段。

* argocd_session_max_cache_size"：中允许的最大条目数

缓存。 默认值：1000

* argocd_max_concurrent_login_requests_count`：限制最大并发登录请求数。

默认值：50。

## SSO

SSO 有两种配置方式：

* [捆绑的 Dex OIDC Provider](#dex) - 如果您当前的 Provider 不支持 OIDC（如 SAML、
LDAP），或希望利用 Dex 连接器的任何功能（如将 GitHub
组织和团队映射到 OIDC 组索赔的功能）。Dex 还直接支持 OIDC，并能在群组声明时从身份提供商处获取用户信息。
当组无法包含在 IDToken 中时，可从身份提供商处获取用户信息。
* [现有 OIDC 提供商](#existing-oidc-provider) - 如果您已经有正在使用的 OIDC 提供商（例如
[Okta]（okta.md）、[OneLogin]（onelogin.md）、[Auth0]（auth0.md）、[Microsoft]（microsoft.md）、[Keycloak]（keycloak.md）、
[Google (G Suite)](google.md)) ，在这里你可以管理用户、群组和成员资格。

## Dex

Argo CD 在安装时嵌入并捆绑了 [Dex](https://github.com/dexidp/dex)，目的是将身份验证委托给外部身份提供者。 支持多种类型的身份提供者（OIDC、SAML、LDAP、GitHub 等......）。Argo CD 的 SSO 配置需要使用 [Dex connector](https://dexidp.io/docs/connectors/) 设置编辑 `argocd-cm` ConfigMap。

本文档以 GitHub（OAuth2）为例，介绍了如何配置 Argo CD SSO，但其他身份 Provider 的步骤也应类似。

#### 1.在身份 Providers 中注册应用程序

在 GitHub 注册一个新应用程序，回调地址应为 Argo CD URL 的 `/api/dex/callback` 端点（例如 `https://argocd.example.com/api/dex/callback`）。

![注册 OAuth 应用程序](.../.../assets/register-app.png "注册 OAuth 应用程序")

注册应用程序后，您将收到一个 OAuth2 客户端 ID 和 secrets。 这些值将被输入 Argo CD configmaps。

![OAuth2 客户端配置](../../assets/oauth2-config.png "OAuth2 客户端配置")

#### 2. 为 SSO 配置 Argo CD

编辑 argocd-cm configmap：

```bash
kubectl edit configmap argocd-cm -n argocd
```

* 在 `url` 键中，输入 Argo CD 的基本 URL。本例中为 `https://argocd.example.com` 。
* 在 `dex.config` 键中，将 `github` 连接器添加到 `connectors` 子字段。有关字段的解释，请参阅 Dex 的 [GitHub 连接器](https://github.com/dexidp/website/blob/main/content/docs/connectors/github.md) 文档。最低配置应填入在步骤 1 中生成的 clientID 和 clientSecret。
* 你很可能想限制登录到一个或多个 GitHub 组织。在 `connectors.config.orgs` 列表中，添加一个或多个 GitHub 组织。这样，该组织的任何成员都能登录 Argo CD 执行管理任务。

```yaml
data:
  url: https://argocd.example.com

  dex.config: |
    connectors:
      # GitHub example
      - type: github
        id: github
        name: GitHub
        config:
          clientID: aabbccddeeff00112233
          clientSecret: $dex.github.clientSecret # Alternatively $<some_K8S_secret>:dex.github.clientSecret
          orgs:
          - name: your-github-org

      # GitHub enterprise example
      - type: github
        id: acme-github
        name: Acme GitHub
        config:
          hostName: github.acme.example.com
          clientID: abcdefghijklmnopqrst
          clientSecret: $dex.acme.clientSecret  # Alternatively $<some_K8S_secret>:dex.acme.clientSecret
          orgs:
          - name: your-github-org
```

保存后，更改将自动生效。

备注：

* 无需如 dex 文档所示在 `connectors.config` 中设置 `redirectURI`。Argo CD 会自动为任何 OAuth2 连接器引用正确的 `redirectURI` 以匹配正确的外部回调 URL（例如 `https://argocd.example.com/api/dex/callback`）。
* 当使用自定义秘密（例如上面的 `some_K8S_secret`）时，它 _must_ 必须有标签 `app.kubernetes.io/part-of: argocd`。

## 使用 DEX 进行 OIDC 配置

Dex 可用于 OIDC 身份验证，而不是直接用于 ArgoCD。 这提供了一套单独的功能，如从 `UserInfo` 端点获取信息和[联合令牌](https://dexidp.io/docs/custom-scopes-claims-clients/#cross-client-trust-and-authorized-party)

#### 配置：

* 在 `argocd-cm` ConfigMap 中，将 `OIDC` 连接器添加到 `dex.config` 中的 `connectors` 子字段。

请参阅 Dex 的 [OIDC connect 文档](https://dexidp.io/docs/connectors/oidc/)，了解还有哪些配置选项可能有用。我们将在此引用最低配置。

* 发行商 URL 应是 Dex 与 OIDC Provider 对话的地址。通常会有一个

该 URL 下的 `.well-known/openid-configuration` 包含有关 Provider 支持内容的信息，例如 https://accounts.google.com/.well-known/openid-configuration。

```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: oidc
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
```

### 申请额外的身份令牌索赔

默认情况下，Dex 只检索个人资料和电子邮件范围。 要检索更多索赔，可以在 Dex 配置中的 "范围 "条目下添加索赔。 要通过 Dex 启用组索赔，还需要启用 "insecureEnableGroups"。 组信息目前只在身份验证时刷新，支持更动态地刷新组信息可在此处跟踪：[dexidp/dex#1065](https://github.com/dexidp/dex/issues/1065)。

```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: OIDC
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
          insecureEnableGroups: true
          scopes:
          - profile
          - email
          - groups
```

警告 由于组信息只在身份验证时刷新，因此在用户重新身份验证之前，从组中添加或删除帐户不会改变用户的成员资格。 这可能是一个安全风险，可根据贵组织的需要，通过更改身份验证令牌的有效期来降低风险。

### 检索不在令牌中的 claims

当 Idp 不支持或无法支持 IDToken 中的某些 claims 时，可使用 UserInfo 端点单独检索这些 claims。 Dex 使用 `getUserInfo` 端点支持此功能。 IDToken 中不支持的最常见的 claims 之一是 `groups` claim，且 `getUserInfo` 和 `insecureEnableGroups` 必须都被引用为 true。

```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: OIDC
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
          insecureEnableGroups: true
          scopes:
          - profile
          - email
          - groups
          getUserInfo: true
```

## 现有 OIDC 提供商

要配置 Argo CD 将身份验证委托给现有的 OIDC Provider，请将 OAuth2 配置添加到 `oidc.config` 键下的 `argocd-cm` ConfigMap 中：

```yaml
data:
  url: https://argocd.example.com

  oidc.config: |
    name: Okta
    issuer: https://dev-123456.oktapreview.com
    clientID: aaaabbbbccccddddeee
    clientSecret: $oidc.okta.clientSecret

    # Optional list of allowed aud claims. If omitted or empty, defaults to the clientID value above (and the 
    # cliClientID, if that is also specified). If you specify a list and want the clientID to be allowed, you must 
    # explicitly include it in the list.
    # Token verification will pass if any of the token's audiences matches any of the audiences in this list.
    allowedAudiences:
    - aaaabbbbccccddddeee
    - qqqqwwwweeeerrrrttt

    # Optional. If false, tokens without an audience will always fail validation. If true, tokens without an audience 
    # will always pass validation.
    # Defaults to true for Argo CD < 2.6.0. Defaults to false for Argo CD >= 2.6.0.
    skipAudienceCheckWhenTokenHasNoAudience: true

    # Optional set of OIDC scopes to request. If omitted, defaults to: ["openid", "profile", "email", "groups"]
    requestedScopes: ["openid", "profile", "email", "groups"]

    # Optional set of OIDC claims to request on the ID token.
    requestedIDTokenClaims: {"groups": {"essential": true}}

    # Some OIDC providers require a separate clientID for different callback URLs.
    # For example, if configuring Argo CD with self-hosted Dex, you will need a separate client ID
    # for the 'localhost' (CLI) client to Dex. This field is optional. If omitted, the CLI will
    # use the same clientID as the Argo CD server
    cliClientID: vvvvwwwwxxxxyyyyzzzz

    # PKCE authentication flow processes authorization flow from browser only - default false
    # uses the clientID
    # make sure the Identity Provider (IdP) is public and doesn't need clientSecret
    # make sure the Identity Provider (IdP) has this redirect URI registered: https://argocd.example.com/pkce/verify
    enablePKCEAuthentication: true
```

注意 回调地址应为 Argo CD URL 的 /auth/callback 端点（如 https://argocd.example.com/auth/callback）。

### 申请额外的身份令牌索赔

并非所有 OIDC Provider 都支持特殊的 "组 "范围。 例如，Okta、OneLogin 和 Microsoft 确实支持特殊的 "组 "范围，并将使用默认的 "requestedScopes "返回组成员信息。

如果明确提出请求，其他 OIDC 提供商可能会返回带有群组成员资格的 claims。 可以使用 `requestedIDTokenClaims` 请求单个 claims，详情请参见 [OpenID Connect Claims Parameter](https://connect2id.com/products/server/docs/guides/requesting-openid-claims#claims-parameter)。 Argo CD 对 claims 的配置如下：

```yaml
oidc.config: |
    requestedIDTokenClaims:
      email:
        essential: true
      groups:
        essential: true
        value: org:myorg
      acr:
        essential: true
        values:
        - urn:mace:incommon:iap:silver
        - urn:mace:incommon:iap:bronze
```

一个简单的例子可以是

```yaml
oidc.config: |
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

### 不在令牌中时检索组索赔

有些 OIDC 提供商即使使用 `requestedIDTokenClaims` 设置明确请求，也不会在 ID 令牌中返回用户的群组信息（例如 Okta）。 它们会在用户信息端点上提供群组信息。 通过以下配置，Argo CD 可在登录时查询用户信息端点，以获取用户的群组信息：

```yaml
oidc.config: |
    enableUserInfoGroups: true
    userInfoPath: /userinfo
    userInfoCacheExpiration: "5m"
```

**注意：如果省略了 `userInfoCacheExpiration` 设置，或者该设置大于 ID 令牌的有效期，则只要 ID 令牌有效，argocd-server 就会缓存组信息！ **

###为 OIDC Provider 配置自定义注销 URL

另外，如果您的 OIDC Provider 提供注销 API，而您又希望配置自定义注销 URL，以便在注销后使任何活动会话失效，则可按如下方式指定该 URL：

```yaml
oidc.config: |
    name: example-OIDC-provider
    issuer: https://example-OIDC-provider.example.com
    clientID: xxxxxxxxx
    clientSecret: xxxxxxxxx
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
    logoutURL: https://example-OIDC-provider.example.com/logout?id_token_hint={{token}}
```

默认情况下，这将在注销后将用户带至其 OIDC Provider 的登录页面。 如果还希望在注销后将用户重定向回 Argo CD，可按如下方式指定注销 URL：

```yaml
...
    logoutURL: https://example-OIDC-provider.example.com/logout?id_token_hint={{token}}&post_logout_redirect_uri={{logoutRedirectURL}}
```

您无需指定注销重定向 URL，因为 ArgoCD 会自动将其生成为您的基本 ArgoCD 网址 + 根路径

注：注销后重定向 URI 可能需要根据 OIDC Provider 的 ArgoCD 客户端设置列入白名单。

### 配置用于与 OIDC Provider 通信的自定义根 CA 证书

如果您的 OIDC 提供商使用的证书不是由知名证书颁发机构签署的，您可以提供自定义证书，该证书将在与 OIDC 提供商通信时用于验证其 TLS 证书。 在您的 `oidc.config` 中添加`rootCA`，其中包含 PEM 编码的根证书：

```yaml
oidc.config: |
    ...
    rootCA: |
      -----BEGIN CERTIFICATE-----
      ... encoded certificate data here ...
      -----END CERTIFICATE-----
```

## SSO 延伸阅读

#### 敏感数据和 SSO 客户端秘密

`argocd-secret` 可用于存储可被 ArgoCD 引用的敏感数据。 configmaps 中以 `$` 开头的值解释如下：

* 如果 Values 的形式为`$<secret>:a.key.in.k8s.secret` 则查找名称为 `<secret>`（去掉 `$`）的 k8s secret，并读取其值。
* 否则，在 k8s secret 中查找名为 `argocd-secret` 的密钥。

#### 示例

因此，SSO 的 "客户端秘密 "可以存储为 Kubernetes 的秘密，配置清单如下

`argocd-secret`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  ...
  # The secret value must be base64 encoded **once** 
  # this value corresponds to: `printf "hello-world" | base64`
  oidc.auth0.clientSecret: "aGVsbG8td29ybGQ="
  ...
```

argocd-cm`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  ...
  oidc.config: |
    name: Auth0
    clientID: aabbccddeeff00112233

    # Reference key in argocd-secret
    clientSecret: $oidc.auth0.clientSecret
  ...
```

#### 替代方案

如果您想将敏感数据存储在另一个*** Kubernetes `Secret`中，而不是`argocd-secret`中，ArgoCD知道，只要配置maps中的值以`$`开头，然后是您的Kubernetes `Secret`名称和`:`（冒号），ArgoCD就会检查您的Kubernetes `Secret`中`data`下的键是否有相应的键。

语法： `$<k8s_secret_name>:<a_key_in_that_k8s_secret>`

&gt; 注意：Secret 必须带有标签`app.kubernetes.io/part-of:argocd`。

##### 示例

`another-secret`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: another-secret
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  ...
  # Store client secret like below.
  # Ensure the secret is base64 encoded
  oidc.auth0.clientSecret: <client-secret-base64-encoded>
  ...
```

argocd-cm`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  ...
  oidc.config: |
    name: Auth0
    clientID: aabbccddeeff00112233
    # Reference key in another-secret (and not argocd-secret)
    clientSecret: $another-secret:oidc.auth0.clientSecret  # Mind the ':'
  ...
```

### 跳过 OIDC Providers 连接上的证书验证

默认情况下，API 服务器与 OIDC Provider（外部 Provider 或捆绑的 Dex 实例）建立的所有连接都必须通过证书验证。 这些连接发生在获取 OIDC Provider 的知名配置、获取 OIDC Provider 的密钥以及交换授权码或验证 ID 令牌（作为 OIDC 登录流程的一部分）时。

在下列情况下，禁用证书验证可能是合理的

* 您正在使用捆绑的 Dex 实例 ****您的 Argo CD 实例使用自签名证书配置了 TLS ****您理解并接受跳过 OIDC 提供商证书验证的风险。
* 您正在使用外部 OIDC Providers ***该 Providers 使用了无效证书 ***您无法通过设置 `oidcConfig.rootCA` 解决问题 ***您理解并接受跳过 OIDC Providers 证书验证的风险。

如果上述两种情况之一适用，则可以通过在 `argocd-cm` ConfigMap 中将 `oidc.tls.insecure.skip.verify` 设置为 `"true"，禁用 OIDC Provider 证书验证。