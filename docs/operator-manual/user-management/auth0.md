<!-- TRANSLATED by md-translate -->
# Auth0

## 用户定义

在 Auth0 中定义用户不在本指南的讨论范围之内。 直接在 Auth0 数据库中添加用户、使用企业注册表或 "社交登录"。 注意：除非通过配置限制访问权限，否则所有用户都可以访问所有 Auth0 定义的应用程序--如果 Argo 暴露在互联网上，或任何人都可以登录，请牢记这一点。

## 使用 Auth0 注册应用程序

按照 [register app](https://auth0.com/docs/dashboard/guides/applications/register-app-spa) 说明在 Auth0 中创建 argocd 应用程序：

* 注意 _clientId_ 和 _clientSecret_ 值。
* 将登录网址注册为 https://your.argoingress.address/login
* 将允许的回调网址设为 https://your.argoingress.address/auth/callback
* 在连接下，选择要与 Argo 一起使用的用户注册表。

任何其他设置都不是身份验证工作所必需的。

## 为 Auth0 添加授权规则

按照 Auth0 [授权指南](https://auth0.com/docs/authorization) 设置授权。这里需要注意的是，群成员资格是一个非标准的权利要求，因此需要放在一个 FQDN 权利要求名称下，例如 `http://your.domain/groups`。

## 配置 Argo

### 为 ArgoCD 配置 OIDC

kubectl edit configmap argocd-cm

```
...
data:
  application.instanceLabelKey: argocd.argoproj.io/instance
  url: https://your.argoingress.address
  oidc.config: |
    name: Auth0
    issuer: https://<yourtenant>.<eu|us>.auth0.com/
    clientID: <theClientId>
    clientSecret: <theClientSecret>
    requestedScopes:
    - openid
    - profile
    - email
    # not strictly necessary - but good practice:
    - 'http://your.domain/groups'
...
```

### 为 ArgoCD 配置 RBAC

`kubectl edit configmap argocd-rbac-cm`（或被引用 helm 值）。

```
...
data:
  policy.csv: |
    # let members with group someProjectGroup handle apps in someProject
    # this can also be defined in the UI in the group-definition to avoid doing it there in the configmap
    p, someProjectGroup, applications, *, someProject/*, allow
    # let the group membership argocd-admins from OIDC become role:admin - needs to go into the configmap
    g, argocd-global-admins, role:admin
  policy.default: role:readonly
  # essential to get argo to use groups for RBAC:
  scopes: '[http://your.domain/groups, email]' 
...
```

<br>

注意 "存储客户秘密" 有关安全、正确存储客户秘密的详细信息，请参阅 [用户管理概览] 页面（index.md#sensitive-data-and-sso-client-secrets）。