<!-- TRANSLATED by md-translate -->
# 核实 Argo CD 文物

## 先决条件

* cosign `v2.0.0` 或更高版本 [安装说明](https://docs.sigstore.dev/cosign/installation)
* slsa-verifier [安装说明](https://github.com/slsa-framework/slsa-verifier#installation)
* 起重机 [安装说明](https://github.com/google/go-containerregistry/blob/main/cmd/crane/README.md) (仅用于集装箱验证)

---

## 发布资产

| Asset                   | Description                   |
|-------------------------|-------------------------------|
| argocd-darwin-amd64     | CLI Binary                    |
| argocd-darwin-arm64     | CLI Binary                    |
| argocd-linux_amd64      | CLI Binary                    |
| argocd-linux_arm64      | CLI Binary                    |
| argocd-linux_ppc64le    | CLI Binary                    |
| argocd-linux_s390x      | CLI Binary                    |
| argocd-windows_amd64    | CLI Binary                    |
| argocd-cli.intoto.jsonl | Attestation of CLI binaries   |
| cli_checksums.txt       | Checksums of binaries         |
| sbom.tar.gz             | Sbom                          |
| sbom.tar.gz.pem         | Certificate used to sign sbom |
| sbom.tar.gz.sig         | Signature of sbom                |

---

## 核实容器镜像

Argo CD 镜像由 [cosign](https://github.com/sigstore/cosign)使用基于身份（"无密钥"）的签名和透明度进行签名。执行以下命令可用于验证容器镜像的签名：

```bash
cosign verify \
--certificate-identity-regexp https://github.com/argoproj/argo-cd/.github/workflows/image-reuse.yaml@refs/tags/v \
--certificate-oidc-issuer https://token.actions.githubusercontent.com \
quay.io/argoproj/argocd:v2.7.0 | jq
```

如果正确验证了容器镜像，命令应输出如下内容：

```bash
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - Any certificates were verified against the Fulcio roots.
[
  {
    "critical": {
      "identity": {
        "docker-reference": "quay.io/argoproj/argo-cd"
      },
      "image": {
        "docker-manifest-digest": "sha256:63dc60481b1b2abf271e1f2b866be8a92962b0e53aaa728902caa8ac8d235277"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "1.3.6.1.4.1.57264.1.1": "https://token.actions.githubusercontent.com",
      "1.3.6.1.4.1.57264.1.2": "push",
      "1.3.6.1.4.1.57264.1.3": "a6ec84da0eaa519cbd91a8f016cf4050c03323b2",
      "1.3.6.1.4.1.57264.1.4": "Publish ArgoCD Release",
      "1.3.6.1.4.1.57264.1.5": "argoproj/argo-cd",
      "1.3.6.1.4.1.57264.1.6": "refs/tags/<version>",
      ...
```

---

## 使用 SLSA 证书验证容器镜像

使用 [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator) 生成[SLSA](https://slsa.dev/) 3 级出处。

以下命令将验证证明的签名和签发方式。 它将包含有效载荷类型、有效载荷和签名。

根据 [slsa-verifier 文档](https://github.com/slsa-framework/slsa-verifier/tree/main#containers)，运行以下命令：

```bash
# Get the immutable container image to prevent TOCTOU attacks https://github.com/slsa-framework/slsa-verifier#toctou-attacks
IMAGE=quay.io/argoproj/argocd:v2.7.0
IMAGE="${IMAGE}@"$(crane digest "${IMAGE}")
# Verify provenance, including the tag to prevent rollback attacks.
slsa-verifier verify-image "$IMAGE" \
    --source-uri github.com/argoproj/argo-cd \
    --source-tag v2.7.0
```

如果只想验证源码库标记的主版本或次版本（而不是完整标记），请使用"--source-versionioned-tag"，它可以执行语义版本验证：

```shell
slsa-verifier verify-image "$IMAGE" \
    --source-uri github.com/argoproj/argo-cd \
    --source-versioned-tag v2 # Note: May use v2.7 for minor version verification.
```

验证有效载荷包含不可伪造的出处，该出处已进行 base64 编码，可通过在上述命令中传递 `--print-provenance` 选项来查看：

```bash
slsa-verifier verify-image "$IMAGE" \
    --source-uri github.com/argoproj/argo-cd \
    --source-tag v2.7.0 \
    --print-provenance | jq
```

如果您更喜欢使用联署，请遵循以下 [说明](https://github.com/slsa-framework/slsa-github-generator/blob/main/internal/builders/container/README.md#cosign)。

提示 `cosign` 或 `slsa-verifier` 都可以被引用来验证镜像证明。 详细说明请查阅每个二进制文件的文档。

---

## 利用 SLSA 证明验证 CLI 工件

提供了每个发布版本的单个证明 (`argocd-cli.intoto.jsonl`)。可与 [slsa-verifier](https://github.com/slsa-framework/slsa-verifier#verification-for-github-builders) 一起使用，以验证 CLI 二进制文件是使用 GitHub 上的 Argo CD 工作流生成的，并确保其经过加密签名。

```bash
slsa-verifier verify-artifact argocd-linux-amd64 \
  --provenance-path argocd-cli.intoto.jsonl \
  --source-uri github.com/argoproj/argo-cd \
  --source-tag v2.7.0
```

如果只想验证源码库标记的主版本或次版本（而不是完整标记），请使用"--source-versionioned-tag"，它可以执行语义版本验证：

```shell
slsa-verifier verify-artifact argocd-linux-amd64 \
  --provenance-path argocd-cli.intoto.jsonl \
  --source-uri github.com/argoproj/argo-cd \
  --source-versioned-tag v2 # Note: May use v2.7 for minor version verification.
```

有效载荷是不可伪造的证明，经过 base64 编码，可以通过在上述命令中传递 `--print-provenance` 选项来查看：

```bash
slsa-verifier verify-artifact argocd-linux-amd64 \
  --provenance-path argocd-cli.intoto.jsonl \
  --source-uri github.com/argoproj/argo-cd \
  --source-tag v2.7.0 \
  --print-provenance | jq
```

## 验证 Sbom

每个版本的单个证明（`argocd-sbom.intoto.jsonl`）与 sbom (`sbom.tar.gz`)一起发布。这可以与 [slsa-verifier](https://github.com/slsa-framework/slsa-verifier#verification-for-github-builders) 一起使用，以验证 SBOM 是使用 GitHub 上的 Argo CD 工作流生成的，并确保其经过加密签名。

```bash
slsa-verifier verify-artifact sbom.tar.gz \
  --provenance-path argocd-sbom.intoto.jsonl \
  --source-uri github.com/argoproj/argo-cd \
  --source-tag v2.7.0
```

---

## 验证 Kubernetes

### 政策控制器

请注意，我们鼓励所有用户在部署到 Kubernetes 集群之前，通过您选择的接入/策略控制器验证签名和出处。 这样做可以验证镜像是由我们构建的。

Cosign 签名和 SLSA 出处与多种类型的准入控制器兼容。有关支持的控制器，请参阅 [cosign 文档](https://docs.sigstore.dev/cosign/overview/#kubernetes-integrations) 和 [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator/blob/main/internal/builders/container/README.md#verification) 。