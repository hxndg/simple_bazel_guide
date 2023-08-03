# ToolChain的简单教程

### 为什么需要工具链 <a href="#motivation" id="motivation"></a>

工具链旨在解决构建平台，目标平台配置之类的一些工具和配置如何组合的问题。假设正在编写规则来支持“bar”编程语言。您的`bar_binary` 规则将`*.bar`使用`barc`编译器（编译器本身是作为工作区中的另一个目标构建的工具）来编译文件。

下面的代码将编译器作为隐式依赖项添加到了属性当中，优点是用户确实不需要关心编译器是怎么实现，在哪里生成，缺点是这个隐藏的依赖是开发者提供的，它必须由开发者保证是可用的。

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

`//bar_tools:barc_linux`现在是每个目标的依赖项`bar_binary`，因此它将在任何目标之前构建`bar_binary`。它可以像任何其他属性一样由规则的实现函数访问：

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

这里的问题是编译器的标签被硬编码到`bar_binary`，但不同的目标可能需要不同的编译器，具体取决于它们构建的平台和构建的平台 - 分别称为目标平台 _和_执行_平台_。“当文章开发出来的时候，文章就不属于作者了”，因此，规则作者不一定知道所有可用的工具和平台，因此在规则定义中对它们进行硬编码是不可行的。

一个不太理想的解决方案是通过将属性设置`_compiler`为非私有来将负担转移给用户。然后，可以对各个目标进行硬编码，以针对一个平台或另一个平台进行构建。

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

[可以通过使用基于平台的](https://bazel.build/docs/configurable-attributes)`select`选择来改进此解决方案：简单来说就是根据配置的选项来选择`compiler`属性是什么

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

这种方法的问题是过于僵硬，所以bazel提供了一个类似接口层的东西来实现这个功能，即`toolchain`框架通过添加额外的间接层来解决这个问题。简单来说就是将声明您的规则从对一堆写死的配置文件依赖，转换为对一个工具链类型的依赖，Bazel会根据适用的平台约束自动将其解析为特定的目标（一个工具链）。这样子，既不需要规则作者也不需要目标作者知道可用平台和工具链的完整集合，毕竟依赖转换为对一个抽象toolchain集合的依赖。

### 如何使用工具链编写规则 <a href="#writing-rules-toolchains" id="writing-rules-toolchains"></a>

在工具链框架下，规则不是直接依赖于工具，而是依赖于_工具链类型_。工具链类型是一个简单的目标，代表一类为不同平台提供相同作用的工具。例如，可以声明一个代表 bar 编译器的类型：

```
# By convention, toolchain_type targets are named "toolchain_type" and
# distinguished by their package path. So the full path for this would be
# //bar_tools:toolchain_type.
toolchain_type(name = "toolchain_type")
```

上一节中的规则定义经过修改，不再将编译器作为属性，而是声明它使用工具链 `//bar_tools:toolchain_type`。

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

现在，实现函数使用工具链类型作为键来访问此依赖项`ctx.toolchains` ，而不是泛大街的`ctx.attr`。

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

`ctx.toolchains["//bar_tools:toolchain_type"]`返回 Bazel 工具链依赖关系提供出来的 [`ToolchainInfo` provider](https://bazel.build/rules/lib/toplevel/platform\_common#ToolchainInfo)[。](https://bazel.build/rules/lib/toplevel/platform\_common#ToolchainInfo)具体 `ToolchainInfo`的内容由底层工具的规则设置；在下一节中，定义了这个实现的规则，会提供一个包装`BarcInfo`对象的`barcinfo`。

#### 强制和可选工具链 <a href="#optional-toolchains" id="optional-toolchains"></a>

默认情况下，当规则使用裸标签表达工具链类型依赖性时（裸标签就是//bar\_tools:toolchain\_type这个东西，就是只有它一个），工具链类型被认为是必须**强制提供的**。如果 Bazel 无法找到匹配的工具链类型，就会报错[。](https://bazel.build/extending/toolchains#toolchain-resolution)

可以改为声明**可选的**工具链类型依赖项，如下所示：

```
bar_binary = rule(
    ...
    toolchains = [
        config_common.toolchain_type("//bar_tools:toolchain_type", mandatory = False),
    ],
)
```

当无法解析可选工具链类型时，分析将继续，结果`ctx.toolchains["//bar_tools:toolchain_type"]`为`None`。

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



#### 使用工具链编写方面 <a href="#writing-aspects-toolchains" id="writing-aspects-toolchains"></a>

方面可以访问与规则相同的工具链 API：您可以定义所需的工具链类型，通过上下文访问工具链，并使用它们通过工具链生成新操作。

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

要定义一些工具链，需要做三件事：

1. 代表工具或工具套件类型的特定于语言的规则。按照惯例，以“\_toolchain”为后缀，以语言类型作为前缀。
2. 此规则类型适用的多个目标，代表该规则所对应不同平台(的工具组合)的原型，在这里用来定义toolchain的所需要的信息（属性）。
3. 针对第二个目标，不同平台的实际组合。

以下面规则的定义`bar_toolchain`为例。注意，这里的示例只有一个编译器，但其他工具（例如链接器）也可以分组在下面。

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

该规则必须返回一个`ToolchainInfo`提供者，供使用该规则实现对象`ctx.toolchains的`具体代码使用。`ToolchainInfo`与 `struct`一样，可以存储任意键值对。具体`ToolchainInfo` 应该包含哪些字段都需要显式写到代码里。之后就是第三步，定义特定的目标`barc`。

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

最后一步，将`toolchain`为和平台绑定到一起。

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

上面相对路径语法的使用表明这些定义都位于同一个包中，但实际上工具链类型、特定于语言的工具链目标和`toolchain`定义目标可以位于单独的包中。

可以参考资料[`go_toolchain`](https://github.com/bazelbuild/rules\_go/blob/master/go/private/go\_toolchain.bzl) 看看真实的例子。

#### 工具链和配置 <a href="#toolchains_and_configurations" id="toolchains_and_configurations"></a>

对于规则作者来说，一个重要的问题是，在分析`bar_toolchain`目标时，它看到的是什么配置，并且对于依赖项应该使用什么转换来确定如何执行构建行为？这个问题因为涉及到了bazel的platform，所以变得复杂起来，因为bazel需要支持跨平台编译，因此需要能够确定使用哪个平台的构建工具来执行。

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

根据目标平台（就是真正执行的平台，这里被称为“父级平台”）转换到具体的工具链的依赖关系，目前采用一种被称为“工具链转换”的特殊配置转换。这种工具链转换和其它的配置（我理解是说配置什么config属性之类）基本一致，只是它要求必须明确是参与构建阶段还是参与到执行阶段，简单来说就是是否参与到最终生成。

这使得`exec`工具链的任何依赖项必须也可以针对父级的构建操作执行，即任何使用`cfg = "target"`（如果未指定`cfg`是什么也会是target`，`“target”是默认值）的工具链依赖项都是在与父级相同的目标平台上构建的。`system_lib`上面的属性和工具（ `compiler`属性）都需要应用或者链接到最终的工件中，因此需要为同一平台构建，必须强制执行属性是一致的。

我个人觉得这段是因为原本bazel的目标平台就是构建平台，所以不得不强制这个要求

这一段我认为我理解的可能有偏差，我建议看[https://github.com/bazelbuild/bazel/issues/11584](https://github.com/bazelbuild/bazel/issues/11584) & [https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md](https://github.com/bazelbuild/proposals/blob/main/designs/2019-02-12-toolchain-transitions.md)

或者看看下面这个英文的原版分析

1. "The dependency from a target (called the 'parent') to a toolchain via toolchain resolution uses a special configuration transition called the 'toolchain transition'."
   * This means that when a target depends on a toolchain using toolchain resolution, there is a specific way the configuration is handled during this resolution, which involves a transition known as the "toolchain transition."
2. "The toolchain transition keeps the configuration the same, except that it forces the execution platform to be the same for the toolchain as for the parent."
   * The toolchain transition ensures that the configuration remains mostly unchanged, but it specifically sets the execution platform of the toolchain to be the same as the parent target's execution platform.
3. "This allows any exec dependencies of the toolchain to also be executable for the parent's build actions."
   * By setting the toolchain's execution platform to be the same as the parent target's execution platform, any "exec" dependencies of the toolchain can also be executed during the build actions of the parent target.
4. "Any of the toolchain's dependencies which use cfg = 'target' (or which don't specify cfg, since 'target' is the default) are built for the same target platform as the parent."
   * The toolchain can have its own dependencies, and any of these dependencies that use cfg = 'target' (or have no specified cfg, using the default which is 'target') are built for the same target platform as the parent target.
5. "This allows toolchain rules to contribute both libraries (the system\_lib attribute above) and tools (the compiler attribute) to the build rules which need them."
   * The toolchain rules can provide both libraries (system\_lib attribute) and tools (compiler attribute) to the build rules that require them, making it possible for the build system to utilize these contributions in the build process.
6. "The system libraries are linked into the final artifact, and so need to be built for the same platform, whereas the compiler is a tool invoked during the build, and needs to be able to run on the execution platform."
   * The system libraries provided by the toolchain are linked into the final output artifact of the build, so they need to be built for the same platform. On the other hand, the compiler is a tool that is used during the build process and must be able to run on the execution platform where the build is taking place.

### 使用工具链注册和构建 <a href="#registering-building-toolchains" id="registering-building-toolchains"></a>

此时，所有构建块都已组装完毕，只需使工具链可用于 Bazel 的解析过程即可。这是通过使用 注册工具链来完成的，或者在`WORKSPACE`文件中使用 `register_toolchains()`，或者使用 标志在命令行上传递工具链的标签`--extra_toolchains`。

```
register_toolchains(
    "//bar_tools:barc_linux_toolchain",
    "//bar_tools:barc_windows_toolchain",
    # Target patterns are also permitted, so you could have also written:
    # "//bar_tools:all",
)
```

现在，当您构建依赖于工具链类型的目标时，将根据目标和执行平台选择适当的工具链。

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

Bazel 会发现`//my_pkg:my_bar_binary`使用`@platforms//os:linux`构建，因此解析了 `//bar_tools:toolchain_type`对`//bar_tools:barc_linux_toolchain`的依赖，最终只会构建`//bar_tools:barc_linux`平台的产物而非 `//bar_tools:barc_windows`平台产物。

### 工具链解析 <a href="#toolchain-resolution" id="toolchain-resolution"></a>

**注意：**[某些 Bazel 规则](https://bazel.build/concepts/platforms#status)尚不支持工具链解析。

对于每个使用工具链的目标，Bazel 的工具链解析过程确定目标的具体工具链依赖关系。该过程将一组所需的工具链类型、目标平台、可用执行平台列表以及可用工具链列表作为输入。其输出为每种工具链类型选择的工具链以及为当前目标选择的执行平台。

可用的执行平台和工具链是通过 register\_execution\_platforms 和 register\_toolchains 在 WORKSPACE 文件中收集的。还可以通过命令行选项 --extra\_execution\_platforms 和 --extra\_toolchains 指定额外的执行平台和工具链。主机平台会自动包括为可用的执行平台。可用的平台和工具链被跟踪为有序列表，以保证确定性，并优先考虑列表中较早的项目。

解析步骤如下。

1. 如果 target\_compatible\_with 或 exec\_compatible\_with 子句匹配了一个平台，则对于其列表中的每个约束值，该平台也必须具有该约束值（可以是显式指定，也可以是默认值）。
2. 如果平台具有约束设置中未被子句引用的约束值，则这些值不会影响匹配
3. 如果正在构建的目标指定了 [`exec_compatible_with`属性](https://bazel.build/reference/be/common-definitions#common.exec\_compatible\_with) （或其规则定义指定了 [`exec_compatible_with`参数](https://bazel.build/rules/lib/globals/bzl#rule.exec\_compatible\_with)），则可用执行平台的列表将被过滤以删除任何与执行约束不匹配的平台。
4. 对于每个可用的执行平台，将每个工具链类型与第一个可用的工具链关联起来。
