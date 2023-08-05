# 使用bazel管理shell对象

### 引言

shell，作为胶水语言，我实际上不推荐用bazel来管理，不过偶尔也别无他法。建议用户直接阅读[https://bazel.build/reference/be/shell](https://bazel.build/reference/be/shell)

### 为什么我需要阅读这一页

但是有的时候需要写一个integration test，多个部分同时作用，除了shell没得什么可选；亦或想写一个通用的工具用来对代码做检查，这个集成到统一的bazel工具链当中，不用bazel管理shell对象也没办法。如果有这样子的场景，那么就需要学会用bazel管理shell对象

### GUIDE

下面的内容是一个sh\_test对象，bazel还支持sh\_binary，同样的道理。

直接看BUILD文件怎么写的，声明了一个run\_compare对象，核心是run\_sompare.sh脚本，依赖了shflags功能，shflags（[https://github.com/kward/shflags](https://github.com/kward/shflags)）实际上是个类似gflags的shell版本实现

```
package(default_visibility = ["//visibility:public"])

sh_library(
    name = "sh_flags",
    data = ["shflags"],
)

sh_test(
    name = "run_compare",
    size = "medium",
    srcs = ["run_compare.sh"],
    data = [
        "//save_money/compare:jandan",
    ],
    tags = ["manual"],
    # Notes: deps should only be sh_library,
    # other depends such as cc_binary and data should include in data field
    deps = [
        ":sh_flags",
    ],
)
```

这里可能需要注意下data属性，data属性提供了shell内部要直接调用的bazel target，这个和脚本是强关联的，脚本里面调用了什么对象，都需要加到这里，可以看到下面的脚本就是直接调用了save\_money/compare/jandan。

而shflags是所谓shell source的对象，放到了deps里面，这个是用户写shell需要注意的情况

```shellscript
TOP_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd -P)"
source "${TOP_DIR}/shflags"

DEFINE_boolean 'chinese' "${FLAGS_FALSE}" 'If true, use save chinese yuan.'
DEFINE_boolean 'US' "${FLAGS_TRUE}" 'If true, save us dollar.'

# Parse the command-line.
FLAGS "$@" || exit 1
eval set -- "${FLAGS_ARGV}"

set -e

echo "Envoking save money mode"
save_money/compare/jandan ${FLAGS_chinese} ${FLAGS_US}
```

最后直接调用bazel run就可以执行具体的shell target了。
