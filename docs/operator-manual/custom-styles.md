<!-- TRANSLATED by md-translate -->
# 自定义样式

Argo CD 的大部分用户界面样式表都是从 [argo-ui](https://github.com/argoproj/argo-ui) 项目中导入的。有时，可能需要自定义用户界面的某些组件，以达到品牌推广的目的，或帮助区分在不同环境中运行的多个 Argo CD 实例。

这种自定义样式可以通过提供远程托管 CSS 文件的 URL 或直接在 argocd-server 容器上加载 CSS 文件来应用。 这两种机制都是通过修改 argocd-cm configMap 来驱动的。

## 通过远程 URL 添加样式

第一种方法只需在 argocd-cm configMap 中添加远程 URL：

### argocd-cm

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: argocd-cm
data:
  ui.cssurl: "https://www.example.com/my-styles.css"
```

## 通过卷挂载添加样式

第二种方法需要将 CSS 文件直接挂载到 argocd-server 容器上，然后向 argocd-cm 提供该文件的正确配置路径。 在下面的示例中，CSS 文件实际上是在单独的 configMap 中定义的（在 initContainer 中生成或下载 CSS 文件也能达到同样的效果）：

### argocd-cm

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: argocd-cm
data:
  ui.cssurl: "./custom/my-styles.css"
```

请注意，"cssurl "应指定为"/shared/app "目录的相对路径，而不是绝对路径。

### argocd-styles-cm

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: argocd-styles-cm
data:
  my-styles.css: |
    .sidebar {
      background: linear-gradient(to bottom, #999, #777, #333, #222, #111);
    }
```

### argocd-server

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  ...
spec:
  template:
    ...
    spec:
      containers:
      - command:
        ...
        volumeMounts:
        ...
        - mountPath: /shared/app/custom
          name: styles
      ...
      volumes:
      ...
      - configMap:
          name: argocd-styles-cm
        name: styles
```

请注意，CSS 文件应安装在"/shared/app "目录下的一个子目录中（例如"/shared/app/custom"）。 否则，浏览器很可能无法导入该文件，并出现 "MIME 类型不正确 "的错误。 该子目录可通过 [argocd-cmd-params-cm.yaml](./argocd-cmd-params-cm.yaml) 配置表中的 "server.staticassets "键更改。

## 开发风格叠加

注入的 CSS 文件中指定的样式应与 [argo-ui](https://github.com/argoproj/argo-ui) 中定义的组件和类相关。建议首先使用浏览器内置的开发工具测试要应用的样式。要获得功能更全面的体验，不妨使用 [Argo CD UI 开发服务器](https://webpack.js.org/configuration/dev-server/) 构建一个单独的项目。

## 标语

Argo CD 可以有选择地显示横幅，用于通知用户即将到来的维护和操作更改。 通过在 `argocd-cm` 配置地图中使用 `ui.bannercontent` 字段指定横幅信息，即可启用此功能，Argo CD 将在每个用户界面页面的顶部显示此信息。 您可以通过设置 `ui.bannerurl` 有选择地添加此信息的链接。您还可以通过设置 `ui.bannerpermanent` 为 true 使横幅具有粘性（永久性），并通过使用 `ui.bannerposition: "both" 将其位置更改为 "both "或 "bottom"，使横幅同时显示在顶部和底部，或使用 `ui.bannerposition: "bottom" 将其仅显示在底部。

### argocd-cm

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  ...
  name: argocd-cm
data:
    ui.bannercontent: "Banner message linked to a URL"
    ui.bannerurl: "www.bannerlink.com"
    ui.bannerpermanent: "true"
    ui.bannerposition: "bottom"
```

![带有链接的横幅](../assets/banner.png)