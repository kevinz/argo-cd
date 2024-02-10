<!-- TRANSLATED by md-translate -->
# 衡量标准

Argo CD 为每台服务器提供不同的 Prometheus 指标集。

## 应用程序控制器指标

从 `argocd-metrics:8082/metrics` 端点获取的应用程序指标。

| Metric | Type | Description |
|--------|:----:|-------------|
| `argocd_app_info` | gauge | Information about Applications. It contains labels such as `sync_status` and `health_status` that reflect the application state in Argo CD. |
| `argocd_app_k8s_request_total` | counter | Number of Kubernetes requests executed during application reconciliation |
| `argocd_app_labels` | gauge | Argo Application labels converted to Prometheus labels. Disabled by default. See section below about how to enable it. |
| `argocd_app_reconcile` | histogram | Application reconciliation performance. |
| `argocd_app_sync_total` | counter | Counter for application sync history |
| `argocd_cluster_api_resource_objects` | gauge | Number of k8s resource objects in the cache. |
| `argocd_cluster_api_resources` | gauge | Number of monitored Kubernetes API resources. |
| `argocd_cluster_cache_age_seconds` | gauge | Cluster cache age in seconds. |
| `argocd_cluster_connection_status` | gauge | The k8s cluster current connection status.

如果您在使用 Argo CD 时创建和删除了大量应用程序和项目，那么度量页面将缓存您的应用程序和项目的历史记录。 如果由于删除资源导致大量度量卡入度出现问题，您可以使用应用程序控制器 flag 安排度量重置以清理历史记录。 示例：`-metrics-cache-expiration="24h0m0s"`。

### 将应用程序标签作为 Prometheus 指标公开

在一些用例中，Argo CD 应用程序所包含的标签希望作为 Prometheus 指标被显示出来。 一些例子如下：

* 将团队名称作为标签，以便将警报发送给特定接收方
* 创建按业务单元细分的仪表板

由于应用程序标签是每个公司特有的，因此该功能默认为禁用。 要启用该功能，请在 Argo CD 应用程序控制器中添加"--metrics-application-labels "标志。

下面的示例将向 Prometheus 公开 Argo CD 应用程序标签 "team-name "和 "business-unit"：

```
containers:
- command:
  - argocd-application-controller
  - --metrics-application-labels
  - team-name
  - --metrics-application-labels
  - business-unit
```

在这种情况下，度量标准就会是这样：

```
# TYPE argocd_app_labels gauge
argocd_app_labels{label_business_unit="bu-id-1",label_team_name="my-team",name="my-app-1",namespace="argocd",project="important-project"} 1
argocd_app_labels{label_business_unit="bu-id-1",label_team_name="my-team",name="my-app-2",namespace="argocd",project="important-project"} 1
argocd_app_labels{label_business_unit="bu-id-2",label_team_name="another-team",name="my-app-3",namespace="argocd",project="important-project"} 1
```

## API 服务器指标

有关 API 服务器 API 请求和响应活动的指标（请求总数、响应代码等）。 从 `argocd-server-metrics:8083/metrics` 端点抓取。

| Metric | Type | Description |
|--------|:----:|-------------|
| `argocd_redis_request_duration` | histogram | Redis requests duration. |
| `argocd_redis_request_total` | counter | Number of Kubernetes requests executed during application reconciliation. |
| `grpc_server_handled_total` | counter | Total number of RPCs completed on the server, regardless of success or failure. |
| `grpc_server_msg_sent_total` | counter | Total number of gRPC stream messages sent by the server. |
| `argocd_proxy_extension_request_total` | counter | Number of requests sent to the configured proxy extensions. |
| `argocd_proxy_extension_request_duration_seconds` | histogram | Request duration in seconds between the Argo CD API server and the proxy extension backend. |

## Repo 服务器指标

Repo 服务器的度量指标。 从 `argocd-repo-server:8084/metrics` 端点抓取。

| Metric | Type | Description |
|--------|:----:|-------------|
| `argocd_git_request_duration_seconds` | histogram | Git requests duration seconds. |
| `argocd_git_request_total` | counter | Number of git requests performed by repo server |
| `argocd_redis_request_duration_seconds` | histogram | Redis requests duration seconds. |
| `argocd_redis_request_total` | counter | Number of Kubernetes requests executed during application reconciliation. |
| `argocd_repo_pending_request_total` | gauge | Number of pending requests requiring repository lock |

## 普罗米修斯操作员

如果使用 Prometheus Operator，可使用以下 ServiceMonitor 配置清单示例。 添加 Argo CD 安装所在的名称空间，并将 `metadata.labels.release` 更改为 Prometheus 选择的标签名称。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  endpoints:
  - port: metrics
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  endpoints:
  - port: metrics
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-applicationset-controller-metrics
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-applicationset-controller
  endpoints:
  - port: metrics
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-dex-server
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-dex-server
  endpoints:
    - port: metrics
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-redis-haproxy-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-redis-ha-haproxy
  endpoints:
  - port: http-exporter-port
```

对于通知控制器，您需要额外添加以下内容：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-notifications-controller
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-notifications-controller-metrics
  endpoints:
    - port: metrics
```

## 仪表板

您可以找到 Grafana 仪表盘示例 [此处](https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json) 或查看演示实例 [仪表盘](https://grafana.apps.argoproj.io)。

![dashboard](../assets/dashboard.jpg)