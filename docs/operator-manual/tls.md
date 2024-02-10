<!-- TRANSLATED by md-translate -->
# TLS 配置

Argo CD 提供三个可配置的入站 TLS 端点：

* argocd-server "工作负载面向用户的端点，为用户界面和应用程序接口提供服务
* argocd-repo-server "的端点，由 "argocd-server "和 "argocd-application-controller "工作负载访问，以请求存储库操作。
* argocd-dex-server "的端点，由 "argocd-server "访问，以处理 OIDC 验证。

默认情况下，无需进一步配置，这些端点将被设置为使用自动生成的自签名证书。 不过，大多数用户都希望为这些 TLS 端点明确配置证书，可以使用自动方法（如 "cert-manager"）或使用自己的专用证书颁发机构。

## 为 argocd-server 配置 TLS

### argocd-server 的入站 TLS 选项

通过设置命令行参数，可以为 `argocd-server` 工作负载配置某些 TLS 选项。 以下是可用参数：

|参数|默认值|描述| |---------|-------|-----------| |`--insecure`|`false`| 完全禁用 TLS| |`--tlsminversion`|`1.2`| 向客户提供的最小 TLS 版本| |`--tlsmaxversion`|`1.3`| 向客户提供的最大 TLS 版本| |`--tlsciphers`|`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_RSA_WITH_AES_256_GCM_SHA384`| 向客户提供的 TLS 密码套件的冒号分隔列表

### argocd-server 被引用的 TLS 证书

有两种方法可以配置 `argocd-server` 被引用的 TLS 证书：

* 设置 `argocd-server-tls` secret 中的 `tls.crt` 和 `tls.key` 密钥，以保存证书的 PEM 数据和相应的私钥。argocd-server-tls` secret 可以是 `tls` 类型，但并非必须如此。
* 设置 `argocd-secret` secret 中的 `tls.crt` 和 `tls.key` 密钥，以保存证书的 PEM 数据和相应的私钥。此方法已被弃用，仅用于向后兼容。不应再使用更改 `argocd-secret` 来覆盖 TLS 证书。

Argo CD 决定为 `argocd-server` 的端点使用哪种 TLS 证书的方法如下：

* 如果 `argocd-server-tls` secret 存在，并且在 `tls.crt` 和 `tls.key` 密钥中包含有效的密钥对，这将被引用为 `argocd-server` 端点的证书。
* 否则，如果 `argocd-secret` secret 中的

tls.crt "和 "tls.key "密钥，这将被引用为 `argocd-server` 端点的证书。

* 如果在上述两个秘密中都没有找到`tls.crt`和`tls.key`密钥，Argo CD 将生成一个自签名证书，并将其保存在`argocd-secret`秘密中。

argocd-server-tls "秘密只包含 TLS 配置信息，供 "argocd-server "使用，可通过第三方工具（如 "cert-manager "或 "SealedSecrets"）安全管理。

要从现有的密钥对中手动创建该秘密，可以使用 `kubectl`：

```shell
kubectl create -n argocd secret tls argocd-server-tls \
  --cert=/path/to/cert.pem \
  --key=/path/to/key.pem
```

Argo CD 会自动被引用 "argocd-server-tls "secret 的更改，无需重启 pod 以使用更新的证书。

## 为 argocd-repo-server 配置入站 TLS

### argocd-repo-server 的入站 TLS 选项

通过设置命令行参数，可以为 `argocd-repo-server` 工作负载配置某些 TLS 选项。 下列参数可用：

|Parameter|Default|Description|
|---------|-------|-----------|
|`--disable-tls`|`false`|Disables TLS completely|
|`--tlsminversion`|`1.2`|The minimum TLS version to be offered to clients|
|`--tlsmaxversion`|`1.3`|The maximum TLS version to be offered to clients|
|`--tlsciphers`|`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_RSA_WITH_AES_256_GCM_SHA384`|A colon separated list of TLS cipher suites to be offered to clients|

### argocd-repo-server 被引用的入站 TLS 证书

要配置 `argocd-repo-server` 工作负载使用的 TLS 证书，请在运行 Argo CD 的 namespace 中创建名为 `argocd-repo-server-tls` 的秘密，证书的密钥对存储在 `tls.crt` 和 `tls.key` 密钥中。 如果该秘密不存在，`argocd-repo-server` 将生成并使用自签名证书。

要创建此秘密，可以使用 `kubectl`：

```shell
kubectl create -n argocd secret tls argocd-repo-server-tls \
  --cert=/path/to/cert.pem \
  --key=/path/to/key.pem
```

如果证书是自签名的，还需要在秘密中添加 `ca.crt` 和 CA 证书的内容。

请注意，与 `argocd-server` 不同的是，`argocd-repo-server` 无法自动获取对该秘密的更改。 如果创建（或更新）了该秘密，则需要重新启动`argocd-repo-server` pod。

还需注意的是，签发证书时应为 `argocd-repo-server` 添加正确的 SAN 条目，至少包含 `DNS:argocd-repo-server` 和 `DNS:argocd-repo-server.argo-cd.svc` 条目，具体取决于工作负载连接版本库服务器的方式。

## 为 argocd-dex-server 配置入站 TLS

### argocd-dex-server 的入站 TLS 选项

通过设置命令行参数，可以为 `argocd-dex-server` 工作负载配置某些 TLS 选项。 以下是可用参数：

|参数|默认值|描述| |---------|-------|-----------| |`--disable-tls`|`false`| 完全禁用 TLS

### argocd-dex-server 被引用的入站 TLS 证书

要配置 `argocd-dex-server` 工作负载使用的 TLS 证书，请在 Argo CD 运行所在的 namespace 中创建名为 `argocd-dex-server-tls` 的秘密，并将证书的密钥对存储在 `tls.crt` 和 `tls.key` 密钥中。 如果该秘密不存在，`argocd-dex-server` 将生成并使用自签名证书。

要创建此秘密，可以使用 `kubectl`：

```shell
kubectl create -n argocd secret tls argocd-dex-server-tls \
  --cert=/path/to/cert.pem \
  --key=/path/to/key.pem
```

如果证书是自签名的，还需要在秘密中添加 `ca.crt` 和 CA 证书的内容。

请注意，相对于 `argocd-server` 而言，`argocd-dex-server` 无法自动接收对该秘密的更改。 如果创建（或更新）了该秘密，则需要重新启动`argocd-dex-server` pod。

还要注意的是，证书应为 `argocd-dex-server` 分配正确的 SAN 条目，至少包含 `DNS:argocd-dex-server` 和 `DNS:argocd-dex-server.argo-cd.svc` 条目，具体取决于工作负载连接到版本库服务器的方式。

## 在 Argo CD 组件之间配置 TLS

### 为 argocd-repo-server 配置 TLS

argocd-server "和 "argocd-application-controller "都使用通过 TLS 的 gRPC API 与 "argocd-repo-server "通信。 默认情况下，"argocd-repo-server "会生成一个非持久、自签名的证书，在启动时用于其 gRPC 端点。 由于 "argocd-repo-server "无法连接到 k8s 控制平面 API，因此外部用户无法使用该证书进行验证。 出于这个原因，"argocd-server "和 "argocd-application-server "都将使用与 "argocd-repo-server "的非验证连接。

要改变这种行为，让 `argocd-server` 和 `argocd-application-controller` 验证 `argocd-repo-server` 端点的 TLS 证书，使其更加安全，需要执行以下步骤：

* 创建持久 TLS 证书供 `argocd-repo-server` 使用，如上所示
* 重启 `argocd-repo-server` pod
* 修改 `argocd-server` 和 `argocd-application-controller` 的 pod 启动参数，加入 `--repo-server-strict-tls` 参数。

现在，"argocd-server "和 "argocd-application-controller "工作负载将通过使用存储在 "argocd-repo-server-tls "secret 中的证书来验证 "argocd-repo-server "的 TLS 证书。

注意 "证书过期" 请确保证书有适当的有效期。 请记住，当您必须更换证书时，所有工作负载都必须重新启动才能再次正常工作。

### 为 argocd-dex-server 配置 TLS

argocd-server "使用通过 TLS 的 HTTPS API 与 "argocd-dex-server "通信。 默认情况下，"argocd-dex-server "会生成一个非持久、自签名的证书，以便在启动时用于其 HTTPS 端点。 由于 "argocd-dex-server "无法连接到 k8s 控制平面 API，因此外部用户无法使用该证书进行验证。 为此，"argocd-server "将使用一个非验证连接连接到 "argocd-dex-server"。

要改变这种行为，让 `argocd-server` 验证 `argocd-dex-server` 端点的 TLS 证书，使其更加安全，需要执行以下步骤：

* 创建持久 TLS 证书供 `argocd-dex-server` 使用，如上所示
* 重启 `argocd-dex-server` pod
* 修改 `argocd-server` 的 pod 启动参数，以包含

`--dex-server-strict-tls` 参数。

现在，"argocd-server "工作负载将通过使用存储在 "argocd-dex-server-tls "存储秘密中的证书来验证 "argocd-dex-server "的 TLS 证书。

注意 "证书过期" 请确保证书有适当的有效期。 请记住，当您必须更换证书时，所有工作负载都必须重新启动才能再次正常工作。

### 停用连接 argocd-repo-server 的 TLS

在某些涉及通过侧车代理使用 mTLS 的情况下（例如在服务网格中），您可能希望将 `argocd-server` 和 `argocd-application-controller` 到 `argocd-repo-server` 之间的连接配置为完全不使用 TLS。

在这种情况下，您需要

* 在配置 `argocd-repo-server` 时，通过在 pod 容器的启动参数中指定 `--disable-tls`参数，禁用 gRPC API 上的 TLS。此外，还可考虑通过指定 `--listen 127.0.0.1` 参数，将监听地址限制为环回接口，这样不安全的端点就不会暴露在 pod 的网络接口上，但仍可供侧车容器使用。
* 通过在 pod 容器的启动参数中指定参数 `--repo-server-plaintext`，配置 `argocd-server` 和 `argocd-application-controller` 在连接到 `argocd-repo-server` 时不使用 TLS。
* 配置 `argocd-server` 和 `argocd-application-controller` 连接到侧车，而不是直接连接到 `argocd-repo-server` 服务，方法是通过 `--repo-server<address>` 参数指定其地址。

更改后，"argocd-server "和 "argocd-application-controller "将使用纯文本连接到侧车代理，该代理将处理与 "argocd-repo-server "的 TLS 侧车代理之间的所有 TLS 事宜。

### 停用连接 argocd-dex-server 的 TLS

在某些涉及通过侧车代理使用 mTLS 的情况下（例如在服务网格中），您可能希望将 `argocd-server` 与 `argocd-dex-server` 之间的连接配置为完全不使用 TLS。

在这种情况下，您需要

* 通过在 pod 容器的启动参数中指定 `--disable-tls` 参数，配置 `argocd-dex-server` 禁用 HTTPS API 上的 TLS
* 通过在 pod 容器的启动参数中指定 `--dex-server-plaintext` 参数，将 `argocd-server` 配置为不使用 TLS 连接到 `argocd-dex-server` 。
* 配置 `argocd-server` 以连接到边车，而不是直接连接到 `argocd-dex-server` 服务，方法是通过 `--dex-server<address>` 参数指定其地址

更改后，"argocd-server "将使用纯文本连接到侧车代理，它将处理到 "argocd-dex-server "的 TLS 侧车代理的所有方面。