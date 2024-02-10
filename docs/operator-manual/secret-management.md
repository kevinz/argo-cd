<!-- TRANSLATED by md-translate -->
# 秘密管理

Argo CD 对如何管理秘密不持异议。 方法有很多种，没有放之四海而皆准的解决方案。

许多解决方案使用插件将秘密注入应用程序配置清单。 请参阅下面的[降低秘密注入插件的风险]（#mitigating-risks-of-secret-injection-plugins），以确保安全地使用这些插件。

下面是一些人们在做 GitOps 秘密的方法：

* [Bitnami封印秘密](https://github.com/bitnami-labs/sealed-secrets)
* [外部秘密操作员](https://github.com/external-secrets/external-secrets)
* [HashiCorp Vault](https://www.vaultproject.io)
* [银行金库](https://bank-vaults.dev/)
* [Helm 秘密](https://github.com/jkroepke/helm-secrets)
* [Kustomize secret generator plugins](https://github.com/kubernetes-sigs/kustomize/blob/fd7a353df6cece4629b8e8ad56b71e30636f38fc/examples/kvSourceGoPlugin.md#secret-values-from-anywhere)
* [aws-secret-operator](https://github.com/mumoshu/aws-secret-operator)
* [KSOPS](https://github.com/viaduct-ai/kustomize-sops#argo-cd-integration)
* [argocd-vault-plugin](https://github.com/argoproj-labs/argocd-vault-plugin)
* [argocd-vault-replacer](https://github.com/crumbhole/argocd-vault-replacer)
* [Kubernetes 存储秘密 CSI 驱动程序](https://github.com/kubernetes-sigs/secrets-store-csi-driver)
* [Vals-Operator](https://github.com/digitalis-io/vals-operator)

讨论内容见 [#1364](https://github.com/argoproj/argo-cd/issues/1364)

## 降低秘密注入插件的风险

Argo CD 将插件生成的配置清单和注入的秘密缓存在 Redis 实例中。 这些配置清单也可以通过 repo-server API（gRPC 服务）获取。 这意味着，任何可以访问 Redis 实例或 repo-server 的人都可以获取这些秘密。

考虑采取以下步骤来降低秘密注入插件的风险：

1.设置网络策略，防止直接访问 Argo CD 组件（Redis 和 repo-server）。确保你的集群支持这些网络策略，并能真正执行它们。
2.考虑在自己的集群上运行 Argo CD，其上不运行其他应用程序。
3.[在 Redis 实例上启用密码验证](https://github.com/argoproj/argo-cd/issues/3130) （目前仅支持非HA Argo CD 安装）。
