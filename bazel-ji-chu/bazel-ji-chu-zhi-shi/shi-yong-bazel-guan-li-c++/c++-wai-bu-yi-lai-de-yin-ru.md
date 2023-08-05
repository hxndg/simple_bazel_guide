# C++外部依赖的引入

### 引言

就C++而言，外部依赖的引入是一个比较大的问题。

### 为什么需要阅读这一页

对于使用C++的很多用户而言，最主要的问题往往是因为外部依赖引起，一些外部依赖包有对应的BUILD文件，直接引用即可，另一部分的外部依赖使用诸如CMake, configure-make, GNU Make, boost, ninja, Meson等构件系统没有BUILD文件，怎么办呢？



### 内容

现在的很多软件同时支持诸如Cmake或者Bazel的构件系统，如果引入的第三方依赖有BUILD文件，已经有了就可以直接引入使用了



#### 方法1 直接找到对应的实现

很多著名的库实际上要么本身支持了bazel，要么有人已经写好了对应的bazel规则。这个最直接的例子就是boost了，参考链接为[https://github.com/nelhage/rules\_boost](https://github.com/nelhage/rules\_boost)，直接在WORKSPACE文件里面写入下面的内容。PS 记得更新url和sha256来使用新一些的版本。

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Boost
# Famous C++ library that has given rise to many new additions to the C++ Standard Library
# Makes @boost available for use: For example, add `@boost//:algorithm` to your deps.
# For more, see https://github.com/nelhage/rules_boost and https://www.boost.org
http_archive(
    name = "com_github_nelhage_rules_boost",

    # Replace the commit hash in both places (below) with the latest, rather than using the stale one here.
    # Even better, set up Renovate and let it do the work for you (see "Suggestion: Updates" in the README).
    url = "https://github.com/nelhage/rules_boost/archive/96e9b631f104b43a53c21c87b01ac538ad6f3b48.tar.gz",
    strip_prefix = "rules_boost-96e9b631f104b43a53c21c87b01ac538ad6f3b48",
    # When you first run this tool, it'll recommend a sha256 hash to put here with a message like: "DEBUG: Rule 'com_github_nelhage_rules_boost' indicated that a canonical reproducible form can be obtained by modifying arguments sha256 = ..."
)
load("@com_github_nelhage_rules_boost//:boost/boost.bzl", "boost_deps")
boost_deps()
```



写代码的时候如果想使用boost对应的库，就可以在对应的规则的deps里面加上对boost依赖即可@boost即可。举个简单例子，比方说我要在一个C++ library里面使用boost的算法库，就直接在在deps里面加上@boost//:algorithm

如果你是用的是std17的C++，那么可能一部分功能直接用标准库即可，不一定要引用boost。

另一个例子是rules\_folly，这个是（前）同事，也是工程治理大佬写的，参考链接为[https://github.com/storypku/rules\_folly](https://github.com/storypku/rules\_folly)

按照github链接的方式，安装好必要的依赖，下面是ubuntu的例子

```
sudo apt-get update \
    && sudo apt-get -y install --no-install-recommends \
    autoconf \
    automake \
    libtool \
    libssl-dev
```

接着在WORKSPACE文件里面添加好对应的依赖，使用

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "com_github_storypku_rules_folly",
    sha256 = "16441df2d454a6d7ef4da38d4e5fada9913d1f9a3b2015b9fe792081082d2a65",
    strip_prefix = "rules_folly-0.2.0",
    urls = [
        "https://github.com/storypku/rules_folly/archive/v0.2.0.tar.gz",
    ],
)

load("@com_github_storypku_rules_folly//bazel:folly_deps.bzl", "folly_deps")
folly_deps()

load("@com_github_nelhage_rules_boost//:boost/boost.bzl", "boost_deps")
boost_deps()
```

这个库有一个点可能需要注意，

#### 方法2 手动改写BUILD文件

手动改写BUILD文件，需要用户自己能够看明白Cmake的构建稳健是怎么生效的，有的时候做编译还得去考虑native编译工具链的事情，总之是个比较麻烦的事情。但也不失为一种办法。这种方法只需要注意：

* 首先需要用http\_archive的方式，将第三方库下载下来
* 其次，用指定Build的方式，指定为自己编写的BUILD文件

给一个简单的例子，将hiredis转换为bazel的库，这里面我就不写指定Build的部分了，只简单写写怎么写BUILD文件。

首先看hiredis的cmakelists.txt文件，重点就是找到对应的SRC和对应的头文件，确定哪些应该暴露，哪些不应该暴露。除此之外，还要关注一些编译选项的东西，构建应该是统一的一套，即依赖hiredis的库编译用的是release模式，那么hiredis也是release模式。

```
# Hiredis requires C99
SET(CMAKE_C_STANDARD 99)    #C99标准，添加到copt里面即可
SET(CMAKE_POSITION_INDEPENDENT_CODE ON)    #fpic标记，编译静态库时需要加上，这样子才能生成的静态库被第三方引用
SET(CMAKE_DEBUG_POSTFIX d)

SET(hiredis_sources   #下面的.c文件对应于bazel src文件，我们可以看到目前的代码实际上是包含sync和async的，
    alloc.c
    async.c
    dict.c
    hiredis.c
    net.c
    read.c
    sds.c
    sockcompat.c)

SET(hiredis_sources ${hiredis_sources})

...

ADD_LIBRARY(hiredis SHARED ${hiredis_sources})     #要生成这两个库文件
ADD_LIBRARY(hiredis_static STATIC ${hiredis_sources})

SET_TARGET_PROPERTIES(hiredis
    PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE
    VERSION "${HIREDIS_SONAME}")
SET_TARGET_PROPERTIES(hiredis_static
    PROPERTIES COMPILE_PDB_NAME hiredis_static)
SET_TARGET_PROPERTIES(hiredis_static
    PROPERTIES COMPILE_PDB_NAME_DEBUG hiredis_static${CMAKE_DEBUG_POSTFIX})


#INSTALL_INTERFACE用于给install的时候指定使用的引用文件，那么INSTALL_INTERFACE又是怎么指定的？看这个https://ravenxrz.ink/archives/e40194d1.html
TARGET_INCLUDE_DIRECTORIES(hiredis PUBLIC $<INSTALL_INTERFACE:include> $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)
TARGET_INCLUDE_DIRECTORIES(hiredis_static PUBLIC $<INSTALL_INTERFACE:include> $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)

#CONFIGURE_FILE替换原本的普通文件的内容，给pkg用的，不需要了，所以去掉
CONFIGURE_FILE(hiredis.pc.in hiredis.pc @ONLY)

...

#Cpack打包的内容，直接省略了，关系并不大
...

IF(ENABLE_SSL)
    IF (NOT OPENSSL_ROOT_DIR)  #这个是export的openssl根目录，用来判断找openssl
        IF (APPLE)
            SET(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
        ENDIF()
    ENDIF()
    FIND_PACKAGE(OpenSSL REQUIRED)  #找到依赖的openssl库
    SET(hiredis_ssl_sources         #编译hiredis_ssl库的源文件
        ssl.c)
    ADD_LIBRARY(hiredis_ssl SHARED  
            ${hiredis_ssl_sources}) #设定生成的库文件
    ADD_LIBRARY(hiredis_ssl_static STATIC
            ${hiredis_ssl_sources})

	...

    SET_TARGET_PROPERTIES(hiredis_ssl_static
        PROPERTIES COMPILE_PDB_NAME hiredis_ssl_static)
    SET_TARGET_PROPERTIES(hiredis_ssl_static
        PROPERTIES COMPILE_PDB_NAME_DEBUG hiredis_ssl_static${CMAKE_DEBUG_POSTFIX})

    TARGET_INCLUDE_DIRECTORIES(hiredis_ssl PRIVATE "${OPENSSL_INCLUDE_DIR}")  #引入openssl包裹的头文件，只给自己用。不会暴露给hiredis_ssl的使用者
    TARGET_INCLUDE_DIRECTORIES(hiredis_ssl_static PRIVATE "${OPENSSL_INCLUDE_DIR}")

    TARGET_LINK_LIBRARIES(hiredis_ssl PRIVATE ${OPENSSL_LIBRARIES})
...

    INSTALL(TARGETS hiredis_ssl hiredis_ssl_static
        EXPORT hiredis_ssl-targets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}    #安装bin文件的位置
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}    #安装动态库的位置
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})   #安装静态库的位置

...

    INSTALL(FILES hiredis_ssl.h   
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/hiredis)        #可以看到安装头文件的位置，将hiredis_ssl.h搞到了安装头文件hiredis里面

    INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/hiredis_ssl.pc    #提供给pkg安装用的，不用关注
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

    export(EXPORT hiredis_ssl-targets
           FILE "${CMAKE_CURRENT_BINARY_DIR}/hiredis_ssl-targets.cmake"
           NAMESPACE hiredis::)

    SET(CMAKE_CONF_INSTALL_DIR share/hiredis_ssl)
    ...
ENDIF()

...
```

按照上面的cmakelist里面写的注释的解析过程，最终得出BUILD文件如下。这个是非SSL的版本，SSL的版本也非常简单，就不写了。

```
cc_library(
    name = "hiredis",
    srcs = [
        "alloc.c",
        "dict.c",
        "async.c",
        "hiredis.c",
        "net.c",
        "read.c",
        "sds.c",
        "sockcompat.c",
    ],
    hdrs = glob(["*.h"])+glob(["adapters/*.h"])+["dict.c",],  #dict.c的引入是因为被async.c引了，而adapters是编译async时候别的库要使用
    include_prefix = "hiredis",
    visibility = ["//visibility:public"],
)
```



#### 方法3 rules\_foreign\_cc

rules\_foreign\_cc好用吗？好用，但是没有充分测试，且不被bazel的官方认可在我看来问题就比较大了，所以我个人不是特别喜欢rules\_foreign\_cc这套，不过也是个用法。

我个人拿rules\_foreign\_cc做测试比较早，所以下面的内容我推荐用最新版，直接参考[https://bazelbuild.github.io/rules\_foreign\_cc/main/index.html](https://bazelbuild.github.io/rules\_foreign\_cc/main/index.html)openssl和libuv做例子

首先需要配置引入rules\_foreign\_cc，并且下载openssl和libuv的源代码

```
workspace(name = "cmake2bazel")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
   name = "rules_foreign_cc",
   strip_prefix = "rules_foreign_cc-4010620160e0df4d894b61496d3d3b6fc8323212",
    sha256 = "07e3414cc841b1f4d16e5231eb818e5c5e03e2045827f5306a55709e5045c7fd",
   url = "https://github.com/bazelbuild/rules_foreign_cc/archive/4010620160e0df4d894b61496d3d3b6fc8323212.zip",
)

load("@rules_foreign_cc//foreign_cc:repositories.bzl", "rules_foreign_cc_dependencies")
rules_foreign_cc_dependencies()

all_content = """filegroup(name = "all", srcs = glob(["**"]), visibility = ["//visibility:public"])"""

# openssl
http_archive(
    name = "openssl",
    build_file_content = all_content,
    strip_prefix = "openssl-OpenSSL_1_1_1d",
    urls = ["https://github.com/openssl/openssl/archive/OpenSSL_1_1_1d.tar.gz"]
)
all_content = """filegroup(name = "all", srcs = glob(["**"]), visibility = ["//visibility:public"])"""

# openssl
http_archive(
    name = "openssl",
    build_file_content = all_content,
    strip_prefix = "openssl-OpenSSL_1_1_1d",
    urls = ["https://github.com/openssl/openssl/archive/OpenSSL_1_1_1d.tar.gz"]
)
# uv
http_archive(
    name = "libuv",
    build_file_content = all_content,
    strip_prefix = "libuv-1.42.0",
    urls = ["https://github.com/libuv/libuv/archive/refs/tags/v1.42.0.tar.gz"]

)
```

新建一个文件，为thirdy\_party/openssl/BUILD

```
# See https://github.com/bazelbuild/rules_foreign_cc
load("@rules_foreign_cc//foreign_cc:defs.bzl", "configure_make")

config_setting(
    name = "darwin_build",
    values = {"cpu": "darwin"},
)

# See https://wiki.openssl.org/index.php/Compilation_and_Installation
# See https://github.com/bazelbuild/rules_foreign_cc/issues/338
#可以通过指定out_lib_dir选项指定编译出来的lib放在哪里，aka The path to where the compiled library binaries will be written to following a successful build
#对于使用configure-make形式的代码编译的方式，
configure_make(
    name = "openssl",
#实际上调用configure的命令，默认是调用configure，这里可以找到openssl里面调用的是config
    configure_command = "config",
#Any options to be put on the 'configure' command line.
    configure_options =
      select({
            ":darwin_build": [
              "shared",
              "ARFLAGS=r",
              "enable-ec_nistp_64_gcc_128",
              "no-ssl2", "no-ssl3", "no-comp"
            ],
            "//conditions:default": [
            ]}),
    #defines = ["NDEBUG"], Don't know how to use -D; NDEBUG seems to be the default anyway
#指定OPENSSL编译lib的源代码文件，aka Where the library source code is for openssl
    lib_source = "@openssl//:all",               
    visibility = ["//visibility:public"],
#Environment variables to be set for the 'configure' invocation.
    configure_env_vars =
        select({
            ":darwin_build": {
              "OSX_DEPLOYMENT_TARGET": "10.14",
              "AR": "",
            },
            "//conditions:default": {}}),
#用来指定共享出来的动态库、动态文件是什么，可以使用static_libraries属性来共享动态库
    out_shared_libs =
        select({
            ":darwin_build": [
                "libssl.dylib",
                "libcrypto.dylib",
            ],
            "//conditions:default": [
                "libssl.so",
                "libcrypto.so",
            ],
        })
)
```

接着新建一个文件为third\_party/libuv/BUILD文件

```
# See https://github.com/bazelbuild/rules_foreign_cc
load("@rules_foreign_cc//foreign_cc:defs.bzl", "cmake")

cmake(
    name = "libuv",
    lib_source = "@libuv//:all",
    #out_static_libs = ["libuv.a"],
    out_static_libs = ["libuv_a.a"],
    #out_shared_libs = ["libuv.so.1.0.0"],  libuv.so是个链接，直接编译会报错
)
```

最后可以编译一下试试

```
qcraft@BJ-HeXiaonan:~/code_test/cmake2bazel$ bazelisk-linux-amd64 build //third_party/libuv:libuv
DEBUG: Rule 'libuv' indicated that a canonical reproducible form can be obtained by modifying arguments sha256 = "371e5419708f6aaeb8656671f89400b92a9bba6443369af1bb70bcd6e4b3c764"
DEBUG: Repository libuv instantiated at:
  /home/qcraft/code_test/cmake2bazel/WORKSPACE:36:13: in <toplevel>
Repository rule http_archive defined at:
  /home/qcraft/.cache/bazel/_bazel_qcraft/66cfb4dff202f299686aa7bc701960fa/external/bazel_tools/tools/build_defs/repo/http.bzl:336:31: in <toplevel>
INFO: Analyzed target //third_party/libuv:libuv (1 packages loaded, 1 target configured).
INFO: Found 1 target...
Target //third_party/libuv:libuv up-to-date:
  bazel-bin/third_party/libuv/libuv/include
  bazel-bin/third_party/libuv/libuv/lib/libuv_a.a
  bazel-bin/third_party/libuv/copy_libuv/libuv
INFO: Elapsed time: 31.269s, Critical Path: 30.97s
INFO: 2 processes: 1 internal, 1 linux-sandbox.
INFO: Build completed successfully, 2 total actions
```
