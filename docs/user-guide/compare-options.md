<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 比较选项

## 忽略无关资源

&gt; v1.1

在某些情况下，您可能希望将资源排除在应用程序的整体同步状态之外。 例如，如果这些资源是由工具生成的，您可以在希望排除的资源上添加此注释：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
```

比较选项需要剪枝](.../assets/compare-option-ignore-needs-pruning.png)

注意，这只会影响同步状态。 如果资源的健康状况下降，那么应用程序也会下降。

kustomize 有一项功能可以生成配置映射 ([阅读更多 ⧉](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/configGeneration.md)您可以设置`生成器选项`添加此注释，以便应用程序保持同步：

```yaml
configMapGenerator:
  - name: my-map
    literals:
      - foo=bar
generatorOptions:
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
kind: Kustomization
```

注意`generatorOptions`为配置映射和秘密添加注释 ([阅读更多 ⧉](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/generatorOptions.md)).

您可以将其与[`Prune= false``](sync-options.md).