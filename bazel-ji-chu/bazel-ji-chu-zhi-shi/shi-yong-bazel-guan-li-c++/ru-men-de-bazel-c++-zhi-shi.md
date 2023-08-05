---
description: 我实际上是抄的bazel官方文档。。。
---

# 入门的Bazel C++知识

### 引言

官方文档的link为[https://bazel.build/start/cpp](https://bazel.build/start/cpp)，建议直接阅读官方文档以获得最原汁原味的体验。如果你不清楚一些基础概念，比方说包，label（标签）是啥意思，看看前面基础概念那一节

###

### 为什么我要阅读这一页？

我建议所有的研发都阅读一下这部分，因为编译型语言C++本身就暗合最直观的输入，输出，头文件，声明定义等概念（当然，你说它最原始，最简单也不是不可以，可能诸如GO的开发者阅读的时候会觉得Bazel的包管理过于麻烦。）

而且比方说WORKSPACE从概念映射到具体的文件什么的，还有基础理念什么的，就C++入门这里写的比较详细，像入门的GO或者Python，基本就是WORKSPACE里面写什么，BUILD里面写什么啥的。。。





#### 先决条件 <a href="#prerequisites" id="prerequisites"></a>

装好bazel和git以后，先把bazel官方给的代码库clone下来

```
git clone https://github.com/bazelbuild/examples
```

本教程的示例项目位于该`examples/cpp-tutorial`目录中。

下面看一下它的结构：

```
examples
└── cpp-tutorial
    ├──stage1
    │  ├── main
    │  │   ├── BUILD
    │  │   └── hello-world.cc
    │  └── WORKSPACE
    ├──stage2
    │  ├── main
    │  │   ├── BUILD
    │  │   ├── hello-world.cc
    │  │   ├── hello-greet.cc
    │  │   └── hello-greet.h
    │  └── WORKSPACE
    └──stage3
       ├── main
       │   ├── BUILD
       │   ├── hello-world.cc
       │   ├── hello-greet.cc
       │   └── hello-greet.h
       ├── lib
       │   ├── BUILD
       │   ├── hello-time.cc
       │   └── hello-time.h
       └── WORKSPACE
```

共有三组文件，每组代表本教程中的一个阶段。在第一阶段，构建驻留在单个[包中的单个](https://bazel.build/reference/glossary#package)[目标](https://bazel.build/reference/glossary#target)。在第二阶段，您将从单个包构建二进制文件和库。在第三个也是最后一个阶段，您将构建一个包含多个包的项目并使用多个目标构建它。



### 入门 <a href="#getting_started" id="getting_started"></a>

#### 设置WorkSpace <a href="#set_up_the_workspace" id="set_up_the_workspace"></a>

在构建项目之前，需要设置WorkSpace。一般将工作区理解为一个保存源代码和 Bazel 构建输出的目录。它必须包含下面两种文件：

* [`WORKSPACE file`](https://bazel.build/reference/glossary#workspace-file)文件，标识一个 Bazel 工作区，一般是位于代码根目录下。
* [`BUILD files`](https://bazel.build/reference/glossary#build-file) ，告诉 Bazel 如何构建项目的不同部分。工作区中包含文件的目录`BUILD`是package。（本教程稍后将详细介绍包。）

在将来的项目中，要将目录指定为 Bazel 工作区，请创建一个`WORKSPACE`在该目录中命名的空文件。

**注意**：当 Bazel 构建项目时，所有输入必须位于同一工作区中。驻留在不同工作区中的文件彼此独立，除非显式地链接到一起。[有关工作区规则的更多详细信息可以在本指南](https://bazel.build/reference/be/workspace)中找到。

#### 了解 BUILD 文件 <a href="#understand_the_build_file" id="understand_the_build_file"></a>

一个`BUILD`文件包含多种不同类型的 Bazel 指令。每个 `BUILD`文件至少需要一个[规则](https://bazel.build/reference/glossary#rule) 作为一组指令，告诉 Bazel 如何构建目标，这些目标包括，例如可执行二进制binary或库。`BUILD`文件中构建rule的每个实例（）称为[目标](https://bazel.build/reference/glossary#target) ，并指向一组特定的源文件和[依赖项](https://bazel.build/reference/glossary#dependency)。一个目标也可以指向其他目标。

查看目录`BUILD`下的文件`cpp-tutorial/stage1/main`：

```
cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
)
```

在我们的示例中，`hello-world`目标实例化 Bazel 的内置 [`cc_binary rule`](https://bazel.build/reference/be/c-cpp#cc\_binary). 该规则告诉 Bazel 从源文件构建一个独立的linux binary， `hello-world.cc`没有依赖项。

#### 摘要：入门 <a href="#summary_getting_started" id="summary_getting_started"></a>

现在您已经熟悉了一些关键术语，以及它们在本项目和 Bazel 上下文中的含义。在下一部分中，您将构建并测试项目的第一阶段。

### 第一阶段：单一目标、单一包 <a href="#stage_1_single_target_single_package" id="stage_1_single_target_single_package"></a>

现在是构建项目第一部分的时候了。这部分的代码结构为：

```
examples
└── cpp-tutorial
    └──stage1
       ├── main
       │   ├── BUILD
       │   └── hello-world.cc
       └── WORKSPACE
```

`首先切换目录到cpp-tutorial/stage1`目录：

```
cd cpp-tutorial/stage1
```

接下来，运行：

```
bazel build //main:hello-world
```

可以看到如下的输出内容，Bazel 生成的东西看起来像这样：

```
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 2.267s, Critical Path: 0.25s
```

恭喜，使用bazel很简单。Bazel 将构建的目标target文件，放置在 `bazel-bin`工作区根目录中。

现在测试您新构建的二进制文件，即：

```
bazel-bin/main/hello-world
```

这会导致打印“ `Hello world`”消息。

这是第一阶段的依赖关系图：

![hello-world 的依赖关系图显示具有单个源文件的单个目标。](https://bazel.build/static/docs/images/cpp-tutorial-stage1.png)



### 第二阶段：多个构建目标 <a href="#stage_2_multiple_build_targets" id="stage_2_multiple_build_targets"></a>

虽然单个目标对于小型项目来说就足够了，但较大的项目，无论是从效率的角度，还是逻辑的角度，需要拆分为多个目标和包，。

第 2 阶段使用的目录：

```
    ├──stage2
    │  ├── main
    │  │   ├── BUILD
    │  │   ├── hello-world.cc
    │  │   ├── hello-greet.cc
    │  │   └── hello-greet.h
    │  └── WORKSPACE
```

下面看一下目录`BUILD`中的文件`cpp-tutorial/stage2/main`：

```
cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
    ],
)
```

使用此`BUILD`文件，Bazel 首先构建`hello-greet`库（使用 Bazel 的内置[`cc_library rule`](https://bazel.build/reference/be/c-cpp#cc\_library)），然后构建`hello-world`二进制文件。`deps`目标中的属性告诉`hello-world`Bazel`hello-greet` 构建二进制文件需要该库`hello-world`。

在构建该项目的新版本之前，您需要更改目录，`cpp-tutorial/stage2`通过运行以下命令切换到该目录：

```
cd ../stage2
```

现在您可以使用以下熟悉的命令构建新的二进制文件：

```
bazel build //main:hello-world
```

Bazel 再一次生成了如下所示的内容：

```
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 2.399s, Critical Path: 0.30s
```

现在您可以测试新构建的二进制文件，它返回另一个“ `Hello world`”：

```
bazel-bin/main/hello-world
```

如果您现在修改`hello-greet.cc`并重建项目，Bazel 只会重新编译该文件。

查看依赖关系图，您可以看到它`hello-world`依赖于名为 的额外输入`hello-greet`：

![“hello-world”的依赖关系图显示文件修改后的依赖关系更改。](https://bazel.build/static/docs/images/cpp-tutorial-stage2.png)

#### &#x20;<a href="#summary_stage_2" id="summary_stage_2"></a>

### 第三阶段：多包 <a href="#stage_3_multiple_packages" id="stage_3_multiple_packages"></a>

下一阶段又增加了一层复杂性，并构建了一个包含多个包的项目。下面看一下该 `cpp-tutorial/stage3`目录的结构和内容：

```
└──stage3
   ├── main
   │   ├── BUILD
   │   ├── hello-world.cc
   │   ├── hello-greet.cc
   │   └── hello-greet.h
   ├── lib
   │   ├── BUILD
   │   ├── hello-time.cc
   │   └── hello-time.h
   └── WORKSPACE
```

您可以看到现在有两个子目录，每个子目录都包含一个`BUILD` 文件。因此，对于 Bazel 来说，工作区现在包含两个包：`lib`和 `main`。

看一下`lib/BUILD`文件：

```
cc_library(
    name = "hello-time",
    srcs = ["hello-time.cc"],
    hdrs = ["hello-time.h"],
    visibility = ["//main:__pkg__"],
)
```

看一下`main/BUILD`文件中：

```
cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
        "//lib:hello-time",
    ],
)
```

主包中的目标`hello-world依赖包hello-time`中的目标`lib`（因此是目标标签`//lib:hello-time`）——Bazel 通过`deps`属性知道这一点。依赖关系图中看到这一点：

![“hello-world”的依赖关系图显示了主包中的目标如何依赖于“lib”包中的目标。](https://bazel.build/static/docs/images/cpp-tutorial-stage3.png)

为了成功构建，需要使用 Visibility 属性使`//lib:hello-time`目标`lib/BUILD` 对目标显式可见。`main/BUILD`这是因为默认情况下目标仅对同一文件中的其他目标可见 `BUILD`。Bazel 使用目标可见性来防止诸如包含实现细节的库泄漏到公共 API 等问题。

现在构建该项目的最终版本。`cpp-tutorial/stage3` 通过运行以下命令切换到目录：

```
cd  ../stage3
```

再次运行以下命令：

```
bazel build //main:hello-world
```

Bazel 生成的东西看起来像这样：

```
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 0.167s, Critical Path: 0.00s
```



好的，大功告成啦
