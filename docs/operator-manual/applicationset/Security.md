<!-- TRANSLATED by md-translate -->
# ApplicationSet 安全性

ApplicationSet 是一个功能强大的工具，在被引用之前，了解其安全含义至关重要。

## 只有管理员可以创建/更新/删除 ApplicationSet

ApplicationSet 可在任意 [Projects](../../user-guide/projects.md) 下创建应用程序。 Argo CD 设置通常包括具有高级权限的 Project（如 `default`），通常包括管理 Argo CD 本身资源的能力（如 RBAC ConfigMap）。

ApplicationSet 还可以快速创建任意数量的应用程序，并同样快速地删除它们。

最后，ApplicationSet 可以泄露权限信息。 例如，[git generator](./Generators-Git.md) 可以读取 Argo CD namespace 中的 Secrets，并将其作为 auth 标头发送到任意 URL（例如为 `api` 字段提供的 URL）（此功能旨在授权向 GitHub 等 SCM Provider 提出的请求，但可能会被恶意用户滥用）。

出于这些原因，***只有管理员**才有权限（通过 Kubernetes RBAC 或任何其他机制）创建、更新或删除 ApplicationSet。

## 管理员必须对 ApplicationSet 的真相来源应用适当的控制措施

即使非管理员不能创建 ApplicationSet 资源，他们也有可能影响 ApplicationSet 的行为。

例如，如果 ApplicationSet 使用 [git 生成器](./Generators-Git.md)，拥有源 git 仓库推送权限的恶意用户可能会生成过多的应用程序，给 ApplicationSet 和应用程序控制器造成压力。 他们还可能导致 SCM Provider 的速率限制启动，从而降低 ApplicationSet 的服务水平。

### 模板化的 "项目 "字段

需要特别注意的是，ApplicationSet 中的 `project` 字段是模板化的。 拥有生成器真实源写入权限的恶意用户（例如，拥有 git 生成器 git 仓库推送权限的人）可以在限制不足的项目下创建应用程序。 拥有在不受限制的项目（如 `default` 项目）下创建应用程序能力的恶意用户可以通过修改 RBAC ConfigMap 等方式控制 Argo CD 本身。

如果 ApplicationSet 模板中没有硬编码 "项目 "字段，则管理员必须控制 ApplicationSet 生成器的所有真相来源。