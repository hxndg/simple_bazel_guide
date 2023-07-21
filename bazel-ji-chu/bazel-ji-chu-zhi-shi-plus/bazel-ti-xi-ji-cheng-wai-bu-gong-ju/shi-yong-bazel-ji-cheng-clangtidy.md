---
description: Clang-Tidy
---

# 使用Bazel集成Clang-Tidy

### 引言

Clang-Tidy是基于Clang的C++ Lint工具，用于检查C++代码中的常见编码错误，执行代码静态分析等，和Bazel的继承使用了bazel的aspect机制实现

建议直接阅读[https://github.com/erenon/bazel\_clang\_tidy](https://github.com/erenon/bazel\_clang\_tidy)

### 为啥我要阅读这一页？

原因很简单，针对语言层面的安全检查是SAST实践的核心。因此怎么在Bazel上集成clang-tidy就变成一种通用的手段，本质是使用Bazel提供的Aspect



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

接下来就会针对build的对象启动clang-tidy的检查

