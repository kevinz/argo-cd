<!-- TRANSLATED by md-translate -->
# ApplicationSet 控制器如何与 Argo CD 交互

创建、更新或删除 "ApplicationSet "资源时，ApplicationSet 控制器会响应创建、更新或删除一个或多个相应的 Argo CD "Application "资源。

事实上，ApplicationSet 控制器的唯一职责就是创建、更新和删除 Argo CD 名称空间中的 "Application "资源。 控制器的唯一工作就是确保 "Application "资源与已定义的声明式 "ApplicationSet "资源保持一致，仅此而已。

因此，ApplicationSet 控制器：

* 不创建/修改/删除 Kubernetes 资源（"应用程序 "CR 除外）
* 除 Argo CD 部署的集群外，不连接其他集群
* 不与 Argo CD 所部署的命名空间以外的命名空间交互

!!重要 "使用 Argo CD 命名空间" 所有 ApplicationSet 资源和 ApplicationSet 控制器必须安装在与 Argo CD 相同的命名空间中。 不同命名空间中的 ApplicationSet 资源将被忽略。

Argo CD 本身负责实际部署生成的子 "应用程序 "资源，如部署、服务和 configmaps。

因此，ApplicationSet 控制器可被视为 "应用程序 "的 "工厂"，它将 "ApplicationSet "资源作为输入，并输出一个或多个与该集参数相对应的 Argo CD "应用程序 "资源。

![ApplicationSet 控制器 vs Argo CD，交互图]( .../../assets/applicationset/Argo-CD-Integration/ApplicationSet-Argo-Relationship-v2.png)

在该图中，定义了一个 "ApplicationSet "资源，ApplicationSet 控制器负责创建相应的 "Application "资源。 然后，生成的 "Application "资源由 Argo CD 管理：也就是说，Argo CD 负责实际部署子资源。

Argo CD 会根据 Application `spec` 字段中定义的 Git 仓库内容生成应用程序的 Kubernetes 资源，例如部署、服务和其他资源。

ApplicationSet 的创建、更新或删除将对 Argo CD 名称空间中的应用程序产生直接影响。 同样，集群事件（使用集群生成器时，Argo CD 集群秘密的添加/删除）或 Git 的更改（使用 Git 生成器时）将被用作 ApplicationSet 控制器构建 "应用程序 "资源的输入。

Argo CD 和 ApplicationSet 控制器协同工作，确保应用程序资源集的一致性，并在目标集群中进行部署。