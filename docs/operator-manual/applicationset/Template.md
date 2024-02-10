<!-- TRANSLATED by md-translate -->
# 模板

ApplicationSet `spec` 的模板字段被引用来生成 Argo CD `Application` 资源。

ApplicationSet 被引用 [fasttemplate](https://github.com/valyala/fasttemplate) 但很快就会被弃用，改用 Go Template。

## 模板字段

Argo CD 应用程序是通过将生成器中的参数与模板中的字段（通过 `{{values}}`）相结合而创建的，并由此生成一个具体的 `Application` 资源并应用于集群。

下面是集群生成器的模板子字段：

```yaml
# (...)
 template:
   metadata:
     name: '{{cluster}}-guestbook'
   spec:
     source:
       repoURL: https://github.com/infra-team/cluster-deployments.git
       targetRevision: HEAD
       path: guestbook/{{cluster}}
     destination:
       server: '{{url}}'
       namespace: guestbook
```

模板子字段直接对应于[Argo CD `Application` 资源的规格]（.../.../声明式设置/#applications）：

* project "指的是被引用的[Argo CD 项目]（../../user-guide/projects.md）（"default "可用于使用默认的 Argo CD 项目）
* source "定义从哪个 Git 仓库提取所需的应用程序配置清单
    - **repoURL**：版本库的 URL（例如 `https://github.com/argoproj/argocd-example-apps.git`)
    - **targetRevision**：版本库的修订版本（标签/分支/提交）（例如 `HEAD`)
    - **path**：版本库中 Kubernetes 配置清单（和/或 Helm、Kustomize、Jsonnet 资源）所在的路径
* 目的地定义要部署到哪个 Kubernetes 集群/名称空间
    - **name**：要部署到的集群（Argo CD 中）的名称
    - **服务器***：集群的 API 服务器 URL（例如：`https://kubernetes.default.svc`）。
    - **namespace**：要从 `source` 中配置清单的目标名称空间（例如：`my-app-namespace`）。

请注意：

* 被引用的集群必须已在 Argo CD 中定义，这样 ApplicationSet 控制器才能使用它们。
* 只能指定 `name` 或 `server` 中的***一个：如果同时指定两个，将返回错误信息。

模板的 `metadata` 字段也可被用来设置应用程序的 `name` 或为应用程序添加标签或 Annotations。

虽然 ApplicationSet 规范提供了一种基本的模板形式，但它并不打算取代 kustomize、Helm 或 Jsonnet 等工具的全面配置管理功能。

### 部署作为 Helm 图表一部分的 ApplicationSet 资源

ApplicationSet 使用与 Helm 相同的模板符号 (`{{}}`)。 如果 ApplicationSet 模板没有写成 Helm 字符串字面量，Helm 会抛出类似`函数 "集群 "未定义`的错误。 为避免出现该错误，请将模板写成 Helm 字符串字面量。 例如：

```yaml
metadata:
      name: '{{`{{.cluster}}`}}-guestbook'
```

这_仅_适用于使用 Helm 部署 ApplicationSet 资源的情况。

## 生成器模板

除了在 `ApplicationSet` 资源的 `.spec.template` 中指定模板外，还可在生成器中指定模板。 这对于覆盖 `spec` 级模板的值非常有用。

生成器的 "模板 "字段优先于 "规格 "的模板字段：

* 如果两个模板都包含相同的字段，则会引用生成器的字段值。
* 如果只有其中一个模板的字段有值，则将引用该值。

因此，生成器模板可被视为针对外部 "规格 "级模板字段的补丁。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
      template:
        metadata: {}
        spec:
          project: "default"
          source:
            revision: HEAD
            repoURL: https://github.com/argoproj/argo-cd.git
            # New path value is generated here:
            path: 'applicationset/examples/template-override/{{cluster}}-override'
          destination: {}

  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        # This 'default' value is not used: it is is replaced by the generator's template path, above
        path: applicationset/examples/template-override/default
      destination:
        server: '{{url}}'
        namespace: guestbook
```

（_完整示例见 [此处](https://github.com/argoproj/argo-cd/tree/master/applicationset/examples/template-override)。_）

在本例中，ApplicationSet 控制器将使用 List 生成器生成的 `path` 而不是 `.spec.template` 中定义的 `path` 值生成一个 `Application` 资源。

## 模板补丁

模板化仅适用于字符串类型，但某些用例可能需要在其他类型上应用模板化。

例如

* 有条件地设置自动同步策略。
* 有条件地将剪枝布尔值切换为 `true`。
* 从列表中添加多个 helm 值文件。

模板补丁 "功能可实现高级模板化，支持 "json "和 "yaml"。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  goTemplate: true
  generators:
  - list:
      elements:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
          autoSync: true
          prune: true
          valueFiles:
            - values.large.yaml
            - values.debug.yaml
  template:
    metadata:
      name: '{{.cluster}}-deployment'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/infra-team/cluster-deployments.git
        targetRevision: HEAD
        path: guestbook/{{ .cluster }}
      destination:
        server: '{{.url}}'
        namespace: guestbook
  templatePatch: |
    spec:
      source:
        helm:
          valueFiles:
          {{- range $valueFile := .valueFiles }}
            - {{ $valueFile }}
          {{- end }}
    {{- if .autoSync }}
      syncPolicy:
        automated:
          prune: {{ .prune }}
    {{- end }}
```

重要 `templatePatch` 可以对模板进行任意修改。 如果参数中包含不可信的用户输入，就有可能向模板中注入恶意修改。 建议仅对可信输入使用 `templatePatch` 或在模板中使用前仔细转义输入。 将输入导入 `toJson` 应有助于防止用户成功注入带换行的字符串等。

```
The `spec.project` field is not supported in `templatePatch`. If you need to change the project, you can use the
`spec.project` field in the `template` field.
```

!!! 重要 当编写 `templatePatch` 时，您正在制作一个补丁。 因此，如果补丁包含一个空的 `spec: # nothing in here`，它将有效地清除现有字段。 有关此行为的示例，请参阅 [#17040](https://github.com/argoproj/argo-cd/issues/17040)。