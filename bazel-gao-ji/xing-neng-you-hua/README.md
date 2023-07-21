# 性能优化

Bazel的性能优化一般集中在以下几个方面：

* 远程执行或远程缓存：共享action cache或文件，提供命中率
  * 远程执行，默认情况下，编译和测试都在本地机器执行. 远程执行运行用户将编译和测试分布到不同的多台机器上执行.
  * 远程缓存，remote cache：一般是相同Project的开发者，或者一个CI体系的研发系统用来共享编译产物，只要产物是确定性且可复用的，那么就能提高效率。参考[https://github.com/buchgr/bazel-remote/](https://github.com/buchgr/bazel-remote/)
* 减少外部依赖下载所使用的时间
  * 简单来说就是distdir和第三方repo的cache
* 减少java对内存的消耗
