# 导入镜像container\_import

为什么第一个写container\_import? 在集成Bazel管理Docker机制到CI体系中的时候，发现Bazel直接用内置的Docker下载机制将镜像下载过程当中时，会非常慢（原因就不多分析拉），因此采用了Docker镜像直接导入，之后将之Import到Bazel内部的方法。

步骤很简单，

1. 在Node机器上下载外部Base镜像，一般情况下，CI机器内部都已经缓存了镜像
2. 使用docker save命令，将Base镜像存储到本地，打开压缩包里面的config.json文件
3. 参考下面的代码，首先将config.json文件配置为filegroup
4. 使用container\_import命令，标记image\_digest和config.json文件导入即可，其中tag加上no-cache和no-remote用来标记不需要缓存到远端，避免占用过多空间和内存。base镜像直接缓存到本地

如下所示，Bazel就可以读取到导入的镜像了



````
```python

load("@io_bazel_rules_docker//cc:image.bzl", "cc_image")
load("@io_bazel_rules_docker//container:container.bzl", "container_bundle", "container_import")

package(default_visibility = ["//visibility:public"])

filegroup(
    name = "base_image_config_json",
    srcs = [
        "base_image_config.json",
    ],
    visibility = ["//visibility:public"],
)

container_import(
    name = "base_image",
    base_image_digest = "sha256:xxxxxxxxxxxxxxxxxxxxxx2e457cd116a429257d8f3615e51a55c56b3705539568d58c",
    base_image_registry = "registry.xxxxxx.com",
    base_image_repository = "global/xxxxxx",
    config = ":base_image_config_json",
    layers = [],
    tags = [
        "no-cache",
        "no-remote",
    ],
)
```
````



接下来，只需要运行docker pull，这个镜像只要在本地就可以直接load进bazel了

