<!-- TRANSLATED by md-translate -->
# 监测

Argo CD 通知控制器通过 9001 端口为 Prometheus 指标提供服务。

注意 可使用 `argocd-notifications-controller` 部署中的 `--metrics-port` flag 更改度量端口。

## 指标

可使用以下指标：

### `argocd_notifications_deliveries_total` 共计

发送的通知数量。 标签：

* template` - 通知模板名称
* `notifier` - 通知服务名称
* `succeeded` - 表示通知发送成功或失败的 flag

### `argocd_notifications_trigger_eval_total` 总计

触发评估次数 标签

* `name` - 触发器名称
* `triggered` - 标志，表示触发条件返回的是真还是假

## 示例

* Grafana 仪表板：[grafana-dashboard.json](grafana-dashboard.json)
