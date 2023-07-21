# 编写Bazel 通用配置

### 引言

建议参考阅读[https://bazel.build/run/bazelrc](https://bazel.build/run/bazelrc)

### 为什么我要阅读这一页？

bazel有很多通用配置，这些配置一般来说对于一个研发体系而言，启用之后就基本不需要修改，一种简单的实现方法就是写.bazelrc文件，将这种配置都明面地写到一起。对于研发而言，知道配置的起点很重要，对于infra而言，知道写到哪里对问题排查非常重要。

### bazelrc的层级

这种bazelrc文件有四个层级，简单来说就是有四个地方可以直接配置

1.  **系统 RC 文件**，除非指定`--nosystem_rc`

    一般参考：

    * 在 Linux/macOS/Unix 上：`/etc/bazel.bazelrc`
    * 在 Windows 上：`%ProgramData%\bazel.bazelrc`
2.  **工作区 RC 文件**，除非`--noworkspace_rc`存在。

    一般参考工作区目录里面的`.bazelrc`在您的中（和 `WORKSPACE` 同目录）
3.  **主 RC 文件**，除非`--nohome_rc`存在。

    一般参考：

    * 在 Linux/macOS/Unix 上：`$HOME/.bazelrc`
    * 在 Windows 上：`%USERPROFILE%\.bazelrc`如果存在，或者`%HOME%/.bazelrc`
4. **用户指定的 RC 文件**（如果指定） `--bazelrc=file`可选的，但也可以指定多次。



### bazelrc的语法

**Imports语法**

`import` 和`try-import` 一般用来加载其它 "rc" files. 一般都用来引入和workspace相对的路径。 `import %workspace%/path/to/bazelrc`. `try-import`导入文件，如果文件不存在也不会报错。后导入的文件优先级更高，简单来说就是后倒入的会比前面导入的生效优先级更高

**默认Option语法**

用来指定配置相关的语法:

* `startup`: 启动选项，不过一般是针对java或者bazel通用的启动选项 `bazel help startup_options`.可以检索该类命令。
* `common`: bazel命令通用配置，如果某个bazel命令不支持这个选项，会直接失败
* _`command`_: 针对具体的bazel命令的配置，比方说 `build` or `query` 级别的配置. 从命令派生的子命令也会继承这些配置. (举个例子, `test` 继承 `build`的配置)

这些行中的每一行都可以多次使用，并且第一个单词后面的参数被组合起来，就像它们出现在一行上一样 啥意思呢？

```
build --test_tmpdir=/tmp/foo --verbose_failures
build --test_tmpdir=/tmp/bar
```

等同于:

```
build --test_tmpdir=/tmp/foo --verbose_failures --test_tmpdir=/tmp/bar
```

Option 优先级:

* 运行的shell命令里面的配置的优先级，总是高于配置文件里面的优先级。 例如，如果 rc 文件显示`build -c opt`但命令行标志为 `-c dbg`，则命令行标志优先
* rc内部，更具体的配置命令选项会覆盖通用的配置选项
* 两行配置，如果是给同样的配置选项组，那么他们生效的顺序和配置里面的顺序一致。
* 建议配置文件里面的配置顺序从粗粒度到细粒度，避免出现可读性的问题。

**`--config`**

如何组合配置选型？config是一种通用的方式来组合配置 ，只需要添加 `:name` 后缀到命令之前即可. 默认情况下会忽略这些选项，但当选项存在时，无论是在命令行上还是在文件中，甚至在另一个配置定义中，都会递归地包含这些选项。由 指定的选项只会按照上述优先顺序针对适用的命令进行扩展

举个简单的例子，这种情况下只需要编译时指定bazel build -c opt --config=c5即可成套编译

```ini
build:c5 --platforms=//bazel/platforms:c5_cross
build:c5 --config=cpu
build:c5 --copt="-D__C5__=1"
build:c5 --platform_suffix=c5

```

### 其它通用配置 <a href="#bazel-behavior-files" id="bazel-behavior-files"></a>

**`.bazelignore`**

Bazel 忽略的目录，使用`.bazelignore`来忽略，添加希望 Bazel 忽略的目录，每行一个即可，这些目录条目需要相对于工作区根目录的。

