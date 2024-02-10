<!-- TRANSLATED by md-translate -->
# 安装

您可以从[本版本库的最新发布页面](https://github.com/argoproj/argo-cd/releases/latest)下载最新的 Argo CD 版本，其中将包含 `argocd` CLI。

## Linux 和 WSL

### ArchLinux

```bash
pacman -S argocd
```

#### 自制

```bash
brew install argocd
```

#### 使用 Curl 下载

#### 下载最新版本

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

#### 下载具体版本

设置 `VERSION` 替换下面命令中的 `<TAG>`，用你想下载的 Argo CD 版本：

```bash
VERSION=<TAG> # Select desired TAG from https://github.com/argoproj/argo-cd/releases
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

#### 下载最新稳定版本

您可以通过以下步骤下载最新的稳定版发布：

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

现在应该可以运行 `argocd` 命令了。

## Mac (M1)

#### 使用 Curl 下载

您可以通过上面的链接查看 Argo CD 的最新版本，或运行以下命令抓取版本：

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
```

将下面命令中的 `VERSION` 替换为您要下载的 Argo CD 版本：

```bash
curl -sSL -o argocd-darwin-arm64 https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-darwin-arm64
```

安装 Argo CD CLI 二进制文件：

```bash
sudo install -m 555 argocd-darwin-arm64 /usr/local/bin/argocd
rm argocd-darwin-arm64
```

## Mac

#### 自制

```bash
brew install argocd
```

#### 使用 Curl 下载

您可以通过上面的链接查看 Argo CD 的最新版本，或运行以下命令抓取版本：

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
```

将下面命令中的 `VERSION` 替换为您要下载的 Argo CD 版本：

```bash
curl -sSL -o argocd-darwin-amd64 https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-darwin-amd64
```

安装 Argo CD CLI 二进制文件：

```bash
sudo install -m 555 argocd-darwin-amd64 /usr/local/bin/argocd
rm argocd-darwin-amd64
```

完成上述任一说明后，现在就可以运行 `argocd` 命令了。

## Windows

### 使用 PowerShell 下载：Invoke-WebRequest

您可以通过上面的链接查看 Argo CD 的最新版本，或运行以下命令抓取版本：

```powershell
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
```

将下面命令中的 `$version` 替换为您要下载的 Argo CD 版本：

```powershell
$url = "https://github.com/argoproj/argo-cd/releases/download/" + $version + "/argocd-windows-amd64.exe"
$output = "argocd.exe"

Invoke-WebRequest -Uri $url -OutFile $output
```

此外，请注意您可能需要将文件移动到 PATH 中。 使用以下命令将 Argo CD 添加到环境变量 PATH 中

```powershell
[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\Path\To\ArgoCD-CLI", "User")
```

完成上述说明后，现在就可以运行 `argocd` 命令了。