# C++跨平台实例

### 引言

建议直接参考[https://ltekieli.com/cross-compiling-with-bazel/](https://ltekieli.com/cross-compiling-with-bazel/)，我这里只是一个简单的翻译



### 内容

简单介绍一下 Bazel 以及如何使用这个工具进行交叉编译。这里会解释如何针对主机平台以及多个不同的目标进行构建。[可以在此处](https://github.com/ltekieli/bazel\_cross\_compile)获取本练习中使用的部分存储库。

#### 安装 Bazelisk <a href="#installingbazelisk" id="installingbazelisk"></a>

我强烈建议使用[Bazelisk](https://github.com/bazelbuild/bazelisk)来管理 Bazel 安装。命令如下：

```
$ wget https://github.com/bazelbuild/bazelisk/releases/download/v1.6.1/bazelisk-linux-amd64
$ sudo mv bazelisk-linux-amd64 /usr/local/bin/bazel
$ sudo chmod +x /usr/local/bin/bazel
```

#### 编译“hello world！” <a href="#compilinghelloworld" id="compilinghelloworld"></a>

一个C++ 项目最简单的结构如下所示，毕竟是hello world，没什么难点：

```
$ tree
.
├── BUILD
├── main.cpp
└── WORKSPACE

$ cat BUILD 
cc_binary(
    name = "hello",
    srcs = ["main.cpp"],
)

$ cat main.cpp 
#include <iostream>

int main() {
    std::cout << "Hello World!" << std::endl;
}
```

代码并不复杂，bazel的基础含义什么的原先也说过，可以直接跳过基础概念的解释

现在，直接使用bazel build就可以构建啦：

```
$ bazel build //:hello
2020/09/02 17:21:34 Downloading https://releases.bazel.build/3.4.1/release/bazel-3.4.1-linux-x86_64...
Extracting Bazel installation...
Starting local Bazel server and connecting to it...
INFO: Analyzed target //:hello (14 packages loaded, 47 targets configured).
INFO: Found 1 target...
Target //:hello up-to-date:
  bazel-bin/hello
INFO: Elapsed time: 8.451s, Critical Path: 0.62s
INFO: 2 processes: 2 linux-sandbox.
INFO: Build completed successfully, 6 total actions
```

如果要运行，用bazel run就可以看到结果了：

```
$ bazel run //:hello 
...
Hello World!
```



#### 下载依赖项 <a href="#downloadingdependencies" id="downloadingdependencies"></a>

为了设置交叉编译环境，我们需要下载工具链。手动执行此操作非常乏味，幸运的是 Bazel 已经包含了必要的工具。重点需要的实际上就是两个东西：编译器和 sysroot。

```
$ tree
.
├── BUILD
├── main.cpp
├── third_party
│   ├── BUILD
│   ├── deps.bzl
│   └── toolchains
│       ├── aarch64-rpi3-linux-gnu-sysroot.BUILD
│       ├── aarch64-rpi3-linux-gnu.BUILD
│       ├── arm-cortex_a8-linux-gnueabihf-sysroot.BUILD
│       ├── arm-cortex_a8-linux-gnueabihf.BUILD
│       ├── BUILD
│       └── toolchains.bzl
└── WORKSPACE
```

所有`*.BUILD"`文件都包含如何使用下载的工件内容的规范。这些`*.bzl`文件包含[Starlark](https://docs.bazel.build/versions/master/skylark/language.html)代码，该代码定义下载文件并将其公开为目标的逻辑。

Bazel的 `WORKSPACE可以获取`它需要下载任何外部的依赖：

```
$ cat WORKSPACE 
load("//third_party:deps.bzl", "deps")
deps()
```

这里的deps函数实际上就是加载工具链。如下面的代码`deps.bzl`：这里的toochains实际上隐藏了外部包的详细信息，下面有toolchains的代码

```
$ cat third_party/deps.bzl 
load("//third_party/toolchains:toolchains.bzl", "toolchains")

def deps():
    toolchains()
```

另一方面`toolchains.bzl`：

```
$ cat third_party/toolchains/toolchains.bzl 
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

URL_TOOLCHAIN = "https://github.com/ltekieli/devboards-toolchains/releases/download/v2020.09.01/"
URL_SYSROOT = "https://github.com/ltekieli/buildroot/releases/download/v2020.09.01/"

def toolchains():
    if "aarch64-rpi3-linux-gnu" not in native.existing_rules():
        http_archive(
            name = "aarch64-rpi3-linux-gnu",
            build_file = Label("//third_party/toolchains:aarch64-rpi3-linux-gnu.BUILD"),
            url = URL_TOOLCHAIN + "aarch64-rpi3-linux-gnu.tar.gz",
            sha256 = "35a093524e35061d0f10e302b99d164255dc285898d00a2b6ab14bfb64af3a45",
        )

    if "aarch64-rpi3-linux-gnu-sysroot" not in native.existing_rules():
        http_archive(
            name = "aarch64-rpi3-linux-gnu-sysroot",
            build_file = Label("//third_party/toolchains:aarch64-rpi3-linux-gnu-sysroot.BUILD"),
            url = URL_SYSROOT + "aarch64-rpi3-linux-gnu-sysroot.tar.gz",
            sha256 = "56f3d84c9adf192981a243f27e6970afe360a60b72083ae06a8aa5c0161077a5",
            strip_prefix = "sysroot",
        )

    if "arm-cortex_a8-linux-gnueabihf" not in native.existing_rules():
        http_archive(
            name = "arm-cortex_a8-linux-gnueabihf",
            build_file = Label("//third_party/toolchains:arm-cortex_a8-linux-gnueabihf.BUILD"),
            url = URL_TOOLCHAIN + "arm-cortex_a8-linux-gnueabihf.tar.gz",
            sha256 = "6176e47be8fde68744d94ee9276473648e2e3d98d22578803d833d189ee3a6f0",
        )

    if "arm-cortex_a8-linux-gnueabihf-sysroot" not in native.existing_rules():
        http_archive(
            name = "arm-cortex_a8-linux-gnueabihf-sysroot",
            build_file = Label("//third_party/toolchains:arm-cortex_a8-linux-gnueabihf-sysroot.BUILD"),
            url = URL_SYSROOT + "arm-cortex_a8-linux-gnueabihf-sysroot.tar.gz",
            sha256 = "89a72cc874420ad06394e2333dcbb17f088c2245587f1147ff9da124bb60328f",
            strip_prefix = "sysroot",
        )

```

这一段实际上就是先加载 http\_archive 规则，通过http下载的方式，将外部的依赖比方说compiler和sysroot都下载下来：

```
if "NAME_OF_THE_EXTERNAL_RESOURCE" not in native.existing_rules():
    http_archive(
        name = "NAME_OF_THE_EXTERNAL_RESOURCE",
        build_file = Label("//path/to:buildfile.BUILD"),
        url = SOME_URL,
        sha256 = SOME_SHA256,
    )
```

其中内容为：如果尚未定义此类规则，则使用具有给定名称、构建文件、URL 和校验和的 http\_archive 来定义它。

如果往toolchain下载下来的文件继续排查，就会发现下面的代码，它将工具链内部的文件都公开了出来。方便任何内部调用使用

`BUILD`文件包含来自下载工件的目标的定义，例如：

```
$ cat third_party/toolchains/aarch64-rpi3-linux-gnu.BUILD 
package(default_visibility = ['//visibility:public'])

filegroup(
  name = 'toolchain',
  srcs = glob([
    '**',
  ]),
)
```

它将该包中所有目标的默认可见性指定为公共，并创建一个目标“工具链”，它是工件内所有文件的句柄。

运行下面的命令就可以拿到这些action执行的最终结果：

```
$ bazel build @aarch64-rpi3-linux-gnu//:toolchain
```

看看 Bazel 为我们做了什么：

```
$ tree -L 1 bazel-02_deps/external/aarch64-rpi3-linux-gnu/
├── aarch64-rpi3-linux-gnu
├── bin
├── BUILD.bazel
├── build.log.bz2
├── include
├── lib
├── libexec
├── share
└── WORKSPACE
```

Bazel 在他的缓存中下载了artifacts，将我们的`aarch64-rpi3-linux-gnu.BUILD`文件复制为`BUILD.bazel`并添加了一个`WORKSPACE`文件，表明这是另一个外部依赖库。我们可以通过指定存储库名称来引用此类包内的所有目标：`@aarch64-rpi3-linux-gnu//:toolchain`。

#### 设置自定义工具链 <a href="#settingupcustomtoolchains" id="settingupcustomtoolchains"></a>

Bazel 支持两种设置自定义工具链的方法，一种是旧方法`crosstool_top，`另一种是用平台的新方法，原文用的是两种方法，我这里翻译就不写旧的方法了，只使用新的方法。

使用具有上述设置的平台需要几个额外的步骤。首先我们需要定义我们的新平台：

```
$ cat bazel/platforms/BUILD 
platform(
    name = "rpi",
    constraint_values = [
        "@platforms//cpu:aarch64",
        "@platforms//os:linux",
    ],
)
```

其次，我们需要创建新的平台兼容的工具链目标：

```
$ cat bazel/toolchain/aarch64-rpi3-linux-gnu/BUILD 
...

toolchain(
    name = "aarch64_linux_toolchain",
    exec_compatible_with = [
        "@platforms//os:linux",
        "@platforms//cpu:x86_64",
    ],
    target_compatible_with = [
        "@platforms//os:linux",
        "@platforms//cpu:aarch64",
    ],
    toolchain = ":aarch64_toolchain",
    toolchain_type = "@bazel_tools//tools/cpp:toolchain_type",
)
```

最后、在WORKSPACE文件中注册工具链：

```
$ cat bazel/toolchain/toolchain.bzl 
def register_all_toolchains():
    native.register_toolchains(
        "//bazel/toolchain/aarch64-rpi3-linux-gnu:aarch64_linux_toolchain",
    )
```

```
$ cat WORKSPACE 
...

load("//bazel/toolchain:toolchain.bzl", "register_all_toolchains")
register_all_toolchains()
```

完成后，Bazel 需要不同的命令行参数来使用平台：

```
$ bazel build \
    --incompatible_enable_cc_toolchain_resolution \
    --platforms=//bazel/platforms:rpi \
    //:hello 
Starting local Bazel server and connecting to it...
INFO: Analyzed target //:hello (19 packages loaded, 7123 targets configured).
INFO: Found 1 target...
Target //:hello up-to-date:
  bazel-bin/hello
INFO: Elapsed time: 25.510s, Critical Path: 1.41s
INFO: 2 processes: 2 linux-sandbox.
INFO: Build completed successfully, 6 total actions

$ file bazel-bin/hello
bazel-bin/hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 5.5.5, not stripped
```



你可能会奇怪，这个怎么就自动生效了？这里实际上是指示 bazel 使用`aarch64`工具链。`gcc_toolchainbazel/toolchain/aarch64-rpi3-linux-gnu`

该文件包含我们在交叉编译时要使用的所有工具的定义：

```
$ cat bazel/toolchain/aarch64-rpi3-linux-gnu/BUILD 
package(default_visibility = ["//visibility:public"])

load(":cc_toolchain_config.bzl", "cc_toolchain_config")

filegroup(name = "empty")

filegroup(
  name = 'wrappers',
  srcs = glob([
    'wrappers/**',
  ]),
)

filegroup(
  name = 'all_files',
  srcs = [
    '@aarch64-rpi3-linux-gnu-sysroot//:sysroot',
    '@aarch64-rpi3-linux-gnu//:toolchain',
    ':wrappers',
  ],
)

cc_toolchain_config(name = "aarch64_toolchain_config")

cc_toolchain(
    name = "aarch64_toolchain",
    toolchain_identifier = "aarch64-toolchain",
    toolchain_config = ":aarch64_toolchain_config",
    all_files = ":all_files",
    compiler_files = ":all_files",
    dwp_files = ":empty",
    linker_files = ":all_files",
    objcopy_files = ":empty",
    strip_files = ":empty",
)

cc_toolchain_suite(
    name = "gcc_toolchain",
    toolchains = {
        "aarch64": ":aarch64_toolchain",
    },
    tags = ["manual"]
)
```

这里实际的操作，是搞了一堆外部文件，包括编译器和sysroot，之后将他们组装到一起，这里的filegroup是从编译器和 sysroot 外部下载的工件引用的文件的便捷包装器，为了简单起见，将引用各处的所有文件都包到了一起。

组装完成之后，使用一个更精细`cc_toolchain`来构建C++的工具链



代码路径下面实际上是这样的目录包装文件：

```
$ tree bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/
bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/
├── aarch64-rpi3-linux-gnu-ar -> wrapper
├── aarch64-rpi3-linux-gnu-cpp -> wrapper
├── aarch64-rpi3-linux-gnu-gcc -> wrapper
├── aarch64-rpi3-linux-gnu-gcov -> wrapper
├── aarch64-rpi3-linux-gnu-ld -> wrapper
├── aarch64-rpi3-linux-gnu-nm -> wrapper
├── aarch64-rpi3-linux-gnu-objdump -> wrapper
├── aarch64-rpi3-linux-gnu-strip -> wrapper
└── wrapper
```

wrapper是一个引用包装器脚本，它最终知道实际工具驻留在哪里：

```
$ cat bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/wrapper 
#!/bin/bash
 
NAME=$(basename "$0")
TOOLCHAIN_BINDIR=external/aarch64-rpi3-linux-gnu/bin
 
exec "${TOOLCHAIN_BINDIR}"/"${NAME}" "$@"
```

Bazel 将调用`bazel/toolchain/aarch64-rpi3-linux-gnu/wrappers/aarch64-rpi3-linux-gnu-gcc，`它将执行驻留在外部目录的 sandobx 内的实际 gcc：`external/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-gcc`。



再向下，看下如何找到对应的执行文件

```
load(":cc_toolchain_config.bzl", "cc_toolchain_config")
...
cc_toolchain_config(name = "aarch64_toolchain_config")
```

从附加文件加载工具链配置，其中包含特定工具链工具的路径，以及编译和链接步骤的默认标志：

```
$ cat bazel/toolchain/aarch64-rpi3-linux-gnu/cc_toolchain_config.bzl 
load("@bazel_tools//tools/build_defs/cc:action_names.bzl", "ACTION_NAMES")
load("@bazel_tools//tools/cpp:cc_toolchain_config_lib.bzl",
    "feature",
    "flag_group",
    "flag_set",
    "tool_path",
)

all_link_actions = [
    ACTION_NAMES.cpp_link_executable,
    ACTION_NAMES.cpp_link_dynamic_library,
    ACTION_NAMES.cpp_link_nodeps_dynamic_library,
]

all_compile_actions = [
    ACTION_NAMES.assemble,
    ACTION_NAMES.c_compile,
    ACTION_NAMES.clif_match,
    ACTION_NAMES.cpp_compile,
    ACTION_NAMES.cpp_header_parsing,
    ACTION_NAMES.cpp_module_codegen,
    ACTION_NAMES.cpp_module_compile,
    ACTION_NAMES.linkstamp_compile,
    ACTION_NAMES.lto_backend,
    ACTION_NAMES.preprocess_assemble,
]

def _impl(ctx):
    tool_paths = [
        tool_path(
            name = "ar",
            path = "wrappers/aarch64-rpi3-linux-gnu-ar",
        ),
        tool_path(
            name = "cpp",
            path = "wrappers/aarch64-rpi3-linux-gnu-cpp",
        ),
        tool_path(
            name = "gcc",
            path = "wrappers/aarch64-rpi3-linux-gnu-gcc",
        ),
        tool_path(
            name = "gcov",
            path = "wrappers/aarch64-rpi3-linux-gnu-gcov",
        ),
        tool_path(
            name = "ld",
            path = "wrappers/aarch64-rpi3-linux-gnu-ld",
        ),
        tool_path(
            name = "nm",
            path = "wrappers/aarch64-rpi3-linux-gnu-nm",
        ),
        tool_path(
            name = "objdump",
            path = "wrappers/aarch64-rpi3-linux-gnu-objdump",
        ),
        tool_path(
            name = "strip",
            path = "wrappers/aarch64-rpi3-linux-gnu-strip",
        ),
    ]

    default_compiler_flags = feature(
        name = "default_compiler_flags",
        enabled = True,
        flag_sets = [
            flag_set(
                actions = all_compile_actions,
                flag_groups = [
                    flag_group(
                        flags = [
                            "--sysroot=external/aarch64-rpi3-linux-gnu-sysroot",
                            "-no-canonical-prefixes",
                            "-fno-canonical-system-headers",
                            "-Wno-builtin-macro-redefined",
                            "-D__DATE__=\"redacted\"",
                            "-D__TIMESTAMP__=\"redacted\"",
                            "-D__TIME__=\"redacted\"",
                        ],
                    ),
                ],
            ),
        ],
    )
    
    default_linker_flags = feature(
        name = "default_linker_flags",
        enabled = True,
        flag_sets = [
            flag_set(
                actions = all_link_actions,
                flag_groups = ([
                    flag_group(
                        flags = [
                            "--sysroot=external/aarch64-rpi3-linux-gnu-sysroot",
                            "-lstdc++",
                        ],
                    ),
                ]),
            ),
        ],
    )

    features = [
        default_compiler_flags,
        default_linker_flags,
    ]

    return cc_common.create_cc_toolchain_config_info(
        ctx = ctx,
        features = features,
        toolchain_identifier = "aarch64-toolchain",
        host_system_name = "local",
        target_system_name = "unknown",
        target_cpu = "unknown",
        target_libc = "unknown",
        compiler = "unknown",
        abi_version = "unknown",
        abi_libc_version = "unknown",
        tool_paths = tool_paths,
    )

cc_toolchain_config = rule(
    implementation = _impl,
    attrs = {},
    provides = [CcToolchainConfigInfo],
)
```



#### 设置自定义平台 <a href="#settingupcustomplatforms" id="settingupcustomplatforms"></a>

### 设置 .bazelrc <a href="#settingupbazelrc" id="settingupbazelrc"></a>

为了方便起见，所有附加命令行参数都可以隐藏在`.bazelrc`文件中：

```
$ cat .bazelrc 
build:rpi --crosstool_top=//bazel/toolchain/aarch64-rpi3-linux-gnu:gcc_toolchain --cpu=aarch64
build:bbb --crosstool_top=//bazel/toolchain/arm-cortex_a8-linux-gnueabihf:gcc_toolchain --cpu=armv7

build:platform_build --incompatible_enable_cc_toolchain_resolution
build:rpi-platform --config=platform_build --platforms=//bazel/platforms:rpi
build:bbb-platform --config=platform_build --platforms=//bazel/platforms:bbb
```

调用简化为：

```
$ bazel build --config=rpi //:hello

$ bazel build --config=rpi-platform //:hello
```

### 概括 <a href="#summary" id="summary"></a>

这些步骤应该适用于大多数 C++ 工具链。尽管最终的设置过程可能会很复杂，但所带来的好处确实是值得的。可以获得开箱即用的缓存、分布式和单命令构建。
