<!-- TRANSLATED by md-translate -->
# 钥匙斗篷

# 整合 Keycloak 和 ArgoCD

这些说明将引导您完成 ArgoCD 应用程序与 Keycloak 进行身份验证的整个过程。 您将在 Keycloak 中创建一个客户端，并配置 ArgoCD 使用 Keycloak 进行身份验证，使用 Keycloak 中设置的组来确定 Argo 中的权限。

## 在 Keycloak 中创建新客户端

首先，我们需要设置一个新客户端。 首先登录你的 keycloak 服务器，选择你要使用的领域（默认为 "master"），然后进入**客户端**，点击顶部的**创建客户端**按钮。

![Keycloak添加客户端](../../assets/keycloak-add-client.png "Keycloak 添加客户端")

启用**客户端验证**。

![Keycloak添加客户端步骤2]( .././assets/keycloak-add-client_2.png "Keycloak添加客户端步骤2")

通过将 **根 URL**、**网络起源**、**管理 URL** 设置为主机名 (https://{hostname})来配置客户端。

此外，您还可以将 ** 主页 URL** 设为 _/applications_ 路径，将 **Valid Post logout 重定向 URIs** 设为 "+"。

有效重定向 URI 应设置为 https://{hostname}/auth/callback（也可以设置安全性较低的 https://{hostname}/*，用于测试/开发目的，但不建议在生产中使用）。

![Keycloak配置客户端](../../assets/keycloak-configure-client.png "Keycloak 配置客户端")

确保点击**保存**。 应该会出现一个名为**凭证**的选项卡。 您可以复制我们在 ArgoCD 配置中将被引用的秘密。

![Keycloak 客户端秘密](../../assets/keycloak-client-secret.png "Keycloak 客户端秘密")

## 配置组索赔

为了让 ArgoCD 提供用户所在的群组，我们需要配置一个可包含在身份验证令牌中的群组 claims。 为此，我们将首先创建一个名为 _groups_ 的新 ** 客户端范围**。

![Keycloak添加范围](../../assets/keycloak-add-scope.png "Keycloak add scope")

创建客户端范围后，现在可以添加令牌映射器，当客户端请求群组范围时，该映射器会将群组权利要求添加到令牌中。 在 "映射器 "选项卡中，点击 "配置新映射器 "并选择 "群组成员 "**。 确保将**名称**和**令牌权利要求名称**设置为_groups_。 同时禁用 "完整群组路径"。

![Keycloak组映射器]( .././assets/keycloak-groups-mapper.png "Keycloak组映射器")

现在我们可以配置客户端以提供 _groups_ 作用域。 回到我们之前创建的客户端，进入 "客户端作用域 "选项卡。 点击 "添加客户端作用域"，选择 _groups_ 作用域并将其添加到 **Default** 或 **Optional** 客户端作用域中。 如果将其放在可选类别中，则需要确保 ArgoCD 在其 OIDC 配置中请求该作用域。 由于我们总是需要组信息，因此我建议使用默认类别。

![Keycloak客户端范围](../../assets/keycloak-client-scope.png "Keycloak客户端范围")

创建一个名为 _ArgoCDAdmins_ 的组，并让当前用户加入该组。

![Keycloak用户组](../../assets/keycloak-user-group.png "Keycloak用户组")

## 配置 ArgoCD OIDC

我们先把之前生成的客户端秘密存储在 argocd secret _argocd-secret_ 中。

1.首先，您需要用 base64 对客户秘密进行编码： `$ echo -n '83083958-8ec6-47b0-a411-a8c55381fbd2' | base64`.
2.然后，您可以使用 `$ kubectl edit secret argocd-secret` 编辑秘密，并将 base64 值添加到名为 _oidc.keycloak.clientSecret_ 的新密钥中。

您的 Secret 应该是这样的：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
data:
  ...
  oidc.keycloak.clientSecret: ODMwODM5NTgtOGVjNi00N2IwLWE0MTEtYThjNTUzODFmYmQy   
  ...
```

现在，我们可以配置 configmaps 并添加 oidc 配置，以启用 keycloak 身份验证。 可以使用 `$ kubectl edit configmap argocd-cm`。

您的 ConfigMap 应该是这样的：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  url: https://argocd.example.com
  oidc.config: |
    name: Keycloak
    issuer: https://keycloak.example.com/realms/master
    clientID: argocd
    clientSecret: $oidc.keycloak.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
```

确保

* **issuer** 以正确的 realm 结束（本例中为 _master_）。
* 在 Keycloak 版本大于 17 的版本中，**issuer** URL 必须包含 /auth（本例中为 /auth/realms/master）。
* **clientID** 设置为您在 Keycloak 中配置的客户端 ID
* **clientSecret** 指向您在 _argocd-secret_ Secret 中创建的正确密钥
* 如果没有添加到默认作用域，**requestedScopes** 包含 _groups_ claims

## 配置 ArgoCD 策略

现在我们有了一个提供组的身份验证，我们想对这些组应用策略。 我们可以使用 `$ kubectl edit configmap argocd-rbac-cm`修改 _argocd-rbac-cm_ ConfigMap。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data:
  policy.csv: |
    g, ArgoCDAdmins, role:admin
```

在本例中，我们将 _role:admin_ 角色赋予 _ArgoCDAdmins_ 组中的所有用户。

## 登录

您现在可以使用我们新的 Keycloak OIDC 身份验证登录：

![Keycloak ArgoCD login](../../assets/keycloak-login.png "Keycloak ArgoCD 登录")