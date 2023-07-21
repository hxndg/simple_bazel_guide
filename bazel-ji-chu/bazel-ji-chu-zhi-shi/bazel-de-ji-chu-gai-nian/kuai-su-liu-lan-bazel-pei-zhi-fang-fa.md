# 快速浏览Bazel配置方法

### 引言

这部分会针对性地讲一下怎么快速看配置相关的东西，



### 为什么我要阅读这一页？

Bazel体系的用户经常只局限于一部分Scope，即我怎么写个library，怎么写个binary。对于配置如何生效。代码怎么启用不同的编译配置不理解。因此我建议用户看看本部分的例子，理解下怎么去梳理配置的逻辑



### Learned By Example

简单来讲，看配置的方法基本就是三步

* 先看WORKSPACE看工具链
  * WORKSPACE中工具链的配置可以方便理解具体的规则上下文，配置是否移动化，可以阅读WORKSPACE中的配置就能看出来
* 递归地看.bazelrc里面的通用配置组
  *   配置组.bazelrc的阅读过程中，要注意try-import语句，正确的情况下会拆分为

      ```
      # Missing CI section
      try-import %workspace%/abc.bazelrc
      ```
* 递归地看BUILD文件里面load的各种语句的实现
  * 具体的各种语句，比方说rules docker的规则细节什么的，都可以直接对应到starlark语句。



简单地看一个例子

WORKSPACE如下，可以发现这是一个go的仓库，其初始化了GO的依赖，同时引入了rules\_docker

```
workspace(name = "debug")

load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
load("//bazel:go_deps.bzl", "go_dependencies")

# gazelle:repository_macro bazel/go_deps.bzl%go_dependencies
go_dependencies()

load("@io_bazel_rules_docker//repositories:deps.bzl", rules_docker_go_deps = "deps")

rules_docker_go_deps()  
```

.bazelrc如下，可以发现环境默认使用python3，编译的C++选项默认为C++17，且开启了fpic，release模式下，开启了thin级别的flto。

```
build --java_runtime_version=remotejdk_11

build --host_force_python=PY3
build --python_version=PY3

build --cxxopt=-std=c++17

build --experimental_cc_shared_library
build --experimental_link_static_libraries_once

# This is to avoid building both PIC and non-PIC versions in opt mode,
# which doubles the build time.
build --force_pic

build:release --copt="-g1" --copt="-flto=thin"
```



