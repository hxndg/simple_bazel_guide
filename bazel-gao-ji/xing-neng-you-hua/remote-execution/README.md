# Remote Execution

远程执行，说白了就是把应该在本地做的事情放到了云端做，换言之“分布式编译（构建）”，随着bazel7.0的到来，构建支持异步式构建，因此远程执行会带来巨大的性能提升。

bazel的远程执行有一套remote execution api，只要兼容就可以，因此有多种远程执行的服务可以选择。

* 非商业化服务
  * [Buildbarn](https://github.com/buildbarn)
  * [Buildfarm](https://github.com/bazelbuild/bazel-buildfarm)
  * [BuildGrid](https://gitlab.com/BuildGrid/buildgrid)
  * [Scoot](https://github.com/twitter/scoot)
* 商业化服务
  * [EngFlow Remote Execution](https://www.engflow.com/) - Remote execution and remote caching service. Can be self-hosted or hosted.
  * [BuildBuddy](https://www.buildbuddy.io/) - Remote build execution, caching, and results UI.
  * [Flare](https://www.flare.build/) - Providing a cache + CDN for Bazel artifacts and Apple-focused remote builds in addition to build & test analytics.

这其中[bazel-buildfarm ](https://github.com/bazelbuild/bazel-buildfarm)是bazel官方出的RBE方案官方文档，目前实际上已经处于可用状态，而且有一个很关键的一点，buildfarm支持worker节点扩缩容，因此我重点介绍这个软件。

关于其架构可以参考下面的链接，来获取一些基本的了解： [https://github.com/bazelbuild/bazel-buildfarm/issues/351](https://github.com/bazelbuild/bazel-buildfarm/issues/351)
