# 传统方法

传统方法讲究化劲，接，化，发！。。。不好意思，走错片场了。。。。

### 实现方法

理解传统方式需要首先明白以下几个概念

* 代码库：包含`WORKSPACE`带有或文件的目录`WORKSPACE.bazel`，包含 Bazel 构建中使用的源文件的一些列文件的组织，称为代码库。代码库不是只针对下面的主仓库，第三方库也是代码库。
* 主仓库： 这个不用多说了，真正的执行Bazel 命令的代码库。
* WorkSpace:工作区间，也是所有 Bazel 命令共享的环境。
* 代码库规则：（第三方）代码库库定义的模式，告诉 Bazel 如何获取代码库。例如，它可以是“从某个 URL 下载 zip 存档并解压它”，或者“获取某个 Maven 工件并使其可用作目标 `java_import`”，或者只是“符号链接本地目录”。每个存储库都是 通过使用适当数量的参数调用存储库规则来**定义的。**到目前为止，最常见的存储库规则是 [`http_archive`](https://bazel.build/rules/lib/repo/http#http\_archive)，它从 URL 下载存档并提取它，以及 [`local_repository`](https://bazel.build/reference/be/workspace#local\_repository)，它符号链接已经是 Bazel 存储库的本地目录。

简单来讲，传统方式可以认为就是http\_archive，官方给的例子如下，引入foo作为外部依赖。

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
http_archive(
    name = "foo",
    urls = ["https://example.com/foo.zip"],
    sha256 = "c9526390a7cd420fdcec2988b4f3626fe9c5b51e2959f685e8f4d170d1a9bd96",
)
```

举个简单例子，在WORKSPACE文件内写入下面的内容，之后在对应依赖grpc的BUILD文件里面写入相应的依赖项即可使用该grpc外部依赖，比方说glog的引入也是同样的道理。其中strip\_prefix用来去除解压的文件夹前缀。

<pre><code>
<strong>version = "1.xx.x"
</strong>http_archive(
    name = "com_github_grpc_grpc",
    strip_prefix = "grpc-{}".format(version),
    sha256 = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    urls = [
        "https://xxxxxxxxxxxxxxxxx/grpc-{}.tar.gz".format(version),
        "https://xxxxxxxxxxxxxxxxxxx{}.tar.gz".format(version),
    ],
    patch_args = ["-p1"],
    patches = [
        clean_dep("//third_party/grpc:p01xxxxxxxxxxxxx.patch"),
        clean_dep("//third_party/grpc:p02xxxxxxxxxxxxx.patch"),
        clean_dep("//third_party/grpc:p03xxxxxxxxxxxxx.patch"),
    ],
)

</code></pre>

```

cc_binary(
    name = "grpc_test_main",
    testonly = True,
    srcs = [
        "grpc_test_main.cc",
    ],
    deps = [
        "//xxxxx/proto:grpc_test_main",
        "@com_github_google_glog//:glog",
        "@com_github_grpc_grpc//:grpc++",
    ],
)

```



### 缺陷

传统方式有几种不足

* Bazel的WORKSPACE级别的外部依赖只能是一个依赖链，不存在灵活的选择。
* 为了绕开这一点，bazel要求用户自己实现宏来选择外部依赖，宏不能灵活load .bzl文件，因此需要用户多次宣式指定依赖，而显示依赖实际上是缺乏版本信息的，简单来说，http\_archive是没有（显式）版本信息的。
