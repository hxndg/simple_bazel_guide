# 入门级别的Bazel Go知识

### 引言

我建议用户直接阅读[https://github.com/bazelbuild/rules\_go](https://github.com/bazelbuild/rules\_go)，获取最标准和最新的使用方法



### 为什么我需要阅读这一页？

如果你想知道怎么从GO原始的go build，迁移到bazel下面的go，那么我建议你看看这个部分

### GUIDE

两个部分，第一个部分适用于从头开始写基于Bazel的Go代码，第二个适用于从GO原生的go build切换到Bazel的情况。

#### 使用原生的方式编写GO Target

首先要引入对rules\_go的支持，在WORKSPACE文件当中写入下面的内容，使用 [git\_repository](https://docs.bazel.build/versions/master/repo/git.html) 替代 [http\_archive](https://docs.bazel.build/versions/master/repo/http.html#http\_archive) 也可以，不过http\_archive提供了sha256，方便缓存&#x20;

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "io_bazel_rules_go",
    sha256 = "51dc53293afe317d2696d4d6433a4c33feedb7748a9e352072e2ec3c0dafd2c6",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
    ],
)

load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")

go_rules_dependencies()

go_register_toolchains(version = "1.20.5")
```

接下来，新建一个BUILD文件，并开发对应的go代码，最后bazel build 即可

```
load("@io_bazel_rules_go//go:def.bzl", "go_binary")

go_binary(
    name = "hello",
    srcs = ["hello.go"],
)
```



#### 迁移原始GO代码到Bazel

从GO迁移到Bazel自然可以直接手写对应的BUILD文件，比较麻烦，推荐使用gazelle来解决这个问题

在WORKSPACE里面添加gazelle

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "io_bazel_rules_go",
    sha256 = "51dc53293afe317d2696d4d6433a4c33feedb7748a9e352072e2ec3c0dafd2c6",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.40.1/rules_go-v0.40.1.zip",
    ],
)

http_archive(
    name = "bazel_gazelle",
    sha256 = "727f3e4edd96ea20c29e8c2ca9e8d2af724d8c7778e7923a854b2c80952bc405",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
    ],
)

load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")

go_rules_dependencies()

go_register_toolchains(version = "1.20.5")

gazelle_dependencies()
```

接着，在代码的根目录添加一个BUILD.bazel，把prefix后面的那个字符串，就是那个什么github.comxxx啥的东西，替换为原始项目的import path，也就是 `go.mod` 里面的部分

```
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix github.com/example/project
gazelle(name = "gazelle")
```

接着执行，就会生成具体的BUILD文件。

```
bazel run //:gazelle
```

\
