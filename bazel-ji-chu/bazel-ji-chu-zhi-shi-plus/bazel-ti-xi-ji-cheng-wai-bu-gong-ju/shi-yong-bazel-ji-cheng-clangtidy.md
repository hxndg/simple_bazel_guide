---
description: Clang-Tidy
---

# 使用Bazel集成Clang-Tidy

### 引言

Clang-Tidy是基于Clang的C++ Lint工具，用于检查C++代码中的常见编码错误，执行代码静态分析等，和Bazel的继承使用了bazel的aspect机制实现

建议直接阅读[https://github.com/erenon/bazel\_clang\_tidy](https://github.com/erenon/bazel\_clang\_tidy)

### 为啥我要阅读这一页？

原因很简单，针对语言层面的安全检查是SAST实践的核心。因此怎么在Bazel上集成clang-tidy就变成一种通用的手段，本质是使用Bazel提供的Aspect。

这种方式的好处有三个

* 可以针对任意的C++对象执行clang-tidy
* 因为用的是bazel的aspect，所以实际上并没有做真正的构建。直接调用clang-tidy对per文件做分析
* Bazel会缓存clang-tidy 的结果，如果文件没变化，那么clang-tidy不会重新运行。



### 使用方法

在WORKSPACE里面添加代码

```
# //:WORKSPACE
load(
    "@bazel_tools//tools/build_defs/repo:git.bzl",
    "git_repository",
)

git_repository(
       name = "bazel_clang_tidy",
       commit = "69aa13e6d7cf102df70921c66be15d4592251e56",
       remote = "https://github.com/erenon/bazel_clang_tidy.git",
)
```

具体调用可以直接显式使用aspect调用，或者修改.bazelrc，贴到了最下面

```
bazel build //... \
  --aspects @bazel_clang_tidy//clang_tidy:clang_tidy.bzl%clang_tidy_aspect \
  --output_groups=report
```

修改bazelrc如下，然后直接调用tidy配置即可

````
```ini
build:tidy --aspects //bazel/clang_tidy:clang_tidy.bzl%clang_tidy_aspect
build:tidy --build_tag_filters=-no-tidy
build:tidy --output_groups=report

```
````

```
bazel build --config=tidy //...
```

接下来就会针对build的对象启动clang-tidy的检查。



我们都非常清楚，clang-tidy是一个复杂检查，其中有很多种的checker可以开启，如果需要配置clang-tidy的config，可以添加一个新的配置文件，调用的时候使用第二条命令。

```
# //:BUILD
filegroup(
       name = "clang_tidy_config",
       srcs = [".clang-tidy"],
       visibility = ["//visibility:public"],
)
```

```
bazel build //... \
  --aspects @bazel_clang_tidy//clang_tidy:clang_tidy.bzl%clang_tidy_aspect \
  --output_groups=report \
  --@bazel_clang_tidy//:clang_tidy_config=//:clang_tidy_config
```
