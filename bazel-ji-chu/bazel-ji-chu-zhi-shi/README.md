# Bazel基础知识

这一部分主要面向bazel体系的用户。希望读者能够明白

*   Bazel是构建平台，不能简单地理解为编译平台，错误理解这一点会导致无法快速找到对应的解决方案。有个用户曾经抱怨：bazel编译报错，而g++编译同样代码不报错，仔细一看报错如下图，这个是非常明显的bazel的工具链启用-Werror和-Wuninitialized选项导致的结果，如果g++启用同样的选项，也会报错。

    <figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>
* Bazel提供的是一种抽象的package，target，库依赖的关系，具体如何构建这些target和package，是对应的bazel rule的实现范畴。具体构建的细节，麻烦用户自己去看工具链的配置。
* Bazel的执行模式有多种：sandbox模式，本地模式等。因此使用sandbox模式的时候，本地临时文件如果不明确地指定到BUILD文件中，就无法在构建过程中看到（调用到）

这部分我建议读者先把C++那部分的例子看了，再根据需求选择使用的语言，选择对应的阅读章节。
