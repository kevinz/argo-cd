<!-- TRANSLATED by md-translate -->
# OneLogin

!!! 注 "您正在使用吗？ 请贡献！" 如果您正在使用此 IdP，请考虑[贡献]（.../.../developer-guide/site.md）到此文档。

<!-- markdownlint-disable MD033 -->

<div style="text-align:center"><img src="../../../assets/argo.png" /></div>
<!-- markdownlint-enable MD033 -->

# 整合 OneLogin 和 ArgoCD

这些说明将引导您完成 ArgoCD 应用程序与 OneLogin 进行身份验证的整个过程。 您将在 OneLogin 中创建一个自定义 OIDC 应用程序，并配置 ArgoCD 使用 OneLogin 进行身份验证，使用在 OneLogin 中设置的 UserRoles 来确定 Argo 中的权限。

## 创建和配置 OneLogin 应用程序

要使 ArgoCD 应用程序与 OneLogin 通信，首先需要在 OneLogin 端创建和配置 OIDC 应用程序。

### 创建 OIDC 应用程序

要创建应用程序，请执行以下操作：

1.导航至 OneLogin 门户，然后导航至管理工具 &gt; 应用程序。
2.单击 "添加应用程序"。
3.在搜索栏中搜索 "OpenID Connect"。
4.选择要创建的 "OpenId Connect (OIDC) "应用程序。
5.更新 "显示名称 "字段（可以是 "ArgoCD (Production) "之类的名称。
6.点击 "保存"。

### 配置 OIDC 应用程序设置

应用程序已创建，您可以配置应用程序的设置。

#### 配置选项卡

更新 "配置 "设置如下：

1.选择左侧的 "配置 "选项卡。
2.将 "Login Url"（登录网址）字段设为 https://argocd.myproject.com/auth/login，用自己的主机名代替。
3.将 "重定向网址 "字段设置为 https://argocd.myproject.com/auth/callback，并将主机名替换为您自己的主机名。
4.点击 "保存"。

注意 "在设置上述字段之前，OneLogin 可能不会让您保存任何其他字段"。

#### 信息选项卡

您可以在此更新 "显示名称"、"描述"、"备注 "或显示在 OneLogin 门户中的显示镜像。

#### 参数选项卡

此选项卡可控制在令牌中发送给 Argo 的信息。 默认情况下，它将包含一个组字段，"凭据是 "被设置为 "由管理员配置"。 请将 "凭据是 "保留为默认值。

如何配置 "组 "字段的 Values 将根据您的需要而有所不同，但要将 OneLogin 用户角色用于 ArgoCD 权限，请按以下方式配置 "组 "字段的 Values：

1.单击 "组"。出现一个模态。
2.将 "未选择值时的默认值 "字段设为 "用户角色"。
3.将转换字段（在其下方）设为 "分号分隔输入"。
4.点击 "保存"。

当用户尝试使用 OneLogin 登录 Argo 时，OneLogin 中的用户角色（如经理、产品团队和测试工程）将包含在令牌中的 Groups（组）字段中。 这些值是 Argo 分配权限所需的值。

令牌中的组字段将与下面的内容相似：

```
"groups": [
    "Manager",
    "ProductTeam",
    "TestEngineering",
  ],
```

#### 规则选项卡

要开始运行，您无需修改此处的任何设置。

#### SSO 标签

该选项卡包含需要放入 ArgoCD 配置文件的大部分信息（API 端点、客户端 ID、客户端秘密）。

确认 "应用程序类型 "设置为 "Web"。

确认 "令牌端点 "设置为 "基本"。

#### 访问选项卡

此选项卡控制谁可以在 OneLogin 门户中查看此应用程序。

选择希望访问此应用程序的角色，然后点击 "保存"。

#### 用户选项卡

此选项卡显示可访问此应用程序的单个用户（通常是在 "访问 "选项卡中指定角色的用户）。

要开始运行，您无需修改此处的任何设置。

#### 特权选项卡

此选项卡显示哪些 OneLogin 用户可以配置此应用程序。

要开始运行，您无需修改此处的任何设置。

## 在 ArgoCD 中更新 OIDC 配置

既然已在 OneLogin 中配置了 OIDC 应用程序，您就可以更新 Argo 配置，以便与 OneLogin 通信，并控制通过 OneLogin 验证的用户的权限。

### 告诉 Argo OneLogin 的位置

Argo 需要更新配置 maps (argocd-cm) 才能与 OneLogin 通信。 请看下面的 yaml：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
data:
  url: https://<argocd.myproject.com>
  oidc.config: |
    name: OneLogin
    issuer: https://<subdomain>.onelogin.com/oidc/2
    clientID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaaaaaaaa
    clientSecret: abcdef123456

    # Optional set of OIDC scopes to request. If omitted, defaults to: ["openid", "profile", "email", "groups"]
    requestedScopes: ["openid", "profile", "email", "groups"]
```

url "键的值应为 Argo 项目的主机名。

客户 ID "取自 OneLogin 应用程序的 SSO 选项卡。

签发人 "来自 OneLogin 应用程序的 SSO 标签，是签发人 api 端点之一。

clientSecret "值是位于 OneLogin 应用程序的 SSO 选项卡中的客户端秘密。

注意："如果您在尝试使用 OneLogin 进行身份验证时出现 "invalid_client "错误，则有可能是您的客户秘密不正确。 请记住，在以前的版本中，"clientSecret "值必须经过 base64 加密，但现在已不再需要。

### 配置 OneLogin 验证用户的权限

ArgoCD 中的权限可通过使用令牌中 Groups 字段传递的 OneLogin 角色名称来配置。 请考虑 argocd-rbac-cm.yaml 中的以下 yaml：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, get, *, allow
    p, role:org-admin, repositories, get, *, allow
    p, role:org-admin, repositories, create, *, allow
    p, role:org-admin, repositories, update, *, allow
    p, role:org-admin, repositories, delete, *, allow

    g, TestEngineering, role:org-admin
```

在 OneLogin 中，具有用户角色 "TestEngineering "的用户通过 OneLogin 登录 Argo 时，将获得 ArgoCD 管理权限。 所有其他用户将获得只读角色。 这里的关键是，"TestEngineering "通过令牌中的组字段传递（在 OneLogin 的 "参数 "选项卡中指定）。