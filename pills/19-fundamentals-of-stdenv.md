# stdenv 基础

欢迎来到第 19 篇 Nix 药丸。在上一篇[第 18 篇](18-nix-store-paths.md)药丸中，我们深入探讨了 Nix 计算存储路径（store path）所用的算法，还介绍了固定输出存储路径。

这次我们来看看 `nixpkgs`，特别是它的一个核心 derivation（派生）：`stdenv`。

Nix 并不把 `stdenv` 当作特殊的 derivation 对待，但它对 `nixpkgs` 仓库而言非常重要。它是打包软件的基础。它用来引入诸如 GCC 工具链、GNU make、coreutils、patch 和 diff 工具等依赖：这些是编译 `nixpkgs` 中现存的大量软件所需要的基本工具。

## 什么是 stdenv？

首先，`stdenv` 是一个 derivation，而且是一个非常简单的 derivation：

```console
$ nix-build '<nixpkgs>' -A stdenv
/nix/store/k4jklkcag4zq4xkqhkpy156mgfm34ipn-stdenv
$ ls -R result/
result/:
nix-support/  setup

result/nix-support:
propagated-user-env-packages
```

它只有两个文件：`/setup` 和 `/nix-support/propagated-user-env-packages`。不用管后者，它实际上是空的。重要的是 `/setup` 文件。

这么简单的一个 derivation 怎么能引入编译软件包所需的全部工具链和基本工具呢？我们来看看它的运行时依赖：

```console
$ nix-store -q --references result
/nix/store/3a45nb37s0ndljp68228snsqr3qsyp96-bzip2-1.0.6
/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24
/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39
/nix/store/47sfpm2qclpqvrzijizimk4md1739b1b-gcc-wrapper-4.9.3
...
```

这怎么可能？这个软件包一定以某种方式引用了那些其他软件包。实际上，它们被硬编码在 `/setup` 文件中：

```console
$ head result/setup
export SHELL=/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39/bin/bash
initialPath="/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24 ..."
defaultNativeBuildInputs="/nix/store/sgwq15xg00xnm435gjicspm048rqg9y6-patchelf-0.8 ..."
```

## setup 文件

还记得[第 8 篇药丸](08-generic-builders.md)中我们那个通用的 `builder.sh` 吗？它会设置基本的 `PATH`、解压源码，并替我们运行常见的 `autotools` 命令。

[stdenv 的 setup 文件](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh)正是这样的东西。它设置若干环境变量（如 `PATH`），并创建一些辅助 bash 函数来构建软件包。我建议你读一读它。

硬编码的工具链和实用工具用来预先填充环境变量，让常用命令的运行更顺手，这与我们在自己的 builder 中使用 `baseInputs` 和 `buildInputs` 的做法类似。

使用 `stdenv` 的构建按阶段（phase）进行。阶段包括 `unpackPhase`、`configurePhase`、`buildPhase`、`checkPhase`、`installPhase`、`fixupPhase` 等。你可以在 `genericBuild` 函数中看到默认的阶段列表。

`genericBuild` 所做的只是依次运行这些阶段。默认阶段就是普通的 bash 函数，你可以轻松地阅读它们。

每个阶段都有在阶段执行前后运行命令的钩子（hook）。阶段可以被覆盖、重新排序，随便你怎么改，它就是 bash 代码。

如何使用这个文件？就像我们以前的 builder 一样。为了测试它，我们进入一个假的空 derivation，source `stdenv` 的 `setup`，解压 hello 的源码并构建它：

```console
$ nix-shell -E 'derivation { name = "fake"; builder = "fake"; system = "x86_64-linux"; }'
nix-shell$ unset PATH
nix-shell$ source /nix/store/k4jklkcag4zq4xkqhkpy156mgfm34ipn-stdenv/setup
nix-shell$ tar -xf hello-2.10.tar.gz
nix-shell$ cd hello-2.10
nix-shell$ configurePhase
...
nix-shell$ buildPhase
...
```

_我 unset 了 `PATH`，进一步说明 `stdenv` 的自包含程度足以构建没有任何其他依赖的 autotools 软件包。_

于是我们运行了 `configurePhase` 函数和 `buildPhase` 函数，它们正常工作。这些 bash 函数应当不言自明，你可以在 `setup` 文件中阅读其代码。

## setup 文件是如何构建的

到目前为止我们一直在用纯 bash 脚本。那 Nix 这边呢？`nixpkgs` 仓库提供了一个实用的函数，就像我们以前的 builder 一样。它是对原始 derivation 函数的封装，会替我们引入 `stdenv`，并运行 `genericBuild`。它就是 [stdenv.mkDerivation](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix)。

注意 `stdenv` 是一个 derivation，但同时也是一个包含其他属性（如 `mkDerivation`）的属性集合。这里没有什么特别的花哨之处，只是为了方便。

让我们用这个新发现的 `stdenv` 编写一个 `hello.nix` 表达式：

```nix
with import <nixpkgs> { };
stdenv.mkDerivation {
  name = "hello";
  src = ./hello-2.10.tar.gz;
}
```

不要被 `with` 表达式吓到。它把 `nixpkgs` 仓库引入作用域，让我们可以直接使用 `stdenv`。它看起来与[第 8 篇药丸](08-generic-builders.md)中的 hello 表达式非常相似。

它可以构建并正常运行：

```console
$ nix-build hello.nix
...
/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello
$ result/bin/hello
Hello, world!
```

## stdenv.mkDerivation 的 builder

我们来看看 `mkDerivation` 所使用的 builder。你可以在 [nixpkgs 中的这里](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix)阅读代码：

```nix
{
  # ...
  builder = attrs.realBuilder or shell;
  args =
    attrs.args or [
      "-e"
      (attrs.builder or ./default-builder.sh)
    ];
  stdenv = result;
  # ...
}
```

同时也看看前几篇药丸中我们自己的 derivation 封装吧！这里的 builder 是 bash（即那个 shell 变量），传给 builder（bash）的参数是 `default-builder.sh`，然后我们在 derivation 中加入环境变量 `$stdenv`，它就是 `stdenv` 这个 derivation。

你可以打开 [default-builder.sh](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/default-builder.sh) 看看它做了什么：

```sh
source $stdenv/setup
genericBuild
```

这正是我们在[第 10 篇药丸](10-developing-with-nix-shell.md)中让 derivation 对 `nix-shell` 友好的做法。进入 shell 时，setup 文件只设置环境而不构建任何东西；执行 `nix-build` 时，它才真正运行构建过程。

为了清楚地理解环境变量，看看 hello 这个 derivation 的 .drv：

```console
$ nix derivation show $(nix-instantiate hello.nix)
warning: you did not specify '--add-root'; the result might be removed by the garbage collector
{
  "/nix/store/abwj50lycl0m515yblnrvwyydlhhqvj2-hello.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello"
      }
    },
    "inputSrcs": [
      "/nix/store/9krlzvny65gdc8s7kpb6lkx8cd02c25b-default-builder.sh",
      "/nix/store/svc70mmzrlgq42m9acs0prsmci7ksh6h-hello-2.10.tar.gz"
    ],
    "inputDrvs": {
      "/nix/store/hcgwbx42mcxr7ksnv0i1fg7kw6jvxshb-bash-4.4-p19.drv": [
        "out"
      ],
      "/nix/store/sfxh3ybqh97cgl4s59nrpi78kgcc8f3d-stdenv-linux.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
    "args": [
      "-e",
      "/nix/store/9krlzvny65gdc8s7kpb6lkx8cd02c25b-default-builder.sh"
    ],
    "env": {
      "buildInputs": "",
      "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
      "configureFlags": "",
      "depsBuildBuild": "",
      "depsBuildBuildPropagated": "",
      "depsBuildTarget": "",
      "depsBuildTargetPropagated": "",
      "depsHostBuild": "",
      "depsHostBuildPropagated": "",
      "depsTargetTarget": "",
      "depsTargetTargetPropagated": "",
      "name": "hello",
      "nativeBuildInputs": "",
      "out": "/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello",
      "propagatedBuildInputs": "",
      "propagatedNativeBuildInputs": "",
      "src": "/nix/store/svc70mmzrlgq42m9acs0prsmci7ksh6h-hello-2.10.tar.gz",
      "stdenv": "/nix/store/6kz2vbh98s2r1pfshidkzhiy2s2qdw0a-stdenv-linux",
      "system": "x86_64-linux"
    }
  }
}
```

它非常短，所以我决定把整段贴在上面。builder 是 bash，参数为 `-e default-builder.sh`。然后你可以看到 `src` 和 `stdenv` 环境变量。

最后一点，setup 中的 `unpackPhase` 用于解压源码并进入目录。同样，和我们以前的 builder 做法一样。

## 结论

`stdenv` 是 `nixpkgs` 仓库的核心。所有软件包都使用 `stdenv.mkDerivation` 封装，而不是原始的 derivation。它替我们完成了一系列操作，还搭建了一个舒适的构建环境。

整体流程很简单：

- `nix-build`

- `bash -e default-builder.sh`

- `source $stdenv/setup`

- `genericBuild`

就是这样。关于 stdenv 各阶段你需要知道的一切都在 [setup 文件](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh)里。

真的，请花点时间读一读那个文件。别忘了 [nixpkgs 手册](http://nixos.org/nixpkgs/manual/#chap-stdenv)里还有丰富的文档。

## 下一篇药丸……

……我们将讨论如何用 `buildInputs` 和 `propagatedBuildInputs` 为软件包添加依赖，以及如何用 setup hook 和 env hook 影响下游构建。这些概念对 `nixpkgs` 软件包的组装方式至关重要。
