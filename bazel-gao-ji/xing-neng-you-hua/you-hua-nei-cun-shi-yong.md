# 优化内存使用

一般优化都是对时间效率的优化，不过另外一些时候希望优化对内存的使用。常见的方式就是指定启动flag --host\_jvm\_args设置最大堆大小，例如--host\_jvm\_args=-Xmx2g。

如果BUILD消耗内存，bazel可能抛出OutOfMemoryError（OOM）异常，不过我目前基本没遇到这个问题，针对一个7w个action的编译库，我配置的启动选项如下，，没啥问题。

```shellscript
bazel --host_jvm_args=-Xmx4g build -c opt
```



有一些选项可以用来控制内存中的graph是否保存，副作用是这次编译完了，下回没办法复用这次的编译结果。



* \--discard\_analysis\_cache将减少执行过程（而非分析过程）中使用的内存。下次构建将不需要重新加载依赖和包，但需要重新进行分析和执行（尽管磁盘上的操作缓存已经存在）
* \--notrack\_incremental\_state将不会存储任何Bazel的内部依赖图中的边，简单来说，依赖关系没啦！因此增量构建无法使用了
* \--nokeep\_state\_after\_build将在构建完成后丢弃所有数据，以便增量构建必须从头开始构建（除了磁盘上的操作缓存）

我目前用的是

```shellscript
bazel --host_jvm_args=-Xmx4g build -c opt xxxxxx --notrack_incremental_state --nokeep_state_after_build --discard_analysis_cache
```

这里可能有一点要注意，不一定是bazel造成的内存消耗过多，对于kubernetes环境，buffer/cache也是一个可能造成内存消耗过多的点，这部分在POD当中也会被当做是内存消耗，尽管不是JAVA导致的（毕竟编译要访问大量文件）。
