# 编译确定性镜像

编译确定性的镜像就比较简单了，这里我以cc\_image为例子，编译镜像并自定义命名Docker镜像

代码如下，这里面复用了上一节的base\_image的target.如下面的代码和流程

1. cc\_image的base指定为Docker Base Image，这里就是上一节导入的base\_image。binary指定为要编译出来的C++ Target
2. container\_bundle用来解决镜像命名的问题，这里的REAL\_IMAGE就是真正的命名对象。两个规则都加no-remote来避免cache

````
```python

load("@io_bazel_rules_docker//cc:image.bzl", "cc_image")
load("@io_bazel_rules_docker//container:container.bzl", "container_bundle", "container_import")

package(default_visibility = ["//visibility:public"])


cc_image(
    name = "image",
    base = ":base_image",
    binary = "//offboard/xxx:abc_target",
    tags = [
        "no-remote",
    ],
)

container_bundle(
    name = "taged_image",
    images = {
        "$(REAL_IMAGE)": ":image",
    },
    tags = [
        "no-remote",
    ],
)
```
````



那么如何将镜像编译出来呢？运行如下命令即可编译出来名字为hxndg:image的镜像

````
```shellscript
bazel run --define REAL_IMAGE=hxndg:image xxxx/xxx/xxx:taged_image -- --norun

```
````

