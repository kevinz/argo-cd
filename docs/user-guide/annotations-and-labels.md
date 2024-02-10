<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# Argo CD 被引用的注释和标签

## 注释

| Annotation key | Target resource(es) | Possible values | Description | |--------------------------------------------|---------------------|---------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| | argocd.argoproj.io/application-set-refresh | ApplicationSet |`"true"`| 当应用集被 webhook 请求刷新时添加。 应用集控制器将在调节结束时移除此注解。[参见比较选项文档](compare-options.md)| 配置如何将应用程序的当前状态与其期望状态进行比较。[参见资源挂钩文档](resource_hooks.md)| 被引用用于配置[资源挂钩](resource_hooks.md)| | argocd.argoproj.io/hook-delete-policy | any |.[参见资源挂钩文档](resource_hooks.md#hook-deletion-policies)| 被引用时，设置"......"。[资源钩的删除

## 标签

| 标签关键字 | 目标资源 | 可能的值 | 说明 | |--------------------------------|---------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------| | argocd.argoproj.io/instance | 应用程序 | 任何 | 建议跟踪标签为[避免与其他被引用的工具发生冲突。](../faq.md#why-is-my-app-out-of-sync-even-after-syncing). | | argocd.argoproj.io/secret-type | Secret | 秘密`cluster`,`repository`,`repo-creds`| 识别 Argo CD 被引用的某些秘密类型。 参见[声明式设置文档](../operator-manual/declarative-setup.md)了解详情。