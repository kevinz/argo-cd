<!-- TRANSLATED by md-translate -->
# Git 文件生成器 Globbing

## 问题陈述

Git 文件生成器的原始默认实现会进行非常贪婪的全局搜索。 这可能会引发错误或让用户措手不及。 例如，请看下面的版本库布局：

```
└── cluster-charts/
    ├── cluster1
    │   ├── mychart/
    │   │   ├── charts/
    │   │   │    └── mysubchart/
    │   │   │        ├── values.yaml
    │   │   │        └── etc…
    │   │   ├── values.yaml
    │   │   └── etc…
    │   └── myotherchart/
    │       ├── values.yaml
    │       └── etc…
    └── cluster2
        └── etc…
```

在 "群组 1 "中，我们有两个图表，其中一个带有子图表。

假设我们需要 ApplicationSet 来模板`values.yaml`中的值，那么我们需要使用 Git 文件生成器而不是目录生成器。 Git 文件生成器的`path`键的值应设置为

```
path: cluster-charts/*/*/values.yaml
```

不过，默认实现会将上述模式解释为

```
path: cluster-charts/**/values.yaml
```

这意味着，对于 `cluster1` 中的 `mychart` 来说，它既会获取图表的 `values.yaml` 也会获取其子图表的 `values.yaml` 。 这很可能会失败，即使不失败也是错误的。

这种不可取的套色还有多种失败方式，例如

```
path: some-path/*.yaml
```

这将返回 `some-path` 下任何层级任何目录中的所有 YAML 文件，而不是只返回其正下方的文件。

## 启用新的 Globbing

由于一些用户可能会依赖旧的行为，因此决定将修复作为可选项，而不是默认启用。

它可以通过以下任何一种方式启用：

1.将 `--enable-new-git-file-globbing` 传递给 ApplicationSet 控制器参数。
2.在 ApplicationSet 控制器环境变量中设置 `ARGOCD_APPLICATIONSET_CONTROLLER_ENABLE_NEW_GIT_FILE_GLOBBING=true` 。
3.在 Argo CD 配置表中设置`applicationsetcontroller.enable.new.git.file.globbing:true`。

请注意，默认值将来可能会更改。

## Usage

新的 Git 文件生成器 globbing 被引用到了 `doublestar` 软件包。你可以在 [这里](https://github.com/bmatcuk/doublestar) 找到它。

下面是其文档的简短摘录。

双星模式以递归方式匹配文件和目录。 例如，如果您有以下目录结构：

```bash
grandparent
`-- parent
    |-- child1
    `-- child2
```

您可以使用以下模式查找子代：`**/child*`、`grandparent/**/child?`、`**/parent/*`，甚至只使用`**`本身（将递归返回所有文件和目录）。

Bash 的 globstar 受到了 doublestar 的启发，因此工作原理类似。 请注意，doublestar 必须单独作为路径组件出现。 `/path**`之类的模式是无效的，其处理方式与 `/path*`相同，但 `/path*/**`应能达到预期效果。 此外，`/path/**`将匹配路径目录下的所有目录和文件，但 `/path/**/`仅匹配目录。