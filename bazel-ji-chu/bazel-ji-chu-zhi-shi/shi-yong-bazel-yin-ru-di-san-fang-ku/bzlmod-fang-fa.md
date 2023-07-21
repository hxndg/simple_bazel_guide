# Bzlmod方法

还是推荐阅读原始文档[https://github.com/bazelbuild/examples/tree/main/bzlmod](https://github.com/bazelbuild/examples/tree/main/bzlmod)和[https://bazel.build/external/module](https://bazel.build/external/module)

### 实现代码

WORKSPACE文件这么写

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "com_github_google_glog",
    sha256 = "eede71f28371bf39aa69b45de23b329d37214016e2055269b3b5e7cfd40b59f5",
    strip_prefix = "glog-0.5.0",
    urls = ["https://github.com/google/glog/archive/refs/tags/v0.5.0.tar.gz"],
)

# We have to define gflags as well because it's a dependency of glog.
http_archive(
    name = "com_github_gflags_gflags",
    sha256 = "34af2f15cf7367513b352bdcd2493ab14ce43692d2dcd9dfc499492966c64dcf",
    strip_prefix = "gflags-2.2.2",
    urls = ["https://github.com/gflags/gflags/archive/refs/tags/v2.2.2.tar.gz"],
)
```

在附加一个[MODULE.bazel](https://github.com/bazelbuild/examples/blob/main/bzlmod/01-depend\_on\_bazel\_module/MODULE.bazel)文件

```
# 1. The metadata of glog is fetched from the BCR, including its dependencies (gflags).
# 2. The `repo_name` attribute allows users to reference this dependency via the `com_github_google_glog` repo name.
bazel_dep(name = "glog", version = "0.5.0", repo_name = "com_github_google_glog")
```

编译的时候，添加属性--enable\_bzelmod即可，并在对应的BUILD文件添加依赖

```
build --enable_bzlmod
```

### 原理分析

Bazel的bzlmod使用类似GO的[Minimal Version Selection](https://research.swtch.com/vgo-mvs) (MVS) 算法，即MVS 假设所有依赖满足前向兼容性，因此挑选第一个满足所有兼容性要求的版本。对这个图里面的例子，会选择1.1的D版本。如果有1.2的D版本，也会用1.1的版本。

```
       A 1.0
      /     \
   B 1.0    C 1.1
     |        |
   D 1.0    D 1.1
```
