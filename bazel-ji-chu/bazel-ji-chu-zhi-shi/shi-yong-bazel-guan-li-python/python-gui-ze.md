# python规则

### py\_binary & py\_library & py\_test

我建议直接阅读，[https://bazel.build/reference/be/python#py\_binary](https://bazel.build/reference/be/python#py\_binary)，或者参考C++规则那一部分，过于基础，我这里打算只简单写一个py\_library了

`py_library`是Bazel中用于构建Python库的规则。用来将Python代码打包成一个库，供其他目标（如`py_binary`或其他`py_library`）使用。下面是关于`py_library`规则的一些重要信息：

```python
pythonCopy codepy_library(
    name = "my_library",
    srcs = ["module1.py", "module2.py"],
    deps = ["//path/to:another_library"],
)
```



### python工具链的配置

首先还是先引入对rules\_python的支持，在WORKSPACE里面写入下面的内容

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

rules_python_version = "740825b7f74930c62f44af95c9a4c1bd428d2c53" # Latest @ 2021-06-23

http_archive(
    name = "rules_python",
    # Bazel will print the proper value to add here during the first build.
    # sha256 = "FIXME",
    strip_prefix = "rules_python-{}".format(rules_python_version),
    url = "https://github.com/bazelbuild/rules_python/archive/{}.zip".format(rules_python_version),
)
```

接着，使用rules\_python的方式注册工具链，在WORKSPACE里面继续添加内容，如下。这里要注意python对象的执行，用的是这里注册的python环境和对应的python解释器，但是这些对象还是会使用默认的系统级别的python环境和解释器去“初始化”

```
load("@rules_python//python:repositories.bzl", "python_register_toolchains")

python_register_toolchains(
    name = "python3_9",
    # Available versions are listed in @rules_python//python:versions.bzl.
    # We recommend using the same version your team is already standardized on.
    python_version = "3.9",
)

load("@python3_9//:defs.bzl", "interpreter")

load("@rules_python//python:pip.bzl", "pip_parse")

pip_parse(
    ...
    python_interpreter_target = interpreter,
    ...
)
```



### 如何引入和使用外部依赖

上一节实际上已经写过了如何引入外部库，这次再重复写一次，不过初始化规则，就是WORKSPACE那部分不多赘述了。注意，这里的对第三方包的管理是一种集中式的管理，我个人比较推崇这种集中化的管理方式。

#### 集中化管理并引入外部包

在WORKSPACE里面写入如下的内容

<pre><code><strong>load("@rules_python//python:pip.bzl", "pip_parse")
</strong>
# Create a central repo that knows about the dependencies needed from
# requirements_lock.txt.
pip_parse(
   name = "my_deps",
   requirements_lock = "//path/to:requirements_lock.txt",
)
# Load the starlark macro which will define your dependencies.
load("@my_deps//:requirements.bzl", "install_deps")
# Call it to define repos for your requirements.
install_deps()
</code></pre>

requirements\_lock.txt内部只需要写入依赖的python包即可，这里我们写入jira作为对应第三方包就行了

```

# requirements.txt for Python3 packages
jira

```

#### 使用外部包

参考下面的代码，简单来说就是直接引入requirement并找到对应的依赖即可。外部包就可以直接使用了

```
load("@my_deps//:requirements.bzl", "requirement")

py_library(
    name = "mylib",
    srcs = ["mylib.py"],
    deps = [
        ":myotherlib",
        requirement("some_pip_dep"),
        requirement("another_pip_dep"),
    ]
)
```

