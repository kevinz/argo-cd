<!-- TRANSLATED by md-translate -->
# Pushover

1.在 [pushover.net](https://pushover.net/apps/build) 上创建应用程序。
2.将 API 密钥存储在 `<secret-name>` Secret 中，并在 `<config-map-name>` ConfigMap 中定义秘密名称：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-map-name>
data:
  service.pushover: |
    token: $pushover-token
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
stringData:
  pushover-token: avtc41pn13asmra6zaiyf7dh6cgx97
```

3.将用户密钥添加到应用程序资源中：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.pushover: uumy8u4owy7bgkapp6mc5mvhfsvpcd
```