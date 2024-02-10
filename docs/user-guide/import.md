<!-- TRANSLATED by md-translate -->
<!-- TRANSLATED by md-translate -->

# 导入 Argo CD go packages

## 问题

在自己的项目中导入 Argo CD 软件包时，下载依赖包时可能会遇到一些错误，比如 "修订未知版 v0.0.0"，这是因为 Argo CD 直接依赖于一些 Kubernetes 软件包，而这些软件包的 go.mod 中就有这些未知的 v0.0.0 版本。

## 解决方案

在你自己的 go.mod 中添加一个替换部分，与相应 Argo CD 版本的 go.mod 的替换部分相同。 为了找到特定版本的 go.mod，请导航到[阿尔戈光盘库](https://github.com/argoproj/argo-cd/)现在您可以查看特定版本的 go.mod 文件和所有其他文件。

## 示例

如果您正在使用 Argo CD v2.4.15，您的 go.mod 应包含以下内容： Argo CD v2.4.15.

```
replace (
    // https://github.com/golang/go/issues/33546#issuecomment-519656923
    github.com/go-check/check => github.com/go-check/check v0.0.0-20180628173108-788fd7840127

    github.com/golang/protobuf => github.com/golang/protobuf v1.4.2
    github.com/gorilla/websocket => github.com/gorilla/websocket v1.4.2
    github.com/grpc-ecosystem/grpc-gateway => github.com/grpc-ecosystem/grpc-gateway v1.16.0
    github.com/improbable-eng/grpc-web => github.com/improbable-eng/grpc-web v0.0.0-20181111100011-16092bd1d58a

    // Avoid CVE-2022-28948
    gopkg.in/yaml.v3 => gopkg.in/yaml.v3 v3.0.1

    // https://github.com/kubernetes/kubernetes/issues/79384#issuecomment-505627280
    k8s.io/api => k8s.io/api v0.23.1
    k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.23.1
    k8s.io/apimachinery => k8s.io/apimachinery v0.23.1
    k8s.io/apiserver => k8s.io/apiserver v0.23.1
    k8s.io/cli-runtime => k8s.io/cli-runtime v0.23.1
    k8s.io/client-go => k8s.io/client-go v0.23.1
    k8s.io/cloud-provider => k8s.io/cloud-provider v0.23.1
    k8s.io/cluster-bootstrap => k8s.io/cluster-bootstrap v0.23.1
    k8s.io/code-generator => k8s.io/code-generator v0.23.1
    k8s.io/component-base => k8s.io/component-base v0.23.1
    k8s.io/component-helpers => k8s.io/component-helpers v0.23.1
    k8s.io/controller-manager => k8s.io/controller-manager v0.23.1
    k8s.io/cri-api => k8s.io/cri-api v0.23.1
    k8s.io/csi-translation-lib => k8s.io/csi-translation-lib v0.23.1
    k8s.io/kube-aggregator => k8s.io/kube-aggregator v0.23.1
    k8s.io/kube-controller-manager => k8s.io/kube-controller-manager v0.23.1
    k8s.io/kube-proxy => k8s.io/kube-proxy v0.23.1
    k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.23.1
    k8s.io/kubectl => k8s.io/kubectl v0.23.1
    k8s.io/kubelet => k8s.io/kubelet v0.23.1
    k8s.io/legacy-cloud-providers => k8s.io/legacy-cloud-providers v0.23.1
    k8s.io/metrics => k8s.io/metrics v0.23.1
    k8s.io/mount-utils => k8s.io/mount-utils v0.23.1
    k8s.io/pod-security-admission => k8s.io/pod-security-admission v0.23.1
    k8s.io/sample-apiserver => k8s.io/sample-apiserver v0.23.1
)
```