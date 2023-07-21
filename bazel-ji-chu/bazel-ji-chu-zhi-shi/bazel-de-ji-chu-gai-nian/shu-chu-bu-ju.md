# 输出布局

### 引言

依然是推荐阅读原始文档[https://bazel.build/remote/output-directories](https://bazel.build/remote/output-directories)。



### 我为什么要阅读这一页？

对于普通研发用户而言，输出布局一般是固定的，他们常常遇到的问题都是权限错误的问题，可以通过对输出布局的理解来定位具体是什么原因导致的问题，最简单的解决方法就是chown。

对Infra研发而言，输出布局需要根据业务需要做变化，尤其在CI环境下，比方说输出目录，CACHE在哪里。还有如何排查一些确定性的问题的时候输出布局知识就体现其重要性



### 输出布局

目前Bazel的文件布局是怎么实现的呢？

* Bazel命令行必须在WORKSPACE文件所在的目录，或者子目录调用
* Linux默认的_outputRoot_目录设置为`~/.cache/bazel`&#x20;
* Bazel用户编译状态位于 `outputRoot/_bazel_$USER`.&#x20;
* 在 `outputUserRoot` 目录有个 `install` 目录，里面放了一堆MD5的编译产物文件
* 在 `outputUserRoot` 目录, an `outputBase` 目录会根据workspace directory的MD5创建 .比方说workspace路径为 `/home/user/src/my-project` (or in a directory symlinked to that one), 就会创建`/home/user/.cache/bazel/_bazel_user/7ffd56a6e4cb724ea575aba15733d113`. 目录。
* 通过配置Bazel's `--output_base` 启动选项来覆盖默认的output base 目录,举个例子 `bazel --output_base=/tmp/bazel/output build x/y:z`.
* 通过配置Bazel's `--output_user_root` 启动选项来覆盖默认的install base 和 output base 目录，比方说`bazel --output_user_root=/tmp/bazel build x/y:z`.



```
<workspace-name>/                         <== workspace文件路径
  bazel-my-project => <...my-project>     <== Symlink to execRoot
  bazel-out => <...bin>                   <== Convenience symlink to outputPath
  bazel-bin => <...bin>                   <== Convenience symlink to most recent written bin dir $(BINDIR)
  bazel-testlogs => <...testlogs>         <== Convenience symlink to the test logs directory

/home/user/.cache/bazel/                  <== Root for all Bazel output on a machine: outputRoot
  _bazel_$USER/                           <== Top level directory for a given user depends on the user name:
                                              outputUserRoot
    install/
      fba9a2c87ee9589d72889caf082f1029/   <== Hash of the Bazel install manifest: installBase
        _embedded_binaries/               <== Contains binaries and scripts unpacked from the data section of
                                              the bazel executable on first run (such as helper scripts and the
                                              main Java file BazelServer_deploy.jar)
    7ffd56a6e4cb724ea575aba15733d113/     <== Hash of the client's workspace directory (such as
                                              /home/user/src/my-project): outputBase
      action_cache/                       <== Action cache directory hierarchy
                                              This contains the persistent record of the file
                                              metadata (timestamps, and perhaps eventually also MD5
                                              sums) used by the FilesystemValueChecker.
      action_outs/                        <== Action output directory. This contains a file with the
                                              stdout/stderr for every action from the most recent
                                              bazel run that produced output.
      command.log                         <== A copy of the stdout/stderr output from the most
                                              recent bazel command.
      external/                           <== The directory that remote repositories are
                                              downloaded/symlinked into.
      server/                             <== The Bazel server puts all server-related files (such
                                              as socket file, logs, etc) here.
        jvm.out                           <== The debugging output for the server.
      execroot/                           <== The working directory for all actions. For special
                                              cases such as sandboxing and remote execution, the
                                              actions run in a directory that mimics execroot.
                                              Implementation details, such as where the directories
                                              are created, are intentionally hidden from the action.
                                              Every action can access its inputs and outputs relative
                                              to the execroot directory.
        <workspace-name>/                 <== Working tree for the Bazel build & root of symlink forest: execRoot
          _bin/                           <== Helper tools are linked from or copied to here.

          bazel-out/                      <== All actual output of the build is under here: outputPath
            local_linux-fastbuild/        <== one subdirectory per unique target BuildConfiguration instance;
                                              this is currently encoded
              bin/                        <== Bazel outputs binaries for target configuration here: $(BINDIR)
                foo/bar/_objs/baz/        <== Object files for a cc_* rule named //foo/bar:baz
                  foo/bar/baz1.o          <== Object files from source //foo/bar:baz1.cc
                  other_package/other.o   <== Object files from source //other_package:other.cc
                foo/bar/baz               <== foo/bar/baz might be the artifact generated by a cc_binary named
                                              //foo/bar:baz
                foo/bar/baz.runfiles/     <== The runfiles symlink farm for the //foo/bar:baz executable.
                  MANIFEST
                  <workspace-name>/
                    ...
              genfiles/                   <== Bazel puts generated source for the target configuration here:
                                              $(GENDIR)
                foo/bar.h                     such as foo/bar.h might be a headerfile generated by //foo:bargen
              testlogs/                   <== Bazel internal test runner puts test log files here
                foo/bartest.log               such as foo/bar.log might be an output of the //foo:bartest test with
                foo/bartest.status            foo/bartest.status containing exit status of the test (such as
                                              PASSED or FAILED (Exit 1), etc)
              include/                    <== a tree with include symlinks, generated as needed. The
                                              bazel-include symlinks point to here. This is used for
                                              linkstamp stuff, etc.
            host/                         <== BuildConfiguration for build host (user's workstation), for
                                              building prerequisite tools, that will be used in later stages
                                              of the build (ex: Protocol Compiler)
        <packages>/                       <== Packages referenced in the build appear as if under a regular workspace
```

