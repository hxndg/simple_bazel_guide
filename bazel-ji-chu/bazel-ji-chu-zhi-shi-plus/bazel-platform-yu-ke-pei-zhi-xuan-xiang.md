# Bazel Platform与可配置选项

### 引言

首先还是推荐阅读[https://bazel.build/configure/attributes](https://bazel.build/configure/attributes)



### 为什么我要阅读这一页

bazel的可配置选项和platform，再集合上工具链等概念，是实现跨平台编译等复杂功能的方法。如果想了解这么实现这些高级功能就需要从这里入手。



### 可配置选项



#### select的功能

_**可配置属性**_（通常称为[`select()`](https://bazel.build/reference/be/functions#select)）是 Bazel 的一项功能，允许用户在命令行切换构建规则属性的值。

```
# myapp/BUILD

cc_binary(
    name = "mybinary",
    srcs = ["main.cc"],
    deps = select({
        ":arm_build": [":arm_lib"],
        ":x86_debug_build": [":x86_dev_lib"],
        "//conditions:default": [":generic_lib"],
    }),
)

config_setting(
    name = "arm_build",
    values = {"cpu": "arm"},
)

config_setting(
    name = "x86_debug_build",
    values = {
        "cpu": "x86",
        "compilation_mode": "dbg",
    },
)
```

接下来可以看下，对应的配置选项会怎么触发怎样编译依赖的选择。

| 命令                                              | 依赖                 |
| ----------------------------------------------- | ------------------ |
| `bazel build //myapp:mybinary --cpu=arm`        | `[":arm_lib"]`     |
| `bazel build //myapp:mybinary -c dbg --cpu=x86` | `[":x86_dev_lib"]` |
| `bazel build //myapp:mybinary --cpu=ppc`        | `[":generic_lib"]` |
| `bazel build //myapp:mybinary -c dbg --cpu=ppc` | `[":generic_lib"]` |

`select()`_用用来根据情况选择不同的配置，简单来说就是选择_ [`config_setting`](https://bazel.build/reference/be/general#config\_setting) 中的细节。注意select是具有传递性的，如果top level启用了某种配置，这个配置会传递到下层的依赖当中。

`select`可以组合，但是不能递归

```
sh_binary(
    name = "my_target",
    srcs = ["always_include.sh"] +
           select({
               ":armeabi_mode": ["armeabi_src.sh"],
               ":x86_mode": ["x86_src.sh"],
           }) +
           select({
               ":opt_mode": ["opt_extras.sh"],
               ":dbg_mode": ["dbg_extras.sh"],
           }),
)
```

#### select的细节语法

select支持多种写法

*   selects.with\_or语法，一种多种情况使用相同配置的简单写法

    ```
    sh_binary(
        name = "my_target",
        srcs = ["always_include.sh"],
        deps = selects.with_or({
            (":config1", ":config2", ":config3"): [":standard_lib"],
            ":config4": [":special_lib"],
        }),
    )
    ```
*   selects.config\_setting\_group，同样是多种匹配任何一种的情况

    ```
    config_setting(
        name = "config1",
        values = {"cpu": "arm"},
    )
    config_setting(
        name = "config2",
        values = {"compilation_mode": "dbg"},
    )
    selects.config_setting_group(
        name = "config1_or_2",
        match_any = [":config1", ":config2"],
    )
    sh_binary(
        name = "my_target",
        srcs = ["always_include.sh"],
        deps = select({
            ":config1_or_2": [":standard_lib"],
            "//conditions:default": [":other_lib"],
        }),
    )
    ```
*   AND chaining，用来同时匹配多种的情况

    ```
    config_setting(
        name = "config1",
        values = {"cpu": "arm"},
    )
    config_setting(
        name = "config2",
        values = {"compilation_mode": "dbg"},
    )
    selects.config_setting_group(
        name = "config1_and_2",
        match_all = [":config1", ":config2"],
    )
    sh_binary(
        name = "my_target",
        srcs = ["always_include.sh"],
        deps = select({
            ":config1_and_2": [":standard_lib"],
            "//conditions:default": [":other_lib"],
        }),
    )
    ```

###

### Platforms

platform是用来方便地组合和索引多配置的方法，简单来说可以认为多种属性构成了平台（反过来，多平台构成了属性）。简单来说可以认为platform是下面的组合

`config_setting` 用来提供可选择的配置。

`constraint_setting` 用来表示一种配置属性，可以认为这个就是枚举类型

`constraint_value` 用来支持[multi-platform behavior](https://bazel.build/configure/attributes#platforms)，这个可以认为是一种枚举类型的值



```
# myapp/BUILD

sh_binary(
    name = "my_rocks",
    srcs = select({
        ":basalt": ["pyroxene.sh"],
        ":marble": ["calcite.sh"],
        "//conditions:default": ["feldspar.sh"],
    }),
)

config_setting(
    name = "basalt",
    constraint_values = [
        ":black",
        ":igneous",
    ],
)

config_setting(
    name = "marble",
    constraint_values = [
        ":white",
        ":metamorphic",
    ],
)

# constraint_setting acts as an enum type, and constraint_value as an enum value.
constraint_setting(name = "color")
constraint_value(name = "black", constraint_setting = "color")
constraint_value(name = "white", constraint_setting = "color")
constraint_setting(name = "texture")
constraint_value(name = "smooth", constraint_setting = "texture")
constraint_setting(name = "type")
constraint_value(name = "igneous", constraint_setting = "type")
constraint_value(name = "metamorphic", constraint_setting = "type")

platform(
    name = "basalt_platform",
    constraint_values = [
        ":black",
        ":igneous",
    ],
)

platform(
    name = "marble_platform",
    constraint_values = [
        ":white",
        ":smooth",
        ":metamorphic",
    ],
)
```

下面的命令，即编译“大理石”平台

```
bazel build //my_app:my_rocks --platforms=//myapp:marble_platform
```

就等同于下面的配置

```
bazel build //my_app:my_rocks --define color=white --define texture=smooth --define type=metamorphic
```



### Bazel的Platforms

上面介绍了基本的语法，现在看看这东西组合出来做什么，下面的概念在Toolchain部分才有用，如果不想理解也可以先不看。

Bazel可以在各种不同的硬件、操作系统和系统配置上构建和测试代码，使用多种不同版本的链接器和编译器等构建工具。为了管理这种复杂性，Bazel引入了`constraint`（约束）和平台的概念。约束是是指构建或者生产环境可能不同的地方，例如CPU架构（Darwin，XXX）、是否存在GPU或系统安装的编译器版本。平台是这些约束的集合，表示某个环境中可用的特定资源。

Bazel将平台区分为三种：

1. **Host（**主机**）** - 运行Bazel的平台，比方说我现在做编译，我在本地起了一个bazel，bazel cli启动了一个java进程，这个启动java进程的机器或者说当前的系统，就是Host
2. **Execution（**执行**）** - 构建工具在其中执行构建操作以生成中间和最终输出的平台，书接上文，java进程分析好了DAG（有向无环图），拆解为多个Action，如果后台启用了Buildfarm远程执行服务，这些Action就会被分配到这些Buildfarm真正的机器上执行，这些真正执行构建的机器，就是Execution
3. **Target(**目标) - 最终输出所在的平台，并在其中执行。继续举例，我比方说在X86编译出来了一个软件，这个软件在ARM平台执行，那么最终执行的平台就是Target平台

关于平台，Bazel支持以下构建场景：

1. 单平台构建（默认）- 主机、执行和目标平台相同。例如，在运行在Intel x64 CPU上的Ubuntu上构建Linux可执行文件。
2. 交叉编译构建 - 主机和执行平台相同，但目标平台不同。例如，在运行在MacBook Pro上的macOS上构建iOS应用程序。
3. 多平台构建 - 主机、执行和目标平台都不同。



对于一部分monorepo而言，内部混杂了诸如X86（X86内部可能还按照CPU架构拆分的更细），ARM64的平台，这些代码库构建的时候都是指定了某种具体的平台，然后对某种pattern的内部package做构建。如果命令行里匹配的模式命中了某个被认为不兼容的目标，目标会自动将被跳过。例如，以下两个调用会跳过目标模式扩展中发现的任何不兼容目标。

```
$ bazel build --platforms=//:myplatform //...
```

```
$ bazel build --platforms=//:myplatform //:all
```

[`test_suite`](https://bazel.build/reference/be/general#test\_suite)如果`test_suite`在命令行上用 指定 ， 则类似地会跳过a 中不兼容的测试[`--expand_test_suites`](https://bazel.build/reference/command-line-reference#flag--expand\_test\_suites)。换句话说，`test_suite`命令行上的目标的行为类似于`:all`和 `...`。使用`--noexpand_test_suites`会阻止扩展并导致 `test_suite`具有不兼容测试的目标也不兼容。

在命令行上显式指定不兼容的目标会导致错误消息和构建失败。

```
$ bazel build --platforms=//:myplatform //:target_incompatible_with_myplatform
...
ERROR: Target //:target_incompatible_with_myplatform is incompatible and cannot be built, but was explicitly requested.
...
FAILED: Build did NOT complete successfully
```

如果启用`--skip_incompatible_explicit_targets`，不兼容的显式目标将被静默跳过 ，这就给提供了一种对CI很方便的bazel构建用法，我可以先bazel query反向依赖查出来修改的东西涉及到的文件，然后再全部编译，并且加上`--skip_incompatible_explicit_targets` flag，这样子，编译错误就不会被平台不兼容的错误所掩盖。

不过这个是bazel 7.0.0才支持。

如果想指定配置来检查改动设计哪些目标，可以用bazel cquery

\
