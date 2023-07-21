# Bazel（配置组和BUILD）部分细节

### 控制任务并发数量

使用下面的语句控制action并发数量

```shellscript
build --jobs=xx
```



### BUILD文件格式化

&#x20;[Buildifier](https://github.com/bazelbuild/buildifier)是官方推荐的格式化工具，一部分研发不是很能理解格式化的意义，我建议这部分人就别做研发了。

### BUILD共享变量

Bazel的BUILD文件可以共享变量，借此提高可读性，在单个文件内共享配置的简单例子如下

```
COPTS = ["-DVERSION=5"]

cc_library(
  name = "foo",
  copts = COPTS,
  srcs = ["foo.cc"],
)

cc_library(
  name = "bar",
  copts = COPTS,
  srcs = ["bar.cc"],
  deps = [":foo"],
)
```

如果想在多个文件共享，可以如下面所示

在 `path/to/variables.bzl`, 有如下的内容:

```
COPTS = ["-DVERSION=5"]
```

在其它的BUILD文件里面可以如此来引用这个共享变量:

```
load("//path/to:variables.bzl", "COPTS")

cc_library(
  name = "foo",
  copts = COPTS,
  srcs = ["foo.cc"],
)

cc_library(
  name = "bar",
  copts = COPTS,
  srcs = ["bar.cc"],
  deps = [":foo"],
)
```

