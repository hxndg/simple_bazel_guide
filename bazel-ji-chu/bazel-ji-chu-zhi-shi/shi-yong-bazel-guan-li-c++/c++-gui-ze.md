# C++规则

这里只写了通用性的bazel规则，具体的细节建议阅读github里面的文档

### CC\_IMPORT

```
cc_import(
  name = "mylib",
  hdrs = ["mylib.h"],
  static_library = "libmylib.a",
  # If alwayslink is turned on,
  # libmylib.a will be forcely linked into any binary that depends on it.
  # alwayslink = 1,
)
```

cc\_import是用来解决外部依赖的问题的，一般用来将C++头文件和静态库从外部包或存储库导入到项目里，这个可以提供给跨平台编译使用





### CC\_LIBRARY

```
cc_library(
    name = "my_library",
    srcs = ["file1.cpp", "file2.cpp"],
    hdrs = ["header1.h", "header2.h"],
    visibility = ["//visibility:public"],
    deps = ["//path/to/dependency"],
    defines = ["DEBUG"],
    copts = ["-Wall", "-O2"],
)
```

`cc_library` 是 Bazel 中专门用于构建 C++ 库的构建规则，有以下属性。

* `name`：指定库目标的名称。
* `srcs`：列出库的 C++ 源文件（`.cpp` 文件）。
* `hdrs`：列出库的 C++ 头文件（`.h` 文件）。
* `visibility`：设置库目标的可见性，以控制其他目标是否可以依赖它。它使用标签模式，如 `"//path/to/package:target"`。
* `deps`：指定库的依赖项。这可以是其他 `cc_library` 目标或库所依赖的其他类型的目标。
* `defines`：定义预处理宏，用于编译过程中使用。
* `copts`：设置库的编译选项，如警告标志或优化级别。

### CC\_BINARY

```python
cc_binary(
    name = "my_binary",
    srcs = ["main.cpp", "util.cpp"],
    hdrs = ["util.h"],
    visibility = ["//visibility:public"],
    deps = ["//path/to/dependency"],
    copts = ["-Wall", "-O2"],
)
```

参数因为和cc\_library过分类似就不写了，就多赘述一句，除了编译，可以使用 Bazel 的 `bazel run` 命令，执行对应的Binary，这个执行一般是在sandbox里

```arduino
bazel run //path/to/package:my_binary
```

### CC\_TEST

```
cc_test(
    name = "my_test",
    srcs = ["test.cpp", "util.cpp"],
    hdrs = ["util.h"],
    visibility = ["//visibility:public"],
    deps = ["//path/to/dependency"],
    size = "small",
    copts = ["-Wall", "-O2"],
)
```

重复的东西不写了，只看几个关键点

*   `size`属性用于指定测试的时间和内存使用约束。它用来限制测试运行的时间和内存，防止测试用例运行时间过长或占用过多的系统内存资源，下面的表格给出来了一些约定俗成的资源消耗规格。一般来说，如果cc\_test是单元测试，那么size指定为small；如果是集成测试，size指定为medium，如果是接受性测试或者端到端测试，一般size指定为large或者enormous

    | Size     | RAM (in MB) | CPU (in CPU cores) | Default timeout      |
    | -------- | ----------- | ------------------ | -------------------- |
    | small    | 20          | 1                  | short (1 minute)     |
    | medium   | 100         | 1                  | moderate (5 minutes) |
    | large    | 300         | 1                  | long (15 minutes)    |
    | enormous | 800         | 1                  | eternal (60 minutes) |



一般调用的时候都是直接

```bash
bazel test //path/to/package:my_test
```



### 例子

这里只提供BUILD文件作为例子参考了，针对C++一般也是直接用Gtest，直接一套集成。。。

```python
load("//bazel:cpplint.bzl", "cpplint")
load("//bazel:rules_cc.bzl", "cc_library", "cc_test")

package(default_visibility = ["//visibility:public"])

cc_library(
    name = "file_debug",
    srcs = ["file_debug.cc"],
    hdrs = ["file_debug.h"],
    deps = [
        "@boost//:filesystem",
        "@com_github_google_glog//:glog",
        "@com_google_absl//absl/strings",
        "@com_google_absl//absl/strings:str_format",
        "@com_googlesource_code_re2//:re2",
    ],
)

filegroup(
    name = "file_pointer_file",
    # only two mock file pointer file, will never add new, so use glob
    srcs = glob([
        "*.txt",
    ]),
)

cc_test(
    name = "file_debug_test",
    srcs = [
        "file_debug_test.cc",
    ],
    data = [
        ":file_pointer_file",
    ],
    deps = [
        ":file_debug",
        "@com_google_googletest//:gtest",
        "@com_google_googletest//:gtest_main",
    ],
)

cpplint()

```



### 常见问题

写几个常见的问题，我目前看到很多研发都问过

1. `cc_test`可以管理什么样子的test？理论上只要是C++的测试程序，有返回值来判断是否执行成功，就都可以写为`cc_test`，因此无论写单元测试，集成测试，acceptance-check都是可以的。使用什么层级的测试，做哪些事情属于CI系统或者程序员自己的层面
2. 为什么`cc_test`超时了？参考cc\_test那部分的size属性，如果size不合适可能会报错TIMEOUT。
3. 为什么`cc_test`好像没测试，直接就输出了成功？我希望用户能理解，bazel本身是有缓存机制的，如果不希望使用已经缓存过的结果，建议在test的时候加上--nocache\_test\_results，来强行reruntest
4. 为什么`cc_test`没输出日志？在test运行成功的情况下（或者说期望情况下），可以认为逻辑正确执行，输入测试用例都得到了正确的输出，那么我们不需要关心具体的细节，bazel就不会输出任何问题。只有test执行失败，才会输出对应的日志
5. 为什么我写的`cc_test`找不到文件？检查写的BUILD文件里面的deps和data是不是都写全了，路径是不是写的相对路径，将依赖的数据路径加到BUILD里的时候，数据依赖应该是使用相对路径，而不是本地绝对路径。



\
