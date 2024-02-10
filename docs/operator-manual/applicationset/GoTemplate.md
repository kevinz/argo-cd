<!-- TRANSLATED by md-translate -->
# 转到模板

## 简介

ApplicationSet 可以使用 [Go Text Template](https://pkg.go.dev/text/template)。要激活此功能，请在 ApplicationSet 配置清单中添加 `goTemplate: true` 。

除了默认的 Go 文本模板函数外，还可使用 [Sprig 函数库](https://masterminds.github.io/sprig/)（`env`、`expandenv` 和 `getHostByName` 除外）。

附加的 "规范化 "功能可将任何字符串参数作为有效的 DNS 名称使用，方法是用连字符替换无效字符，并在 253 个字符处截断。 这在使参数安全用于应用程序名称等情况时非常有用。

该函数接受几个参数：

* 第一个参数（如果 Provider 提供）是一个整数，用于指定 slug 的最大长度。
* 第二个参数（如有 Providers）是一个布尔值，表示是否启用智能截断。
* 最后一个参数（如果 Providers 提供）是需要被 slug 化的输入名称。

#### Usage 示例

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: test-appset
spec:
  ... 
  template:
    metadata:
      name: 'hellos3-{{.name}}-{{ cat .branch | slugify 23 }}'
      annotations:
        label-1: '{{ cat .branch | slugify }}'
        label-2: '{{ cat .branch | slugify 23 }}'
        label-3: '{{ cat .branch | slugify 50 false }}'
```

如果要自定义[由文本/模板定义的选项](https://pkg.go.dev/text/template#Template.Option)，可以在 ApplicationSet 中的`goTemplate: true` 旁边添加`goTemplateOptions:["opt1", "opt2", ...]`键。请注意，在撰写本文时，只定义了一个有用的选项，即`missingkey=error`。

建议将 `goTemplateOptions` 设置为 `["missingkey=error"]`，这样可以确保在模板查找到未定义的值时报告错误，而不是默默忽略。 为了向后兼容，目前默认行为并非如此。

##激励

Go Template 是字符串模板的 Go 标准，它比 fasttemplate（默认模板引擎）功能更强大，因为它允许执行复杂的模板逻辑。

## 限制

Go 模板是按字段应用的，而且只应用于字符串字段。 以下是一些使用 Go 文本模板无法***的示例：

* 模板布尔字段。
    ```
    ::yaml
      apiVersion: argoproj.io/v1alpha1
      类型ApplicationSet
      spec：
        goTemplate: true
        goTemplateOptions：["missingkey=error"]
        模板：
          spec：
            source：
              helm：
                useCredentials："{{.useCredentials}}"  # 该字段不可模板化，因为它是一个布尔字段。
    ```
* 模板化对象字段：
    ```
    ::yaml
      apiVersion: argoproj.io/v1alpha1
      种类：ApplicationSet
      spec：
        goTemplate: true
        goTemplateOptions：["missingkey=error"]
        模板：
          spec：
            syncPolicy："{{.syncPolicy}}"  # 该字段不可模板化，因为它是一个对象字段。
    ```
* 跨字段使用控制关键字：
    ```
    ::yaml
      apiVersion: argoproj.io/v1alpha1
      种类ApplicationSet
      spec：
        goTemplate: true
        goTemplateOptions：["missingkey=error"]
        模板：
          spec：
            source：
              helm：
                parameters：
                # 这些字段中的每一个都会作为独立模板进行 evaluated，因此第一个模板会出错。
                - name："{{range .parameters}}"
                - name："{{.name}}"
                  value："{{.values}}"
                - name: 丢弃
                  Values："{{end}}"
    ```

## 迁移指南

###Globals

您的所有模板都必须使用 GoTemplate 语法替换参数：

例如：`{{ some.value }}` 变成`{{ .some.value }}`

#### 集群发电机

激活 Go Templating 后，`{{ .metadata }}` 将成为一个对象。

* `{{ metadata.labels.my-label }}` 变成 `{{ index .metadata.labels "my-label" }}`
* `{{ metadata.annotations.my/annotation }}` 变成 `{{ index .metadata.annotations "my/annotation" }}`

### Git 生成器

激活 Go Templating 后，"{{ .path }}"将成为一个对象。 因此，必须对 Git 生成器的模板进行一些修改：

* `{{ path }}` 变成 `{{ .path.path }}`
* `{{ path.basename }}` 变成 `{{ .path.basename }}`
* `{{ path.basenameNormalized }}` 变成 `{{ .path.basenameNormalized }}`
* `{{ path.filename }}` 变成 `{{ .path.filename }}`
* `{{ path.filenameNormalized }}` 变成 `{{ .path.filenameNormalized }}`
* `{{ path[n] }}` 变成 `{{ index .path.segments n }}`
* `{{ Values }}` 如果被引用到文件生成器中，则变为 `{{ .values }}`

下面就是一个例子：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
spec:
  generators:
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      directories:
      - path: applicationset/examples/git-generator-directory/cluster-addons/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

成为

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://github.com/argoproj/argo-cd.git
      revision: HEAD
      directories:
      - path: applicationset/examples/git-generator-directory/cluster-addons/*
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'
```

也可以被引用 Sprig 函数来手动构建路径变量：

| with `goTemplate: false` | with `goTemplate: true` | with `goTemplate: true` + Sprig |
| ------------ | ----------- | --------------------- |
| `{{path}}` | `{{.path.path}}` | `{{.path.path}}` |
| `{{path.basename}}` | `{{.path.basename}}` | `{{base .path.path}}` |
| `{{path.filename}}` | `{{.path.filename}}` | `{{.path.filename}}` |
| `{{path.basenameNormalized}}` | `{{.path.basenameNormalized}}` | `{{normalize .path.path}}` |
| `{{path.filenameNormalized}}` | `{{.path.filenameNormalized}}` | `{{normalize .path.filename}}` |
| `{{path[N]}}` | `-` | `{{index .path.segments N}}` |

## 可用的模板功能

ApplicationSet 控制器提供：

* 除 `env`, `expandenv` 和 `getHostByName` 以外的所有 [sprig](http://masterminds.github.io/sprig/) Go 模板函数
* normalize`：对输入进行消毒，使其符合以下规则：
    1. 不包含超过 253 个字符
    2. 只包含小写字母数字字符、"-"或".
    3. 以字母数字字符开始和结束
* `slugify`：像`normalize`一样进行消毒，并像[introduction](#introduction)部分所述的那样进行智能截断（不会将一个单词截成2个）。
* `toYaml` / `fromYaml` / `fromYamlArray` helm 类似函数

## 示例

#### Go 模板的基本 Usage

本例展示了基本的字符串参数替换。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://1.2.3.4
      - cluster: engineering-prod
        url: https://2.4.6.8
      - cluster: finance-preprod
        url: https://9.8.7.6
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: my-project
      source:
        repoURL: https://github.com/infra-team/cluster-deployments.git
        targetRevision: HEAD
        path: guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

### 未设置参数的回退

对于某些生成器，某个名称的参数可能并不总是会被填充（例如，在 Values 生成器或 git 文件生成器中）。 在这种情况下，您可以使用 Go 模板来提供一个后备值。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://kubernetes.default.svc
      - cluster: engineering-prod
        url: https://kubernetes.default.svc
        nameSuffix: -my-name-suffix
  template:
    metadata:
      name: '{{.cluster}}{{dig "nameSuffix" "" .}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: applicationset/examples/list-generator/guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

此 ApplicationSet 将生成一个名为 "engineering-dev "和另一个名为 "engineering-prod-my-name-ffix "的应用程序。

请注意，未设置的参数是一个错误，因此需要避免查找不存在的属性。 相反，可以使用模板函数，如 `dig` 以默认值进行查找。 如果希望未设置的参数默认为零，可以删除 `goTemplateOptions: ["missingkey=error"]` 或将其设置为 `goTemplateOptions: ["missingkey=invalid"]` 。