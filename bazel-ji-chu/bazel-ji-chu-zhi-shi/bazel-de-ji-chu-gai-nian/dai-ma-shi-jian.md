# 代码实践

### 引言

俗话说“知行合一”，没有了实践，仅有理论或者思维的前行是不能得到正确的结论的，下面开始实际的写一套代码，来理解bazel的实际运用

### 为什么要阅读这一部分

这一部分是可以不看的，我建议读者直接用类似的方法自己在本地写一些代码，试试就行

### 内容

#### 创建WORKSPACE文件

开始利用bazel体系构建一个继承C++,Golang,Rust,Python的monorepo，该仓库需要支持grpc,glog,absiel,boost,gtest等外部仓库，我将这个仓库命名为cyber\_security。

新建一个folder，写入下面的内容到WORKSPACE文件，这里我们先引入llvm c++ & grpc的支持，写一个grpc的同步模式服务器。下面的代码可以看到注释掉了protobuf的引入（因为有个bug还没解决），引入了grpc和llvm toolchain。

我之所以不使用llvm 16是因为在mac m1上会报错，所以使用的是15的llvm版本

```
workspace(name = "cyber_security")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# portable llvm toolchain

BAZEL_TOOLCHAIN_TAG = "0.8.2"
BAZEL_TOOLCHAIN_SHA = "0fc3a2b0c9c929920f4bed8f2b446a8274cad41f5ee823fd3faa0d7641f20db0"

http_archive(
    name = "com_grail_bazel_toolchain",
    sha256 = BAZEL_TOOLCHAIN_SHA,
    strip_prefix = "bazel-toolchain-{tag}".format(tag = BAZEL_TOOLCHAIN_TAG),
    canonical_id = BAZEL_TOOLCHAIN_TAG,
    url = "https://github.com/grailbio/bazel-toolchain/archive/refs/tags/{tag}.tar.gz".format(tag = BAZEL_TOOLCHAIN_TAG),
)

load("@com_grail_bazel_toolchain//toolchain:deps.bzl", "bazel_toolchain_dependencies")

bazel_toolchain_dependencies()

load("@com_grail_bazel_toolchain//toolchain:rules.bzl", "llvm_toolchain")

llvm_toolchain(
    name = "llvm_toolchain",
    llvm_version = "15.0.0",
)

load("@llvm_toolchain//:toolchains.bzl", "llvm_register_toolchains")

llvm_register_toolchains()

# protobuf and grpc section
# http_archive(
#     name = "rules_proto",
#     sha256 = "dc3fb206a2cb3441b485eb1e423165b231235a1ea9b031b4433cf7bc1fa460dd",
#     strip_prefix = "rules_proto-5.3.0-21.7",
#     urls = [
#         "https://github.com/bazelbuild/rules_proto/archive/refs/tags/5.3.0-21.7.tar.gz",
#     ],
# )
# load("@rules_proto//proto:repositories.bzl", "rules_proto_dependencies", "rules_proto_toolchains")
# rules_proto_dependencies()
# rules_proto_toolchains()

http_archive(
    name = "com_github_grpc_grpc",
    strip_prefix = "grpc-1.57.0",
    sha256 = "8393767af531b2d0549a4c26cf8ba1f665b16c16fb6c9238a7755e45444881dd",
    urls = ["https://github.com/grpc/grpc/archive/refs/tags/v1.57.0.tar.gz"],
)
 
load("@com_github_grpc_grpc//bazel:grpc_deps.bzl", "grpc_deps")
grpc_deps()
 
load("@com_github_grpc_grpc//bazel:grpc_extra_deps.bzl", "grpc_extra_deps")
grpc_extra_deps()
```

#### 编写BUILD文件和代码

环境初始化好了以后新建proto文件和BUILD文件

<pre><code># experimental/sync_grpc_server/protos/stream.proto
syntax="proto3";

package stream;

// The service definition.
service Parser {
  // Sends a request
  rpc SendRequest (Request) returns (Response) {}
}

message Request {
    bytes client_ip = 1;
}

message Response {
    bytes event_id = 1;
}

<strong># experimental/sync_grpc_server/protos/BUILD
</strong>```starlark
package(default_visibility = ["//visibility:public"])

load("@rules_proto//proto:defs.bzl", "proto_library")
load("@rules_cc//cc:defs.bzl", "cc_proto_library")
load("@com_github_grpc_grpc//bazel:cc_grpc_library.bzl", "cc_grpc_library")

# The following three rules demonstrate the usage of the cc_grpc_library rule in
# in a mode compatible with the native proto_library and cc_proto_library rules.
proto_library(
    name = "stream_proto",
    srcs = ["stream.proto"],
)

cc_proto_library(
    name = "stream_cc_proto",
    deps = [":stream_proto"],
)

cc_grpc_library(
    name = "stream_cc_grpc",
    srcs = [":stream_proto"],
    grpc_only = True,
    deps = [":stream_cc_proto"],
)

```
</code></pre>

接着新建server和client的代码

````
#experimental/sync_grpc_server/src/sync_client.cc
```cpp
#include <iostream>
#include <memory>
#include <string>

#include "grpcpp/grpcpp.h"

#ifdef BAZEL_BUILD
#include "experimental/sync_grpc_server/protos/stream.grpc.pb.h"
#else
#include "stream.grpc.pb.h"
#endif

using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;
using stream::Parser;
using stream::Response;
using stream::Request;

class ParserClient {
 public:
  ParserClient(std::shared_ptr<Channel> channel)
      : stub_(Parser::NewStub(channel)) {}

  // Assembles the client's payload, sends it and presents the response back
  // from the server.
  std::string SendRequest() {
    // Data we are sending to the server.
    Request request;
    request.set_client_ip("127.0.0.1");

    // Container for the data we expect from the server.
    Response response;

    // Context for the client. It could be used to convey extra information to
    // the server and/or tweak certain RPC behaviors.
    ClientContext context;

    // The actual RPC.
    Status status = stub_->SendRequest(&context, request, &response);

    // Act upon its status.
    if (status.ok()) {
      return std::string(response.event_id());
    } else {
      std::cout << status.error_code() << ": " << status.error_message() << std::endl;
      return "RPC failed";
    }
  }

 private:
  std::unique_ptr<Parser::Stub> stub_;
};

int main(int argc, char** argv) {
  std::string address = "localhost";
  std::string port = "50051";
  std::string server_address = address + ":" + port;
  std::cout << "Client querying server address: " << server_address << std::endl;


  // Instantiate the client. It requires a channel, out of which the actual RPCs
  // are created. This channel models a connection to an endpoint (in this case,
  // localhost at port 50051). We indicate that the channel isn't authenticated
  // (use of InsecureChannelCredentials()).
  ParserClient Parser(grpc::CreateChannel(
      server_address, grpc::InsecureChannelCredentials()));

  std::string response = Parser.SendRequest();
  std::cout << "Parser received: " << response << std::endl;

  return 0;
}
```
#experimental/sync_grpc_server/src/sync_server.cc
```cpp
#include <iostream>
#include <memory>
#include <string>

#include <grpcpp/grpcpp.h>

#ifdef BAZEL_BUILD
#include "experimental/sync_grpc_server/protos/stream.grpc.pb.h"
#else
#include "stream.grpc.pb.h"
#endif

using grpc::Server;
using grpc::ServerBuilder;
using grpc::ServerContext;
using grpc::Status;
using stream::Request;
using stream::Response;
using stream::Parser;

// Logic and data behind the server's behavior.
class ParserServiceImpl final : public Parser::Service {
  Status SendRequest(ServerContext* context, const Request* request,
                  Response* response) override {
    response->set_event_id("47F1F2FF-7679-4378-ACC7-051F72D5679A");
    return Status::OK;
  }
};

void RunServer() {
  std::string address = "0.0.0.0";
  std::string port = "50051";
  std::string server_address = address + ":" + port;
  ParserServiceImpl service;

  ServerBuilder builder;
  // Listen on the given address without any authentication mechanism.
  builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
  // Register "service" as the instance through which we'll communicate with
  // clients. In this case it corresponds to an *synchronous* service.
  builder.RegisterService(&service);
  // Finally assemble the server.
  std::unique_ptr<Server> server(builder.BuildAndStart());
  std::cout << "Server listening on " << server_address << std::endl;

  // Wait for the server to shutdown. Note that some other thread must be
  // responsible for shutting down the server for this call to ever return.
  server->Wait();
}

int main(int argc, char** argv) {
  RunServer();

  return 0;
}
```
#experimental/sync_grpc_server/src/BUILD
```starlark
package(default_visibility = ["//visibility:public"])

load("@rules_cc//cc:defs.bzl", "cc_binary")

cc_binary(
    name = "sync_client",
    srcs = ["sync_client.cc"],
    defines = ["BAZEL_BUILD"],
    deps = [
        "//experimental/sync_grpc_server/protos:stream_cc_grpc",
        "@com_github_grpc_grpc//:grpc++",
    ],
)

cc_binary(
    name = "sync_server",
    srcs = ["sync_server.cc"],
    defines = ["BAZEL_BUILD"],
    deps = [
        "//experimental/sync_grpc_server/protos:stream_cc_grpc",
        "@com_github_grpc_grpc//:grpc++",
    ],
)
```
````



#### 运行

<pre><code><strong># 窗口1 运行server
</strong><strong>bazel run experimental/sync_grpc_server/src:sync_server
</strong>
# 窗口2 运行client
bazel run experimental/sync_grpc_server/src:sync_client

#输出结果
➜  cyber_security git:(main) bazel run experimental/sync_grpc_server/src:sync_client
INFO: Analyzed target //experimental/sync_grpc_server/src:sync_client (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //experimental/sync_grpc_server/src:sync_client up-to-date:
  bazel-bin/experimental/sync_grpc_server/src/sync_client
INFO: Elapsed time: 0.168s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/experimental/sync_grpc_server/src/sync_client
Client querying server address: localhost:50051
Parser received: 47F1F2FF-7679-4378-ACC7-051F72D5679A
</code></pre>
