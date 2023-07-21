# 使用Bazel生成Compile DataBase

### 引言

建议直接阅读[https://github.com/grailbio/bazel-compilation-database](https://github.com/grailbio/bazel-compilation-database)，或者参考[https://github.com/hedronvision/bazel-compile-commands-extractor](https://github.com/hedronvision/bazel-compile-commands-extractor)

### 为什么我要阅读这一页？

很多第三方SAST检查工具，比方说CODECHECKER，都需要生成compile database来获取具体的编译行为，从而从语义的角度检查代码上下文是不是符合一些通用的安全开发准则。因此了解如何在bazel体系生成compile database就很有必要性。

### 用法

以[https://github.com/grailbio/bazel-compilation-database](https://github.com/grailbio/bazel-compilation-database)为例

添加如下内容在WORKSPACE里面

```
http_archive(
    name = "com_grail_bazel_compdb",
    strip_prefix = "bazel-compilation-database-0.5.2",
    urls = ["https://github.com/grailbio/bazel-compilation-database/archive/0.5.2.tar.gz"],
)

load("@com_grail_bazel_compdb//:deps.bzl", "bazel_compdb_deps")
bazel_compdb_deps()
```

针对想生成compile database的target，写入下面的代码，target可以直接写到targets里面

```
## Replace workspace_name and dir_path as per your setup.
load("@com_grail_bazel_compdb//:defs.bzl", "compilation_database")
load("@com_grail_bazel_output_base_util//:defs.bzl", "OUTPUT_BASE")

compilation_database(
    name = "example_compdb",
    targets = [
        "//a_cc_binary_label",
        "//a_cc_library_label",
    ],
    # OUTPUT_BASE is a dynamic value that will vary for each user workspace.
    # If you would like your build outputs to be the same across users, then
    # skip supplying this value, and substitute the default constant value
    # "__OUTPUT_BASE__" through an external tool like `sed` or `jq` (see
    # below shell commands for usage).
    output_base = OUTPUT_BASE,
)
```



在命令行运行下面的命令，就可以拿到对应target的compile database了

```
# Command to generate the compilation database file.
bazel build //path/to/pkg/dir:example_compdb

# Location of the compilation database file.
outfile="$(bazel info bazel-bin)/path/to/pkg/dir/compile_commands.json"

# [Optional] Command to replace the marker for output_base in the file if you
# did not use the dynamic value in the example above.
output_base=$(bazel info output_base)
sed -i.bak "s@__OUTPUT_BASE__@${output_base}@" "${outfile}"

# The compilation database is now ready to use at this location.
echo "Compilation Database: ${outfile}"
```
