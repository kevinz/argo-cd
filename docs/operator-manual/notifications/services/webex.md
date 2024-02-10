<!-- TRANSLATED by md-translate -->
# Webex 团队

## 参数

Webex Teams 通知服务配置包括以下设置：

* `token` - 应用程序令牌

## 配置

1.创建 Webex [机器人](https://developer.webex.com/docs/bots)
2.复制机器人访问 [token](https://developer.webex.com/my-apps) 并将其存储在 `argocd-notifications-secret` Secret 中，并在 `argocd-notifications-cm` ConfigMap 中配置 Webex Teams 集成
    yaml
    apiVersion: v1
    类型: Secret
    元数据：
    name：<secret-name>
    stringData：
    webex-token：<bot access token>
    `````` yaml
    apiVersion: v1
    kind：configmaps
    metadata：
    name：<config-map-name>
    data：
    service.webex：|
        token: $webex-token
    ```
3.为 Webex Teams 集成创建订阅
    ``` yaml
    apiVersion: argoproj.io/v1alpha1
    类型：应用程序
    元数据：
    Annotations：
        notifications.argoproj.io/subscribe.<trigger-name>.webex：<personal email or room id>
    ```
