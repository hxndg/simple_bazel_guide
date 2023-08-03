# 编写自定义规则

### 引言

依然是建议直接阅读[https://bazel.build/rules/rules-tutorial](https://bazel.build/rules/rules-tutorial)

### 为什么我需要阅读这一页？



### 内容

Bazel用StarLark编写规则，嗯，可以认为是python的一种方言。写起来比较简单

#### 基础概念

理解bazel的规则之前需要先清楚一些基本的概念，这几个概念比较重要是因为不理解就不能看明白定义rule到底写了一些什么。可以直接阅读[https://www.jayconrod.com/posts/107/writing-bazel-rules--library-rule--depsets--providers](https://www.jayconrod.com/posts/107/writing-bazel-rules--library-rule--depsets--providers)来获取最地道的理解。

#### 结构体（struct)

结构体是 Starlark 中的基本数据结构（从技术上讲，结构体不是 Starlark 语言的一部分；它们由 Bazel 提供）。结构体值本质上是一个带着一堆名字的元组。通过调用struct函数来创建结构体值[`struct`](https://docs.bazel.build/versions/master/skylark/lib/struct.html)：

```
my_value = struct(
    foo = 12,
    bar = 34,
)
```

Python 中的对象中的字段访问方式在这里一样适用

```
print (my_value.foo + my_value.bar )
```

#### 供应者 （provider）

[`provider`](https://docs.bazel.build/versions/master/skylark/lib/Provider.html)是一个具名结构（如果写过C++会觉得这个词比较好了解），包含有关规则的信息。规则实现函数在执行末尾返回provider的结构。任何依赖于当前规则的其他行为（规则？）都可以读取提供者。可以通过调用该[`provider`](https://docs.bazel.build/versions/master/skylark/lib/globals.html#provider)函数来定义新的provider。

```
MyProvider = provider (
     doc = "我的自定义提供程序" ,
     fields = {
         "foo" : "一个 foo 值" ,
         "bar" : "一个 bar 值" ,
    },
）
```

#### 深度set（depset)

Bazel 提供了一种称为[depset](https://docs.bazel.build/versions/master/skylark/lib/depset.html)的特殊用途数据结构。与任何集合一样，depset 是一组唯一值。Depsets 与其他set的区别在于它可以快速合并并具有非常快的迭代顺序。

Depset 通常用于在可能较大的依赖关系图上累积信息，例如源文件或头文件。

depset 包含直接元素列表、传递子元素列表和迭代顺序。

![depset 图](https://www.jayconrod.com/images/depset.png)

构造 depset 很快，因为它只涉及创建具有直接列表和传递列表的对象。这需要 O(D+T) 时间，其中 D 是直接列表中的元素数量，T 是传递子元素的数量。Bazel 在构造集合时会删除两个列表中的重复元素。迭代 depset 或将其转换为列表需要 O(n) 时间，其中 n 是集合及其所有子元素（包括重复元素）中的元素数量。



#### 编写规则前理解执行顺序 <a href="#the_empty_rule" id="the_empty_rule"></a>

理解bazel的规则前，就需要理解bazel的构建顺序，比方说有下面的rule和BUILD文件，

```
def _foo_binary_impl(ctx):
    print("analyzing", ctx.label)

foo_binary = rule(
    implementation = _foo_binary_impl,
)

print("bzl file evaluation")
```

```
load(":foo.bzl", "foo_binary")

print("BUILD file")
foo_binary(name = "bin1")
foo_binary(name = "bin2")
```



执行下面的命令，其为何会输出诸如bzl file evaluation的结果呢？

```
$ bazel query :all
DEBUG: /usr/home/bazel-codelab/foo.bzl:8:1: bzl file evaluation
DEBUG: /usr/home/bazel-codelab/BUILD:2:1: BUILD file
//:bin2
//:bin1
```

有如下的原因

* "bzl file evaluation"先打印是因为bazel会先执行它家在的BUILD文件
* 自定义的规则`_foo_binary_impl` 实现部分不会执行，因为bazel query只会load Build文件，它不会去真正的分析target

#### 编写规则

看下面的规则实力，out定义了一个输出，ctx.actions.write表明了一个行为，这个行为会写出来一个输出文件。如果不加上return后面的语句，这个文件只会被当成临时文件不会产生，只有return了才会真正地生成。执行bazel build

<pre><code><strong>def _foo_binary_impl(ctx):
</strong>    out = ctx.actions.declare_file(ctx.label.name)
    ctx.actions.write(
        output = out,
        content = "Hello!\n",
    )
    return [DefaultInfo(files = depset([out]))]

foo_binary = rule(
    implementation = _foo_binary_impl,
)
</code></pre>



#### 编写支持属性的规则

下面编写更复杂一些的规则，需要显式地声明属性，从而可以在对应的规则实现中使用这些属性，如下面的代码所示，并不复杂

```
def _foo_binary_impl(ctx):
    out = ctx.actions.declare_file(ctx.label.name)
    ctx.actions.write(
        output = out,
        content = "Hello {}!\n".format(ctx.attr.username),
    )
    return [DefaultInfo(files = depset([out]))]

foo_binary = rule(
    implementation = _foo_binary_impl,
    attrs = {
        "username": attr.string(),
    },
)


```



#### 更复杂一些的例子

最后来看一个更复杂的例子，这个例子是抄的[https://zhuanlan.zhihu.com/p/266510840](https://zhuanlan.zhihu.com/p/266510840)，建议直接看他写的，比较清楚

```
def _cmd_binary_impl(ctx):
  temp_log = ctx.actions.declare_file("temp.log")
  output_log = ctx.actions.declare_file("output.log")

  cmd_str = "%s > %s" % (ctx.attr.command, temp_log.path)

  ctx.actions.run_shell(
      outputs = [temp_log],
      command = cmd_str,
  )

  ctx.actions.write(
      output=output_log,
      content = "Finish running shell",
  )
  return [DefaultInfo(files = depset([temp_log, output_logl))

cmd_binary = rule(
   implementation = _cmd_binary_impl,
   attrs = {
      "command": attr.string(),
   },
)

```

这里：

* ctx.actions.declare\_file: 声明文件，这跟一般C++/Java这些里面要先定义输入输出一个道理
* ctx.actions.run\_shell: 执行命令。注意第5行中我们定义了所执行的命令，实际上是把用户在BUILD文件中定义的命令输出写入temp.log文件
* ctx.actions.write: 写文件，这只是往文件随便写点东西

