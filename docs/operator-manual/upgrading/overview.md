<!-- TRANSLATED by md-translate -->
# 概览

注意

```
This section contains information on upgrading Argo CD. Before upgrading please make sure to read details about
the breaking changes between Argo CD versions.
```

Argo CD 被引用 semver 版本控制，并确保遵循以下规则：

* 该补丁发布不引入任何破坏性更改。因此，如果您要从 v1.5.1 升级到 v1.5.3

应该没有特别的说明需要遵循。

* 次要发布可能会引入一些小改动，并提供一个变通办法。如果您要从 v1.3.0 升级到 v1.5.2

请务必查看 [v1.3 to v1.4](./1.3-1.4.md) 和 [v1.4 to v1.5](./1.4-1.5.md) 升级说明中的升级详情。

* 主要发布版本引入了向后不兼容的行为更改。建议备份

Argo CD 设置被引用灾难恢复 [指南](.../disaster_recovery.md)。

在阅读了 Argo CD 版本中可能引入的破坏性更改的相关说明后，请使用以下命令升级 Argo CD。请确保将 `<version>` 替换为所需的版本号：

**非哈**：

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/<version>/manifests/install.yaml
```

**HA**：

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/<version>/manifests/ha/install.yaml
```

警告

```
Even though some releases require only image change it is still recommended to apply whole manifests set.
Manifest changes might include important parameter modifications and applying the whole set will protect you from
introducing misconfiguration.
```

<hr/>

* [v2.9至v2.10](./2.9-2.10.md)
* [v2.8至v2.9](./2.8-2.9.md)
* [v2.7至v2.8](./2.7-2.8.md)
* [v2.6至v2.7](./2.6-2.7.md)
* [v2.5至v2.6](./2.5-2.6.md)
* [v2.4至v2.5](./2.4-2.5.md)
* [v2.3至v2.4](./2.3-2.4.md)
* [v2.2至v2.3](./2.2-2.3.md)
* [v2.1至v2.2](./2.1-2.2.md)
* [v2.0至v2.1](./2.0-2.1.md)
* [v1.8 至 v2.0](./1.8-2.0.md)
* [v1.7 至 v1.8](./1.7-1.8.md)
* [V1.6至V1.7](./1.6-1.7.md)
* [V1.5至V1.6](./1.5-1.6.md)
* [V1.4至V1.5](./1.4-1.5.md)
* [V1.3至V1.4](./1.3-1.4.md)
* [V1.2至V1.3](./1.2-1.3.md)
* [V1.1至V1.2](./1.1-1.2.md)
* [v1.0 至 v1.1](./1.0-1.1.md)
