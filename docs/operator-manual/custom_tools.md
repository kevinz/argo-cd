<!-- TRANSLATED by md-translate -->
# 定制工具

Argo CD 捆绑了其支持的模板工具（helm、kustomize、ks、jsonnet）的首选版本，作为其容器镜像的一部分。 有时，可能需要使用 Argo CD 捆绑的工具之外的特定版本。 这样做的一些原因可能是：

* 升级/降级到某个工具的特定版本，因为存在错误或修复了错误。
* 安装额外的依赖项，以便被 kustomize 的 configmaps 和 secrets 生成器引用。(例如：curl、vault、gpg、AWS CLI）
* 安装[配置管理插件]（config-management-plugins.md）。

由于 Argo CD 版本库服务器是负责生成 Kubernetes 配置清单的单一服务，因此可以对其进行定制，以使用环境所需的其他工具链。

## 通过卷挂载添加工具

第一种技术是使用一个 "init "容器和一个 "volumeMount "将不同版本的工具复制到 repo-server 容器中。 在下面的示例中，init 容器正在用与 Argo CD 中捆绑的版本不同的二进制文件覆盖 helm：

```yaml
spec:
      # 1. Define an emptyDir volume which will hold the custom binaries
      volumes:
      - name: custom-tools
        emptyDir: {}
      # 2. Use an init container to download/copy custom binaries into the emptyDir
      initContainers:
      - name: download-tools
        image: alpine:3.8
        command: [sh, -c]
        args:
        - wget -qO- https://storage.googleapis.com/kubernetes-helm/helm-v2.12.3-linux-amd64.tar.gz | tar -xvzf - &&
          mv linux-amd64/helm /custom-tools/
        volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools
      # 3. Volume mount the custom binary to the bin directory (overriding the existing version)
      containers:
      - name: argocd-repo-server
        volumeMounts:
        - mountPath: /usr/local/bin/helm
          name: custom-tools
          subPath: helm
```

## BYOI（打造自己的镜像）

有时，仅替换二进制文件是不够的，还需要安装其他依赖项。 下面的示例从 Docker 文件构建了一个完全定制的版本服务器，安装了生成配置清单可能需要的额外依赖项。

```Dockerfile
FROM argoproj/argocd:v2.5.4 # Replace tag with the appropriate argo version

# Switch to root for the ability to perform install
USER root

# Install tools needed for your repo-server to retrieve & decrypt secrets, render manifests 
# (e.g. curl, awscli, gpg, sops)
RUN apt-get update && \
    apt-get install -y \
        curl \
        awscli \
        gpg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    curl -o /usr/local/bin/sops -L https://github.com/mozilla/sops/releases/download/3.2.0/sops-3.2.0.linux && \
    chmod +x /usr/local/bin/sops

# Switch back to non-root user
USER $ARGOCD_USER_ID
```