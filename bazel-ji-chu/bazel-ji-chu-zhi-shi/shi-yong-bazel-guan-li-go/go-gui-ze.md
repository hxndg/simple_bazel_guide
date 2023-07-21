# GO规则

说起来，还是推荐直接看，ttps://github.com/bazelbuild/rules\_go



### 工具链的配置

GO的工具链有三个层面， [the SDK](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#the-sdk), [the toolchain](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#the-toolchain), 和 [the context](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#the-context).，一般来说，用户只需要关注SDK层面的事情，比方说外部依赖什么的参考下面的部分

#### GO SDK

SDK说白了就是GO的标准库，工具链的源代码还有一些预编译的库之类的东西。这部分一般是用[go\_download\_sdk](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#go-download-sdk)来显示地选择一个版本，有一些通用的命令

* [go\_download\_sdk](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#go-download-sdk)：为特定操作系统和架构下载特定版本 Go 的工具链。
* [go\_host\_sdk](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#go-host-sdk)：使用运行 Bazel 的系统上安装的工具链。工具链的位置通过`GOROOT`或 运行来 指定`go env GOROOT`。

如果想引用1.15.5的GO版本，那么可以在WORKSPACE里面添加下面内容

```
# WORKSPACE

load("@io_bazel_rules_go//go:deps.bzl", "go_rules_dependencies", "go_register_toolchains")

go_rules_dependencies()

go_register_toolchains(version = "1.15.5")
```

如果想用机器安装的GO版本，那么改下为如下的内容

```
# WORKSPACE

load("@io_bazel_rules_go//go:deps.bzl", "go_rules_dependencies", "go_register_toolchains")

go_rules_dependencies()

go_register_toolchains(version = "host")
```

#### Toolchain

SDK 特定于主机平台（例如`linux_amd64`）和 Go 版本，而Tollchain就是解决这个问题，一般来说，运行Bazel构建时，Bazel会根据指定的执行平台和目标平台（分别用--host\_platform和--platforms选项指定）自动选择已注册的工具链。这样，就无需手动配置工具链，Bazel会自动根据平台的特性和需求选择合适的工具链，以确保构建的正确性和可靠性。

举个例子，很多时候我编译都是直接调用

```shellscript
bazel build -c opt --platforms=@io_bazel_rules_go//go/toolchain:linux_amd64 //save_money:money
```

#### Context

就是指引用具体的规则，不过这个用户一般不关心，如果要自定义规则才需要使用相关的东西，参考[https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#go-context](https://github.com/bazelbuild/rules\_go/blob/master/go/toolchains.rst#go-context)



### 引入外部依赖

说起来很简单，首先在WORKSPACE里面添加对gazelle的使用，接着直接使用go\_repository即可，

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Download the Go rules.
http_archive(
    name = "io_bazel_rules_go",
    sha256 = "51dc53293afe317d2696d4d6433a4c33feedb7748a9e352072e2ec3c0dafd2c6",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
    ],
)

# Download Gazelle.
http_archive(
    name = "bazel_gazelle",
    sha256 = "727f3e4edd96ea20c29e8c2ca9e8d2af724d8c7778e7923a854b2c80952bc405",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
    ],
)

# Load macros and repository rules.
load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies", "go_repository")

# Declare Go direct dependencies.
go_repository(
    name = "org_golang_x_net",
    importpath = "golang.org/x/net",
    sum = "h1:zK/HqS5bZxDptfPJNq8v7vJfXtkU7r9TLIoSr1bXaP4=",
    version = "v0.0.0-20200813134508-3edf25e44fcc",
)

# Declare indirect dependencies and register toolchains.
go_rules_dependencies()

go_register_toolchains(version = "1.20.5")

gazelle_dependencies()
```

如果希望从原始的go.mod里面导入依赖，用下面的语句就可以直接添加到依赖了

```
bazel run //:gazelle update-repos repo-uri
```

如果只想添加Kafka 的 segmentio 的 go client 包，只需要在项目根目录下执行命令：

```
bazel run //:gazelle update-repos github.com/segmentio/kafka-g
```

Gazelle 便会自动增加一条依赖到 WORKSPACE 文件：

```
go_repository(
    name = "com_github_segmentio_kafka_go",
    importpath = "github.com/segmentio/kafka-go",
    sum = "h1:Mv9AcnCgU14/cU6Vd0wuRdG1FBO0HzXQLnjBduDLy70=",
    version = "v0.3.4",
)
```



对于Infra同学而言，都写到WORKSPACE中可能会导致WORKSPACE贼大，那么统一组织这些依赖就比较方便：

先创建一个文件，比方说叫做bazel/go\_deps.bzl，这个实际上定义了一个go\_dependencies的规则，同时定义了大量的第三方依赖包

```
load("@bazel_gazelle//:deps.bzl", "go_repository")

def grouped_go_dependencies():
    go_repository(
        name = "com_github_golang_glog",
        build_file_proto_mode = "disable_global",
        importpath = "github.com/golang/glog",
        sum = "h1:nfP3RFugxnNRyKgeWd4oI1nYvXpxrx8ck8ZrcizshdQ=",
        version = "v1.0.0",
    )
```

在WORKSPACE里面写入下面的内容，实际上就是展开了grouped\_go\_dependencies规则，也就是把各种go\_repository写在了Workspace里面，当然本质还是没变

```julia
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
load("//bazel:go_deps.bzl", "grouped_go_dependencies")

grouped_go_dependencies()
```

