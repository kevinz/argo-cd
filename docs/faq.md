<!-- TRANSLATED by md-translate -->
# 常见问题

## 我删除/破坏了我的软件源，无法删除我的应用程序。

Argo CD 如果无法生成配置清单，就无法删除应用程序。 你也需要这样做：

1. 恢复/修复您的 repo。
2. 使用 `--cascade=false` 删除应用程序，然后手动删除资源。

## 为什么我的应用程序在成功同步后仍会立即`OutOfSync`？

请参阅 [Diffing](user-guide/diffing.md)文档，了解资源可能出现不同步（OutOfSync）的原因，以及配置 Argo CD 以在预计出现差异时忽略字段的方法。

## 为什么我的应用程序停留在 `Progressing` 状态？

Argo CD 为几种标准的 Kubernetes 类型提供了健康检查，但 `Ingress`、`StatefulSet` 和 `SealedSecret` 类型存在已知问题，可能导致健康检查返回 `Progressing` 状态，而不是 `Healthy` 状态。

* 如果`status.loadBalancer.ingress`列表非空，且`hostname`或`IP`至少有一个值，则认为`Ingress`是健康的。
主机名 "或 "IP"。某些 ingress 控制器
([contour](https://github.com/heptio/contour/issues/403)
, [traefik](https://github.com/argoproj/argo-cd/issues/968#issuecomment-451082913)) 不更新
status.loadBalancer.ingress`字段，这会导致 `Ingress` 永远停留在 `Progressing` 状态。
* 如果`status.updatedReplicas`字段的值与`spec.replicas`字段匹配，则认为`StatefulSet`是健康的。由于
Kubernetes bug
[kubernetes/kubernetes#68573](https://github.com/kubernetes/kubernetes/issues/68573) `status.updatedReplicas` 没有被填充。
未填充。因此，除非您运行的 Kubernetes 版本包含
修复 [kubernetes/kubernetes#67570](https://github.com/kubernetes/kubernetes/pull/67570) 的 Kubernetes 版本。
状态。
* 您的 `StatefulSet` 或 `DaemonSet` 正在使用 `OnDelete` 而不是 `RollingUpdate` 策略。
请参阅 [#1881](https://github.com/argoproj/argo-cd/issues/1881)。
* 关于 `SealedSecret`, 请参阅 [为什么 `SealedSecret` 类型的资源停留在 `Progressing` 状态？](#sealed-secret-stuck-progressing)

作为变通办法，Argo CD 允许提供 [health check]（operator-manual/health.md）自定义功能，以覆盖默认行为。

如果您的 ingress 被引用了 Traefik，您可以更新 Traefik 配置，使用 [publishservice](https://doc.traefik.io/traefik/providers/kubernetes-ingress/#publishedservice) 发布 loadBalancer IP，这样就能解决这个问题。

```yaml
providers:
  kubernetesIngress:
    publishedService:
      enabled: true
```

## 我忘记了管理员密码，如何重置？

对于 Argo CD v1.8 及更早版本，初始密码按照[入门指南](getting_started.md)设置为服务器 pod 的名称。 对于 Argo CD v1.9 及更高版本，初始密码可从名为 "argocd-initial-admin-secret "的秘密中获取。

要更改密码，请编辑 `argocd-secret` 秘密，并用新的 bcrypt 哈希值更新 `admin.password` 字段。

注意 "生成 bcrypt 哈希值" 使用以下命令为 `admin.password` 生成 bcrypt 哈希值

```
argocd account bcrypt --password <YOUR-PASSWORD-HERE>
```

要引用新密码哈希值，请使用以下命令（用自己的哈希值替换）：

```bash
# bcrypt(password)=$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

另一种方法是删除 `admin.password` 和 `admin.passwordMtime` 密钥，然后重启 argocd-server。 这将根据[入门指南](getting_started.md) 生成一个新密码，可以是 pod 的名称（Argo CD 1.8 及更早版本），也可以是存储在某个存储密钥中的随机生成密码（Argo CD 1.9 及更高版本）。

## 如何禁用管理员用户？

在 `argocd-cm` ConfigMap 中添加 `admin.enabled: "false"` （请参阅 [user management](./operator-manual/user-management/index.md) ）。

## Argo CD 在无法上网的情况下无法部署基于 helm chart 的应用程序，如何解决？

如果图表依赖于外部版本库，Argo CD 可能无法生成 Helm 图表清单。 要解决这个问题，您需要确保 `requirements.yaml` 只使用内部可用的 Helm 版本库。 即使图表只使用内部版本库的依赖，Helm 也可能决定刷新 `stable` 版本库。 作为解决方法，在 `argocd-cm` 配置映射中覆盖 `stable` 版本库 URL：

```yaml
data:
  repositories: |
    - type: helm
      url: http://<internal-helm-repo-host>:8080
      name: stable
```

## 在使用 Argo CD 部署 Helm 应用程序后，我无法使用 `helm ls` 和其他 Helm 命令看到它。

在部署 Helm 应用程序时，Argo CD 仅将 Helm 用作模板机制。 它运行 "helm template"，然后将生成的配置清单部署到集群上，而不是执行 "helm install"。 这意味着您无法使用任何 Helm 命令来查看/验证应用程序。 它完全由 Argo CD 管理。 请注意，Argo CD 原生支持 Helm 可能会遗漏的一些功能（如历史记录和回滚命令）。

做出这一决定是为了使 Argo CD 对所有配置清单生成器保持中立。

## 我配置了 [集群秘密](./operator-manual/declarative-setup.md#clusters)，但它没有显示在 CLI/UI 中，我该如何修复？

检查集群秘密是否有 "argocd.argoproj.io/secret-type: cluster "标签。 如果秘密有标签，但集群仍不可见，则确定可能是权限问题。 尝试使用 "admin "用户列出集群（例如，"argocd login --username admin &amp;&amp; argocd cluster list"）。

## Argo CD 无法连接到我的集群，如何排除故障？

使用以下步骤重构已配置的集群配置，并使用 kubectl 手动连接到集群：

```bash
kubectl exec -it <argocd-pod-name> bash # ssh into any argocd server pod
argocd admin cluster kubeconfig https://<cluster-url> /tmp/config --namespace argocd # generate your cluster config
KUBECONFIG=/tmp/config kubectl get pods # test connection manually
```

现在，您可以手动验证集群是否可从 Argo CD pod 访问。

## 如何终止同步？

要终止同步，请单击 "同步"，然后单击 "终止"：

同步](assets/synchronization-button.png) 终止](assets/terminate-button.png)

## 为何我的应用程序在同步后仍会 "不同步"？

在某些情况下，通过添加 `app.kubernetes.io/instance` 标签，您使用的工具可能会与 Argo CD 冲突。 例如，被引用 kustomize 常用标签功能。

Argo CD 会自动设置 `app.kubernetes.io/instance` 标签，并使用它来确定哪些资源构成应用程序。 如果工具也这样做，就会造成混乱。 您可以通过设置 `argocd-cm` 中的 `application.instanceLabelKey` 值来更改此标签。 我们建议您引用 `argocd.argoproj.io/instance`。

注意 当您进行此更改时，您的应用程序将不同步，需要重新同步。

参见 [#1482](https://github.com/argoproj/argo-cd/issues/1482)。

## Argo CD 多久检查一次我的 Git 或 Helm 仓库的变更？

默认轮询间隔为 3 分钟（180 秒），可配置抖动。您可以通过更新 [argocd-cm](https://github.com/argoproj/argo-cd/blob/2d6ce088acd4fb29271ffb6f6023dbb27594d59b/docs/operator-manual/argocd-cm.yaml#L279-L282) config map 中的 `timeout.reconciliation` 值和 `timeout.reconciliation.jitter` 来更改设置。 如果有任何 Git 变动，Argo CD 将只更新启用了 [auto-sync setting](user-guide/auto_sync.md) 的应用程序。如果将其设置为 `0`，Argo CD 将停止自动轮询 Git 仓库，您只能使用 [webhooks](operator-manual/webhook.md) 和/或手动同步等替代方法来部署应用程序。

## Why Are My Resource Limits `Out Of Sync`?

Kubernetes 在配置您的资源限制时已将其归一化，然后 Argo CD 将您生成的配置清单中的版本与 Kubernetes 归一化后的版本进行了比较，结果发现两者并不匹配。

例如

* 1000米 "归一化为 "1米
* 0.1 "归一化为 "100米
* 3072米 "归一化为 "3Gi
* `3072`归一化为`'3072'`（引号已添加）

要解决这个问题，请使用 diffing 自定义 [设置]（./user-guide/diffing.md#kknown-kubernetes-types-in-crds-resource-limits-volume-mounts-etc）。

## 如何修复 `invalid cookie, longer than max length 4093`?

Argo CD 使用 JWT 作为认证令牌。 您可能加入了许多群组，并已超过了为 Cookie 设置的 4KB 限制。 您可以打开 "开发工具 -&gt; 网络"，获取群组列表。

* 点击登录
* 查找对 `<argocd_instance>/auth/callback?code=<random_string>` 的调用

解码 [https://jwt.io/](https://jwt.io/)上的令牌。这将提供您可以删除自己的团队列表。

参见 [#2165](https://github.com/argoproj/argo-cd/issues/2165)。

## 为什么我在使用 CLI 时会收到 `rpc error: code = Unavailable desc = transport is closing` 的提示？

也许您的代理服务器不支持 HTTP 2？ 试试 `--grpc-web` flag：

```bash
argocd ... --grpc-web
```

## 为什么我在使用 CLI 时会收到 `x509: certificate signed by unknown authority` 的提示？

Argo CD 默认创建的证书不会被 Argo CD CLI 自动识别，为了创建一个安全的系统，您必须按照说明[安装证书]（/operator-manual/tls/），并配置您的客户端操作系统以信任该证书。

如果不是在生产系统中运行（例如测试 Argo CD），请尝试使用 `--insecure` flag：

```bash
argocd ... --insecure
```

!!! 警告 "请勿在生产中使用 `--insecure`"

## 我通过 `argocd-cm` 中的 `dex.config` 配置了 Dex，但它仍然显示 Dex 未配置。 为什么？

您很可能忘记设置 `argocd-cm` 中的 `url` 以指向 Argo CD。 另请参阅 [文档](./operator-manual/user-management/index.md#2-configure-argo-cd-for-sso)。

## 为什么`SealedSecret`资源会报告`Status`？

包括 `v0.15.0` 在内的 `SealedSecret` 版本（尤其是 helm `1.15.0-r3`）不包含 [modern CRD](https://github.com/bitnami-labs/sealed-secrets/issues/555)，因此状态字段不会被暴露（在 k8s `1.16+`上）。如果你的 Kubernetes 部署是 [modern](https://www.openshift.com/blog/a-look-into-the-technical-details-of-kubernetes-1-16)，如果你想让此功能正常工作，请确保你使用的是固定 CRD。

##<a name="sealed-secret-stuck-progressing"></a>`SealedSecret`类型的资源为何停留在`Progressing`状态？

封密 "资源的控制器可能会暴露其所提供资源的状态条件。 自版本 "v2.0.0 "起，Argo CD 会获取该状态条件，从而推导出 "封密 "的健康状态。

SealedSecret "控制器的 "v0.15.0 "之前的版本受到状态条件更新问题的影响，因此这些版本默认禁用此功能。 可以通过使用"--update-status "命令行参数启动 "SealedSecret "控制器或设置 "SEALED_SECRETS_UPDATE_STATUS "环境变量来启用状态条件更新。

要禁止 Argo CD 检查`SealedSecret`资源的状态条件，请通过`resource.customizations.health.<group_kind>`键在`argocd-cm`配置地图中添加以下资源定制。

```yaml
resource.customizations.health.bitnami.com_SealedSecret: |
  hs = {}
  hs.status = "Healthy"
  hs.message = "Controller doesn't report resource status"
  return hs
```

## 如何修复`补丁列表中的顺序......与 $setElementOrder 列表不匹配:......`？

应用程序可能会触发标有 "ComparisonError"（比较错误）的同步错误，并显示类似信息：

&gt; The order in patch list: [map[name:**KEY_BC** value:150] map[name:**KEY_BC** value:500] map[name:**KEY_BD** value:250] map[name:**KEY_BD** value:500] map[name:KEY_BI value:something]] doesn't match $setElementOrder list: [map[name:KEY_AA] map[name:KEY_AB] map[name:KEY_AC] map[name:KEY_AD] map[name:KEY_AE] map[name:KEY_AF] map[name:KEY_AG] map[name:KEY_AH] map[name:KEY_AI] map[name:KEY_AJ] map[name:KEY_AK] map[name:KEY_AL] map[name:KEY_AM] map[name:KEY_AN] map[name:KEY_AO] map[name:KEY_AP] map[name:KEY_AQ] map[name:KEY_AR] map[name:KEY_AS] map[name:KEY_AT] map[name:KEY_AU] map[name:KEY_AV] map[name:KEY_AW] map[name:KEY_AX] map[name:KEY_AY] map[name:KEY_AZ] map[name:KEY_BA] map[name:KEY_BB] map[name:**KEY_BC**] map[name:**KEY_BD**] map[name:KEY_BE] map[name:KEY_BF] map[name:KEY_BG] map[name:KEY_BH] map[name:KEY_BI] map[name:**KEY_BC**] map[name:**KEY_BD**]]

信息有两个部分：

1. `补丁列表中的顺序：[`
    这将识别项目的值，尤其是多次出现的项目：&gt; map[name:**KEY_BC** value:150] map[name:**KEY_BC** value:500] map[name:**KEY_BD** value:250] map[name:**KEY_BD** value:500] map[name:KEY_BI value:something]
    您需要识别重复的键 - 您可以关注第一部分，因为每个重复的键都会出现一次，其值与第一个列表中的值各一次。第二个列表实际上只是`]`。
2. 不匹配 $setElementOrder 列表：[`
    这包括所有的键。它是为了调试目的而包含的 -- 你不需要太在意它。它将提示您重复键在列表中的精确位置：&gt; map[name:KEY_AA] map[name:KEY_AB] map[name:KEY_AC] map[name:KEY_AD] map[name:KEY_AE] map[name：KEY_AF] 地图[名称:KEY_AG] 地图[名称:KEY_AH] 地图[名称:KEY_AI] 地图[名称:KEY_AJ] 地图[名称:KEY_AK] 地图[名称:KEY_AL] 地图[名称:KEY_AM] 地图[名称:KEY_AN] 地图[名称:KEY_AO] 地图[名称:KEY_AP] 地图[名称：Map[name:KEY_AQ] map[name:KEY_AR] map[name:KEY_AS] map[name:KEY_AT] map[name:KEY_AU] map[name:KEY_AV] map[name:KEY_AW] map[name:KEY_AX] map[name:KEY_AY] map[name:KEY_AZ] map[name:KEY_BA] map[name：地图[名称:KEY_BB] 地图[名称:**KEY_BC**] 地图[名称:**KEY_BD**] 地图[名称:KEY_BE] 地图[名称:KEY_BF] 地图[名称:KEY_BG] 地图[名称:KEY_BH] 地图[名称:KEY_BI] 地图[名称:**KEY_BC**] 地图[名称:**KEY_BD**]
    `]`

在本例中，重复的键已被**强调，以帮助您识别有问题的键。 许多编辑器都有高亮显示字符串所有实例的功能，使用这样的编辑器可以帮助解决此类问题。

这种错误最常见的情况是 `containers` 的 `env:` 字段。

注意 "动态应用程序" 您的应用程序可能是由一个工具生成的，在这种情况下，单个文件范围内的重复可能并不明显。 如果您在调试这个问题时遇到困难，可以考虑向生成工具的 Owners 提交一份报告，要求他们改进其验证和错误报告。
