# ToolChain的简单教程

### 引言 <a href="#motivation" id="motivation"></a>

下面的内容了里面执行平台和构建平台是一个意思，我可能串来串去地用

### 为什么需要工具链 <a href="#motivation" id="motivation"></a>

工具链旨在解决构建平台 & 目标平台配置之类的一些细节问题，比方说工具和标准库如何组合的问题。假设现在正在编写规则来支持“bar”编程语言。`bar_binary` 规则将产生对应的文件，这个过程需要使用使用`barc`编译器（编译器本身是作为工作区中的另一个目标构建的工具）来编译文件。

下面的代码将编译器作为隐式依赖项添加到了属性当中，优点是用户确实不用关心编译器是怎么调用的，在哪里生成。缺点是这个隐藏的依赖是开发者提供的，它必须由开发者保证是可用的，某个用户在linux上跑这个规则没啥问题，然后换了windows崩了，开发者说：“我妹想到会在windows上跑呀”。。。

```
bar_binary = rule(
    implementation = _bar_binary_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = True),
        ...
        "_compiler": attr.label(
            default = "//bar_tools:barc_linux",  # the compiler running on linux
            providers = [BarcInfo],
        ),
    },
)
```

造成这个问题的原因就是，`//bar_tools:barc_linux`现在是每个`bar_binary`目标的依赖项了，每次构建`bar_binary`，它都必须是和任何其他属性一样，是可用的。看下面的代码，每次调用bar\_binary的时候，都需要显示地找到 `ctx.attr._compiler[BarcInfo]`的`compiler_path`，很明显，不应该将编译器的label被硬编码到`bar_binary`，不同的目标可能需要不同的编译器，具体取决于它们构建的平台和构建的平台 - 分别称为目标平台和执行平台。

可是“当文章开发出来的时候，文章就不属于作者了”，因此，规则作者不一定知道所有可用的工具和平台，总之，在规则定义里就做硬编码是不可行的。PS下面的内容里面的Provider不知道是什么的话，看一下编写自定义规则那部分：

```
BarcInfo = provider(
    doc = "Information about how to invoke the barc compiler.",
    # In the real world, compiler_path and system_lib might hold File objects,
    # but for simplicity they are strings for this example. arch_flags is a list
    # of strings.
    fields = ["compiler_path", "system_lib", "arch_flags"],
)

def _bar_binary_impl(ctx):
    ...
    info = ctx.attr._compiler[BarcInfo]
    command = "%s -l %s %s" % (
        info.compiler_path,
        info.system_lib,
        " ".join(info.arch_flags),
    )
    ...
```



一个不太合适的解决方案，将编译器属性暴露给用户，但这样子意味着内部的东西暴露给了外部。用这种方法，可以对各个目标进行硬编码，来只针对一个平台进行构建。用户表示，我写一个目标，针对三个平台，代码量\*3。

```
bar_binary(
    name = "myprog_on_linux",
    srcs = ["mysrc.bar"],
    compiler = "//bar_tools:barc_linux",
)

bar_binary(
    name = "myprog_on_windows",
    srcs = ["mysrc.bar"],
    compiler = "//bar_tools:barc_windows",
)
```

固然，可以通过使用基于[平台的`select`](https://bazel.build/docs/configurable-attributes)选择特定的配置来少写代码：简单来说就是根据配置的选项来选择`compiler`属性是什么

```
config_setting(
    name = "on_linux",
    constraint_values = [
        "@platforms//os:linux",
    ],
)

config_setting(
    name = "on_windows",
    constraint_values = [
        "@platforms//os:windows",
    ],
)

bar_binary(
    name = "myprog",
    srcs = ["mysrc.bar"],
    compiler = select({
        ":on_linux": "//bar_tools:barc_linux",
        ":on_windows": "//bar_tools:barc_windows",
    }),
)
```

这种方法的问题是过于僵硬，一个研发用户还得写一堆配置文件，普通的研发又得升级为infra研发了。

所以bazel提供了一个类似接口层的东西来实现这个功能，即通过添加额外的间接层（`toolchain`框架）来解决这个问题。简单来说就是将规则从对一堆写死的配置文件依赖，转换为对一个工具链类型的依赖，Bazel会根据适用的平台约束自动将其解析为特定的目标（一个工具链）。这样子，既不需要规则作者也不需要目标作者知道可用平台和工具链的完整集合，毕竟依赖具体的对象转换为依赖一个抽象toolchain集合

### 如何使用工具链编写规则 <a href="#writing-rules-toolchains" id="writing-rules-toolchains"></a>

在工具链框架下，规则不直接依赖于具体的工具，而是依赖于工具链类型。工具链类型是一个简单的目标，代表一类为不同平台提供相同作用的工具的组合。例如，可以声明一个代表 bar 编译器的类型：

```
# By convention, toolchain_type targets are named "toolchain_type" and
# distinguished by their package path. So the full path for this would be
# //bar_tools:toolchain_type.
toolchain_type(name = "toolchain_type")
```

上一节中的规则定义经过修改，不再将编译器作为属性，而是使用`//bar_tools:toolchain_type` 。

```
bar_binary = rule(
    implementation = _bar_binary_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = True),
        ...
        # No `_compiler` attribute anymore.
    },
    toolchains = ["//bar_tools:toolchain_type"],
)
```

现在，具体的impl实现函数使用工具链类型作为键来访问对应的依赖项。从用`ctx.attr`变成用`ctx.toolchains`&#x20;

```
def _bar_binary_impl(ctx):
    ...
    info = ctx.toolchains["//bar_tools:toolchain_type"].barcinfo
    # The rest is unchanged.
    command = "%s -l %s %s" % (
        info.compiler_path,
        info.system_lib,
        " ".join(info.arch_flags),
    )
    ...
```

`ctx.toolchains["//bar_tools:toolchain_type"]`返回 Bazel 工具链返回的 [`ToolchainInfo` provider](https://bazel.build/rules/lib/toplevel/platform\_common#ToolchainInfo)[。](https://bazel.build/rules/lib/toplevel/platform\_common#ToolchainInfo)具体 `ToolchainInfo`的内容由底层工具的规则设置；在下一节中，定义了这个实现的规则，会提供一个包装`BarcInfo`对象的`barcinfo`。

#### 强制和可选工具链 <a href="#optional-toolchains" id="optional-toolchains"></a>

默认情况下，当rule给出toolchains的依赖的时候，工具链类型被认为是必须**提供的**（这里的工具链类型就是//bar\_tools:toolchain\_type这个东西，就是只有它一个）。如果 Bazel 无法找到匹配的工具链类型，就会报错[。](https://bazel.build/extending/toolchains#toolchain-resolution)

但是可以有意识的生命为依赖**可选的**工具链类型依赖项，如下所示：

```
bar_binary = rule(
    ...
    toolchains = [
        config_common.toolchain_type("//bar_tools:toolchain_type", mandatory = False),
    ],
)
```

当无法找到可选工具链类型时，解析会把`ctx.toolchains["//bar_tools:toolchain_type"]`当为`None`。

这个[`config_common.toolchain_type`](https://bazel.build/rules/lib/toplevel/config\_common#toolchain\_type) 默认为强制类型，就是必须有一个对应的工具链类型。

以下几种写法都是合法的，只不过意义不同：

* 强制工具链类型：
  * `toolchains = ["//bar_tools:toolchain_type"]`
  * `toolchains = [config_common.toolchain_type("//bar_tools:toolchain_type")]`
  * `toolchains = [config_common.toolchain_type("//bar_tools:toolchain_type", mandatory = True)]`
* 可选工具链类型：
  * `toolchains = [config_common.toolchain_type("//bar_tools:toolchain_type", mandatory = False)]`

```
bar_binary = rule(
    ...
    toolchains = [
        "//foo_tools:toolchain_type",
        config_common.toolchain_type("//bar_tools:toolchain_type", mandatory = False),
    ],
)
```



#### 使用工具链编写aspects  <a href="#writing-aspects-toolchains" id="writing-aspects-toolchains"></a>

aspects 可以访问与规则相同的工具链 API：您可以定义所需的工具链类型，通过上下文访问工具链，并使用它们通过工具链生成新操作。aspects 是什么可以参考

```
bar_aspect = aspect(
    implementation = _bar_aspect_impl,
    attrs = {},
    toolchains = ['//bar_tools:toolchain_type'],
)

def _bar_aspect_impl(target, ctx):
  toolchain = ctx.toolchains['//bar_tools:toolchain_type']
  # Use the toolchain provider like in a rule.
  return []
```

### 定义工具链 <a href="#toolchain-definitions" id="toolchain-definitions"></a>

要定义一些工具链，需要确定三方面信息：

1. 适用于特定语言的工具链命名，按照惯例，以“\_toolchain”为后缀，以语言类型作为前缀。
2. 使用该toolchain的诸多规则的所用到的工具是哪些？这些工具，代表具体rule所对应不同平台用到的的工具组合的原型，都是定义toolchain的所需要的信息（属性）。
3. 针对第二个方面，如何将这些原信息组合起来。

以下面规则的定义`bar_toolchain`为例。注意，这里的示例只写了编译器，但其他工具（例如链接器）也可以一并写在下面。

```
def _bar_toolchain_impl(ctx):
    toolchain_info = platform_common.ToolchainInfo(
        barcinfo = BarcInfo(
            compiler_path = ctx.attr.compiler_path,
            system_lib = ctx.attr.system_lib,
            arch_flags = ctx.attr.arch_flags,
        ),
    )
    return [toolchain_info]

bar_toolchain = rule(
    implementation = _bar_toolchain_impl,
    attrs = {
        "compiler_path": attr.string(),
        "system_lib": attr.string(),
        "arch_flags": attr.string_list(),
    },
)
```

该规则必须返回一个`ToolchainInfo`的Procider，供其他规则（依赖该toolchain）使用。`ToolchainInfo`与 `struct`一样，可以存储任意键值对。具体`ToolchainInfo` 应该包含哪些字段都需要显式写到代码里。

接下来就可以定义具体的工具链目标了，下面定义了两种工具链，分别针对linux和windows

```
bar_toolchain(
    name = "barc_linux",
    arch_flags = [
        "--arch=Linux",
        "--debug_everything",
    ],
    compiler_path = "/path/to/barc/on/linux",
    system_lib = "/usr/lib/libbarc.so",
)

bar_toolchain(
    name = "barc_windows",
    arch_flags = [
        "--arch=Windows",
        # Different flags, no debug support on windows.
    ],
    compiler_path = "C:\\path\\on\\windows\\barc.exe",
    system_lib = "C:\\path\\on\\windows\\barclib.dll",
)
```

最后一步，将`toolchain`为和平台绑定到一起，注意这里的exec\_compatible\_with  \&target\_compatible\_with 。

```
toolchain(
    name = "barc_linux_toolchain",
    exec_compatible_with = [
        "@platforms//os:linux",
        "@platforms//cpu:x86_64",
    ],
    target_compatible_with = [
        "@platforms//os:linux",
        "@platforms//cpu:x86_64",
    ],
    toolchain = ":barc_linux",
    toolchain_type = ":toolchain_type",
)

toolchain(
    name = "barc_windows_toolchain",
    exec_compatible_with = [
        "@platforms//os:windows",
        "@platforms//cpu:x86_64",
    ],
    target_compatible_with = [
        "@platforms//os:windows",
        "@platforms//cpu:x86_64",
    ],
    toolchain = ":barc_windows",
    toolchain_type = ":toolchain_type",
)
```

上面用法使用相对路径语法，这似乎表明这些定义都需要位于同一个包中，但实际上并不是这个样子的，工具链类型、特定于语言的工具链目标和`toolchain`定义目标可以位于单独的包中。（下面还有一部注册工具链。）

可以参考资料[`go_toolchain`](https://github.com/bazelbuild/rules\_go/blob/master/go/private/go\_toolchain.bzl) 看看真实的例子。

#### 工具链和配置 <a href="#toolchains_and_configurations" id="toolchains_and_configurations"></a>

如果对下面的内容有点疑问，建议看下[https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-execution-transitions.md](https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-execution-transitions.md)

[https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md](https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md)

对于规则作者来说，一个重要的问题是，在分析`bar_toolchain`目标时，它看到的是什么配置，并且对于依赖项应该使用什么转换来确定如何执行构建行为？这个问题因为涉及到了bazel的platform，所以变得复杂起来，因为bazel需要支持跨平台编译，因此需要能够确定使用构建平台的构建工具如何执行。

最早期的时候，bazel只支持在宿主机上编译的时候，它只支持一种[**host configuration**](https://docs.bazel.build/versions/master/guide.html#configurations),这个工具链说白了在analysis阶段就确定了：我就是要用宿主机的工具链。但是随着remote execution的生效，这套机制无效了，需要支持在其它的平台（即不等同于target平台）上做构建，但是做构建有的工具是目标平台的（比方说链接的库），有的工具是构建平台（执行平台，比方说编译器）

因此，在Bazel做analysis的阶段，它就需要能够确定构建工具链的一些属性。比方说我想编译一个arm64的target，target看到的工具链需要能够access两个平台，一个是目标平台即（the target platform of the original target that requires a toolchain），另一个是构建平台（the execution platform of the original target）。这两个区分开，我就能够指定构建平台的工具（比方说编译器），同时指定一些目标平台的工具（比方说我想连接arm64平台的libstdc++代码库）。但另外一个问题就引入了，可以随便指定构建平台吗？

让我们看一个更复杂的版本`bar_toolchain`：

```
def _bar_toolchain_impl(ctx):
    # The implementation is mostly the same as above, so skipping.
    pass

bar_toolchain = rule(
    implementation = _bar_toolchain_impl,
    attrs = {
        "compiler": attr.label(
            executable = True,
            mandatory = True,
            cfg = "exec",
        ),
        "system_lib": attr.label(
            mandatory = True,
            cfg = "target",
        ),
        "arch_flags": attr.string_list(),
    },
)
```

可以看到使用[`attr.label`](https://bazel.build/rules/lib/toplevel/attr#label)与标准规则没什么不同，但`cfg`参数的含义略有不同。

根据目标平台（就是真正执行的平台，这里被称为“父级平台”）转换到具体的工具链的依赖关系，目前采用一种被称为“工具链转换”的特殊配置转换。这种工具链转换和其它的配置（我理解是说配置什么config属性之类）基本一致，只是它要求target看到的构建平台（执行平台也行，whatever名字吧）和工具链看到的构建平台必须是一致的。

这使得一个target的`exec`工具链的必须是通用的，即任何使用`cfg = "target"`（如果未指定`cfg`是什么也会是target`，`“target”是默认值）的工具链依赖项都是可以在对应的构建平台执行。

贴一下原文：The dependency from a target (called the "parent") to a toolchain via toolchain resolution uses a special configuration transition called the "toolchain transition". The toolchain transition keeps the configuration the same, except that it forces the execution platform to be the same for the toolchain as for the parent (otherwise, toolchain resolution for the toolchain could pick any execution platform, and wouldn't necessarily be the same as for parent). This allows any `exec` dependencies of the toolchain to also be executable for the parent's build actions. Any of the toolchain's dependencies which use `cfg = "target"` (or which don't specify `cfg`, since "target" is the default) are built for the same target platform as the parent. This allows toolchain rules to contribute both libraries (the `system_lib` attribute above) and tools (the `compiler` attribute) to the build rules which need them. The system libraries are linked into the final artifact, and so need to be built for the same platform, whereas the compiler is a tool invoked during the build, and needs to be able to run on the execution platform.

这一段我认为我理解的可能有偏差，我建议看[https://github.com/bazelbuild/bazel/issues/11584](https://github.com/bazelbuild/bazel/issues/11584) & [https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md](https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md)



### 使用工具链注册和构建 <a href="#registering-building-toolchains" id="registering-building-toolchains"></a>

此时，所有基础代码块都组装完毕，只需将工具链暴露出来，另其可用于 Bazel 的解析过程即可。此时，就需要注册工具链了，有两种方法

* 要么直接在`WORKSPACE`文件中使用 `register_toolchains()`，
* 要么在构建时命令行上传递选择工具链的标签`--extra_toolchains`。

```
register_toolchains(
    "//bar_tools:barc_linux_toolchain",
    "//bar_tools:barc_windows_toolchain",
    # Target patterns are also permitted, so you could have also written:
    # "//bar_tools:all",
)
```

现在，就可以根据目标和执行平台选择适当的工具链了。

```
# my_pkg/BUILD

platform(
    name = "my_target_platform",
    constraint_values = [
        "@platforms//os:linux",
    ],
)

bar_binary(
    name = "my_bar_binary",
    ...
)
```

```
bazel build //my_pkg:my_bar_binary --platforms=//my_pkg:my_target_platform
```

Bazel 会发现`//my_pkg:my_bar_binary`使用`@platforms//os:linux`构建，因此解析出 `//bar_tools:toolchain_type`对`//bar_tools:barc_linux_toolchain`的依赖，最终只会构建`//bar_tools:barc_linux`平台的产物而非 `//bar_tools:barc_windows`平台产物。

### 工具链解析的细节 <a href="#toolchain-resolution" id="toolchain-resolution"></a>

**注意：**[某些 Bazel 规则](https://bazel.build/concepts/platforms#status)尚不支持工具链解析。

对于每个使用工具链的目标，Bazel 的工具链解析过程确定目标的具体工具链依赖关系。该过程将一组所需的工具链类型、目标平台、可用构建平台列表以及可用工具链列表作为输入。其输出为选定的工具链以及选定的构建平台。

可用的执行平台和工具链是通过 register\_execution\_platforms 和 register\_toolchains 在 WORKSPACE 文件中注册的。还可以通过命令行选项 --extra\_execution\_platforms 和 --extra\_toolchains 指定额外的执行平台和工具链。主机平台会自动作为可用的执行平台添加到备选执行平台列表中。在执行的时候，可用的平台靠前的项目，会优先被选用。

解析步骤如下。

1. 如果 target\_compatible\_with 或 exec\_compatible\_with 子句匹配了一个平台，则对于其列表中的每个约束值，对应的构建平台也必须match对应的约束值（可以是显式指定，也可以是默认值）。
2. 如果正在构建的目标指定了 [`exec_compatible_with`属性](https://bazel.build/reference/be/common-definitions#common.exec\_compatible\_with) （或其规则定义指定了 [`exec_compatible_with`参数](https://bazel.build/rules/lib/globals/bzl#rule.exec\_compatible\_with)），则可用执行平台的列表将被过滤以删除任何与contrant\_setting约束不匹配的平台。
3. 对于每个可用的执行平台，将每个工具链类型与第一个可用的工具链类型关联起来。
