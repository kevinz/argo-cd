<!-- TRANSLATED by md-translate -->
# Zitadel

还请查阅 [Zitadel 文档](https://zitadel.com/docs)。

## 整合 Zitadel 和 ArgoCD

您将在 Zitadel 中创建一个应用程序，并配置 ArgoCD 使用 Zitadel 进行身份验证，使用 Zitadel 中设置的角色来确定 ArgoCD 中的权限。

将 ArgoCD 与 Zitadel 整合需要以下步骤：

1.在 Zitadel 中创建新项目和新应用程序
2.在 Zitadel 中配置应用程序
3.在 Zitadel 中设置角色
4.在 Zitadel 中设置操作
5.配置 ArgoCD configmaps
6.测试设置

本例将被引用以下 Values 值：

* Zitadel FQDN：`auth.example.com`。
* Zitadel 项目： `argocd-project
* Zitadel 应用程序：`argocd-application
* Zitadel 操作：`groupsClaim
* ArgoCD FQDN：`argocd.example.com
* ArgoCD 管理员角色：`argocd_administrators
* ArgoCD 用户角色：`argocd_users

您可以在设置中选择不同的 Values；这些 Values 被引用来保持指南的一致性。

## 在 Zitadel 中设置项目和应用程序

首先，我们将在 Zitadel 中创建一个新项目。 进入**项目**，选择**创建新项目**。 现在您应该看到以下屏幕。

![Zitadel项目](../../assets/zitadel-project.png "Zitadel项目")

检查以下选项：

* 在身份验证时证明角色
* 在身份验证时检查授权

![Zitadel 项目设置](.../../assets/zitadel-project-settings.png "Zitadel 项目设置")

### 角色

转到**角色**，点击**新建**。 创建以下两个角色。 在**键**和**组**字段中都被引用指定值。

* 管理员
* 用户

您的角色现在应该是这样的：

![Zitadel 项目角色](../../assets/zitadel-project-roles.png "Zitadel 项目角色")

#### 授权

接下来，进入**授权**，为用户分配角色 "argocd_administrators"。 单击**新建**，输入用户名，然后单击**继续**。 选择角色 "argocd_administrators"，然后单击**保存**。

您的授权现在应该是这样的：

![Zitadel 项目授权](../../assets/zitadel-project-authorizations.png "Zitadel 项目授权")

### 创建应用程序

转到**常规**，创建一个新应用程序。 将应用程序命名为 "argocd-application"。

在应用程序类型中，选择 **WEB**，然后单击继续。

![Zitadel 应用程序设置步骤 1](../../assets/zitadel-application-1.png "Zitadel 应用程序设置步骤 1")

选择 **代码**，然后继续。

![Zitadel 应用程序设置步骤 2](../../assets/zitadel-application-2.png "Zitadel 应用程序设置步骤 2")

接下来，我们将设置重定向和注销后 URI。 设置以下 Values：

* 重定向 URI： `https://argocd.example.com/auth/callback`
* 注销后 URI： `https://argocd.example.com`

注销后 URI 为可选项，在示例设置中，用户注销后将返回 ArgoCD 登录页面。

![Zitadel 应用程序设置步骤 3](../../assets/zitadel-application-3.png "Zitadel 应用程序设置步骤 3")

在下一屏幕中确认配置，然后单击 **Create** 创建应用程序。

![Zitadel 应用程序设置步骤 4](../../assets/zitadel-application-4.png "Zitadel 应用程序设置步骤 4")

点击 **Create** 后，您将看到应用程序的 "ClientId "和 "ClientSecret"。 请务必复制 "ClientSecret"，因为关闭此窗口后将无法检索。 在我们的示例中，被引用的 Values 如下：

* ClientId：`227060711795262483@argocd-project`
* ClientSecret: `UGvTjXVFAQ8EkMv2x4GbPcrEwrJGWZ0sR2KbwHRNfYxeLsDurCiVEpa5bkgW0pl0`

![Zitadel 应用秘密]( ../../assets/zitadel-application-secrets.png "Zitadel 应用秘密")

将客户机密保存到安全位置后，单击**关闭**完成应用程序的创建。

进入**令牌设置**，启用以下选项：

* ID 令牌内的用户角色
* ID 令牌内的用户信息

Zitadel应用程序设置](.../../assets/zitadel-application-settings.png "Zitadel 应用程序设置")

## 在 Zitadel 中设置行动

要在 Zitadel 签发的令牌中包含用户的角色，我们需要设置一个 Zitadel 操作。 ArgoCD 中的授权将由授权令牌中包含的角色决定。 进入**操作**，点击**新建**，选择 "groupsClaim "作为操作的名称。

将以下代码粘贴到操作中：

```javascript
/**
 * sets the roles an additional claim in the token with roles as value an project as key
 *
 * The role claims of the token look like the following:
 *
 * // added by the code below
 * "groups": ["{roleName}", "{roleName}", ...],
 *
 * Flow: Complement token, Triggers: Pre Userinfo creation, Pre access token creation
 *
 * @param ctx
 * @param api
 */
function groupsClaim(ctx, api) {
  if (ctx.v1.user.grants === undefined || ctx.v1.user.grants.count == 0) {
    return;
  }

  let grants = [];
  ctx.v1.user.grants.grants.forEach((claim) => {
    claim.roles.forEach((role) => {
      grants.push(role);
    });
  });

  api.v1.claims.setClaim("groups", grants);
}
```

选中**允许失败**，然后单击**添加**添加您的操作。

_注意：如果未选中 __允许失败_*，且用户未指定角色，则用户可能无法再登录 Zitadel，因为当操作失败时，登录流程会失败。

接下来，将操作添加到**补充令牌**流程中。 从下拉菜单中选择**补充令牌**流程，然后点击**添加触发器**。 将操作添加到**创建用户信息前**和**创建访问令牌前**这两个触发器中。

您的 "操作 "页面现在应该与下面的截图相似：

![Zitadel行动](../../assets/zitadel-actions.png "Zitadel行动")

## 配置 ArgoCD configmaps

接下来，我们将配置两个 ArgoCD configmaps：

* [argocd-cm.yaml](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-cm.yaml)
* [argocd-rbac-cm.yaml](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-rbac-cm.yaml)

请按以下步骤配置您的 configmaps，同时确保将`url`、`issuer`、`clientID`、`clientSecret` 和`logoutURL`等相关值替换为与您的设置相匹配的值。

### argocd-cm.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
data:
  admin.enabled: "false"
  url: https://argocd.example.com
  oidc.config: |
    name: Zitadel
    issuer: https://auth.example.com
    clientID: 227060711795262483@argocd-project
    clientSecret: UGvTjXVFAQ8EkMv2x4GbPcrEwrJGWZ0sR2KbwHRNfYxeLsDurCiVEpa5bkgW0pl0
    requestedScopes:
      - openid
      - profile
      - email
      - groups
    logoutURL: https://auth.example.com/oidc/v1/end_session
```

### argocd-rbac-cm.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
data:
  scopes: '[groups]'
  policy.csv: |
    g, argocd_administrators, role:admin
    g, argocd_users, role:readonly
  policy.default: ''
```

policy.csv "中指定的角色必须与 Zitadel 中配置的角色相匹配。 Zitadel 角色 "argocd_administrators "将被指定为 ArgoCD 角色 "admin"，允许管理员访问 ArgoCD。 Zitadel 角色 "argocd_users "将被指定为 ArgoCD 角色 "readonly"，允许只读访问 ArgoCD。

部署您的 ArgoCD configmaps。 ArgoCD 和 Zitadel 现在应该已正确设置，允许用户使用 Zitadel 登录 ArgoCD。

## 测试设置

进入 ArgoCD 实例，在通常的用户名/密码登录界面上方，您会看到 "用 ZITADEL 登录"（**LOG INH WITH ZITADEL**）按钮。

![Zitadel ArgoCD Login](../../assets/zitadel-argocd-login.png "Zitadel ArgoCD Login")

使用 Zitadel 用户登录后，转到 ** 用户信息**。 如果一切设置正确，你现在应该可以看到`argocd_administrators`组，如下图所示。

![Zitadel ArgoCD 用户信息]( .../../assets/zitadel-argocd-user-info.png "Zitadel ArgoCD 用户信息")