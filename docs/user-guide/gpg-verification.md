<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# GnuPG 签名验证

## 概览

从 v1.7 开始，ArgoCD 可以配置为只与 Git 中使用 GnuPG 签名的提交同步。 签名验证在项目级别进行配置。

如果某个项目被配置为强制签名验证，则与该项目相关的所有应用程序都必须在源代码库中使用 ArgoCD 已知的 GnuPG 公钥对提交内容进行签名。 ArgoCD 将拒绝同步任何未使用配置密钥进行有效签名的修订版本。

默认情况下，签名验证是启用的，但并不强制执行。 如果您希望完全禁用 ArgoCD 中的 GnuPG 功能，则必须设置环境变量`ARGOCD_GPG_ENABLED`至`"false"`的 pod 模板中的`argocd-server`,`argocd-repo-server`和`argocd-application-controller`部署清单。

GnuPG 签名验证仅被 Git 软件源引用，无法使用 Helm 软件源。

注意 "关于信任的几句话" ArgoCD 对您被引用的密钥使用非常简单的信任模型：一旦密钥被导入，ArgoCD 就会信任它。 ArgoCD 不支持更复杂的信任模型，也没有必要（也不可能）对您要导入 ArgoCD 的公钥进行签名。

## 签名验证目标

如果执行签名验证，ArgoCD 将使用以下策略验证签名：

* 如果 `target revision` 是指向提交对象的指针（即分支名称、引用名称（如 `HEAD` 或提交 SHA）），ArgoCD 将对该名称指向的提交对象（即提交）执行签名验证。 如果 `target revision` 解析到一个标签，且该标签是一个轻量级标签，则行为与target revision` 指向提交对象的指针相同。 但是，如果该标签是注释的，则 target revision 将指向 _tag_ 对象，因此签名验证将在标签对象上执行，即标签本身必须签名（使用 `git tags`）。

## 执行签名验证

要配置执行签名验证，必须执行以下步骤：

* 在 ArgoCD 中导入被引用用于签名提交的 GnuPG 公钥 * 配置项目以强制对给定密钥进行签名验证

一旦为某个项目配置了一个或多个验证所需的密钥，则与此项目相关的所有应用程序都会启用强制执行。

警告 如果执行签名验证，您将无法从本地来源同步（即`argocd app sync --local`）了。

## 管理 GnuPG 密钥的 RBAC 规则

Argo CD 的 RBAC 实现允许管理 GnuPG 密钥的适当资源符号是`gpgkeys`。

允许列出名为`role:myrole`, 被引用：

```
p, role:myrole, gpgkeys, get, *, allow
```

要允许为名为`role:myrole`, 被引用：

```
p, role:myrole, gpgkeys, create, *, allow
```

最后，允许删除名为`role:myrole`, 被引用：

```
p, role:myrole, gpgkeys, delete, *, allow
```

## 导入 GnuPG 公钥

您可以使用 CLI、Web UI 或声明式设置配置 ArgoCD 用于验证提交签名的 GnuPG 公钥。

注意 在导入 GnuPG 密钥后，即使已按配置列出，也可能需要一段时间才能在集群内传播该密钥。 如果仍无法同步到已导入密钥签名的提交，请参阅下面的故障排除部分。

要管理 GnuPG 公钥配置的用户需要具备以下 RBAC 权限`gpgkeys`资源

##使用 CLI 管理公钥

要使用 CLI 配置 GnuPG 公钥，请使用`argocd gpg`指挥。

#### 列出所有已配置的密钥

要列出 ArgoCD 已知的所有配置密钥，请使用`argocd gpg list`子命令：

```bash
argocd gpg list
```

#### 显示有关某个键的信息

要获取特定密钥的信息，请使用`argocd gpg get`子命令：

```bash
argocd gpg get <key-id>
```

#### 导入密钥

要将新密钥引用到 ArgoCD，请使用`argocd gpg add`子命令：

```bash
argocd gpg add --from <path-to-key>
```

要导入的密钥可以是二进制格式，也可以是 ascii 加密格式。

#### 从配置中删除密钥

要从配置中删除先前配置的密钥，请使用`argocd gpg rm`子命令：

```bash
argocd gpg rm <key-id>
```

###使用网络用户界面管理公钥

用于列出、导入和删除 GnuPG 公钥的基本密钥管理功能在 Web UI 中实现。**设置**页面中的**GnuPG 密钥**模块。

请注意，使用 Web UI 配置密钥时，密钥必须暂时以 ASCII 装甲格式引用。

#### 在声明式设置中管理公钥

ArgoCD 在内部将公钥存储在`argocd-gpg-keys-cm`ConfigMap 资源，以 GnuPG 公钥的 ID 作为名称，以 ASCII 加密密钥数据作为字符串值，即 GitHub 的网络流签名密钥条目如下所示： GnuPG 公钥的 ID 作为名称，以 ASCII 加密密钥数据作为字符串值，即 GitHub 的网络流签名密钥条目如下所示

```yaml
4AEE18F83AFDEB23: |
    -----BEGIN PGP PUBLIC KEY BLOCK-----

    mQENBFmUaEEBCACzXTDt6ZnyaVtueZASBzgnAmK13q9Urgch+sKYeIhdymjuMQta
    x15OklctmrZtqre5kwPUosG3/B2/ikuPYElcHgGPL4uL5Em6S5C/oozfkYzhwRrT
    SQzvYjsE4I34To4UdE9KA97wrQjGoz2Bx72WDLyWwctD3DKQtYeHXswXXtXwKfjQ
    7Fy4+Bf5IPh76dA8NJ6UtjjLIDlKqdxLW4atHe6xWFaJ+XdLUtsAroZcXBeWDCPa
    buXCDscJcLJRKZVc62gOZXXtPfoHqvUPp3nuLA4YjH9bphbrMWMf810Wxz9JTd3v
    yWgGqNY0zbBqeZoGv+TuExlRHT8ASGFS9SVDABEBAAG0NUdpdEh1YiAod2ViLWZs
    b3cgY29tbWl0IHNpZ25pbmcpIDxub3JlcGx5QGdpdGh1Yi5jb20+iQEiBBMBCAAW
    BQJZlGhBCRBK7hj4Ov3rIwIbAwIZAQAAmQEH/iATWFmi2oxlBh3wAsySNCNV4IPf
    DDMeh6j80WT7cgoX7V7xqJOxrfrqPEthQ3hgHIm7b5MPQlUr2q+UPL22t/I+ESF6
    9b0QWLFSMJbMSk+BXkvSjH9q8jAO0986/pShPV5DU2sMxnx4LfLfHNhTzjXKokws
    +8ptJ8uhMNIDXfXuzkZHIxoXk3rNcjDN5c5X+sK8UBRH092BIJWCOfaQt7v7wig5
    4Ra28pM9GbHKXVNxmdLpCFyzvyMuCmINYYADsC848QQFFwnd4EQnupo6QvhEVx1O
    j7wDwvuH5dCrLuLwtwXaQh0onG4583p0LGms2Mf5F+Ick6o/4peOlBoZz48=
    =Bvzs
    -----END PGP PUBLIC KEY BLOCK-----
```

## 配置项目以执行签名验证

将 GnuPG 密钥导入 ArgoCD 后，现在必须对项目进行配置，以便使用导入的密钥强制验证提交签名。

###使用 CLI 进行配置

#### 在允许的密钥列表中添加密钥 id

要将密钥 ID 添加到项目允许使用的 GnuPG 密钥列表中，可以使用`argocd proj add-signature-key`命令，即以下命令将添加密钥 ID`4AEE18F83AFDEB23`到名为`myproj`：

```bash
argocd proj add-signature-key myproj 4AEE18F83AFDEB23
```

#### 从允许密钥列表中删除密钥 id

同样，您可以使用`argocd proj remove-signature-key`命令，即从项目`myproj`, 使用该命令： 删除-signature-key`命令。

```bash
argocd proj remove-signature-key myproj 4AEE18F83AFDEB23
```

#### 显示项目允许的密钥 ID

要查看特定项目允许使用哪些密钥 ID，可以检查`argocd proj get`命令，即对于名为`gpg`：

```bash
$ argocd proj get gpg
Name:                        gpg
Description:                 GnuPG verification
Destinations:                *,*
Repositories:                *
Allowed Cluster Resources:   */*
Denied Namespaced Resources: <none>
Signature keys:              4AEE18F83AFDEB23, 07E34825A909B250
Orphaned Resources:          disabled
```

#### 覆盖密钥 ID 列表

您还可以通过使用`argocd proj set`命令与`--signature-keys`标志，可以用它来指定一个逗号分隔的允许密钥 ID 列表：

```bash
argocd proj set myproj --signature-keys 4AEE18F83AFDEB23,07E34825A909B250
```

--签名密钥`标记也可在创建项目时被引用，即`argocd proj create`指挥。

###使用 Web UI 进行配置

您可以使用 Web UI 在项目配置中配置签名验证所需的 GnuPG 密钥 ID。 导航至***设置**页面，然后选择***项目***模块，然后单击要配置的项目。

在项目详细信息页面，点击***编辑***并找到*** 所需的签名密钥***部分，您可以在此添加或删除用于签名验证的密钥 id。 修改项目后，单击***更新**保存更改。

###使用声明式设置进行配置

您可以在项目清单的`signatureKeys`节，即

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gpg
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  description: GnuPG verification
  destinations:
  - namespace: '*'
    server: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
  signatureKeys:
  - keyID: 4AEE18F83AFDEB23
  sourceRepos:
  - '*'
```

signatureKeys`是一个`SignatureKey`对象，其唯一属性是`keyID`现在

## 疑难解答

### 禁用该功能

如果需要，可以完全禁用 GnuPG 功能。 要禁用它，请设置环境变量`ARGOCD_GPG_ENABLED`至`false`的 pod 模板的`argocd-server`,`argocd-repo-server`和`argocd-application-controller`部署。

重新启动 pod 后，GnuPG 功能将被禁用。

### GnuPG 钥匙圈

用于签名验证的 GnuPG 密钥环维护在`argocd-repo-server`密钥环中的密钥与存储在`argocd-gpg-keys-cm`ConfigMap 资源，该资源的卷挂载到`argocd-repo-server`豆荚。

注意 pod 中的 GnuPG 密钥环是瞬时的，每次重启 pod 时都会根据配置重新创建。 千万不要手动添加或删除密钥环中的密钥，因为您所做的更改会丢失。 此外，密钥环中的任何私钥都是瞬时的，每次重启时都会重新生成。 私钥仅被用于为运行中的 pod 建立信任 DB。

要检查密钥是否真正同步，可以`kubectl exec`进入版本库服务器的 pod，检查位于路径`/app/config/gpg/keys`中。

```bash
$ kubectl exec -it argocd-repo-server-7d6bdfdf6d-hzqkg bash
argocd@argocd-repo-server-7d6bdfdf6d-hzqkg:~$ GNUPGHOME=/app/config/gpg/keys gpg --list-keys
/app/config/gpg/keys/pubring.kbx
--------------------------------
pub rsa2048 2020-06-15 [SC] [expires: 2020-12-12]
      D48F075D818A813C436914BC9324F0D2144753B1
uid           [ultimate] Anon Ymous (ArgoCD key signing key) <noreply@argoproj.io>

pub rsa2048 2017-08-16 [SC]
      5DE3E0509C47EA3CF04A42D34AEE18F83AFDEB23
uid           [ultimate] GitHub (web-flow commit signing) <noreply@github.com>

argocd@argocd-repo-server-7d6bdfdf6d-hzqkg:~$
```

如果在添加或删除钥匙较长时间后，钥匙圈仍与配置不同步，您可能需要重新启动您的`argocd-repo-server`如果问题仍然存在，请考虑提交错误报告。