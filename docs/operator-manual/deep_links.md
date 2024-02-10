<!-- TRANSLATED by md-translate -->
# 深度链接

深度链接允许用户从 Argo CD 用户界面快速重定向到第三方系统，如 Splunk、Datadog 等。

Argo CD 管理员可以通过 Provider-cm 中配置的深度链接模板来配置与第三方系统的链接。 这些模板可以有条件地渲染，并能够引用与链接显示位置相关的不同类型资源，其中包括项目、应用程序或单个资源（pod、服务等）。

## 配置深度链接

深度链接的配置在 `argocd-cm` 中以 `<location>.links` 字段的形式存在，其中 `<location>` 决定在哪里显示。 `<location>` 的可能值是：

* 项目"：该字段下的所有链接将显示在 Argo CD 用户界面的项目选项卡中
* 应用程序"：该字段下的所有链接将显示在应用程序摘要选项卡中
* 资源"：此字段下的所有链接将显示在资源（部署、pod、服务等）摘要选项卡中

列表中的每个链接都有五个子字段：

1.title"： 该链接对应的用户界面中将显示的标题/标记
2.`url`：深层链接将重定向到的实际 URL，该字段可以模板化，以使用相应应用程序、项目或资源对象（取决于其位置）中的数据。这被引用 [text/template](https://pkg.go.dev/text/template) pkg 进行模板化
3.description`（可选）：关于深层链接的描述
4.icon.class`（可选）：在下拉菜单中显示链接时被引用的字体漂亮的图标类
5.if`（可选）：结果为`true`或`false`的条件语句，它也可以访问与`url`字段相同的数据。如果条件解析为 `true`，则显示深层链接，否则隐藏。如果省略该字段，默认情况下将显示深层链接。这被引用 [antonmedv/expr](https://github.com/antonmedv/expr/tree/master/docs) 来评估条件

注意：对于 Secret 类型的资源，数据字段被编辑，但其他字段可用于模板化深层链接。

警告 请确保验证 url 模板和输入，以防止数据泄漏或可能生成任何恶意链接。

如前所述，链接和条件可以通过模板化来使用资源中的数据，每类链接都可以访问与该资源链接的不同类型的数据。 总体而言，我们在系统中拥有这 4 种可供模板化的资源：

* app "或 "application"：该键被引用用于访问应用程序资源数据。
* `resource`: 该键被用来引用实际 k8s 资源的值。
* `cluster`: 该键用于访问相关的目标集群数据，如名称、服务器、namespace 等。
* project`：该键被引用用于访问项目资源数据。

可通过特定链接类别访问上述资源，以下是每个类别的可用资源列表：

* 资源链接"："资源"、"应用程序"、"集群 "和 "项目
* `application.links`: `app`/`application` 和 `cluster
* `project.links`: `project` 项目

包含深度链接及其变化的示例文件 `argocd-cm.yaml` ：

```yaml
# sample project level links
  project.links: |
    - url: https://myaudit-system.com?project={{.project.metadata.name}}
      title: Audit
      description: system audit logs
      icon.class: "fa-book"
  # sample application level links
  application.links: |
    # pkg.go.dev/text/template is used for evaluating url templates
    - url: https://mycompany.splunk.com?search={{.app.spec.destination.namespace}}&env={{.project.metadata.labels.env}}
      title: Splunk
    # conditionally show link e.g. for specific project
    # github.com/antonmedv/expr is used for evaluation of conditions
    - url: https://mycompany.splunk.com?search={{.app.spec.destination.namespace}}
      title: Splunk
      if: application.spec.project == "default"
    - url: https://{{.app.metadata.annotations.splunkhost}}?search={{.app.spec.destination.namespace}}
      title: Splunk
      if: app.metadata.annotations.splunkhost != ""
  # sample resource level links
  resource.links: |
    - url: https://mycompany.splunk.com?search={{.resource.metadata.name}}&env={{.project.metadata.labels.env}}
      title: Splunk
      if: resource.kind == "Pod" || resource.kind == "Deployment"
```