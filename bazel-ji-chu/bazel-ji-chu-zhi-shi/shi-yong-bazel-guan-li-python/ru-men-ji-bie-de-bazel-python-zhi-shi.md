# 入门级别的Bazel Python知识

### 引言

我建议用户直接阅读[https://github.com/bazelbuild/rules\_python](https://github.com/bazelbuild/rules\_python)，获取最标准和最新的使用方法



### 为什么我需要阅读这一页？

这部分提供了一种简单地集成python产物和依赖的方法，对于初学者而言，环境的配置是个很大的问题，在我看来只要把环境配置的问题解决，用户就可以方便地去写python相关的代码了



### GUIDE

下面的内容，解决了两个问题

* 如何编写python binary对象？
* 如何引入python的外部依赖库

#### 前提

前提就是引入对rules\_python的使用，用bzlmod或者WORKSPACE的方式加载rules\_python，简单来说就是写入下面的内容

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

rules_python_version = "740825b7f74930c62f44af95c9a4c1bd428d2c53" # Latest @ 2021-06-23

http_archive(
    name = "rules_python",
    # Bazel will print the proper value to add here during the first build.
    # sha256 = "FIXME",
    strip_prefix = "rules_python-{}".format(rules_python_version),
    url = "https://github.com/bazelbuild/rules_python/archive/{}.zip".format(rules_python_version),
)
```

#### 如何编写python binary对象

使用rules\_python相关规则，即可方便的编写对应的binary和library

首先我们看一下对应的BUILD文件和py文件

```
load("@my_deps//:requirements.bzl", "requirement")
load("@rules_python//python:defs.bzl", "py_binary")

py_binary(
    name = "validate_jira_issue",
    srcs = ["validate_jira_issue.py"],
    deps = [
        requirement("jira"),
    ],
)
```

validate\_jira\_issue.py如下，我省略了部分的代码，可以看到这就是一个jira issue的校验代码，除了调用系统提供的基础库之外，还引用了jira的python库。那么问题就来了，怎么安装jira的库呢？使用requirement即可，它会在WORKSPACE初始化阶段自动引入外部依赖（注意，这可能不是最佳实践）

```python
#!/usr/bin/python3

import argparse
import logging
import os
import re
import sys
import traceback

import requests
from jira import JIRA, JIRAError

...

_JIRA_SERVER_ADDR = "https://daddy.jira.issues.net"
_JIRA_USER = "i am your daddy"


def jira_issue_link_check(jira_issue_text):

    try:
        jira_pass = "who is your daddy?"

        jira_server = JIRA(server=_JIRA_SERVER_ADDR, basic_auth=(_JIRA_USER, jira_pass))
    except Exception as e:
        logging.error(
            "Exception occurred when trying to connect to jira server, check connection?"
        )
        logging.error(traceback.format_exc())
        return False
    else:
        logging.info("Connection to jira & kms server success,  Congratulations")

    for issue in issues:
        logging.info(f"Get one jira issue: {issue}")
        try:
            jira_issue = jira_server.issue(issue.upper())
        except JIRAError as e:
            logging.error(
                f"Does the {issue} really exists? Exception {e.status_code}:{e.text} met "
            )
            return False
        else:
            logging.info(f"Find JIRA issue '{issue}'")

    return True
...
```

#### 如何引入python的外部依赖库

接下来看下requirement部分怎么实现，在WORKSPACE里面写入如下的内容

<pre><code><strong>load("@rules_python//python:pip.bzl", "pip_parse")
</strong>
# Create a central repo that knows about the dependencies needed from
# requirements_lock.txt.
pip_parse(
   name = "my_deps",
   requirements_lock = "//path/to:requirements_lock.txt",
)
# Load the starlark macro which will define your dependencies.
load("@my_deps//:requirements.bzl", "install_deps")
# Call it to define repos for your requirements.
install_deps()
</code></pre>

requirements\_lock.txt内部只需要写入依赖的python包即可，这里我们写入jira就行了

```

# requirements.txt for Python3 packages
jira

```



最后直接bazel build对应的target即可了
