<!-- TRANSLATED by md-translate -->
# 插件生成器

Plugins 允许您提供自己的生成器。

* 可使用任何语言编写
* 简单：插件只需响应 RPC HTTP 请求。
* 可被引用到侧载或独立部署中。
* 您可以立即运行插件，无需等待 3-5 个月的审核、批准、合并和 Argo 软件发布。
* 您可以将它与 Matrix 或 Merge 结合使用。

要开始制作自己的插件，可以根据 [ApplicationSet-hello-plugin](https://github.com/argoproj-labs/applicationset-hello-plugin) 示例生成一个新的版本库。

## 简单示例

被引用生成器插件而未与 Matrix 或 Merge 结合使用。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myplugin
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - plugin:
        # Specify the configMap where the plugin configuration is located.
        configMapRef:
          name: my-plugin
        # You can pass arbitrary parameters to the plugin. `input.parameters` is a map, but values may be any type.
        # These parameters will also be available on the generator's output under the `generator.input.parameters` key.
        input:
          parameters:
            key1: "value1"
            key2: "value2"
            list: ["list", "of", "values"]
            boolean: true
            map:
              key1: "value1"
              key2: "value2"
              key3: "value3"

        # You can also attach arbitrary values to the generator's output under the `values` key. These values will be
        # available in templates under the `values` key.
        values:
          value1: something

        # When using a Plugin generator, the ApplicationSet controller polls every `requeueAfterSeconds` interval (defaulting to every 30 minutes) to detect changes.
        requeueAfterSeconds: 30
  template:
    metadata:
      name: myplugin
      annotations:
        example.from.input.parameters: "{{ index .generator.input.parameters.map "key1" }}"
        example.from.values: "{{ .values.value1 }}"
        # The plugin determines what else it produces.
        example.from.plugin.output: "{{ .something.from.the.plugin }}"
```

* `configMapRef.name`：ConfigMap "名称，包含被引用用于 RPC 调用的插件配置。
* `input.parameters`：包含在插件 RPC 调用中的输入参数。可选

注意 插件的概念不应破坏 GitOps 的精神，将 Git 外部的数据外部化。 插件的目标是在特定情况下起到补充作用。 例如，当使用其中一个 PullRequest 生成器时，不可能检索到与 CI 相关的参数（只有提交哈希值可用），这就限制了可能性。 通过使用插件，可以从一个单独的数据源检索到必要的参数，并使用它们来扩展生成器的功能。

### 添加 configmaps 以配置插件的访问权限

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-plugin
  namespace: argocd
data:
  token: "$plugin.myplugin.token" # Alternatively $<some_K8S_secret>:plugin.myplugin.token
  baseUrl: "http://myplugin.plugin-ns.svc.cluster.local."
```

* 令牌用于验证 HTTP 请求的预共享令牌（指向您在 `argocd-secret` Secret 中创建的正确密钥）
* `baseUrl`：集群中公开插件的 k8s 服务的 BaseUrl。

### 存储凭证

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  # ...
  # The secret value must be base64 encoded **once**.
  # this value corresponds to: `printf "strong-password" | base64`.
  plugin.myplugin.token: "c3Ryb25nLXBhc3N3b3Jk"
  # ...
```

#### 替代方案

如果你想把敏感数据存储在**另一个**的 Kubernetes "秘密 "中，而不是 "argocd-secret"，ArgoCD 知道如何检查你的 Kubernetes "秘密 "中 "data "下的密钥，以查找相应的密钥，只要 configmaps 中的值以"$"开头，然后是你的 Kubernetes "秘密 "名称和":"（冒号），接着是密钥名称。

语法： `$<k8s_secret_name>:<a_key_in_that_k8s_secret>`

&gt; 注意：Secret 必须带有标签`app.kubernetes.io/part-of:argocd`。

##### 示例

`another-secret`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: another-secret
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  # ...
  # Store client secret like below.
  # The secret value must be base64 encoded **once**.
  # This value corresponds to: `printf "strong-password" | base64`.
  plugin.myplugin.token: "c3Ryb25nLXBhc3N3b3Jk"
```

### HTTP 服务器

#### 一个简单的 Python 插件

您可以将其作为一个辅助工具或独立部署（建议使用后者）。

在示例中，令牌存储在以下位置的文件中：`/var/run/argo/token`。

```
strong-password
```

```python
import json
from http.server import BaseHTTPRequestHandler, HTTPServer

with open("/var/run/argo/token") as f:
    plugin_token = f.read().strip()

class Plugin(BaseHTTPRequestHandler):

    def args(self):
        return json.loads(self.rfile.read(int(self.headers.get('Content-Length'))))

    def reply(self, reply):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(json.dumps(reply).encode("UTF-8"))

    def forbidden(self):
        self.send_response(403)
        self.end_headers()

    def unsupported(self):
        self.send_response(404)
        self.end_headers()

    def do_POST(self):
        if self.headers.get("Authorization") != "Bearer " + plugin_token:
            self.forbidden()

        if self.path == '/api/v1/getparams.execute':
            args = self.args()
            self.reply({
                "output": {
                    "parameters": [
                        {
                            "key1": "val1",
                            "key2": "val2"
                        },
                        {
                            "key1": "val2",
                            "key2": "val2"
                        }
                    ]
                }
            })
        else:
            self.unsupported()

if __name__ == '__main__':
    httpd = HTTPServer(('', 4355), Plugin)
    httpd.serve_forever()
```

使用 curl 执行 getparams ：

```
curl http://localhost:4355/api/v1/getparams.execute -H "Authorization: Bearer strong-password" -d \
'{
  "applicationSetName": "fake-appset",
  "input": {
    "parameters": {
      "param1": "value1"
    }
  }
}'
```

这里有几点需要注意：

* 您只需执行`/api/v1/getparams.execute`调用。
* 您应检查 `Authorization` 标头是否包含与 `/var/run/argo/token`相同的承载值。否则返回 403
* 输入参数包含在请求正文中，可使用 `input.parameters` 变量访问。
* 输出必须始终是一个对象映射列表，嵌套在映射中的 `output.parameters` 键下。
* `generator.input.parameters` 和 `values` 是保留键。如果出现在插件输出中，这些键将被 ApplicationSet 插件生成器规范中的 `input.parameters` 和 `values` 键的内容覆盖。

## 带有矩阵和拉取请求示例

在下面的示例中，插件实现正在返回一组给定分支的镜像摘要。 返回的列表只包含一个项目，对应于该分支最新构建的镜像。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: fb-matrix
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          - pullRequest:
              github: ...
              requeueAfterSeconds: 30
          - plugin:
              configMapRef:
                name: cm-plugin
              input:
                parameters:
                  branch: "{{.branch}}" # provided by generator pull request
              values:
                branchLink: "https://git.example.com/org/repo/tree/{{.branch}}"
  template:
    metadata:
      name: "fb-matrix-{{.branch}}"
    spec:
      source:
        repoURL: "https://github.com/myorg/myrepo.git"
        targetRevision: "HEAD"
        path: charts/my-chart
        helm:
          releaseName: fb-matrix-{{.branch}}
          valueFiles:
            - values.yaml
          values: |
            front:
              image: myregistry:{{.branch}}@{{ .digestFront }} # digestFront is generated by the plugin
            back:
              image: myregistry:{{.branch}}@{{ .digestBack }} # digestBack is generated by the plugin
      project: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{.branch}}"
      info:
        - name: Link to the Application's branch
          value: "{{values.branchLink}}"
```

举例说明：

* 例如，生成器 pullRequest 会返回 2 个分支：特性分支-1 "和 "特性分支-2"。
* 然后，生成器插件将执行以下 2 个请求：

```shell
curl http://localhost:4355/api/v1/getparams.execute -H "Authorization: Bearer strong-password" -d \
'{
  "applicationSetName": "fb-matrix",
  "input": {
    "parameters": {
      "branch": "feature-branch-1"
    }
  }
}'
```

那么

```shell
curl http://localhost:4355/api/v1/getparams.execute -H "Authorization: Bearer strong-password" -d \
'{
  "applicationSetName": "fb-matrix",
  "input": {
    "parameters": {
      "branch": "feature-branch-2"
    }
  }
}'
```

每次调用，它都会返回一个唯一的结果，如 ：

```json
{
  "output": {
    "parameters": [
      {
        "digestFront": "sha256:a3f18c17771cc1051b790b453a0217b585723b37f14b413ad7c5b12d4534d411",
        "digestBack": "sha256:4411417d614d5b1b479933b7420079671facd434fd42db196dc1f4cc55ba13ce"
      }
    ]
  }
}
```

那么

```json
{
  "output": {
    "parameters": [
      {
        "digestFront": "sha256:7c20b927946805124f67a0cb8848a8fb1344d16b4d0425d63aaa3f2427c20497",
        "digestBack": "sha256:e55e7e40700bbab9e542aba56c593cb87d680cefdfba3dd2ab9cfcb27ec384c2"
      }
    ]
  }
}
```

在本例中，通过将两者结合，您可以确保一个或多个拉取请求可用，并确保已正确生成标签。 如果仅使用提交哈希值，这是不可能实现的，因为仅有哈希值并不能证明构建成功。