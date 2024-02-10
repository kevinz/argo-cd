<!-- TRANSLATED by md-translate -->
# 网络终端

Argo CD 终端](../assets/terminal.png)

自 2.4 版起，Argo CD 提供了一个基于 Web 的终端，让您可以在运行中的 pod 中获取 shell，就像使用 `kubectl exec` 一样。 这基本上就是在浏览器中使用 SSH，完全支持 ANSI 颜色！不过，为了安全起见，该功能默认是禁用的。

这是一项功能强大的权限。 它允许用户在拥有 "exec/create "权限的应用程序管理的任何 Pod 上运行任意代码。 如果 Pod 挂载了 ServiceAccount 令牌（这是 Kubernetes 的默认行为），则用户实际上拥有与该 ServiceAccount 相同的权限。

## 启用终端

<!-- Use indented code blocks for the numbered list to prevent breaking the numbering. See #11590 -->

1.将 `argocd-cm` ConfigMap 上的 `exec.enabled` 键设置为 `"true"。
2.给 `argocd-server` 角色（如果使用 namespaced Argo）或 ClusterRole（如果使用集群 Argo）打补丁，允许 `argocd-server

将其装入豆荚

```
- apiGroups:
      - ""
      resources:
      - pods/exec
      verbs:
      - create
```

3.添加 RBAC 规则，允许用户 "创建"`exec`资源，即
    ```
    p,角色:myrole,执行,创建,*/*,允许
    ```

更多信息请参阅 [RBAC 配置](rbac.md#exec-resource)。

## 更改允许的外壳

默认情况下，Argo CD 会尝试按此顺序执行 shell：

1. bash
2. sh
3. powershell
4. cmd

如果找不到任何 shell，终端会话将失败。 要添加或更改允许的 shell，请更改 `argocd-cm` configMap 中的 `exec.shells` 键，用逗号分隔。