# Remote Cache

Remote Cache的使用，相比Buildfarm就简单很多了，推荐直接参考[https://github.com/buchgr/bazel-remote/](https://github.com/buchgr/bazel-remote/)。

说白了，在本地就是运行

```
# Dockerhub example:
$ docker pull buchgr/bazel-remote-cache
$ docker run -u 1000:1000 -v /path/to/cache/dir:/data \
	-p 9090:8080 -p 9092:9092 buchgr/bazel-remote-cache \
	--max_size 5
```

在本地启动了对应的remote cache之后，在bazelrc文件当中写入下面的内容即可使用

```shellscript
common --remote_cache=xxx地址
```

