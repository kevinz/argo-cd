<!-- TRANSLATED by md-translate -->
# 发电机

生成器负责生成_参数_，然后将其呈现到 ApplicationSet 资源的 `template:` 字段中。 有关生成器如何与模板配合创建 Argo CD 应用程序的示例，请参阅[Introduction](index.md)。

生成器主要基于它们用来生成模板参数的数据源。 例如：List 生成器从 _literal list_ 中提供一组参数，Cluster 生成器使用 _Argo CD 集群列表_ 作为数据源，Git 生成器使用 _Git 仓库_ 中的文件/目录，等等。

截至目前，共有 9 台发电机：

* [列表生成器]（Generators-List.md）：列表生成器允许您根据任意选择的键/值元素对的固定列表将 Argo CD 应用程序定位到集群。
* [集群生成器](Generators-Cluster.md)：集群生成器允许您根据 Argo CD 内部定义（并由 Argo CD 管理）的集群列表，将 Argo CD 应用程序定位到集群（包括自动响应 Argo CD 的集群添加/删除事件）。
* [Git 生成器](Generators-Git.md)：Git 生成器允许你根据 Git 仓库中的文件或 Git 仓库的目录结构创建应用程序。
* [矩阵生成器](Generators-Matrix.md)：矩阵生成器可被用来合并两个独立生成器生成的参数。
* [合并生成器](Generators-Merge.md)：合并生成器可用于合并两个或多个生成器的生成参数。附加生成器可以覆盖基本生成器的值。
* [SCM Provider 生成器](Generators-SCM-Provider.md)：SCM Provider 生成器使用 SCM 提供者（如 GitHub）的 API 自动发现组织内的资源库。
* [拉取请求生成器](Generators-Pull-Request.md)：拉取请求生成器使用 SCMaaS 提供商（如 GitHub）的 API，自动发现某个版本库中的开放拉取请求。
* [集群决策资源生成器](Generators-Cluster-Decision-Resource.md)：集群决策资源生成器用于与 Kubernetes 自定义资源接口，这些资源使用特定于资源的自定义逻辑来决定部署到哪一组 Argo CD 集群。
* [插件生成器](Generators-Plugin.md)：插件生成器通过 RPC HTTP 请求来提供参数。

所有生成器都可以通过被引用的 [职位选择器]（Generators-Post-Selector.md）进行筛选

如果你是生成器的新手，请从**列表**和**集群**生成器开始学习。 如需了解更多高级用例，请参阅上述其余生成器的文档。