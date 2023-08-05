# 通用实践和部分细节

### 引言

这部分可以直接阅读[https://bazel.build/configure/best-practices](https://bazel.build/configure/best-practices)，初次之外还有一些常用的控制选项也一并列出



### 为什么要阅读这部分

这部分我个人觉得也可以不看。。。



### 最佳实践

下面的内容是一些提供给Infra或者普通用户的建议，这些规则可以使得CI或者研发的平时工作轻松很多。

#### 开放用户自定配置

有的时候Infra会提供一套通用的配置给研发使用，但是不同研发的本地环境不同，需要一些个性化且不被track到代码库的配置，可以在.bazelrc里面加上

```
try-import %workspace%/user.bazelrc
```

从而方便用户可以自己制定user.bazelrc里面的内容，最常见的使用情况就是并发的action数量控制，我们遇到过的是并发数量太高，用户电脑卡死了。。。

#### 尽量使用源码编译

对于C或C++软件，一个经常遇到的问题就是引用了不该用的库，或者是版本错误，或者是符号重名。这种情况下，推荐尽量用源码来管理第三方库，从而避免使用预定义的库可能带来的编译flag不一致的问题。这几天就遇到一个问题，有一个库内部打包了一个外部libcurl，(srcs="libcurl.so")然后打出来的包同时包含了系统的libcurl和这个libcurl，然后程序只要一启动就crash。

一部分人，那我不得所有依赖都得自己用源码管理吗？的确如此，毕竟只要用prebuild库，就始终有兼容性风险。

#### 整个源码库可构建

一般情况下整个源代码仓库可以直接调用`bazel build //...` 和 `bazel test //...，`对于CI的同学这种方式会简化管理难度。

#### 尽量明确依赖而不是混杂依赖

我们实践中遇到一个大坑是，一些用户会把不该汇总到一起的东西放到一起，它会用glob写出来一个如下语句，并最终把这个大包裹汇聚给外面。

```
glob(["**/**"])
```

这个BUILD文件所在目录包含模型，音频，xml文件，有很多的TEST只用音频，反而还打包一堆模型，从而导致多了很多不必要的依赖和拷贝操作。

#### BUILD文件不要跨级

一般情况下BUILD文件应该只管理当前目录的文件，不应该管理下级目录，如果真的需要管理下级目录，需要考虑文件的位置分布是否合理，是不是可以去掉没用的文件层级



### 常用选项



* 控制action并发数量build --jobs=xx
* 控制是否强行重跑某个test时，指定--nocache\_test\_results，即不利用缓存。排查flaky的问题很有用。cache\_test\_results本来是用来判断是否缓存test结果的
* 只做远端编译，不下载文件--[remote\_download\_minimal](https://bazel.build/reference/command-line-reference#flag--remote\_download\_minimal)
* 有时候做bazel test，希望看到test的完整输出，[--test\_output](https://bazel.build/reference/command-line-reference#flag--test\_output)=\<summary, errors, all or streamed>指定为streamed即可。不过这个会强制rerun test
* 有时候做query或者build，希望遇到错误不停止，加上[`--[no]keep_going`](https://bazel.build/reference/command-line-reference#flag--keep\_going) \[`-k`] default: "false"
* 在CI的环境，很多时候可以反向依赖查找出来改动影响到哪些target，之后针对这些target做编译。可是有个问题，这些target在明确指定的情况下，可能是不兼容的目标平台target。可以加上--skip\_incompatible\_explicit\_targets来显示忽略，从而降低CI的工作量
