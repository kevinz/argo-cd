<!-- TRANSLATED by md-translate -->
# 动态集群分布

_当前状态：[alpha](https://github.com/argoproj/argoproj/blob/master/community/feature-status.md) （自 v2.9.0 起）_

默认情况下，集群会被无限期地分配给分片。 对于使用基于哈希算法的默认分片算法的用户来说，这种静态分配是没有问题的：分片将始终按照基于哈希算法的算法进行大致平衡。 但对于使用 [round-robin](high_availability.md#argocd-application-controller) 或其他自定义分片分配算法的用户来说，这种静态分配可能会在添加或删除副本时导致分片不平衡。

从 v2.9 版开始，Argo CD 支持动态集群分布功能。 当添加或删除副本时，分片算法会重新运行，以确保集群按照算法分布。 如果算法是均衡的，比如轮循算法，那么分片也会是均衡的。

以前，分片计数是通过 `ARGOCD_CONTROLLER_REPLICAS` 环境变量设置的。 更改环境变量会强制重启所有应用程序控制器 pod。 现在，分片计数是通过部署的 `replicas` 字段设置的，不需要重启应用程序控制器 pod。

## 启用集群的动态分配

该功能在 alpha 阶段默认为禁用。 为了使用该功能，可将配置清单 `manifests/ha/base/controller-deployment/` 作为 kustomize overlay 应用。 该 overlay 会将 StatefulSet 复制设置为 `0`，并将应用程序控制器作为部署部署。 此外，在将应用程序控制器作为部署运行时，必须将环境 `ARGOCD_ENABLE_DYNAMIC_CLUSTER_DISTRIBUTION` 设置为 true。

使用部署而不是 StatefulSet 是一个实现细节，在此功能的未来版本中可能会发生变化。 因此，kustomize 叠加的目录名称也可能会发生变化。 请关注发布说明以避免出现问题。

请注意引入了新的环境变量 `ARGOCD_CONTROLLER_HEARTBEAT_TIME`。 该环境变量在 [动态分配心跳进程的工作](#working-of-dynamic-distribution) 中有解释。

## 动态分配的工作原理

为了完成集群的运行时分配，应用程序控制器会被引用一个 configmaps 来将控制器 pod 与分片编号和心跳关联起来，以确保控制器 pod 仍然存活并处理其分片，实际上就是处理其分担的工作。

应用程序控制器将创建一个名为 "argocd-app-controller-shard-cm "的新 ConfigMap，用于存储控制器<-> Shard 映射。每个 Shard 的映射如下所示：

```yaml
ShardNumber    : 0
ControllerName : "argocd-application-controller-hydrxyt"
HeartbeatTime  : "2009-11-17 20:34:58.651387237 +0000 UTC"
```

* 控制器名称存储应用程序控制器 pod 的主机名
* ShardNumber：存储控制器 pod 管理的分片编号
* 心跳时间存储上次更新心跳的时间。

Controller Shard Mapping 会在 pod 的每次就绪探针检查期间更新 ConfigMap，即每 10 秒更新一次（其他情况按配置进行）。 控制器会在每次就绪探针检查迭代期间获取 Shard，并尝试用 `HeartbeatTime` 更新 ConfigMap。 更新心跳的默认 `HeartbeatDuration` 为 `10` 秒。 如果任何控制器 pod 的 ConfigMap 未更新时间超过 `3 * HeartbeatDuration` ，则应用程序 pod 的就绪探针会被标记为 `不健康'。 要增加默认 `HeartbeatDuration` ，可将环境变量 `ARGOCD_CONTROLLER_HEARTBEAT_TIME` 设置为所需值。

新的分片机制不会监控环境变量 `ARGOCD_CONTROLLER_REPLICAS`，而是直接从应用程序控制器部署中读取副本数量。 控制器通过比较应用程序控制器部署中的副本数量和 `argocd-app-controller-shard-cm` ConfigMap 中的映射数量，来确定副本数量的变化。

在应用程序控制器副本数量增加的情况下，"argocd-app-controller-shard-cm "ConfigMap 中的映射列表中会添加一个新条目，并触发集群分布以重新分配集群。

在应用控制器副本数量减少的情况下，"argocd-app-controller-shard-cm "ConfigMap 中的映射将被重置，每个控制器将再次获取分片，从而触发集群的重新分配。