# 使用 nix-shell 开发

欢迎来到第 10 篇 Nix 药丸。在上一篇[第 9 篇药丸](09-automatic-runtime-dependencies.md)中，我们见识了 Nix 的一个强大特性：自动发现运行时依赖。我们还完成了 GNU `hello` 软件包的收尾工作。

在这一篇药丸中，我们将介绍 `nix-shell` 工具，并用它来研究（hack）GNU `hello` 程序。我们将看到，`nix-shell` 如何在我们修改项目源码文件时提供一个隔离的环境，就像 `nix-build` 在构建 derivation（派生）时提供了一个隔离的环境一样。

最后，我们将修改自己的 builder，使其更符合以 `nix-shell` 为中心的工作流。

## 什么是 `nix-shell`？

[nix-shell](https://nix-doc.mczihan.link/command-ref/nix-shell) 工具在设置好研究（hack）一个 derivation 所需的环境变量之后，把我们带进一个 shell。它不会构建这个 derivation；它只是做准备工作，以便我们手动运行构建步骤。

回想一下，在 nix 环境中，除非库或程序是用 `nix-env` 安装的，否则我们无法访问它们。然而，用 `nix-env` 安装库并不是好做法。我们更希望为开发提供隔离的环境，而 `nix-shell` 正是为此提供支持的。

我们可以对任何返回 derivation 的 Nix 表达式调用 `nix-shell`，但得到的 `bash` shell 的 `PATH` 中并没有我们想要的工具：

```console
$ nix-shell hello.nix
[nix-shell]$ make
bash: make: command not found
[nix-shell]$ echo $baseInputs
/nix/store/jff4a6zqi0yrladx3kwy4v6844s3swpc-gnutar-1.27.1 [...]
```

这个 shell 相当没用。合理的预期是 GNU `hello` 的构建输入应该在 `PATH` 中可用，包括 GNU `make`，但事实并非如此。

不过，我们确实拥有在 derivation 中设置的那些环境变量，比如 `$baseInputs`、`$buildInputs`、`$src` 等等。

这意味着我们可以 `source` 我们的 `builder.sh`，它就会构建这个 derivation。你可能会在安装阶段遇到错误，因为你的用户可能没有写入 `/nix/store` 的权限：

```console
[nix-shell]$ source builder.sh
...
```

derivation 没有安装成功，但确实构建了。请注意以下几点：

- 我们 `source` 了 `builder.sh`，它执行了所有构建步骤，包括为我们设置好 `PATH`。

- 工作目录不再是 `nix-build` 创建的临时目录，而是我们进入 shell 时所在的目录。因此，`hello-2.10` 被解压到了当前目录。

我们能够 `cd` 进入 `hello-2.10` 并输入 `make`，因为 `make` 现在可用了。

要点是：`nix-shell` 把我们带进一个与运行 builder 时所用环境相同（或非常相似）的 shell。

## 为 nix-shell 而设的 builder

前面的步骤需要手动运行一些命令，对于以 `nix-shell` 为中心的工作流来说并不是最优的。现在我们将改进 builder，使其对 `nix-shell` 更加友好。

有几处我们想要修改的地方。

首先，当我们 `source` `builder.sh` 文件时，拿到的是当前目录中的文件。而我们真正想要的是存储在 nix store 中的那个 `builder.sh`，因为 `nix-build` 用的就是那个文件。要做到这一点，正确的技术是通过 derivation 传递一个环境变量。（注意，`$builder` 已经定义了，但它指向的是 bash 可执行文件而不是我们的 `builder.sh`。我们的 `builder.sh` 是作为参数传给 bash 的。）

其次，我们不想运行整个 builder：我们只想为手动构建项目设置必要的环境。因此，我们可以把 `builder.sh` 拆成两个文件：一个用于设置环境的 `setup.sh`，以及 `nix-build` 所期望的真正的 `builder.sh`。

在这次重构中，我们会把构建阶段包装进函数，为我们的设计赋予更多结构。此外，我们会把 `set -e` 移到 builder 文件而不是 setup 文件中。`set -e` 在 `nix-shell` 中很烦人，因为一旦遇到错误（比如敲错命令），它就会终止 shell。

下面是我们修改后的 `autotools.nix`。值得注意的是 derivation 中的 `setup = ./setup.sh;` 属性，它把 `setup.sh` 加入 nix store，并相应地在 builder 中添加了一个 `$setup` 环境变量。

```nix
pkgs: attrs:
let
  defaultAttrs = {
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./builder.sh ];
    setup = ./setup.sh;
    baseInputs = with pkgs; [
      gnutar
      gzip
      gnumake
      gcc
      coreutils
      gawk
      gnused
      gnugrep
      binutils.bintools
      patchelf
      findutils
    ];
    buildInputs = [ ];
    system = builtins.currentSystem;
  };
in
derivation (defaultAttrs // attrs)
```

多亏了这个，我们可以把 `builder.sh` 拆分为 `setup.sh` 和 `builder.sh`。`builder.sh` 要做的就是 `source` `$setup` 并调用 `genericBuild` 函数。其余只是一些对 bash 脚本的改动。

下面是修改后的 `builder.sh`：

```sh
set -e
source $setup
genericBuild
```

下面是新增的 `setup.sh`：

```sh
unset PATH
for p in $baseInputs $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
done

function unpackPhase() {
    tar -xzf $src

    for d in *; do
    if [ -d "$d" ]; then
        cd "$d"
        break
    fi
    done
}

function configurePhase() {
    ./configure --prefix=$out
}

function buildPhase() {
    make
}

function installPhase() {
    make install
}

function fixupPhase() {
    find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
}

function genericBuild() {
    unpackPhase
    configurePhase
    buildPhase
    installPhase
    fixupPhase
}
```

最后，这是 `hello.nix`：

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "hello";
  src = ./hello-2.12.1.tar.gz;
}
```

现在回到 nix-shell：

```console
$ nix-shell hello.nix
[nix-shell]$ source $setup
[nix-shell]$
```

现在，举例来说，你可以运行 `unpackPhase`，它会解压 `$src` 并进入该目录。你也可以手动运行诸如 `./configure`、`make` 之类的命令，或者用各自的函数来执行各个阶段。

流程就是这么简单直接。`nix-shell` 会构建 `.drv` 文件及其输入依赖，然后通过设置构建该 `.drv` 所需的环境变量进入一个 shell。特别是，shell 中的环境变量与传给 `derivation` 函数的变量相匹配。

## 结论

借助 `nix-shell`，我们能够进入一个适合开发项目的隔离环境。这个环境为开发 shell 提供了必要的依赖，就像 `nix-build` 为 builder 提供必要的依赖一样。此外，我们可以手动构建和调试项目，像在任何其他操作系统中那样一步步执行。注意，我们从未把 `gcc` 或 `make` 这类工具安装到系统全局；这些工具和库是隔离的，并且按构建各自独立可用。

## 下一篇药丸

在下一篇药丸中，我们将清理 nix store。我们已经编写并构建了会往 nix store 中添加内容的 derivation，但到目前为止我们还没有操心过清理 store 中已占用空间的问题。
