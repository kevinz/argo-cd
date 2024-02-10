<!-- TRANSLATED by md-translate -->
# 电报

1.使用 [@Botfather](https://t.me/Botfather) 获取 API 令牌。
2.将令牌存储在 `<secret-name>` Secret 中，并配置 telegram 集成

in `<config-map-name>` configMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.telegram: |
    token: $telegram-token
```

3.创建新的 Telegram [频道](https://telegram.org/blog/channels)。
4.将您的机器人添加为管理员。
5.在订阅 Telegram 整合时，使用此频道 `username`（公共频道）或 `chatID`（私人频道）：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.telegram: username
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.telegram: -1000000000000
```