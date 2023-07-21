# 基础概念

### 引言

具体名词我不会翻译为中文，因为英文更明确，避免二义性。但是其意义我会用中文表达，从而方便理解。下面的内容可能有点多，可以先大致浏览理解概念，以后再回来细看。



### 为什么我要阅读这一页？

Bazel体系的用户在交流的时候需要有明确的名词来统一上下文，我看到很多次研发在别人给出具体的开发建议的时候，不知道怎么写BUILD的情况。因此，确定基础术语的概念会很有用



### 基础概念



在执行构建或者test任务的时候，bazel可以拆分为一下几个流程:

1. **Loads** 加载BUILD文件，并且将外部依赖
2. **Analyzes** 分析输入和输出的依赖，在内存中生成对应的action图.
3. **Executes** 执行真正的构建操作并输出相应的文件和日志



下面给出一些术语，方便统一上下文：

#### Action <a href="#action" id="action"></a>

执行构建过程的命令，比方说将一个完整的调用过程，改过程将其它Action的产物作为输入，并产生对应的其它编译产物。需要包含包括命令行参数、action索引（action key）、环境变量和在BUILD中声明的输入/输出工件等元数据

**参考** [Rules documentation](https://bazel.build/extending/rules#actions)

#### Action cache <a href="#action-cache" id="action-cache"></a>

磁盘上的缓存，存储执行的[操作](https://bazel.build/reference/glossary#action)到它们创建的输出的映射。缓存键称为action索引(action key)。Bazel 增量模型的核心组件。缓存存储在输出根目录（output base directory）中，因为是文件形式存储，所以在 Bazel 服务器重新启动后仍然存在。

#### Action graph <a href="#action-graph" id="action-graph"></a>

内存中的action关系图，表明action和具体的action读取和生成的产物[。](https://bazel.build/reference/glossary#artifact)该图读取`BUILD`。生成，从本质来说是一种依赖/生成关系的具象化。该图[在分析阶段](https://bazel.build/reference/glossary#analysis-phase)产生并在[执行阶段](https://bazel.build/reference/glossary#execution-phase)使用。目前还不需要了解分析阶段，执行阶段的作用是什么

#### Action graph query (aquery) <a href="#action-graph-query" id="action-graph-query"></a>

一个[查询](https://bazel.build/reference/glossary#query-concept)工具，可以查询action的依赖关系等。用户就可以了解从具体的规则，到转化为实际构建工作的过程是如何的。

#### Action key <a href="#action-key" id="action-key"></a>

action的缓存索引。根据action元数据计算得出，其中可能包括要在操作中执行的命令、编译器标志、库位置或系统头文件，具体取决于action的实现。从而使 Bazel 能够确定性地缓存或使单个action无效。

#### Analysis phase <a href="#analysis-phase" id="analysis-phase"></a>

构建的第二阶段。处理BUILD[文件](https://bazel.build/reference/glossary#build-file) 中指定的target之间的关系，并根据该[目标图](https://bazel.build/reference/glossary#target-graph)以生成内存中action graph。当前阶段是规则的实现被真正的解析的阶段。

#### Artifact <a href="#artifact" id="artifact"></a>

成品，指的是源文件或者代指生成的文件。也可以是文件目录，称为 [tree artifacts](https://bazel.build/reference/glossary#tree-artifact)

成品可以作为多个action的输入，参与到构建中，但最多只能由一个操作生成。

与[文件目标](https://bazel.build/reference/glossary#target)相对应的成品可以通过label来寻址。



#### Aspect <a href="#aspect" id="aspect"></a>

一种在现有rule的基础上拓展以执行更多action的机制，这种机制会传递到具体规则的依赖上，对于初级用户用不到，不过bazel-clang-tidy是用这个实现的。如果目标 A 依赖于 B，则可以在 A 上应用一个aspect，该aspect向上遍历到B中，并在 B 中运行其他action来生成和收集其他输出文件。这些附加操作同样被缓存并在重用。

参考**:** [Aspects documentation](https://bazel.build/extending/aspects)

#### Aspect-on-aspect <a href="#aspect-on-aspect" id="aspect-on-aspect"></a>

A composition mechanism whereby aspects can be applied to the results of other aspects. For example, an aspect that generates information for use by IDEs can be applied on top of an aspect that generates `.java` files from a proto.

For an aspect `A` to apply on top of aspect `B`, the [providers](https://bazel.build/reference/glossary#provider) that `B` advertises in its [`provides`](https://bazel.build/rules/lib/globals#aspect.provides) attribute must match what `A` declares it wants in its [`required_aspect_providers`](https://bazel.build/rules/lib/globals#aspect.required\_aspect\_providers) attribute.

#### Attribute <a href="#attribute" id="attribute"></a>

&#x20;[rule](https://bazel.build/reference/glossary#rule)的属性，用于表达每个目标的构建信息。例如，srcs（源文件）、deps（依赖项）和copts（自定义编译选项）分别声明了目标的源文件、依赖项和自定义编译选项。对于特定目标而言，可用的属性取决于其规则类型。

#### .bazelrc <a href="#bazelrc" id="bazelrc"></a>

Bazel的配置文件用于更改启动标志和命令标志的默认值，并定义常见选项组，可以使用--config标志在Bazel命令行上一起设置。Bazel可以从多个bazelrc文件（系统范围、工作空间范围、用户范围或自定义位置）中组合设置，而bazelrc文件也可以从其他bazelrc文件导入设置。

#### Blaze <a href="#blaze" id="blaze"></a>

google内部的编译构建系统，说实在的，google的infra真是令人羡慕呀

#### BUILD File <a href="#build-file" id="build-file"></a>

BUILD文件是Bazel的主要配置文件，用于告诉Bazel要构建哪些软件输出，它们的依赖关系是什么，以及如何构建它们。Bazel将BUILD文件作为输入，并使用该文件创建依赖图，并推导出构建中间和最终软件输出所必须完成的操作。BUILD文件将目录以及不包含BUILD文件的任何子目录标记为一个包（package），并且可以包含由规则创建的目标（targets）。该文件也可以被命名为BUILD.bazel。

#### BUILD.bazel File <a href="#build-bazel-file" id="build-bazel-file"></a>

参考 [`BUILD` File](https://bazel.build/reference/glossary#build-file).优先级高于同目录的BUILD

#### .bzl File <a href="#bzl-file" id="bzl-file"></a>

一个文件，用Starlark语言编写，用于定义规则（rules）、宏（macros）和常量（constants）。然后可以使用load()函数将其导入到BUILD文件中。实际上后面看什么bazel管理C++，bazel管理GO就能看到类似的load使用方法。

#### Build graph <a href="#build-graph" id="build-graph"></a>

Bazel构建过程中构建和遍历的依赖图。包括目标（targets）、配置后的目标（configured targets）、操作（actions）和构件（artifacts）等节点。当所有被请求的目标所依赖的构件都被验证为最新时，构建被视为完成。



#### Build setting <a href="#build-setting" id="build-setting"></a>

一个由Starlark定义的部分配置。transitions（配置传递）可以设置构建设置以更改子图的配置。对用户而言，这个可以理解为一种编译配置，比方说--config=cpu



#### Clean build <a href="#clean-build" id="clean-build"></a>

一个不使用之前构建结果的构建过程。这通常比增量构建更慢，但通常被认为更正确。保证无论是清理构建（clean build）还是增量构建（incremental build），都始终是正确的是Bazel的功能，不过确定性这点bazel在极低情况下确实会出问题，我明确遇到过在aliyun的内部pod下载的外部依赖版本不正确的问题，最后是通过指定url的下载地址避免这个问题。

#### Client-server model <a href="#client-server-model" id="client-server-model"></a>

bazel会自动在本地机器上启动一个后台服务器来执行Bazel命令。该服务器在命令之间保持持续存在，但在一段时间的无活动后（或通过bazel shutdown显式停止）会自动关闭。将Bazel拆分为服务器和客户端有助于分摊JVM启动时间，并支持更快的增量构建，因为操作图在命令之间保留在内存中。说白了，分离控制端和策略端

#### Command <a href="#command" id="command"></a>

简单理解就是Bazel的子函数，例如bazel build、bazel test、bazel run和bazel query。

#### Command flags <a href="#command-flags" id="command-flags"></a>

一组影响相关命令的标志（flags）。先指定具体命令，再指定flag（例如：先bazel build，然后在--keep going啥的 ）。这些标志可以适用于一个或多个命令。例如，--configure是仅用于bazel sync命令的标志，而--keep\_going适用于sync、build、test等多个命令。command flag通常用于配置目的，因此command flag的更改可能会导致Bazel内存中存储的action graph无效，并重新启动分析阶段。



#### Configuration <a href="#configuration" id="configuration"></a>

配置，rule定义之外的信息，会影响rule生成action的方式。每个build都至少有一个配置，用于指定目标平台、操作环境变量和命令行选项。 [Transitions](https://bazel.build/reference/glossary#transition)可能会创建额外的配置，例如用于指定主机的工具链或交叉编译。

参考**:** [Configurations](https://bazel.build/extending/rules#configurations)

#### Configuration trimming <a href="#config-trimming" id="config-trimming"></a>

配置建材，简单来说就是只包含目标实际需要的配置的过程。例如，如果你使用C++依赖项//:c构建Java二进制文件//:j，在//:c的配置中包含--javacopt的值是不必要的，因为更改--javacopt会不必要地破坏C++构建的缓存性能。因此，按需配置确保每个目标仅包含其自身所需的配置信息，以避免不必要的配置冗余和影响构建缓存性能。

#### Configured query (cquery) <a href="#configured-query" id="configured-query"></a>

检索启用了配置之后的 [configured targets](https://bazel.build/reference/glossary#configured-target)的检索工具，`select()` and [build flags](https://bazel.build/reference/glossary#command-flags) (such as `--platforms`)的选择是已经明确化了。这个在[analysis phase](https://bazel.build/reference/glossary#analysis-phase)之后才执行

参考**:** [cquery documentation](https://bazel.build/query/cquery)

#### Configured target <a href="#configured-target" id="configured-target"></a>

启用了某种配置的target

#### Correctness <a href="#correctness" id="correctness"></a>

当构建的输出忠实地反映其传递输入的状态时，构建就是正确的。为了实现正确的构建，Bazel努力做到具有隔离性、可重现性，并使构建分析和操作执行具有确定性。这意味着在构建过程中，Bazel会尽力确保每次构建都以一致的方式处理输入，从而产生可预测和可重复的结果。这样可以确保构建的正确性，并减少不确定性带来的问题。

不过这个概念对用户一般来说没啥用，毕竟正确性大部分永不不会涉及到，很多时候问题也不是bazel的问题。

#### Dependency <a href="#dependency" id="dependency"></a>

依赖关系，一般是指两个target直接的依赖关系，就是BUILD文件里面规则的deps

#### Depset <a href="#depset" id="depset"></a>

一种用于收集传递依赖关系数据的数据结构。它经过优化，使得合并依赖集（depset）在时间和空间上更加高效，因为依赖集往往非常大（成千上万个文件）。为了节省空间，该数据结构被实现为可以递归引用其他依赖集。规则实现在不是构建图顶层的情况下，不应将依赖集“展开”为列表形式。展开大型依赖集会导致巨大的内存消耗。在Bazel的内部实现中，它也被称为嵌套集合（nested sets）

**参考:** [Depset documentation](https://bazel.build/extending/depsets)

#### Disk cache <a href="#disk-cache" id="disk-cache"></a>

本地磁盘的Blob存储用于远程缓存功能。它可以与实际的远程Blob存储结合使用。在构建过程中，Bazel会将构建输出结果转换为Blob，并将其存储在本地磁盘的Blob存储中。这样可以提高后续构建的速度，因为它可以避免重复构建相同的输出结果。实际的远程Blob存储通常用于分布式构建环境，可以将Blob存储在远程服务器上，以便多个构建节点共享和访问这些Blob。

#### Distdir <a href="#distdir" id="distdir"></a>

只读目录，包含了Bazel本来会通过代码库规则从互联网获取的文件。它使得构建可以完全离线运行，不需要依赖于网络资源。通过将这些文件预先存储在只读目录中，Bazel可以在没有网络连接的情况下执行构建过程，并使用本地文件来满足构建所需的依赖。这种方式对于处于隔离环境或无法访问互联网的构建系统非常有用。不过说起来，distdir对于我的实践过程而言，最重要的还是提供对第三方依赖的缓存

#### Dynamic execution <a href="#dynamic-execution" id="dynamic-execution"></a>

动态执行策略是根据各种启发式规则，在本地和远程执行之间进行选择，并使用更快速成功的方法的执行结果。某些操作在本地执行速度更快（例如链接操作），而其他操作在远程执行速度更快（例如高度可并行化的编译操作）。动态执行策略可以提供最佳的增量构建和清理构建时间。通过动态地选择本地或远程执行，Bazel可以根据操作类型和执行环境的特点，最大程度地优化构建过程，从而获得更快的构建时间。



#### Execution phase <a href="#execution-phase" id="execution-phase"></a>

构建的第三个阶段，执行在分析阶段创建的操作图中的操作。这些操作调用可执行文件（编译器、脚本）来读取和写入构件。生成策略控制这些操作的执行方式：本地执行、远程执行、动态执行、沙盒化执行、使用Docker等等。

注意，具体执行的环境很多时候都是用户显示指定，其中配置的传递是个很蛋疼的东西，因为很多本地的配置是不能在sandbox环境完全一致的。

#### Execution root <a href="#execution-root" id="execution-root"></a>

执行根目录（Execution Root）是workspace的输出目录中的一个目录，这里的action是在在非沙盒的环境中执行。该目录的内容主要放的是各种输入产物的hard link。执行根目录还包含指向外部依赖的符号软链接作为输入，以及用于存储输出的bazel-out目录。

在加载阶段通过创建大量符号软链接的方式来生成该根目录。可以通过在命令行中使用"bazel info execution\_root"来访问该目录。

实际上，在buildfarm里面这个execution root的概念比较常用，因为有大量的分发的执行任务。

#### File <a href="#file" id="file"></a>

参考 [Artifact](https://bazel.build/reference/glossary#artifact).

#### Hermeticity <a href="#hermeticity" id="hermeticity"></a>

如果构建和测试操作没有外部影响，那么构建就是隔离的（hermetic），这有助于确保结果具有确定性和正确性。例如，隔离的构建通常禁止操作访问网络，限制对声明的输入的访问，使用固定的时间戳和时区，限制对环境变量的访问，并为随机数生成器使用固定的种子。

通过确保构建过程中没有外部的不可控因素干扰，隔离的构建可以消除构建结果的不确定性，并提高构建的可靠性和可重复性。这样可以更容易地定位和解决构建中的问题，并确保不同环境下的构建结果一致，从而提高构建的可移植性和可靠性。

#### Incremental build <a href="#incremental-build" id="incremental-build"></a>

我一般叫做，增量编译。更标准应该是增量构建。增量构建是通过重复使用先前构建的结果来减少构建时间和资源使用的一种方式。依赖检查和缓存旨在为此类型的构建生成正确的结果。增量构建是清理构建的反义词。

在增量构建中，Bazel会检查先前构建生成的构件，并根据构建过程中的依赖关系来确定哪些构件是最新的和有效的。只有在需要更新的构件上才会执行必要的操作，从而避免不必要的重新构建。这种方式可以显著减少构建时间，提高开发效率，并降低资源消耗。

#### Label <a href="#label" id="label"></a>

标签（label）是用于唯一标识Bazel构建系统中的目标的方式。通过标签，可以准确定位和引用构建过程中所涉及的不同目标。标签的结构使得能够精确指定目标所在的位置，并指定目标的名称。这种标识符的使用使得Bazel可以在构建和依赖管理过程中准确地追踪和处理不同的目标

完全限定的标签（fully-qualified label），例如//path/to/package:target，由以下部分组成：//用于标记工作空间的根目录，path/to/package表示包含声明该目标的BUILD文件的目录，:target表示在前述BUILD文件中声明的目标的名称。也可以在前面添加@my\_repository//<..>，表示该目标在名为my\_repository的外部依赖代码库中声明。



#### Loading phase <a href="#loading-phase" id="loading-phase"></a>

构建的第一阶段，Bazel在此阶段解析WORKSPACE、BUILD和.bzl文件，创建包（packages）。在此阶段，还会评估宏（macros）和某些函数，如glob()。该阶段与构建的第二阶段，即分析阶段，交替进行，以建立目标图（target graph）。

在加载阶段，Bazel会解析工作空间（WORKSPACE）文件，确定项目的配置和依赖关系。同时，它会解析BUILD文件和.bzl文件，创建和配置目标（targets）。宏和函数的评估也在此阶段进行，这有助于在构建过程中执行各种动态操作。加载阶段与分析阶段交替进行，以逐步构建目标图，为后续的构建阶段做好准备。

#### Macro <a href="#macro" id="macro"></a>

这个机制允许开发者将多个规则目标的声明逻辑封装在一个自定义的Starlark函数中，以提高代码的可重用性和可维护性。通过定义这样的函数，可以在多个BUILD文件中调用该函数来创建相同的规则目标，从而避免了重复编写相似的规则声明代码。在加载阶段，Bazel会将这些组合规则函数展开为实际的规则目标声明，以便后续的构建阶段使用。这种机制使得规则声明的复用变得更加方便和灵活。

**参考:** [Macro documentation](https://bazel.build/extending/macros)

#### Mnemonic <a href="#mnemonic" id="mnemonic"></a>

通过使用助记符，规则作者可以为规则中的操作赋予具有描述性和易记性的名称，使其更容易理解和识别。助记符通常与特定的操作类型或规则相关联，帮助开发者快速了解操作的用途和功能。这种命名方式有助于提高代码的可读性和可维护性，并促进团队之间的协作和理解。

#### Native rules <a href="#native-rules" id="native-rules"></a>

内置规则（Built-in rules）是Bazel中内置的、由Java实现的规则。这些规则在.bzl文件中以原生模块中的函数形式出现（例如native.cc\_library或native.java\_library）。而用户定义的规则（非原生规则）则是使用Starlark创建的。

#### Output base <a href="#output-base" id="output-base"></a>

在Bazel中，工作空间专用目录（workspace-specific directory）通常被称为"bazel-out"。它是一个由Bazel自动创建的目录，用于存储构建过程中生成的各种输出文件，如编译生成的二进制文件、中间文件、测试结果、日志文件等。该目录的位置位于Bazel的输出用户根目录下，用于将构建过程中产生的文件与源代码目录分离开，以保持项目结构的清晰性和可维护性。

#### Output groups <a href="#output-groups" id="output-groups"></a>

输出组（Output group）是指在Bazel完成构建一个目标后，预计将生成的一组文件。规则通常会将其常规输出放在"default output group"中（例如，Java规则的.jar文件、C++规则目标的.a和.so文件）。默认输出组是在命令行上请求目标时构建的输出组。

规则可以定义更多的命名输出组，可以在BUILD文件（使用filegroup规则）或命令行（--output\_groups标志）中显式指定。命名输出组允许开发者将特定类型的输出文件分组，以便在构建过程中对其进行特定的处理或操作。

通过输出组的概念，Bazel提供了一种灵活的机制来管理构建过程中生成的文件。开发者可以根据需要将文件分组，以便更好地组织、处理和操作构建输出。这样可以提高构建过程的灵活性和可定制性，并为构建系统的构建结果提供更好的结构和组织。

#### Output user root <a href="#output-user-root" id="output-user-root"></a>

用户特定的目录，用于存储Bazel的输出。目录名称来自于用户系统的用户名。如果多个用户同时在系统上构建同一项目，这样的目录结构可以避免输出文件冲突。它包含与各个工作空间的构建输出相对应的子目录，也被称为输出基目录（output bases）。

#### Package <a href="#package" id="package"></a>

一个BUILD文件定义的目标集合。一个包（package）的名称是相对于工作空间根目录的BUILD文件路径。一个包可以包含子包，或者包含BUILD文件的子目录，从而形成一个包的层级结构。

在Bazel中，每个BUILD文件定义了一个或多个目标。这些目标可以是编译的二进制文件、库、测试等。一个包是一组相关目标的集合，用于组织和管理相关代码和资源。包的名称是BUILD文件相对于工作空间根目录的路径，通过名称可以唯一标识和引用该包。

一个包可以包含多个子包，这些子包可以是在包内部的子目录，每个子目录都包含一个或多个BUILD文件。这种层级结构的组织方式使得代码和资源可以按照逻辑和功能进行分组，便于项目的管理和维护。通过包的层级结构，可以在构建过程中方便地引用和操作不同级别的目标，从而形成灵活和可扩展的项目结构.

#### Package group <a href="#package-group" id="package-group"></a>

一个目标，代表一组包（packages）。通常在可见性属性（visibility attribute）的值中使用。

#### Platform <a href="#platform" id="platform"></a>

在构建过程中涉及的“机器类型”（machine type）。这包括Bazel运行的主机平台（"host"平台）、构建工具在其上执行的机器（"exec"平台）以及构建目标的目标机器（"target"平台）。

主机平台（host platform）指的是运行Bazel的计算机系统，它提供了构建环境和资源。

执行平台（exec platform）是指在构建过程中实际执行构建工具的机器。这些构建工具可以是编译器、链接器、测试运行器等。执行平台可能与主机平台不同，特别是在跨平台构建或远程构建的情况下。

目标平台（target platform）是指构建的目标所要运行的目标机器。它可以是不同的操作系统、处理器架构或设备。Bazel支持在不同的目标平台上构建和运行代码，这使得跨平台开发和构建成为可能。

通过区分主机平台、执行平台和目标平台，Bazel可以根据不同的平台要求和配置，有效地管理和执行构建过程，并生成适用于特定目标平台的构建结果。

#### Provider <a href="#provider" id="provider"></a>

提供者（Provider）是一种描述在依赖关系中在规则目标之间传递的信息单元的模式。通常，它包含编译器选项、传递的源文件或输出文件以及构建元数据等信息。它通常与依赖集（depsets）结合使用，以高效地存储累积的传递数据。内置的提供者之一是DefaultInfo。

需要注意的是，针对特定规则目标保存具体数据的对象被称为“提供者实例”（provider instance），尽管有时它与“提供者”（provider）这个术语混淆使用。

提供者的概念用于在Bazel构建系统中传递和共享关于目标的相关信息。通过定义和使用提供者，不同的规则目标之间可以传递和访问彼此所需的数据和元数据。提供者实例是提供者模式的具体实现，用于为特定的规则目标提供特定的数据。这种机制使得规则目标之间可以有效地共享和传递信息，并为构建系统提供了更灵活和可扩展的方式来处理目标之间的依赖关系和数据传递。

**参考:** [Provider documentation](https://bazel.build/extending/rules#providers)

#### Query (concept) <a href="#query-concept" id="query-concept"></a>

构建分析是指对构建图进行分析，以了解目标属性和依赖关系结构的过程。Bazel支持三种查询变体：query、cquery和aquery。

* query：这是最常用的查询命令，用于从构建图中获取目标的属性信息、依赖关系、输出文件等。可以使用query命令来查询目标的属性值、依赖的目标以及目标的输出文件等详细信息。
* cquery：这是一个更复杂和强大的查询命令，用于执行复杂的构建图查询操作。它支持使用复杂的查询条件和过滤器来获取目标的详细信息，并可以根据查询结果执行特定的操作。
* aquery：这是一个用于分析构建图的高级查询命令。它可以提供有关目标之间依赖关系、构建过程和执行过程的详细信息。通过aquery命令，可以深入了解构建图的结构和执行细节，有助于进行构建性能分析和优化。



#### Repository cache <a href="#repo-cache" id="repo-cache"></a>

共享内容寻址缓存（Content-Addressable Cache）是Bazel为构建过程中下载的文件提供的一个共享缓存，可以跨工作空间共享。它使得在初始下载后可以进行离线构建成为可能。通常用于缓存通过repository rules（如http\_archive）和repository rule APIs（如repository\_ctx.download）下载的文件。只有在下载时指定了文件的SHA-256校验和，才会将文件缓存起来。

共享内容寻址缓存是一种通过文件内容的哈希值来寻址和存储文件的机制。在Bazel中，通过将文件的SHA-256校验和用作其唯一标识符，将文件存储在共享缓存中。当下次构建需要相同文件时，Bazel会首先检查共享缓存，如果缓存中已存在相应的文件，则可以直接使用缓存中的副本，而无需重新下载。

#### Reproducibility <a href="#reproducibility" id="reproducibility"></a>

无论环境，时间或者具体调用方式的变化，结果总是确定的，且可复现的。

#### Rule <a href="#rule" id="rule"></a>

规则（Rule）是用于在BUILD文件中定义规则目标（rule targets）的一种模式，例如cc\_library。从BUILD文件作者的角度来看，规则由一组属性和黑盒逻辑组成。逻辑告诉规则目标如何生成输出构件并将信息传递给其他规则目标。从.bzl文件作者的角度来看，规则是扩展Bazel以支持新的编程语言和环境的主要方式。

在加载阶段，规则被实例化为规则目标，用于生成构建图。在分析阶段，规则目标通过提供者（providers）的形式向下游依赖传递信息，并注册描述如何生成输出构件的操作。这些操作在执行阶段运行。

**参考:** [Rules documentation](https://bazel.build/extending/rules)



#### Runfiles <a href="#runfiles" id="runfiles"></a>

Runfiles是指测试在运行时所需的文件和目录。这些文件和目录可以是测试执行过程中需要访问的数据文件、配置文件、资源文件等。Bazel会根据构建规则中指定的运行时依赖项，将相应的Runfiles复制到与测试可执行文件并列的目录结构中。这样，在运行测试时，可执行文件就可以访问其所需的运行时数据。

**参考:** [Runfiles documentation](https://bazel.build/extending/rules#runfiles)

#### Sandboxing <a href="#sandboxing" id="sandboxing"></a>

沙盒，这个不用介绍了

#### Skyframe <a href="#skyframe" id="skyframe"></a>

[Skyframe](https://bazel.build/reference/skyframe) Bazel的核心框架，用来提供并发，执行等功能



#### Starlark <a href="#starlark" id="starlark"></a>

Starlark是用于编写规则和宏的扩展语言。它是Python的一个受限子集（在语法和语法上），旨在用于配置目的和提高性能。Starlark文件使用.bzl扩展名。BUILD文件使用Starlark的一个更受限制的版本（例如，不支持def函数定义），曾被称为Skylark。

**参考:** [Starlark language documentation](https://bazel.build/rules/language)

#### Startup flags <a href="#startup-flags" id="startup-flags"></a>

The set of flags specified between `bazel` and the [command](https://bazel.build/reference/glossary#query-command), for example, bazel `--host_jvm_debug` build. These flags modify the [configuration](https://bazel.build/reference/glossary#configuration) of the Bazel server, so any modification to startup flags causes a server restart. Startup flags are not specific to any command.

#### Target <a href="#target" id="target"></a>

目标（Target）是在BUILD文件中定义的对象，由标签（label）标识。从最终用户的角度来看，目标代表了工作空间的可构建单元。

通过实例化规则而声明的目标称为规则目标（rule target）。根据规则的不同，这些目标可以是可执行的（如cc\_binary）或可测试的（如cc\_test）。规则目标通常通过其属性（如deps）依赖于其他目标；这些依赖关系构成了目标图的基础。

除了规则目标，还有文件目标和包组目标。文件目标对应于在BUILD文件中引用的构件。作为特例，任何包的BUILD文件始终被视为该包中的源文件目标。

在加载阶段，目标被发现和解析。在分析阶段，目标与构建配置关联起来，形成配置目标（configured target）。

#### Target graph <a href="#target-graph" id="target-graph"></a>

构建图（Build graph）是一种目标及其依赖关系的内存中表示形式。它在加载阶段生成，并作为分析阶段的输入。

构建图是一个有向无环图（Directed Acyclic Graph，DAG），其中节点表示目标，边表示依赖关系。通过构建图，Bazel可以了解每个目标之间的依赖关系，以及构建过程中各个目标的顺序和执行逻辑

#### Target pattern <a href="#target-pattern" id="target-pattern"></a>

用于指定针对哪些target的方式，常用的模式有:all（所有规则目标），:\*（所有规则目标 + 文件目标），...（当前包和所有子包递归）。可以组合使用，例如，//...:\*表示从工作空间根目录递归地包含所有包中的所有规则和文件目标。



#### Tests <a href="#tests" id="tests"></a>

这个不用多介绍了吧，测试规则，一般要包含一个可执行对象，返回码啥的

#### Toolchain <a href="#toolchain" id="toolchain"></a>

用于构建某种语言输出的一组工具。通常，工具链包括编译器、链接器、解释器或/和代码检查工具。工具链也可以根据平台的不同而有所变化，也就是说，Unix编译器工具链的组件可能与Windows的不同，尽管工具链是用于相同的语言。选择适合平台的正确工具链称为工具链解析（toolchain resolution）。



#### Transition <a href="#transition" id="transition"></a>

将配置变换的过程。即使从同一个规则实例化，build graph中的目标也可以具有不同的配置，这就是转换的作用。

**参考:** [User-defined transitions](https://bazel.build/extending/config#user-defined-transitions)

#### Tree artifact <a href="#tree-artifact" id="tree-artifact"></a>

表示一堆产物的集合。由于这些文件本身不是artifacts，因此对它们进行操作的action必须将Tree artifact明确为其输入或输出。

#### Visibility <a href="#visibility" id="visibility"></a>

构建系统中防止不必要的依赖关系的两种机制之一：目标可见性用于控制一个目标是否可以被其他目标依赖；加载可见性用于控制BUILD或.bzl文件是否可以加载给定的.bzl文件。通常情况下，没有上下文的情况下，"可见性"通常指的是目标可见性。

**参考:** [Visibility documentation](https://bazel.build/concepts/visibility)

#### Workspace <a href="#workspace" id="workspace"></a>

工作区，实际上就是包含WORKSPACE文件的目录

#### WORKSPACE file <a href="#workspace-file" id="workspace-file"></a>

用来定义工作区的文件，一般来说还包含一些外部信息和初始化流程

