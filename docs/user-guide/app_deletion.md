<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 应用程序删除

删除应用程序时，可以使用或不使用级联选项。 A***级联删除**删除应用程序及其资源，而不是只删除应用程序。

## 使用 `argocd` 进行删除

执行非级联删除：

```bash
argocd app delete APPNAME --cascade=false
```

执行级联删除：

```bash
argocd app delete APPNAME --cascade
```

或

```bash
argocd app delete APPNAME
```

## 使用 `kubectl` 进行删除

要执行非级联删除，请确保终结器未设置，然后删除应用程序：

```bash
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": null}}' --type merge
kubectl delete app APPNAME
```

要执行级联删除，可设置终结器，例如被引用为`kubectl patch`：

```bash
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": ["resources-finalizer.argocd.argoproj.io"]}}' --type merge
kubectl delete app APPNAME
```

## 关于删除终结器

```yaml
metadata:
  finalizers:
    # The default behaviour is foreground cascading deletion
    - resources-finalizer.argocd.argoproj.io
    # Alternatively, you can use background cascading deletion
    # - resources-finalizer.argocd.argoproj.io/background
```

使用此终结器删除应用程序时，Argo CD 应用程序控制器将对应用程序的资源执行级联删除。

在执行[应用程序的应用程序模式](.../operator-manual/cluster-bootstrapping.md#cascading-deletion)。

级联删除的默认传播策略是[前台级联删除](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#foreground-deletion).阿尔戈 CD 演出[背景级联删除](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#background-deletion)当`resources-finalizer.argocd.argoproj.io/background`已设定。

当您调用`argocd app delete`与`--cascade`会自动添加终结器。 您可以使用`--propagation-policy<foreground|background>`。