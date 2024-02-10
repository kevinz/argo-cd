<!-- TRANSLATED by md-translate -->
### **时间**

与时间相关的功能。

<hr>
**`time.Now() Time`**

执行函数内置的 Golang [time.Now](https://golang.org/pkg/time/#Now) 函数。返回 Golang [Time](https://golang.org/pkg/time/#Time) 的实例。

<hr>
**`time.Parse(val string) Time`**

使用 RFC3339 布局解析指定字符串。返回 Golang [Time](https://golang.org/pkg/time/#Time) 的实例。

### **字符串**

字符串相关函数

<hr>
**`strings.ReplaceAll() string`**

执行 Golang [strings.ReplaceAll](https://pkg.go.dev/strings#ReplaceAll) 内置函数。

<hr>
**`strings.ToUpper() string`**

执行 Golang [strings.ToUpper](https://pkg.go.dev/strings#ToUpper) 内置函数。

<hr>
**`strings.ToLower() string`**

执行 Golang [strings.ToLower](https://pkg.go.dev/strings#ToLower) 内置函数。

### **sync**

<hr>
**`sync.GetInfoItem(app map, name string) string`**
Returns the `info` item value by given name stored in the Argo CD App sync operation.

### **repo**

提供有关应用程序源码库附加信息的函数。

<hr>
**`repo.RepoURLToHTTPS(url string) string`**

将给定的 GIT URL 转换为 HTTPs 格式。

<hr>
**`repo.FullNameByRepoURL(url string) string`**

Returns repository URL full name `(<owner>/<repoName>)`。目前仅支持 Github、GitLab 和 Bitbucket。

<hr>
**`repo.GetCommitMetadata(sha string) CommitMetadata`**

返回提交元数据。 提交必须属于应用程序源代码库：

* `Message string` 提交信息
* `Author string` - 提交作者
* `Date time.Time` - 提交创建日期
* `Tags []string` - 相关标签

<hr>
**`repo.GetAppDetails() AppDetail`**

返回应用程序的详细信息：

* `Type string` - AppDetail 类型
* `Helm HelmAppSpec` - Helm 详情
    - 字段 ：
        + `Name string
        + `ValueFiles []string`.
        + `Parameters []*v1alpha1.HelmParameter`.
        +`Values string` +`FileParameters []*v1alpha1.HelmParameter` +`Values string`
        +`FileParameters []*v1alpha1.HelmFileParameter` +`ValueFiles []string
    - 方法 ：
        + `GetParameterValueByName(Name string)` 按名称读取参数字段中的值
        + `GetFileParameterPathByName(Name string)` 在 FileParameters 字段中按名称读取路径
* 
* `Kustomize *apiclient.KustomizeAppSpec` - 定制细节
* 目录 *apiclient.DirectoryAppSpec` - 目录详情
