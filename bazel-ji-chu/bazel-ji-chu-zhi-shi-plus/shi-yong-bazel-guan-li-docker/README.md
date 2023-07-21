---
description: Docker
---

# 使用Bazel管理Docker

使用Docker编译镜像方便快捷，但是使用Docker打包出来的定向并不是确定的，换言之两个内部打包的binary一样的Docker镜像，其sha256是不同的，因此使用Bazel管理Docker镜像能带来的好处就是镜像确定化，即用来检查某些文件是不是发生了变化。

下面我会以rules\_docker作为基础工具讲讲一些基础的编译操作
