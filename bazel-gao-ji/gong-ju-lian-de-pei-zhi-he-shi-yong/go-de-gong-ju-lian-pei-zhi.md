# go的工具链配置

参考[https://github.com/bazelbuild/rules\_go/blob/master/go/private/go\_toolchain.bzl](https://github.com/bazelbuild/rules\_go/blob/master/go/private/go\_toolchain.bzl)，来看看GO的Toolchain是怎么写的。

这部分我建议就不要看我废话了，因为我自己都没搞清楚。。。。

文件为`rules_go/go/private/go_toolchain.bzl`,可以看到定义了几个关键属性，包括builder,goos,goarch,sdk等。这些属性被揉到一起以后，交给\_go\_toolchain\_impl，这个内部调用platform\_common.ToolchainInfo。说白了，这些属性就是提交给内部的规则索要使用的属性。

<pre><code># Copyright 2016 The Bazel Go Rules Authors. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
"""
Toolchain rules used by go.
"""

load("//go/private:platforms.bzl", "PLATFORMS")
load("//go/private:providers.bzl", "GoSDK")
load("//go/private/actions:archive.bzl", "emit_archive")
load("//go/private/actions:binary.bzl", "emit_binary")
load("//go/private/actions:link.bzl", "emit_link")
load("//go/private/actions:stdlib.bzl", "emit_stdlib")
load("@bazel_skylib//lib:selects.bzl", "selects")

GO_TOOLCHAIN = "@io_bazel_rules_go//go:toolchain"

def _go_toolchain_impl(ctx):
    sdk = ctx.attr.sdk[GoSDK]
    cross_compile = ctx.attr.goos != sdk.goos or ctx.attr.goarch != sdk.goarch
    return [platform_common.ToolchainInfo(
        # Public fields
        name = ctx.label.name,
        cross_compile = cross_compile,
        default_goos = ctx.attr.goos,
        default_goarch = ctx.attr.goarch,
        actions = struct(
            archive = emit_archive,
            binary = emit_binary,
            link = emit_link,
            stdlib = emit_stdlib,
        ),
        flags = struct(
            compile = (),
            link = ctx.attr.link_flags,
            link_cgo = ctx.attr.cgo_link_flags,
        ),
        sdk = sdk,

        # Internal fields -- may be read by emit functions.
        _builder = ctx.executable.builder,
    )]

go_toolchain = rule(
    _go_toolchain_impl,
    attrs = {
        # Minimum requirements to specify a toolchain
        "builder": attr.label(
            mandatory = True,
            cfg = "exec",
            executable = True,
            doc = "Tool used to execute most Go actions",
        ),
        "goos": attr.string(
            mandatory = True,
            doc = "Default target OS",
        ),
        "goarch": attr.string(
            mandatory = True,
            doc = "Default target architecture",
        ),
        "sdk": attr.label(
            mandatory = True,
            providers = [GoSDK],
            cfg = "exec",
            doc = "The SDK this toolchain is based on",
        ),
        # Optional extras to a toolchain
        "link_flags": attr.string_list(
            doc = "Flags passed to the Go internal linker",
        ),
        "cgo_link_flags": attr.string_list(
            doc = "Flags passed to the external linker (if it is used)",
        ),
    },
    doc = "Defines a Go toolchain based on an SDK",
    provides = [platform_common.ToolchainInfo],
)

<strong>...
</strong></code></pre>

platform\_common的含义啥的参考[https://bazel.build/rules/lib/toplevel/platform\_common](https://bazel.build/rules/lib/toplevel/platform\_common)

好的，现在toolchain明确地指明了诸如builder，是否跨平台，sdk等具体规则要用到的东西，我们看看go的规则里面到底是怎么使用的，走到go/private/rules/binary.bzl继续浏览，

go\_binary先走到go\_binary\_macro，再走到go/private/rules/binary.bzl里面的go\_binary里面，再往里面看是go/private/context.bzl里面的go的context来真正地实现对toolchain的实现，这个context我理解实际上是go native编译的真正实现

````


def _go_binary_impl(ctx):
    """go_binary_impl emits actions for compiling and linking a go executable."""
    go = go_context(ctx)

    is_main = go.mode.link not in (LINKMODE_SHARED, LINKMODE_PLUGIN)
    library = go.new_library(go, importable = False, is_main = is_main)
    source = go.library_to_source(go, ctx.attr, library, ctx.coverage_instrumented())
    name = ctx.attr.basename
    if not name:
        name = ctx.label.name
    executable = None
    if ctx.attr.out:
        # Use declare_file instead of attr.output(). When users set output files
        # directly, Bazel warns them not to use the same name as the rule, which is
        # the common case with go_binary.
        executable = ctx.actions.declare_file(ctx.attr.out)
    archive, executable, runfiles = go.binary(
        go,
        name = name,
        source = source,
        gc_linkopts = gc_linkopts(ctx),
        version_file = ctx.version_file,
        info_file = ctx.info_file,
        executable = executable,
    )

    providers = [
        library,
        source,
        archive,
        OutputGroupInfo(
            cgo_exports = archive.cgo_exports,
            compilation_outputs = [archive.data.file],
        ),
    ]

    if go.mode.link in LINKMODES_EXECUTABLE:
        env = {}
        for k, v in ctx.attr.env.items():
            env[k] = ctx.expand_location(v, ctx.attr.data)
        providers.append(RunEnvironmentInfo(environment = env))

        # The executable is automatically added to the runfiles.
        providers.append(DefaultInfo(
            files = depset([executable]),
            runfiles = runfiles,
            executable = executable,
        ))
    else:
        # Workaround for https://github.com/bazelbuild/bazel/issues/15043
        # As of Bazel 5.1.1, native rules do not pick up the "files" of a data
        # dependency's DefaultInfo, only the "data_runfiles". Since transitive
        # non-data dependents should not pick up the executable as a runfile
        # implicitly, the deprecated "default_runfiles" and "data_runfiles"
        # constructor parameters have to be used.
        providers.append(DefaultInfo(
            files = depset([executable]),
            default_runfiles = runfiles,
            data_runfiles = runfiles.merge(ctx.runfiles([executable])),
        ))

    # If the binary's linkmode is c-archive or c-shared, expose CcInfo
    if go.cgo_tools and go.mode.link in (LINKMODE_C_ARCHIVE, LINKMODE_C_SHARED):
        cc_import_kwargs = {
            "linkopts": {
                "darwin": [],
                "ios": [],
                "windows": ["-mthreads"],
            }.get(go.mode.goos, ["-pthread"]),
        }
        cgo_exports = archive.cgo_exports.to_list()
        if cgo_exports:
            header = ctx.actions.declare_file("{}.h".format(name))
            ctx.actions.symlink(
                output = header,
                target_file = cgo_exports[0],
            )
            cc_import_kwargs["hdrs"] = depset([header])
        if go.mode.link == LINKMODE_C_SHARED:
            cc_import_kwargs["dynamic_library"] = executable
        elif go.mode.link == LINKMODE_C_ARCHIVE:
            cc_import_kwargs["static_library"] = executable
            cc_import_kwargs["alwayslink"] = True
        ccinfo = new_cc_import(go, **cc_import_kwargs)
        ccinfo = cc_common.merge_cc_infos(
            cc_infos = [ccinfo, source.cc_info],
        )
        providers.append(ccinfo)

    return providers
。。。

go_binary = rule(executable = True, **_go_binary_kwargs)
go_non_executable_binary = rule(executable = False, **_go_binary_kwargs)

def _go_tool_binary_impl(ctx):
    sdk = ctx.attr.sdk[GoSDK]
    name = ctx.label.name
    if sdk.goos == "windows":
        name += ".exe"

    out = ctx.actions.declare_file(name)
    if sdk.goos == "windows":
        gopath = ctx.actions.declare_directory("gopath")
        gocache = ctx.actions.declare_directory("gocache")
        cmd = "@echo off\nset GOMAXPROCS=1\nset GOCACHE=%cd%\\{gocache}\nset GOPATH=%cd%\\{gopath}\n{go} build -o {out} -trimpath {srcs}".format(
            gopath = gopath.path,
            gocache = gocache.path,
            go = sdk.go.path.replace("/", "\\"),
            out = out.path,
            srcs = " ".join([f.path for f in ctx.files.srcs]),
        )
        bat = ctx.actions.declare_file(name + ".bat")
        ctx.actions.write(
            output = bat,
            content = cmd,
        )
        ctx.actions.run(
            executable = bat,
            inputs = sdk.headers + sdk.tools + sdk.srcs + ctx.files.srcs + [sdk.go],
            outputs = [out, gopath, gocache],
            mnemonic = "GoToolchainBinaryBuild",
        )
    else:
        # Note: GOPATH is needed for Go 1.16.
        cmd = "GOMAXPROCS=1 GOCACHE=$(mktemp -d) GOPATH=$(mktemp -d) {go} build -o {out} -trimpath {srcs}".format(
            go = sdk.go.path,
            out = out.path,
            srcs = " ".join([f.path for f in ctx.files.srcs]),
        )
        ctx.actions.run_shell(
            command = cmd,
            inputs = sdk.headers + sdk.tools + sdk.srcs + sdk.libs + ctx.files.srcs + [sdk.go],
            outputs = [out],
            mnemonic = "GoToolchainBinaryBuild",
        )

    return [DefaultInfo(
        files = depset([out]),
        executable = out,
    )]

go_tool_binary = rule(
    implementation = _go_tool_binary_impl,
    attrs = {
        "srcs": attr.label_list(
            allow_files = True,
            doc = "Source files for the binary. Must be in 'package main'.",
        ),
        "sdk": attr.label(
            mandatory = True,
            providers = [GoSDK],
            doc = "The SDK containing tools and libraries to build this binary",
        ),
    },
    executable = True,
    doc = """Used instead of go_binary for executables used in the toolchain.

go_tool_binary depends on tools and libraries that are part of the Go SDK.
It does not depend on other toolchains. It can only compile binaries that
just have a main package and only depend on the standard library and don't
require build constraints.
""",
)

def gc_linkopts(ctx):
    gc_linkopts = [
        ctx.expand_make_variables("gc_linkopts", f, {})
        for f in ctx.attr.gc_linkopts
    ]
    return gc_linkopts

```
````

go context的核心实现，里面调用到了toolchain的东西

```starlark

def go_context(ctx, attr = None):
    """Returns an API used to build Go code.

    See /go/toolchains.rst#go-context
    """
    if not attr:
        attr = ctx.attr
    toolchain = ctx.toolchains[GO_TOOLCHAIN]
    cgo_context_info = None
    go_config_info = None
    stdlib = None
    coverdata = None
    nogo = None
    if hasattr(attr, "_go_context_data"):
        go_context_data = _flatten_possibly_transitioned_attr(attr._go_context_data)
        if CgoContextInfo in go_context_data:
            cgo_context_info = go_context_data[CgoContextInfo]
        go_config_info = go_context_data[GoConfigInfo]
        stdlib = go_context_data[GoStdLib]
        coverdata = go_context_data[GoContextInfo].coverdata
        nogo = go_context_data[GoContextInfo].nogo
    if getattr(attr, "_cgo_context_data", None) and CgoContextInfo in attr._cgo_context_data:
        cgo_context_info = attr._cgo_context_data[CgoContextInfo]
    if getattr(attr, "cgo_context_data", None) and CgoContextInfo in attr.cgo_context_data:
        cgo_context_info = attr.cgo_context_data[CgoContextInfo]
    if hasattr(attr, "_go_config"):
        go_config_info = attr._go_config[GoConfigInfo]
    if hasattr(attr, "_stdlib"):
        stdlib = _flatten_possibly_transitioned_attr(attr._stdlib)[GoStdLib]

    mode = get_mode(ctx, toolchain, cgo_context_info, go_config_info)
    tags = mode.tags
    binary = toolchain.sdk.go

    if stdlib:
        goroot = stdlib.root_file.dirname
    else:
        goroot = toolchain.sdk.root_file.dirname

    env = {
        "GOARCH": mode.goarch,
        "GOOS": mode.goos,
        "GOEXPERIMENT": ",".join(toolchain.sdk.experiments),
        "GOROOT": goroot,
        "GOROOT_FINAL": "GOROOT",
        "CGO_ENABLED": "0" if mode.pure else "1",

        # If we use --action_env=GOPATH, or in other cases where environment
        # variables are passed through to this builder, the SDK build will try
        # to write to that GOPATH (e.g. for x/net/nettest). This will fail if
        # the GOPATH is on a read-only mount, and is generally a bad idea.
        # Explicitly clear this environment variable to ensure that doesn't
        # happen. See #2291 for more information.
        "GOPATH": "",
    }

    # The level of support is determined by the platform constraints in
    # //go/constraints/amd64.
    # See https://github.com/golang/go/wiki/MinimumRequirements#amd64
    if mode.amd64:
        env["GOAMD64"] = mode.amd64
    if not cgo_context_info:
        crosstool = []
        cgo_tools = None
    else:
        env.update(cgo_context_info.env)
        crosstool = cgo_context_info.crosstool

        # Add C toolchain directories to PATH.
        # On ARM, go tool link uses some features of gcc to complete its work,
        # so PATH is needed on ARM.
        path_set = {}
        if "PATH" in env:
            for p in env["PATH"].split(ctx.configuration.host_path_separator):
                path_set[p] = None
        cgo_tools = cgo_context_info.cgo_tools
        tool_paths = [
            cgo_tools.c_compiler_path,
            cgo_tools.ld_executable_path,
            cgo_tools.ld_static_lib_path,
            cgo_tools.ld_dynamic_lib_path,
        ]
        for tool_path in tool_paths:
            tool_dir, _, _ = tool_path.rpartition("/")
            path_set[tool_dir] = None
        paths = sorted(path_set.keys())
        if ctx.configuration.host_path_separator == ":":
            # HACK: ":" is a proxy for a UNIX-like host.
            # The tools returned above may be bash scripts that reference commands
            # in directories we might not otherwise include. For example,
            # on macOS, wrapped_ar calls dirname.
            if "/bin" not in path_set:
                paths.append("/bin")
            if "/usr/bin" not in path_set:
                paths.append("/usr/bin")
        env["PATH"] = ctx.configuration.host_path_separator.join(paths)

    # TODO(jayconrod): remove this. It's way too broad. Everything should
    # depend on more specific lists.
    sdk_files = ([toolchain.sdk.go] +
                 toolchain.sdk.srcs +
                 toolchain.sdk.headers +
                 toolchain.sdk.libs +
                 toolchain.sdk.tools)

    _check_importpaths(ctx)
    importpath, importmap, pathtype = _infer_importpath(ctx)
    importpath_aliases = tuple(getattr(attr, "importpath_aliases", ()))

    return struct(
        # Fields
        toolchain = toolchain,
        sdk = toolchain.sdk,
        mode = mode,
        root = goroot,
        go = binary,
        stdlib = stdlib,
        sdk_root = toolchain.sdk.root_file,
        sdk_files = sdk_files,
        sdk_tools = toolchain.sdk.tools,
        actions = ctx.actions,
        exe_extension = goos_to_extension(mode.goos),
        shared_extension = goos_to_shared_extension(mode.goos),
        crosstool = crosstool,
        package_list = toolchain.sdk.package_list,
        importpath = importpath,
        importmap = importmap,
        importpath_aliases = importpath_aliases,
        pathtype = pathtype,
        cgo_tools = cgo_tools,
        nogo = nogo,
        coverdata = coverdata,
        coverage_enabled = ctx.configuration.coverage_enabled,
        coverage_instrumented = ctx.coverage_instrumented(),
        env = env,
        tags = tags,
        stamp = mode.stamp,
        label = ctx.label,
        cover_format = mode.cover_format,
        # Action generators
        archive = toolchain.actions.archive,
        binary = toolchain.actions.binary,
        link = toolchain.actions.link,

        # Helpers
        args = _new_args,  # deprecated
        builder_args = _builder_args,
        tool_args = _tool_args,
        new_library = _new_library,
        library_to_source = _library_to_source,
        declare_file = _declare_file,
        declare_directory = _declare_directory,

        # Private
        # TODO: All uses of this should be removed
        _ctx = ctx,
    )

def _go_context_data_impl(ctx):
    if "race" in ctx.features:
        print("WARNING: --features=race is no longer supported. Use --@io_bazel_rules_go//go/config:race instead.")
    if "msan" in ctx.features:
        print("WARNING: --features=msan is no longer supported. Use --@io_bazel_rules_go//go/config:msan instead.")
    nogo = ctx.files.nogo[0] if ctx.files.nogo else None
    providers = [
        GoContextInfo(
            coverdata = ctx.attr.coverdata[GoArchive],
            nogo = nogo,
        ),
        ctx.attr.stdlib[GoStdLib],
        ctx.attr.go_config[GoConfigInfo],
    ]
    if ctx.attr.cgo_context_data and CgoContextInfo in ctx.attr.cgo_context_data:
        providers.append(ctx.attr.cgo_context_data[CgoContextInfo])
    return providers

```
