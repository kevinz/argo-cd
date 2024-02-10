<!-- TRANSLATED by md-translate -->
# 灾难恢复

您可以使用 `argocd admin` 来引用和导出所有 Argo CD 数据。

确保 `~/.kube/config` 指向 Argo CD 集群。

了解您运行的 Argo CD 版本：

```bash
argocd version | grep server
# ...
export VERSION=v1.0.1
```

导出到备份：

```bash
docker run -v ~/.kube:/home/argocd/.kube --rm quay.io/argoproj/argocd:$VERSION argocd admin export > backup.yaml
```

从备份导入：

```bash
docker run -i -v ~/.kube:/home/argocd/.kube --rm quay.io/argoproj/argocd:$VERSION argocd admin import - < backup.yaml
```

注意 如果在不同于默认的 namespace 上运行 Argo CD，请记住传递 namespace 参数 (-n<namespace>)。如果在错误的 namespace 上运行，"argocd admin export "不会失败。