# Bazel Platform与可配置选项

### 可配置选项

首先还是推荐阅读[https://bazel.build/configure/attributes](https://bazel.build/configure/attributes)

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

\
