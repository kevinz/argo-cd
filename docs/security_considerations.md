<!-- TRANSLATED by md-translate -->
# 安全考虑因素

!!警告 "弃用通知" 本页面现已弃用，仅作为存档。 如需了解最新信息，请查看我们的 [安全策略](https://github.com/argoproj/argo-cd/security/policy) 和 [已发布的安全公告](https://github.com/argoproj/argo-cd/security/advisories)。

作为一个部署工具，Argo CD 需要有生产访问权限，这使得安全成为一个非常重要的话题。 Argoproj 团队非常重视安全问题，并不断努力改进。 在 [Security](./operator-manual/security.md) 部分了解更多与安全相关的功能。

## 过去和当前问题概览

下表概括介绍了 Argo CD 项目过去和现在已知的问题。 如果无法更新或尚未修复，请参阅[已知问题](#known-issues-and-workarounds)部分是否有可用的解决方法。

|Date|CVE|Title|Risk|Affected version(s)|Fix version|
|----|---|-----|----|-------------------|-----------|
|2020-06-16|[CVE-2020-1747](https://nvd.nist.gov/vuln/detail/CVE-2020-1747)|PyYAML library susceptible to arbitrary code execution|High|all|v1.5.8|
|2020-06-16|[CVE-2020-14343](https://nvd.nist.gov/vuln/detail/CVE-2020-14343)|PyYAML library susceptible to arbitrary code execution|High|all|v1.5.8|
|2020-04-14|[CVE-2020-5260](https://nvd.nist.gov/vuln/detail/CVE-2020-5260)|Possible Git credential leak|High|all|v1.4.3,v1.5.2|
|2020-04-08|[CVE-2020-11576](https://nvd.nist.gov/vuln/detail/CVE-2020-11576)|User Enumeration|Medium|v1.5.0|v1.5.1|
|2020-04-08|[CVE-2020-8826](https://nvd.nist.gov/vuln/detail/CVE-2020-8826)|Session-fixation|High|all|n/a|
|2020-04-08|[CVE-2020-8827](https://nvd.nist.gov/vuln/detail/CVE-2020-8827)|Insufficient anti-automation/anti-brute force|High|all &lt;= 1.5.3|v1.5.3|
|2020-04-08|[CVE-2020-8828](https://nvd.nist.gov/vuln/detail/CVE-2020-8828)|Insecure default administrative password|

## 已知问题和解决方法

最近的一次安全审计（非常感谢[https://soluble.ai](https://soluble.ai) 的[Matt Hamilton](https://github.com/Eriner) ）揭示了 Argo CD 中可能危及安全的几个局限性。 大多数问题都与内置的用户管理实现有关。

### CVE-2020-1747, CVE-2020-14343 - PyYAML 库易被执行任意代码

**摘要：**

|Risk|Reported by|Fix version|Workaround|
|----|-----------|-----------|----------|
|High|[infa-kparida](https://github.com/infa-kparida)|v1.5.8|No|

**细节：**

PyYAML 库在处理不受信任的 YAML 文件时，易受任意代码执行影响。 我们认为 Argo CD 不受此漏洞影响，因为 CVE-2020-1747 和 CVE-2020-14343 的影响仅限于使用 awscli。 `awscli` 仅被引用用于 AWS IAM 身份验证，端点是 AWS API。

### CVE-2020-5260 - Git 凭据可能泄漏

**摘要：**

|风险|报告人|修复版本|工作方法||----|-----------|-----------|----------||关键|谷歌零项目的Felix Wilhelm|v1.4.3,v1.5.2|是

**细节：**

Argo CD 的许多操作都依赖于 Git。Git 项目于 2020-04-14 发布了一条[安全公告](https://github.com/git/git/security/advisories/GHSA-qm7j-c969-7j4q)，描述了 Git 中的一个严重漏洞，该漏洞可通过向 "git clone "操作输入恶意 URL，从而通过凭证助手导致凭证泄漏。

我们认为 Argo CD 不受此漏洞影响，因为 ArgoCD 既没有使用 Git 证书助手，也没有使用 `git clone` 进行版本库操作。 不过，我们不知道我们的用户是否可能自行配置了 Git 证书助手，并选择发布包含 Git 漏洞修复的新镜像。

**缓解和/或解决方法：**

我们强烈建议将您的 ArgoCD 安装升级到 `v1.4.3` (如果使用 v1.4 分支) 或 `v1.5.2` (如果使用 v1.5 分支)

当您正在运行 `v1.4.x` 时，只需将 `argocd-server`、`argocd-repo-server` 和 `argocd-controller` 的镜像标签更改为 `v1.4.3`，即可升级到 `v1.4.3`。 `v1.4.3`发布不包含额外的功能性错误修复。

同样，如果您正在运行 `v1.5.x`，只需将 `argocd-server`、`argocd-repo-server` 和 `argocd-controller` 的镜像标签更改为 `v1.5.2`，即可升级到 `v1.5.2`。 `v1.5.2`发布版不包含额外的功能性错误修复。

### CVE-2020-11576 - 用户枚举

**摘要：**

|风险|报告人|修复版本|工作方法| |----|-----------|-----------|----------| |中|[https://soluble.ai](https://soluble.ai)的[Matt Hamilton](https://github.com/Eriner)|v1.5.1|是|

**细节：**

Argo v1.5.0 版存在用户枚举漏洞，攻击者可利用该漏洞确定 Argo 中有效（非 SSO）账户的用户名。

**缓解和/或解决方法：**

升级到 ArgoCD v1.5.1 或更高版本。 作为一种解决方法，禁用本地用户并只使用 SSO 身份验证。

### CVE-2020-8828 - 不安全的默认管理密码

**摘要：**

|风险|报告人|修复版本|工作方法| |----|-----------|-----------|----------| |高|[https://soluble.ai](https://soluble.ai)的[Matt Hamilton](https://github.com/Eriner)|1.8.0|是|

**细节：**

Argo CD 使用 `argocd-server` pod 名称（例如：`argocd-server-55594fbdb9-ptsf5`）作为默认管理员密码。

能够在 Argo 名称空间中列出 pod 的 Kubernetes 用户能够检索默认密码。

此外，在大多数安装程序中，[Pod 名称包含随机 "跟踪 "字符](https://github.com/kubernetes/kubernetes/blob/dda530cfb74b157f1d17b97818aa128a9db8e711/staging/src/k8s.io/apiserver/pkg/storage/names/generate.go#L37)。这些字符是使用[时间种子 PRNG](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/rand/rand.go#L26)而不是 CSPRNG 生成的。攻击者可能会利用这些信息来推断内部 PRNG 的状态，从而协助暴力攻击。

**缓解和/或解决方法：**

如用户文档中所述，建议采取的缓解措施是使用 SSO 集成。 默认的管理员密码只能用于初始配置，然后 [禁用]（.../operator-manual/user-management/#disable-admin-user）或至少改为更安全的密码。

### CVE-2020-8827 - 反自动化/反蛮力不足

**摘要：**

|风险|报告人|修复版本|工作方法| |----|-----------|-----------|----------| |高|[https://soluble.ai](https://soluble.ai)的[Matt Hamilton](https://github.com/Eriner)|n/a|是|

**细节：**

ArgoCD v1.5.3 之前的版本不强制执行速率限制或其他反自动化机制，而这些机制可以减少管理员密码暴力破解。

**缓解和/或解决方法：**

ArgoCD v1.5.3 引入了本地用户账户的速率限制和反自动机制。

如果您还不能将 ArgoCD 升级到 v1.5.3，我们建议您禁用本地用户，改用 SSO 作为缓解问题的变通办法。

### CVE-2020-8826 - 会话修复

**摘要：**

|风险|报告人|修复版本|工作方法| |----|-----------|-----------|----------| |高|[https://soluble.ai](https://soluble.ai)的[Matt Hamilton](https://github.com/Eriner)|n/a|是|

**细节：**

为内置用户生成的身份验证令牌没有有效期。

这些问题在受控的隔离环境中可能可以接受，但如果 Argo CD 用户界面暴露在互联网上，就无法接受了。

**缓解和/或解决方法：**

建议采取的缓解措施是定期更改密码，使身份验证令牌失效。

### CVE-2018-21034 - 敏感信息泄露

**摘要：**

|风险|报告人|修复版本|工作方法| |----|-----------|-----------|----------| |中|[https://soluble.ai](https://soluble.ai)的[Matt Hamilton](https://github.com/Eriner)|v1.5.0|否|

**细节：**

在 Argo v1.5.0-rc1 之前的版本中，通过身份验证的 Argo 用户可以提交 API 调用，以获取存储在 git 中的秘密和其他配置清单。

**缓解和/或解决方法：**

升级至 ArgoCD v1.5.0 或更高版本。 无可用的解决方法

## 报告漏洞

有关如何报告 Argo CD 安全漏洞的更多详情，请参阅我们的[安全政策](https://github.com/argoproj/argo-cd/security/policy)。