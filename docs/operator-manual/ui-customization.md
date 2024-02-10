<!-- TRANSLATED by md-translate -->
# 用户界面定制

## 默认应用程序详细信息视图

默认情况下，"应用程序详细信息 "将显示 "树 "视图。

可以通过设置 `pref.argocd.argoproj.io/default-view` 注解，接受 `tree`、`pods`、`network`、`list` 中的一个作为 Values，在应用程序基础上进行配置。

对于 Pods 视图，可使用 `pref.argocd.argoproj.io/default-pod-sort` 注解配置默认分组机制，并接受 `node`、`parentResource`、`topLevelResource` 中的一个作为值。