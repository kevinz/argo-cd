<!-- TRANSLATED by md-translate -->
# 高可用性

Argo CD 在很大程度上是无状态的。 所有数据都以 Kubernetes 对象的形式持久化，而 Kubernetes 对象又存储在 Kubernetes 的 etcd 中。 Redis 只被用作丢弃式缓存，可能会丢失。 丢失后，它会在不损失服务的情况下重建。

为希望以高可用方式运行 Argo CD 的用户提供了一套 [HA 配置清单](https://github.com/argoproj/argo-cd/tree/master/manifests/ha)。这样可以运行更多容器，并以 HA 模式运行 Redis。

&gt; 此外，不支持仅 IPv6 的集群。

## 扩大规模

### argocd-repo-server

**设置：**

argocd-repo-server "负责克隆 Git 仓库、保持更新并使用相应工具生成配置清单。

* `argocd-repo-server` fork/exec 配置管理工具生成配置清单。分叉可能会因内存不足或操作系统线程数限制而失败。

parallelismlimit"（并行限制）配置清单标志可控制同时运行多少代配置清单，有助于避免 OOM kills。

* 在配置清单生成过程中，"argocd-repo-server "会使用配置管理工具（如 kustomize、Helm）确保版本库处于干净状态。

因此，带有多个应用程序的 Git 仓库可能会影响仓库服务器的性能。 请阅读 [Monorepo 扩展注意事项](#monorepo-scaling-considerations) 了解更多信息。

* argocd-repo-server "会将版本库克隆到"/tmp"（或 "TMPDIR "环境变量中指定的路径）。如果有太多版本库，Pod 可能会耗尽磁盘空间。

为了避免这个问题，请挂载持久卷。

* `argocd-repo-server` 被引用 `git ls-remote` 来解决诸如 `HEAD`、分支或标签名之类的模糊修订。这种操作经常发生

为避免同步失败，可使用 `ARGOCD_GIT_ATTEMPTS_COUNT` 环境变量重试失败的请求。

* `argocd-repo-server` Argo CD 每 3 米（默认情况下）检查一次应用程序配置清单的更改。Argo CD 默认认为配置清单只有在软件仓库发生变化时才会发生变化，因此它会缓存生成的配置清单（默认为 24 小时）。如果使用 Kustomize 远程库，或者 helm chart 在版本号未发生变化的情况下发生更改，即使软件版本库未发生变化，预期配置清单也会发生变化。通过缩短缓存时间，就可以在不等待 24 小时的情况下获得更改。被引用"--repo-cache-expiration duration"（版本库缓存过期持续时间），我们建议在低容量环境中尝试 "1h"。请注意，如果设置过低，会抵消缓存的好处。
* argocd-repo-server "会执行配置管理工具，如 "helm "或 "kustomize"，并强制执行 90 秒超时。该超时可通过 `ARGOCD_EXEC_TIMEOUT` 环境变量进行更改。该值应采用 Go 时间长度字符串格式，例如 `2m30s`。

**指标：**

* `argocd_git_request_total` - git 请求数。该指标提供了两个标签：`repo` - Git 仓库 URL；`request_type` - `ls-remote` 或 `fetch`。
* `ARGOCD_ENABLE_GRPC_TIME_HISTOGRAM` - 环境变量，用于收集 RPC 性能指标。如果需要排除性能问题，请启用它。注意：该指标对查询和存储都很昂贵！

### argocd-application-controller

**设置：**

argocd-application-controller "被引用 "argocd-repo-server "来获取生成的配置清单，而 "Kubernetes API server "则获取实际的集群状态。

* 每个控制器副本被引用两个独立的队列来处理应用程序调节（毫秒）和应用程序同步（秒）。每个队列的队列处理器数量由

状态处理器"（默认为 20 个）和 "操作处理器"（默认为 10 个）标志。 如果 Argo CD 实例管理的应用程序过多，可增加处理器数量。 对于 1000 个应用程序，我们使用 50 个 "状态处理器 "和 25 个 "操作处理器"。

* 在调节过程中，配置清单的生成通常耗时最长。配置清单生成的持续时间受到限制，以确保控制器刷新队列不会溢出。

如果配置清单生成耗时过长，应用程序调节失败时会出现 "Context deadline exceeded "错误。 作为一种解决方法，可增加"--repo-server-timeout-seconds "的值，并考虑扩大 "argocd-repo-server "的部署规模。

* 控制器被引用 Kubernetes 观察 API 来维护轻量级 Kubernetes 集群缓存。这就避免了在应用程序调节过程中查询 Kubernetes，并显著改善了应用程序的性能。

出于性能考虑，控制器只监控和缓存资源的首选版本。 在调节过程中，控制器可能需要将缓存的资源从首选版本转换为存储在 Git 中的资源版本。 如果 "kubectl convert "因不支持转换而失败，控制器就会退回到 Kubernetes API 查询，从而减慢调节速度。 在这种情况下，我们建议使用 Git 中的首选资源版本。

* 控制器默认每 3m 轮询一次 Git。你可以使用 `argocd-cm` configMap 中的 `timeout.reconciliation` 和 `timeout.reconciliation.jitter` 设置来更改这一持续时间。字段的值是持续时间字符串，例如 `60s`、`1m`、`1h` 或 `1d`。
* 如果控制器管理的集群过多，被引用的内存过大，那么可以将集群分片到多个

要启用分片功能，请增加 `ARGOCD-APPERATION- CONTROLLER``StatefulSet` 中的副本数量，并重复 `ARGOCD_CONTROLLER_REPLICAS` 环境变量中的副本数量。 下面的战略合并补丁演示了配置两个控制器副本所需的更改。

* 默认情况下，控制器将每 10 秒更新一次集群信息。如果集群网络环境出现问题导致更新时间过长，可以尝试修改环境变量 `ARGO_CD_UPDATE_CLUSTER_INFO_TIMEOUT` 来增加超时时间（单位为秒）。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: argocd-application-controller
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: argocd-application-controller
        env:
        - name: ARGOCD_CONTROLLER_REPLICAS
          value: "2"
```

* 要手动设置集群的分片编号，请在创建集群时指定可选的 `shard` 属性。如果不指定，应用控制器将自动计算。
* argocd-application-controller "的分片分配算法可通过使用"--sharding-method "参数来设置。支持的分片方法有[传统（默认）、循环]。传统模式 "使用基于 "uid "的分布（非均匀）。round-robin "模式在所有分片之间使用平均分配。还可以通过在 `argocd-cmd-params-cm` 配置表中设置 `controller.sharding.algorithm` 关键字（最好），或通过设置 `ARGOCD_CONTROLLER_SHARDING_ALGORITHM` 环境变量并指定相同的可能值，来覆盖 `--sharding-method` 参数。

!!! 警告 "alpha 功能" "round-robin "碎片分配算法是一项实验性功能。 众所周知，在某些移除集群的情况下会发生重新洗牌。 如果位于 0 级的集群被移除，则会发生跨碎片重新洗牌所有集群的情况，可能会暂时对性能产生负面影响。

* 可通过修补集群秘密中的 "shard "字段，使其包含分片编号，从而手动分配集群并强制其使用 "分片"，例如

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  shard: 1
  name: mycluster.example.com
  server: https://mycluster.example.com
  config: |
    {
      "bearerToken": "<authentication token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64 encoded certificate>"
      }
    }
```

* ARGOCD_ENABLE_GRPC_TIME_HISTOGRAM` - 用于收集 RPC 性能指标的环境变量。如果需要排除性能问题，请启用它。注意：该指标的查询和存储成本都很高！
* `ARGOCD_CLUSTER_CACHE_LIST_PAGE_BUFFER_SIZE` - 环境变量，控制执行列表缓存时控制器在内存中缓冲的页数。
在同步集群缓存时对 k8s api 服务器执行列表操作时在内存中缓冲的页数。这个
当集群包含大量资源，且集群同步时间超过默认的 etcd
压缩间隔超时。在这种情况下，当尝试同步集群缓存时，应用程序控制器
可能会抛出一个错误，即 "继续参数太旧，无法显示一致的列表结果"。为该环境变量设置一个较高的
环境变量的值，可为控制器配置更大的缓冲区，用于存储预先抓取的
页面，从而提高在 etcd
压缩间隔超时之前提取完所有页面的可能性。在最极端的情况下，操作员可以将此值设置为
ARGOCD_CLUSTER_CACHE_LIST_PAGE_SIZE * ARGOCD_CLUSTER_CACHE_LIST_PAGE_BUFFER_SIZE "超过最大资源数（按 k8s 分组）。
数（按 k8s api 版本分组，即列表操作的并行粒度）。在这种情况下，所有资源都将
都将在内存中缓冲，不会阻塞 api 服务器请求的处理。

**metrics**

* `argocd_app_reconcile` - 报告应用程序对账持续时间。可被引用来构建对账持续时间热图，以获得高水平的对账性能图像。
* `argocd_app_k8s_request_total` - 每个应用程序的 k8s 请求数。回退 Kubernetes API 查询的数量 - 用于识别哪个应用程序的资源带有

非首选版本，导致性能问题。

### argocd-server

argocd-server "是无状态的，可能是最不可能引起问题的。 为确保升级期间不会出现停机，可考虑将副本数量增加到 "3 "或更多，并在 "ARGOCD_API_SERVER_REPLICAS "环境变量中重复该数量。 下面的战略合并补丁演示了这一点。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: argocd-server
        env:
        - name: ARGOCD_API_SERVER_REPLICAS
          value: "3"
```

**设置：**

* ARGOCD_API_SERVER_REPLICAS` 环境变量用于在每个副本之间分配 [并发登录请求的限制 (`ARGOCD_MAX_CONCURRENT_LOGIN_REQUESTS_COUNT`)](./user-management/index.md#failed-logins-rate-limiting) 。
* ARGOCD_GRPC_MAX_SIZE_MB` 环境变量允许以 MB 为单位指定服务器响应信息的最大大小。

默认值为 200，对于管理 3000 多个应用程序的 Argo CD 实例，可能需要增加该值。

### argocd-dex-server, argocd-redis

argocd-dex-server "使用内存数据库，两个或更多实例将产生不一致的数据。"argocd-redis "在预配置时了解到总共只有三个 redis 服务器/sentinels。

## Monorepo 扩展考虑因素

Argo CD 版本库服务器会在本地维护一个版本库克隆，并将其用于应用程序配置清单的生成。 如果配置清单的生成需要更改本地版本库克隆中的文件，则每个服务器实例只允许同时生成一个配置清单。 如果您的单版本库中有多个应用程序（50 个以上），则此限制可能会大大降低 Argo CD 的运行速度。

### 启用并行处理

Argo CD 会根据配置管理工具和应用程序设置来确定清单生成是否会更改本地版本库克隆中的本地文件。 如果清单生成没有副作用，则会并行处理请求，而不会影响性能。 以下是可能导致速度变慢的已知情况及其解决方法：

* ** 基于 Helm 的多个应用程序指向一个 Git 仓库中的同一目录：** 确保您的 Helm 图表不存在有条件的

[依赖项](https://helm.sh/docs/chart_best_practices/dependencies/#conditions-and-tags)，并在 chart 目录中创建 `.argocd-allow-concurrency` 文件。

* **基于自定义插件的多个应用程序：**避免在配置清单生成过程中创建临时文件，并在应用程序目录中创建`.argocd-allow-concurrency`文件，或使用 sidecar 插件选项，该选项使用版本库的临时副本处理每个应用程序。
* **在同一版本库中使用 [parameter overrides](../user-guide/parameters.md) 的多个 kustomize 应用程序：**抱歉，暂时没有解决方法。

### Webhook 和配置清单路径注释

Argo CD 会主动缓存生成的清单，并使用版本库的提交 SHA 作为缓存密钥。 Git 版本库的新提交会使版本库中配置的所有应用程序的缓存失效。 这可能会对具有多个应用程序的版本库产生负面影响。您可以使用 [webhooks](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/webhook.md) 和 `argocd.argoproj.io/manifest-generate-paths` Application CRD 注释来解决这个问题并提高性能。

`argocd.argoproj.io/manifest-generate-paths` 注解包含一个分号分隔的 Git 仓库路径列表，在配置清单生成过程中使用。 webhook 会将注解中指定的路径与 webhook 有效负载中指定的更改文件进行比较。 如果没有更改文件与 `argocd.argoproj.io/manifest-generate-paths` 中指定的路径相匹配，则 webhook 不会触发应用程序对账，现有缓存将被视为对新提交有效。

为每个应用程序使用不同版本库的安装***不受此行为的限制，并且很可能无法从使用这些 Annotations 中受益。

应用程序配置清单路径 Annotations 支持取决于应用程序所引用的 git 提供商，目前仅支持基于 GitHub、GitLab 和 Gogs 的 repos。

* **相对路径** Annotations 可能包含相对路径。在这种情况下，路径被视为相对于应用程序源中指定的路径：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  annotations:
    # resolves to the 'guestbook' directory
    argocd.argoproj.io/manifest-generate-paths: .
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
# ...
```

* **绝对路径** Annotations 的值可能是以"/"开头的绝对路径。在这种情况下，路径被视为 Git 仓库中的绝对路径：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  annotations:
    argocd.argoproj.io/manifest-generate-paths: /guestbook
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
# ...
```

* **多个路径** 可以在 Annotations 中放入多个路径。路径之间必须用分号 (`;`) 分隔：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  annotations:
    # resolves to 'my-application' and 'shared'
    argocd.argoproj.io/manifest-generate-paths: .;../shared
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: my-application
# ...
```

### 应用程序同步超时和抖动

Argo CD 为应用程序同步设置了一个超时时间。 当超时时间到期时，它会定期触发每个应用程序的刷新。 如果应用程序数量较多，刷新队列中的刷新次数就会激增，从而导致 repo-server 组件的刷新次数激增。 为避免这种情况，可以为同步超时设置一个抖动，这样就可以分散刷新次数，让 repo-server 有时间赶上。

抖动是可以添加到同步超时中的最大持续时间，因此，如果同步超时为 5 分钟，抖动为 1 分钟，那么实际超时将在 5 至 6 分钟之间。

要配置抖动，可以设置以下环境变量：

* ARGOCD_RECONCILIATION_JITTER` - 应用于同步超时的抖动。Values 为 0 时禁用，默认为 0。

## 限制费率应用调节

为防止因行为不端的应用程序或其他特定环境因素造成控制器资源使用率过高或同步循环，我们可以对应用程序控制器使用的工作队列配置速率限制。 可配置的速率限制有两种类型：

* global 费率限制
* 每个项目的费率限制

Finalizer 的速率限制器被引用两者的组合，并以 `max(globalBackoff,perItemBackoff)`计算最终的后退。

### Global rate limits

默认情况下已启用，这是一个简单的基于桶的速率限制器，可限制每秒可排队的项目数。 这对于防止大量应用程序同时排队非常有用。

要配置水桶限制器，可以设置以下环境变量：

* WORKQUEUE_BUCKET_SIZE` - 单个突发中可排队的项目数。默认为 500。
* `WORKQUEUE_BUCKET_QPS` - 每秒可排队的项目数。默认为 50。

### 每个项目的费率限制

默认情况下，它会返回一个固定的基本延迟/回退值，但也可配置为返回指数值。 每个项目速率限制器可限制特定项目的排队次数。 它基于指数回退，如果一个项目在短时间内多次排队，则其回退时间会以指数形式不断增加，但如果自上次排队以来已过了配置的 "冷却 "期，则回退会自动重置。

要配置每个项目限制器，可以设置以下环境变量：

* WORKQUEUE_FAILURE_COOLDOWN_NS：以纳秒为单位的冷却时间，一旦某个项目的冷却时间过期，将重置回退。如果设置为 0（默认值），指数延迟将被禁用，例如 Values : 10 * 10^9 (=10s)
* WORKQUEUE_BASE_DELAY_NS：以纳秒为单位的基本延迟，这是指数延迟公式中被引用的初始延迟。默认为 1000 (=1μs)
* WORKQUEUE_MAX_DELAY_NS`：最大延迟（纳秒），这是最大回退限制。默认为 3 * 10^9 (=3 秒)
* `WORKQUEUE_BACKOFF_FACTOR` ：延迟因子，即每次重试时延迟增加的因子。默认为 1.5

被引用用于计算项目延迟时间的公式，其中 `numRequeue` 是项目排队的次数，`lastRequeueTime` 是项目最后一次排队的时间：

* 当 `WORKQUEUE_FAILURE_COOLDOWN_NS` != 0 时：

```
backoff = time.Since(lastRequeueTime) >= WORKQUEUE_FAILURE_COOLDOWN_NS ?
          WORKQUEUE_BASE_DELAY_NS :
          min(
              WORKQUEUE_MAX_DELAY_NS,
              WORKQUEUE_BASE_DELAY_NS * WORKQUEUE_BACKOFF_FACTOR ^ (numRequeue)
              )
```

* 当 `WORKQUEUE_FAILURE_COOLDOWN_NS` = 0 时：

```
backoff = WORKQUEUE_BASE_DELAY_NS
```

## HTTP 请求重试策略

在网络不稳定或服务器出现短暂错误的情况下，重试策略会自动重新发送失败的请求，从而确保 HTTP 通信的稳健性。 它将最大重试次数和回退间隔结合起来使用，以防止服务器不堪重负或网络崩溃。

### 配置重试

重试逻辑可通过以下环境变量进行微调：

* `ARGOCD_K8SCLIENT_RETRY_MAX` - 每个请求的最大重试次数。达到此次数后，请求将被放弃。默认值为 0（无重试）。
* ARGOCD_K8SCLIENT_RETRY_BASE_BACKOFF` - 首次重试尝试的初始回退延迟（毫秒）。随后的重试将使该延迟时间加倍，直至达到最大阈值。默认为 100ms。

#### 后退策略

采用的回退策略是一种简单的无抖动指数回退，回退时间随着每次重试呈指数增长，直至达到最大回退持续时间。

后退时间的计算公式为

```
backoff = min(retryWaitMax, baseRetryBackoff * (2 ^ retryAttempt))
```

其中，"retryAttempt "从 0 开始，以后每次重试都以 1 为单位递增。

### 最长等待时间

回退时间有一个上限，以防止重试之间等待时间过长。 该上限由以下参数定义：

retryWaitMax` - 重试前的最长等待时间。 这将确保重试在合理的时间范围内进行。 默认为 10 秒。

### 不可逆转的条件

并非所有 HTTP 响应都可以重试，以下情况不会触发重试：

* 状态代码为 4xx 的回复，表明客户端出错（429 请求过多除外）。
* 状态代码为 501 未执行的响应。
