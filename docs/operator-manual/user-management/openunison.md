<!-- TRANSLATED by md-translate -->
# OpenUnison

## 整合 OpenUnison 和 ArgoCD

这些说明将带您完成整合 OpenUnison 和 ArgoCD 的步骤，以支持单点登录，并在 OpenUnison 门户上添加 "徽章"，为 Kubernetes 和 ArgoCD 创建一个单一访问点。 这些说明假定您将同时使用 ArgoCD 的 Web 界面和命令行界面。这些说明假定您正在运行 [OpenUnison 1.0.20+](https://www.tremolosecurity.com/products/orchestra-for-kubernetes)。

![带有 ArgoCD 的 OpenUnison 门户网站](.../../assets/openunison-portal.png)

## 创建 OpenUnison 信托

更新下面的 `Trust` 对象，并将其添加到 `openunison` 名称空间。 唯一需要做的改动是将 `argocd.apps.domain.com` 替换为 ArgoCD URL 的主机名。 cli 需要本地主机 URL 才能工作。 ArgoCD 没有被引用客户端秘密，因为 cli 无法使用它。

```
apiVersion: openunison.tremolo.io/v1
kind: Trust
metadata:
  name: argocd
  namespace: openunison
spec:
  accessTokenSkewMillis: 120000
  accessTokenTimeToLive: 1200000
  authChainName: login-service
  clientId: argocd
  codeLastMileKeyName: lastmile-oidc
  codeTokenSkewMilis: 60000
  publicEndpoint: true
  redirectURI:
  - https://argocd.apps.domain.com/auth/callback
  - http://localhost:8085/auth/callback
  signedUserInfo: true
  verifyRedirect: true
```

## 在 OpenUnison 中创建 "徽章

下载[`PortalUrl`对象的 yaml](../../assets/openunison-argocd-url.yaml)并更新`url`以指向您的 ArgoCD 实例。 将更新后的`PortalUrl`添加到集群的`openunison`命名空间。

## 在 ArgoCD 中配置 SSO

接下来，更新 `argocd` namespace 中的 `argocd-cm` ConfigMap。 添加`url`和`oidc.config`部分，如下所示。 用 OpenUnison 的主机更新`issuer`。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  url: https://argocd.apps.domain.com
  oidc.config: |-
    name: OpenUnison
    issuer: https://k8sou.apps.192-168-2-144.nip.io/auth/idp/k8sIdp
    clientID: argocd
    requestedScopes: ["openid", "profile", "email", "groups"]
```

如果一切顺利，请登录 OpenUnison 实例，这时应该会出现 ArgoCD 的徽章。 点击该徽章，ArgoCD 就会在新窗口中打开，并已登录！此外，启动 argocd cli 工具将启动浏览器登录 OpenUnison。

## 配置 ArgoCD 策略

OpenUnison 将群组放在 "groups"（群组）声称中。 当您点击 ArgoCD 门户的用户信息部分时，这些声称将显示出来。 如果您使用 LDAP、Active Directory 或 Active Directory 联合服务，群组将以完整的区分名称 (DN) 提供给 ArgoCD。 由于 DN 包含逗号 (`,`)，您需要在策略中引用群组名称。 例如，将 "CN=k8s_login_cluster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com "指定为管理员的情况如下：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    g, "CN=k8s_login_cluster_admins,CN=Users,DC=ent2k12,DC=domain,DC=com", role:admin
```