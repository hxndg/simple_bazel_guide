# 使用Bazel引入第三方库

首先依然推荐阅读官方文档[https://bazel.build/external/overview](https://bazel.build/external/overview)，伴随着Bazel 6.0的发布，目前有两种方法来引入第三方库，一种是传统的写到Workspace的方法，另一种是Bzlmod的方法。注意，永远都是推荐新的方式即Bzlmod的方式来import第三方库
