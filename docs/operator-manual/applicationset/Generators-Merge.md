<!-- TRANSLATED by md-translate -->
# 合并生成器

合并生成器将基本（第一个）生成器生成的参数与后续生成器生成的匹配参数集合并。 匹配参数集的配置_合并键_值相同，非匹配参数集将被丢弃。 覆盖优先级是自下而上的：生成器 3 生成的匹配参数集中的值优先于生成器 2 生成的相应参数集中的值。

当需要覆盖参数集的子集时，使用合并生成器是合适的。

## 示例：基本集群生成器 + 覆盖集群生成器 + 列表生成器

举个例子，设想我们有两个集群：

* 暂存 "集群（位于 `https://1.2.3.4`)
* 生产 "集群（位于 `https://2.4.6.8`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-git
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    # merge 'parent' generator
    - merge:
        mergeKeys:
          - server
        generators:
          - clusters:
              values:
                kafka: 'true'
                redis: 'false'
          # For clusters with a specific label, enable Kafka.
          - clusters:
              selector:
                matchLabels:
                  use-kafka: 'false'
              values:
                kafka: 'false'
          # For a specific cluster, enable Redis.
          - list:
              elements: 
                - server: https://2.4.6.8
                  values.redis: 'true'
  template:
    metadata:
      name: '{{.name}}'
    spec:
      project: '{{index .metadata.labels "environment"}}'
      source:
        repoURL: https://github.com/argoproj/argo-cd.git
        targetRevision: HEAD
        path: app
        helm:
          parameters:
            - name: kafka
              value: '{{.values.kafka}}'
            - name: redis
              value: '{{.values.redis}}'
      destination:
        server: '{{.server}}'
        namespace: default
```

基本集群生成器会扫描[Argo CD 中定义的集群集](Generators-Cluster.md)，找到暂存集群和生产集群秘密，并生成两组相应的参数：

```yaml
- name: staging
  server: https://1.2.3.4
  values.kafka: 'true'
  values.redis: 'false'

- name: production
  server: https://2.4.6.8
  values.kafka: 'true'
  values.redis: 'false'
```

覆盖集群生成器会扫描[Argo CD 中定义的集群集](Generators-Cluster.md)，找到暂存集群 secret（具有所需的标签），并生成以下参数：

```yaml
- name: staging
  server: https://1.2.3.4
  values.kafka: 'false'
```

与基本生成器的参数合并后，暂存集群的 `values.kafka` 值将设为 `'false'`。

```yaml
- name: staging
  server: https://1.2.3.4
  values.kafka: 'false'
  values.redis: 'false'

- name: production
  server: https://2.4.6.8
  values.kafka: 'true'
  values.redis: 'false'
```

Finally, the List cluster generates a single set of parameters：

```yaml
- server: https://2.4.6.8
  values.redis: 'true'
```

与更新的基础参数合并后，生产集群的 `values.redis` 值将设为 `'true'`。 这就是合并生成器的最终输出：

```yaml
- name: staging
  server: https://1.2.3.4
  values.kafka: 'false'
  values.redis: 'false'

- name: production
  server: https://2.4.6.8
  values.kafka: 'true'
  values.redis: 'true'
```

## 示例：在合并中被引用插值法

有些生成器支持附加值和从生成变量到选定值的插值，这可以用来教合并生成器使用哪些生成变量来合并不同的生成器。

下面的示例通过集群标签和分支名称将已发现的集群和 git 仓库结合起来：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-git
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    # merge 'parent' generator:
    # Use the selector set by both child generators to combine them.
    - merge:
        mergeKeys:
          # Note that this would not work with goTemplate enabled,
          # nested merge keys are not supported there.
          - values.selector
        generators:
          # Assuming, all configured clusters have a label for their location:
          # Set the selector to this location.
          - clusters:
              values:
                selector: '{{index .metadata.labels "location"}}'
          # The git repo may have different directories which correspond to the
          # cluster locations, using these as a selector.
          - git:
              repoURL: https://github.com/argoproj/argocd-example-apps/
              revision: HEAD
              directories:
              - path: '*'
              values:
                selector: '{{.path.path}}'
  template:
    metadata:
      name: '{{.name}}'
    spec:
      project: '{{index .metadata.labels "environment"}}'
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        # The cluster values field for each generator will be substituted here:
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: '{{.server}}'
        namespace: default
```

假设有一个名为 "germany01"、标签为 "metadata.labels.location=Germany "的集群，以及一个包含名为 "Germany "的目录的 git 仓库，这可以组合成如下值：

```yaml
# From the cluster generator
- name: germany01
  server: https://1.2.3.4
  # From the git generator
  path: Germany
  # Combining selector with the merge generator
  values.selector: 'Germany'
  # More values from cluster & git generator
  # […]
```

## 限制

1.每个数组条目只能指定一个发生器。这种情况无效：
    ```
    - 合并：
         generators：
         - list：# (...)
           git：# (...)
    ```- 虽然 Kubernetes API 验证会接受这种方法，但控制器在生成时会报错。每个生成器都应在单独的数组元素中指定，如上面的示例。
2.合并生成器不支持在子生成器上指定 [`模板`覆盖](Template.md#generator-templates)。该 `template` 将不会被处理：
    ```
    - 合并：
         生成器：
           - list：
               elements：
                 - # (...)
               模板：{ }# 未处理
    ```
3.组合型生成器（矩阵或合并）只能嵌套一次。例如，这样做是不行的：
    ```
    - 合并：
         生成器：
           - merge：
               generators：
                 - 合并：  # 第三层无效。
                     生成器
                       - list：
                           elements：
                             - # (...)
    ```
4.目前不支持在使用 `goTemplate: true` 时对嵌套值进行合并，这将不起作用
    ```
    规格：
       goTemplate: true
       生成器：
       - merge：
           mergeKeys：
             - values.merge
    ```
5.当使用嵌套在另一个矩阵或合并生成器内部的合并生成器时，只有通过 `spec.applyNestedSelectors` 启用后，才会引用此嵌套生成器的生成器的 [Post Selectors](Generators-Post-Selector.md)。
    ```
    - 合并：
         生成器：
           - 合并
               生成器
                 - 列表
                     元素：
                       - # (...)
                   选择器{ }# 仅在 applyNestedSelectors 为 true 时应用
    ```
