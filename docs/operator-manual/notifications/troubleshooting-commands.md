<!-- TRANSLATED by md-translate -->
## argocd 管理通知模板获取

打印已配置模板的信息

```
argocd admin notifications template get [flags]
```

### 示例

```
# prints all templates
argocd admin notifications template get
# print YAML formatted app-sync-succeeded template definition
argocd admin notifications template get app-sync-succeeded -o=yaml
```

### 选项

```
-h, --help help for get
  -o, --output string Output format. One of:json|yaml|wide|name (default "wide")
```

###从上级命令继承的选项

```
--argocd-repo-server string Argo CD repo server address (default "argocd-repo-server:8081")
      --argocd-repo-server-plaintext Use a plaintext client (non-TLS) to connect to repository server
      --argocd-repo-server-strict-tls Perform strict validation of TLS certificates when connecting to repo server
      --as string Username to impersonate for the operation
      --as-group stringArray Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string UID to impersonate for the operation
      --certificate-authority string Path to a cert file for the certificate authority
      --client-certificate string Path to a client certificate file for TLS
      --client-key string Path to a client key file for TLS
      --cluster string The name of the kubeconfig cluster to use
      --config-map string argocd-notifications-cm.yaml file path
      --context string The name of the kubeconfig context to use
      --disable-compression If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string Path to a kube config. Only required if out-of-cluster
  -n, --namespace string If present, the namespace scope for this CLI request
      --password string Password for basic authentication to the API server
      --proxy-url string If provided, this URL will be used to connect via proxy
      --request-timeout string The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
      --secret string argocd-notifications-secret.yaml file path. Use empty secret if provided value is ':empty'
      --server string The address and port of the Kubernetes API server
      --tls-server-name string If provided, this name will be used to validate server certificate. If this is not provided, hostname used to contact the server is used.
      --token string Bearer token for authentication to the API server
      --user string The name of the kubeconfig user to use
      --username string Username for basic authentication to the API server
```

## argocd 管理通知模板通知

使用指定模板生成通知并发送给指定收件人

```
argocd admin notifications template notify NAME RESOURCE_NAME [flags]
```

### 示例

```
# Trigger notification using in-cluster config map and secret
argocd admin notifications template notify app-sync-succeeded guestbook --recipient slack:my-slack-channel

# Render notification render generated notification in console
argocd admin notifications template notify app-sync-succeeded guestbook
```

### 选项

```
-h, --help help for notify
      --recipient stringArray List of recipients (default [console:stdout])
```

###从上级命令继承的选项

```
--argocd-repo-server string Argo CD repo server address (default "argocd-repo-server:8081")
      --argocd-repo-server-plaintext Use a plaintext client (non-TLS) to connect to repository server
      --argocd-repo-server-strict-tls Perform strict validation of TLS certificates when connecting to repo server
      --as string Username to impersonate for the operation
      --as-group stringArray Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string UID to impersonate for the operation
      --certificate-authority string Path to a cert file for the certificate authority
      --client-certificate string Path to a client certificate file for TLS
      --client-key string Path to a client key file for TLS
      --cluster string The name of the kubeconfig cluster to use
      --config-map string argocd-notifications-cm.yaml file path
      --context string The name of the kubeconfig context to use
      --disable-compression If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string Path to a kube config. Only required if out-of-cluster
  -n, --namespace string If present, the namespace scope for this CLI request
      --password string Password for basic authentication to the API server
      --proxy-url string If provided, this URL will be used to connect via proxy
      --request-timeout string The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
      --secret string argocd-notifications-secret.yaml file path. Use empty secret if provided value is ':empty'
      --server string The address and port of the Kubernetes API server
      --tls-server-name string If provided, this name will be used to validate server certificate. If this is not provided, hostname used to contact the server is used.
      --token string Bearer token for authentication to the API server
      --user string The name of the kubeconfig user to use
      --username string Username for basic authentication to the API server
```

## argocd 管理通知触发器获取

打印已配置触发器的信息

```
argocd admin notifications trigger get [flags]
```

### 示例

```
# prints all triggers
argocd admin notifications trigger get
# print YAML formatted on-sync-failed trigger definition
argocd admin notifications trigger get on-sync-failed -o=yaml
```

### 选项

```
-h, --help help for get
  -o, --output string Output format. One of:json|yaml|wide|name (default "wide")
```

###从上级命令继承的选项

```
--argocd-repo-server string Argo CD repo server address (default "argocd-repo-server:8081")
      --argocd-repo-server-plaintext Use a plaintext client (non-TLS) to connect to repository server
      --argocd-repo-server-strict-tls Perform strict validation of TLS certificates when connecting to repo server
      --as string Username to impersonate for the operation
      --as-group stringArray Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string UID to impersonate for the operation
      --certificate-authority string Path to a cert file for the certificate authority
      --client-certificate string Path to a client certificate file for TLS
      --client-key string Path to a client key file for TLS
      --cluster string The name of the kubeconfig cluster to use
      --config-map string argocd-notifications-cm.yaml file path
      --context string The name of the kubeconfig context to use
      --disable-compression If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string Path to a kube config. Only required if out-of-cluster
  -n, --namespace string If present, the namespace scope for this CLI request
      --password string Password for basic authentication to the API server
      --proxy-url string If provided, this URL will be used to connect via proxy
      --request-timeout string The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
      --secret string argocd-notifications-secret.yaml file path. Use empty secret if provided value is ':empty'
      --server string The address and port of the Kubernetes API server
      --tls-server-name string If provided, this name will be used to validate server certificate. If this is not provided, hostname used to contact the server is used.
      --token string Bearer token for authentication to the API server
      --user string The name of the kubeconfig user to use
      --username string Username for basic authentication to the API server
```

## argocd 管理通知触发器运行

评估指定的触发条件并打印结果

```
argocd admin notifications trigger run NAME RESOURCE_NAME [flags]
```

### 示例

```
# Execute trigger configured in 'argocd-notification-cm' ConfigMap
argocd admin notifications trigger run on-sync-status-unknown ./sample-app.yaml

# Execute trigger using my-config-map.yaml instead of 'argocd-notifications-cm' ConfigMap
argocd admin notifications trigger run on-sync-status-unknown ./sample-app.yaml \
    --config-map ./my-config-map.yaml
```

### 选项

```
-h, --help help for run
```

###从上级命令继承的选项

```
--argocd-repo-server string Argo CD repo server address (default "argocd-repo-server:8081")
      --argocd-repo-server-plaintext Use a plaintext client (non-TLS) to connect to repository server
      --argocd-repo-server-strict-tls Perform strict validation of TLS certificates when connecting to repo server
      --as string Username to impersonate for the operation
      --as-group stringArray Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string UID to impersonate for the operation
      --certificate-authority string Path to a cert file for the certificate authority
      --client-certificate string Path to a client certificate file for TLS
      --client-key string Path to a client key file for TLS
      --cluster string The name of the kubeconfig cluster to use
      --config-map string argocd-notifications-cm.yaml file path
      --context string The name of the kubeconfig context to use
      --disable-compression If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string Path to a kube config. Only required if out-of-cluster
  -n, --namespace string If present, the namespace scope for this CLI request
      --password string Password for basic authentication to the API server
      --proxy-url string If provided, this URL will be used to connect via proxy
      --request-timeout string The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
      --secret string argocd-notifications-secret.yaml file path. Use empty secret if provided value is ':empty'
      --server string The address and port of the Kubernetes API server
      --tls-server-name string If provided, this name will be used to validate server certificate. If this is not provided, hostname used to contact the server is used.
      --token string Bearer token for authentication to the API server
      --user string The name of the kubeconfig user to use
      --username string Username for basic authentication to the API server
```