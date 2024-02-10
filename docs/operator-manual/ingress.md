<!-- TRANSLATED by md-translate -->
# ingress 配置

Argo CD API 服务器同时运行 gRPC 服务器（被 CLI 引用）和 HTTP/HTTPS 服务器（被用户界面引用）。 这两种协议都通过 argocd-server 服务对象在以下端口上公开：

* 443 - gRPC/HTTPS
* 80 - HTTP（重定向至 HTTPS）

有几种方法可以配置 ingress。

## [Ambassador](https://www.getambassador.io/)

Ambassador Edge Stack 可被用作 Kubernetes ingress 控制器，具有 CLI 和 UI 的[自动 TLS 终止](https://www.getambassador.io/docs/latest/topics/running/tls/#host) 和路由功能。

API 服务器应在禁用 TLS 的情况下运行。 编辑 `argocd-server` 部署，在 argocd-server 命令中添加 `--insecure` 标志，或在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"` [如此处所述](server-commands/additional-configuration-method.md)。 鉴于 `argocd` CLI 在请求的 `host` 头中包含端口号，因此需要 2 个 configmaps。

### 选项 1：为基于主机的路由选择映射 CRD

```yaml
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
  name: argocd-server-ui
  namespace: argocd
spec:
  host: argocd.example.com
  prefix: /
  service: argocd-server:443
---
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
  name: argocd-server-cli
  namespace: argocd
spec:
  # NOTE: the port must be ignored if you have strip_matching_host_port enabled on envoy
  host: argocd.example.com:443
  prefix: /
  service: argocd-server:80
  regex_headers:
    Content-Type: "^application/grpc.*$"
  grpc: true
```

使用 `argocd` CLI 登录：

```shell
argocd login <host>
```

### 选项 2：为基于路径的路由选择映射 CRD

API 服务器必须配置为在非根目录下可用（例如 `/argo-cd`）。 编辑 `argocd-server` 部署，在 argocd-server 命令中添加 `--rootpath=/argo-cd` flag。

```yaml
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
  name: argocd-server
  namespace: argocd
spec:
  prefix: /argo-cd
  rewrite: /argo-cd
  service: argocd-server:443
```

使用 `argocd` CLI 登录，对非 root 路径使用额外的 `--grpc-web-root-path` flag。

```shell
argocd login <host>:<port> --grpc-web-root-path /argo-cd
```

## [Contour](https://projectcontour.io/)

Contour ingress 控制器可在边缘终止 TLS ingress 流量。

Argo CD API 服务器应在禁用 TLS 的情况下运行。 编辑 `argocd-server` 部署，在 argocd-server 容器命令中添加 `--insecure` flag，或在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"[如此处所述](server-commands/additional-configuration-method.md)。

还可以通过部署两个 Contour 实例来提供仅限内部的 ingress 路径和仅限外部的 ingress 路径：一个部署在私有子网 LoadBalancer 服务后面，另一个部署在公有子网 LoadBalancer 服务后面。 私有 Contour 部署将拾取注释为 "kubernetes.io/ingress.class: contour-internal "的 ingresses，公有 Contour 部署将拾取注释为 "kubernetes.io/ingress.class: contour-external "的 ingresses。

这就提供了私下部署 Argo CD UI 的机会，但仍允许 SSO 回调成功。

### 具有多个 ingress 对象和 BYO 证书的私人 Argo CD UI

由于 Contour Ingress 每个 Ingress 对象只支持一个协议，因此要定义三个 Ingress 对象：一个用于私有 HTTP/HTTPS，一个用于私有 gRPC，一个用于公共 HTTPS SSO 回调。

内部 HTTP/HTTPS ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http
  annotations:
    kubernetes.io/ingress.class: contour-internal
    ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: internal.path.to.argocd.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: http
  tls:
  - hosts:
    - internal.path.to.argocd.io
    secretName: your-certificate-name
```

内部 gRPC ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc
  annotations:
    kubernetes.io/ingress.class: contour-internal
spec:
  rules:
  - host: grpc-internal.path.to.argocd.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - grpc-internal.path.to.argocd.io
    secretName: your-certificate-name
```

外部 HTTPS SSO 回调 ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-external-callback-http
  annotations:
    kubernetes.io/ingress.class: contour-external
    ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: external.path.to.argocd.io
    http:
      paths:
      - path: /api/dex/callback
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: http
  tls:
  - hosts:
    - external.path.to.argocd.io
    secretName: your-certificate-name
```

argocd-server 服务需要注释为 `projectcontour.io/upstream-protocol.h2c: "https,443"`，以连接 gRPC 协议代理。

编辑 `argocd-server` 部署，在 argocd-server 命令中添加 `--insecure` flag，或者直接在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"[如此处所述](server-commands/additional-configuration-method.md)。

## [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

### 选项 1：SSL-直通

Argo CD 在同一个端口（443）上为多个协议（gRPC/HTTPS）提供服务，这给尝试为 argocd-service 定义单个 nginx ingress 对象和规则带来了挑战，因为 `nginx.ingress.kubernetes.io/backend-protocol` [annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-protocol) 只接受后端协议的单个值（例如 HTTP、HTTPS、GRPC、GRPCS）。

为了使用单一的 ingress 规则和主机名暴露 Argo CD API 服务器，必须引用 `nginx.ingress.kubernetes.io/ssl-passthrough` [notations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#ssl-passthrough)来直通 TLS 连接，并在 Argo CD API 服务器上终止 TLS。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
```

上述规则会在 Argo CD API 服务器上终止 TLS，该服务器会检测正在使用的协议，并作出适当的响应。 请注意，"nginx.ingress.kubernetes.io/ssl-passthrough "注释要求在 "nginx-ingress-controller "的命令行参数中添加"--enable-ssl-passthrough "标志。

#### 使用 cert-manager 和 Let's Encrypt 的 SSL 直通功能

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    #
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-server-tls # as expected by argocd-server
```

### 选项 2：在 ingress 控制器上终止 SSL

另一种方法是在 Ingress 上执行 SSL 终止。 由于 `ingress-nginx` Ingress 每个 Ingress 对象只支持一个协议，因此需要使用 `nginx.ingress.kubernetes.io/backend-protocol` 注解定义两个 Ingress 对象，一个用于 HTTP/HTTPS，另一个用于 gRPC。

每个 ingress 将用于不同的域（`argocd.example.com` 和 `grpc.argocd.example.com`）。这就要求 ingress 资源使用不同的 TLS `secretName`s 以避免意外行为。

HTTP/HTTPS ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: http
    host: argocd.example.com
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-ingress-http
```

gRPC ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
    host: grpc.argocd.example.com
  tls:
  - hosts:
    - grpc.argocd.example.com
    secretName: argocd-ingress-grpc
```

编辑 `argocd-server` 部署，在 argocd-server 命令中添加 `--insecure` flag，或者直接在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"[如此处所述](server-commands/additional-configuration-method.md)。

这种方法的明显缺点是，API 服务器需要两个独立的主机名--一个用于 gRPC，另一个用于 HTTP/HTTPS。 不过，它允许在 ingress 控制器上终止 TLS。

## [Traefik (v2.2)](https://docs.traefik.io/)

Traefik 可被引用为边缘路由器，并在同一部署中提供 [TLS](https://docs.traefik.io/user-guides/grpc/) 终端。

与 NGINX 相比，它目前的优势在于可以在同一端口上终止 TCP 和 HTTP 连接，这意味着你不需要多个主机或路径。

运行 API 服务器时应禁用 TLS。 编辑 `argocd-server` 部署，在 argocd-server 命令中添加 `--insecure` flag，或在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"`[如此处所述](server-commands/additional-configuration-method.md)。

### 入口路线 CRD

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.example.com`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: Host(`argocd.example.com`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default
```

### AWS 应用程序负载平衡器 (ALB) 和经典 ELB（HTTP 模式）

AWS ALB 可被用作 UI 和 gRPC 流量的 L7 负载平衡器，而 Classic ELB 和 NLB 可被用作 UI 和 gRPC 流量的 L4 负载平衡器。

使用 ALB 时，需要为 argocd-server 创建第二个服务，因为我们需要告诉 ALB 将 GRPC 流量发送到与 UI 流量不同的目标组，因为后端协议是 HTTP2 而不是 HTTP1。

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    alb.ingress.kubernetes.io/backend-protocol-version: HTTP2 #This tells AWS to send traffic from the ALB using HTTP2. Can use GRPC as well if you want to leverage GRPC specific features
  labels:
    app: argogrpc
  name: argogrpc
  namespace: argocd
spec:
  ports:
  - name: "443"
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: NodePort
```

创建此服务后，我们就可以使用 `alb.ingress.kubernetes.io/conditions` 注解配置 Ingress，使其有条件地将所有 `application/grpc` 流量路由到新的 HTTP2 后端，如下所示。 注意：条件注解中 .后面的值必须与您希望流量路由到的服务名称相同，并且会被引用到具有匹配服务名称的任何路径上。

```yaml
apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      alb.ingress.kubernetes.io/backend-protocol: HTTPS
      # Use this annotation (which must match a service name) to route traffic to HTTP2 backends.
      alb.ingress.kubernetes.io/conditions.argogrpc: |
        [{"field":"http-header","httpHeaderConfig":{"httpHeaderName": "Content-Type", "values":["application/grpc"]}}]
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    name: argocd
    namespace: argocd
  spec:
    rules:
    - host: argocd.argoproj.io
      http:
        paths:
        - path: /
          backend:
            service:
              name: argogrpc
              port:
                number: 443
          pathType: Prefix
        - path: /
          backend:
            service:
              name: argocd-server
              port:
                number: 443
          pathType: Prefix
    tls:
    - hosts:
      - argocd.argoproj.io
```

## [Istio](https://www.istio.io)

您可以使用以下配置将 Argo CD 置于 Istio 之后。 在这里，我们将同时实现将 Argo CD 置于 Istio 之后和在 Istio 上使用子路径这两个目标

首先，我们需要确保可以通过子路径（即 /argocd）运行 Argo CD。 为此，我们被引用了 argocd 项目中的 install.yaml 文件，如下所示

```bash
curl -kLs -o install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

将以下文件保存为 kustomize.yml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./install.yaml

patches:
- path: ./patch.yml
```

以下各行作为 patch.yml

```yaml
# Use --insecure so Ingress can send traffic with HTTP
# --bashref /argocd is the subpath like https://IP/argocd
# env was added because of https://github.com/argoproj/argo-cd/issues/3572 error
---
apiVersion: apps/v1
kind: Deployment
metadata:
 name: argocd-server
spec:
 template:
   spec:
     containers:
     - args:
       - /usr/local/bin/argocd-server
       - --staticassets
       - /shared/app
       - --redis
       - argocd-redis-ha-haproxy:6379
       - --insecure
       - --basehref
       - /argocd
       - --rootpath
       - /argocd
       name: argocd-server
       env:
       - name: ARGOCD_MAX_CONCURRENT_LOGIN_REQUESTS_COUNT
         value: "0"
```

然后安装 Argo CD（当前目录下应该只有上面定义的 3 个 yml 文件）

```bash
kubectl apply -k ./ -n argocd --wait=true
```

确保为 Isito 创建了 Secret（在我们的例子中，secretname 是 argocd 名称空间中的 argocd-server-tls）。 之后，我们创建 Istio 资源

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: argocd-gateway
  namespace: argocd
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
    tls:
     httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*"
    tls:
      credentialName: argocd-server-tls
      maxProtocolVersion: TLSV1_3
      minProtocolVersion: TLSV1_2
      mode: SIMPLE
      cipherSuites:
        - ECDHE-ECDSA-AES128-GCM-SHA256
        - ECDHE-RSA-AES128-GCM-SHA256
        - ECDHE-ECDSA-AES128-SHA
        - AES128-GCM-SHA256
        - AES128-SHA
        - ECDHE-ECDSA-AES256-GCM-SHA384
        - ECDHE-RSA-AES256-GCM-SHA384
        - ECDHE-ECDSA-AES256-SHA
        - AES256-GCM-SHA384
        - AES256-SHA
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: argocd-virtualservice
  namespace: argocd
spec:
  hosts:
  - "*"
  gateways:
  - argocd-gateway
  http:
  - match:
    - uri:
        prefix: /argocd
    route:
    - destination:
        host: argocd-server
        port:
          number: 80
```

现在我们可以浏览 http://{{ IP }}/argocd （它将被改写为 https://{{ IP }}/argocd

## 使用 Kubernetes ingress 的谷歌云负载平衡器

您可以利用 GKE 与 Google Cloud 的集成，只需引用 Kubernetes 对象就能部署负载平衡器。

为此，我们需要这五个对象：

* A 服务
* 一个后台配置
* 前端配置
* 带有 SSL 证书的秘密
* 一个 GKE 的 ingress

如果您需要详细了解这些 Google 集成的所有可用选项，可以查看 [关于配置 ingress 功能的 Google 文档](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-features)

### 禁用内部 TLS

首先，为避免从 HTTP 到 HTTPS 的内部重定向循环，应禁用 TLS 运行 API 服务器。

编辑 argocd-server 部署的 `argocd-server` 命令中的 `--insecure` flag，或在 `argocd-cmd-params-cm` ConfigMap [如此处所述](server-commands/additional-configuration-method.md) 中简单设置 `server.insecure: "true"。

### 创建服务

现在，您需要一个可从外部访问的服务。 这实际上与 Argo CD 的内部服务相同，但使用了 Google Cloud 注释。请注意，该服务被引用为使用 [Network Endpoint Group](https://cloud.google.com/load-balancing/docs/negs) (NEG)，以允许负载平衡器直接向您的 pod 发送流量，而无需使用 kube-proxy，因此如果您不希望这样，请移除 `neg` 注释。

服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"ports": {"http":"argocd-backend-config"}}'
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

### 创建 BackendConfig

看到之前的服务被引用了一个名为 `argocd-backend-config` 的后端配置了吗？ 让我们用这个 yaml 来部署它：

```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: argocd-backend-config
  namespace: argocd
spec:
  healthCheck:
    checkIntervalSec: 30
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 2
    type: HTTP
    requestPath: /healthz
    port: 8080
```

它被引用的健康检查与豆荚相同。

### 创建 FrontendConfig

现在，我们可以部署一个 HTTP 到 HTTPS 重定向的前端配置：

```yaml
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: argocd-frontend-config
  namespace: argocd
spec:
  redirectToHttps:
    enabled: true
```

---

注意

```
The next two steps (the certificate secret and the Ingress) are described supposing that you manage the certificate yourself, and you have the certificate and key files for it. In the case that your certificate is Google-managed, fix the next two steps using the [guide to use a Google-managed SSL certificate](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs#creating_an_ingress_with_a_google-managed_certificate).
```

---

### 创建证书秘密

现在，我们需要在负载平衡器中创建一个包含 SSL 证书的 Secret。 在存储证书密钥的路径上执行此命令即可：

```
kubectl -n argocd create secret tls secret-yourdomain-com \
  --cert cert-file.crt --key key-file.key
```

### 创建一个 ingress

最后，最重要的是我们的 ingress。 请注意我们的前端配置、服务和证书秘密的引用。

---

注意

运行版本早于 `1.21.3-gke.1600`的 GKE 集群，[pathType 字段唯一支持的值](https://cloud.google.com/kubernetes-engine/docs/how-to/load-balance-ingress#creating_an_ingress) 是 `ImplementationSpecific`。因此，您必须检查 GKE 集群的版本。您需要根据版本使用不同的 YAML。

---

如果使用的版本早于 `1.21.3-gke.1600`，则应使用以下 ingress 资源：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    networking.gke.io/v1beta1.FrontendConfig: argocd-frontend-config
spec:
  tls:
    - secretName: secret-example-com
  rules:
    - host: argocd.example.com
      http:
        paths:
        - pathType: ImplementationSpecific
          path: "/*"   # "*" is needed. Without this, the UI Javascript and CSS will not load properly
          backend:
            service:
              name: argocd-server
              port:
                number: 80
```

如果使用 `1.21.3-gke.1600` 或更高版本，则应使用以下 ingress 资源：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    networking.gke.io/v1beta1.FrontendConfig: argocd-frontend-config
spec:
  tls:
    - secretName: secret-example-com
  rules:
    - host: argocd.example.com
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: argocd-server
              port:
                number: 80
```

您可能已经知道，部署负载平衡器并准备好接受连接可能需要几分钟时间。 准备就绪后，获取负载平衡器的公共 IP 地址，转到 DNS 服务器（谷歌或第三方），将您的域名或子域（如 argocd.example.com）指向该 IP 地址。

您可以像这样获取描述 ingress 对象的 IP 地址：

```
kubectl -n argocd describe ingresses argocd | grep Address
```

DNS 更改传播后，您就可以将 Argo 与 Google Cloud Load Balancer 结合使用了。

## 通过多层验证反向代理进行验证

Argo CD 端点可能受到一个或多个反向代理层的保护，在这种情况下，您可以通过 `argocd` CLI `--header` 参数提供额外的标头，以便通过这些层进行身份验证。

```shell
$ argocd login <host>:<port> --header 'x-token1:foo' --header 'x-token2:bar' # can be repeated multiple times
$ argocd login <host>:<port> --header 'x-token1:foo,x-token2:bar' # headers can also be comma separated
```

## ArgoCD 服务器和用户界面根路径（v1.5.3）

Argo CD 服务器和用户界面可配置为在非根路径下可用（例如 `/argo-cd`）。 为此，请在 `argocd-server` 部署命令中添加 `--rootpath` 标志：

```yaml
spec:
  template:
    spec:
      name: argocd-server
      containers:
      - command:
        - /argocd-server
        - --repo-server
        - argocd-repo-server:8081
        - --rootpath
        - /argo-cd
```

注意：flag `--rootpath`会同时更改 API 服务器和用户界面的基本 URL。 nginx.conf 示例：

```
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    server {
        listen 443;

        location /argo-cd/ {
            proxy_pass https://localhost:8080/argo-cd/;
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            # buffering should be disabled for api/v1/stream/applications to support chunked response
            proxy_buffering off;
        }
    }
}
```

flag `--grpc-web-root-path`用于提供非根目录路径（如 /argo-cd）

```shell
$ argocd login <host>:<port> --grpc-web-root-path /argo-cd
```

## UI 基本路径

如果 Argo CD 用户界面在非根目录下可用（例如，"/argo-cd "而非"/"），则应在 API 服务器中配置用户界面路径。 要配置用户界面路径，请在 "argocd-server "部署命令中添加"--basehref "相应的标志：

```yaml
spec:
  template:
    spec:
      name: argocd-server
      containers:
      - command:
        - /argocd-server
        - --repo-server
        - argocd-repo-server:8081
        - --basehref
        - /argo-cd
```

注意：flag `--basehref`只能更改用户界面的基本 URL。 API 服务器将继续使用 `/` 路径，因此需要在代理配置中添加 URL 重写规则。带有 URL 重写的 nginx.conf 示例：

```
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    server {
        listen 443;

        location /argo-cd {
            rewrite /argo-cd/(.*) /$1 break;
            proxy_pass https://localhost:8080;
            proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host $server_name;
            # buffering should be disabled for api/v1/stream/applications to support chunked response
            proxy_buffering off;
        }
    }
}
```