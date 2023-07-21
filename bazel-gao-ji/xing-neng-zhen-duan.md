# 性能诊断

### 默认profile文件

对于一个黑盒的增量编译系统，做性能诊断是比较困难的，不过Bazel内置了Profile功能，每执行完一次编译行为，就会在输出根目录生成一个command.profile.gz文件，即路径为下面的文件

```shellscript
$(bazel info output_base)/command.profile.gz
```

这个文件的分析，建议打开chrome://tracing/，然后load对应的profile文件进行分析。

###

### 如何阅读profile文件

要看剖析结果，请在Chrome浏览器标签中打开chrome://tracing，点击“Load”并选择对应的profile文件

![Example profile](https://bazel.build/static/docs/images/json-trace-profile.png)



* 按1进入“选择”模式。单击特定的方框以查看事件的详细信息，说白了就是看具体的时间消耗，比方说wall time，cpu time啥的
* 按2进入“平移”模式。然后拖动鼠标来移动视图
* 按3进入“缩放”模式。然后拖动鼠标进行缩放。用w/s键进行放大和缩小
* 按4进入“定时”模式，可以测量两个事件之间的时间间隔



### 阅读Profile文件的关键指标

bazel analyze-profile是bazel提供的子命令，不过这个我个人感觉不如自己写jq分析json的profile文件或者直接看时间消耗更方便。。。

#### 如何阅读bazel的时间消耗

一般阅读profile文件都是追求时间的效率，针对具体的bazel一个action，有意思的点击该action之后会显示相应的信息。有以下集中类型的时间消耗

* **Wall time** 是实际经过的现实世界时间.
  * 一般推荐使用[JSON trace profile](https://bazel.build/advanced/performance/json-trace-profile) 来分析性能消耗
* **CPU time**是CPU执行用户代码所花费的时间
  * 很多时候具体问题处在了用户代码还是bazel本身的问题很难说，这种情况下一般加上
* System time是CPU在内核中花费的时间.
  * 主要与Bazel从文件系统读取文件时的I/O相关，这个实际上也一般不出问题

如果怀疑时间可能和系统负载相关，可以加上 [`--experimental_collect_load_average_in_profiler`](https://github.com/bazelbuild/bazel/blob/6.0.0/src/main/java/com/google/devtools/build/lib/runtime/CommonCommandOptions.java#L306-L312) flag，这个flag是bazel 6.0引入，对应的profile文件里面会有对应系统负载的信息，如下图

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>



#### 显示网络的使用情况 <a href="#monitoring_network_traffic_for_remote_builds" id="monitoring_network_traffic_for_remote_builds"></a>

启用远端执行的情况下，网络带宽会影响编译的效率（因为需要上传下载文件和输入）

添加 `--experimental_collect_system_network_usage` 可以在profile文件里面看到网络使用情况。

启用BEP（构建事件协议）中的NetworkMetrics.SystemNetworkStats proto可以收集网络使用情况（需要启用--experimental\_collect\_system\_network\_usage参数）进行监控，不过我还没实践过监控EBP的情况

![Profile that includes system-wide network usage](https://bazel.build/static/docs/images/json-trace-profile-network-usage.png)



### 常见的profile问题点

* 分析阶段（runAnalysisPhase）比预期慢，尤其是增量构建的情况。这可能是规则实现不佳的迹象，例如将depsets展开的规则实现。包加载可能变慢是因为存在过多的目标、复杂的宏或递归的全局匹配。也有可能是文件下载过慢（比方说不用url，走bazel本身的下载），nas性能不够啥的，下一节写了这个情况
* 个别的慢速动作，特别是那些位于关键路径上的动作。可以尝试将大型动作拆分为多个较小的动作，或减少（传递）依赖项的数量以加快速度。比方说写了一个规则，单线程的跑test，这种常见于自己写的sh\_test
* 瓶颈问题，这个一般是分配任务不均匀导致。优化这个问题可能需要修改规则实现或Bazel本身，我倒是基本没遇到过。



### 阅读profile做文件诊断的例子

下面是一个profile文件的analysis结果截图，对profile文件进行的分析可以看到共编译了4000多个文件，耗时50min，还是比较正常的。

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

看下面一个曾经遇到的例子，发现Bazel编译耗时很长，然后编译行为并不多，也就编译了几十个action就花了50mins，排查是analysis stage外部依赖花了很多时间下载，然而这些外部依赖都缓存在了对应的nas里面，并配置到编译CI环境的distdir，针对性排查网络和nas性能，发现是nas性能打满了，导致analysis stage过慢，最终更换高性能nas，并独占解决该问题

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>



###
