# Nixpkgs 参数

欢迎来到第 16 篇 Nix 药丸。在上一篇[第 15 篇](15-nix-search-paths.md)药丸中，我们弄清了 nix 如何通过尖括号语法查找表达式，所以我们终于知道了 `<nixpkgs>` 在系统中的位置。

我们可以开始深入 [nixpkgs 仓库](https://github.com/NixOS/nixpkgs)，领略其中各种各样的工具和设计模式。请注意，`nixpkgs` 也有自己的手册，它强调了通用的 `nix` 语言与 `nixpkgs` 仓库之间的区别。

## default.nix 表达式

我们不会一开始就去检查各个软件包，而是先看 `nixpkgs` 的整体结构。

在我们的自定义仓库中，我们创建了一个 `default.nix` 来组合各个软件包的表达式。

`nixpkgs` 也有自己的 [default.nix](https://github.com/NixOS/nixpkgs/blob/master/default.nix)，当引用 `<nixpkgs>` 时加载的就是它。它做一件简单的事：检查 `nix` 版本是否至少为 1.7（在撰写这篇博文时）。然后导入 [pkgs/top-level/all-packages.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/all-packages.nix)。从现在起，我们将把这个软件包集合称为 **pkgs**。

`all-packages.nix` 就是组合所有软件包的文件。注意 `pkgs/` 子目录，而 nixos 位于 `nixos/` 子目录中。

`all-packages.nix` 有几分刻意为之。首先，它是一个函数。它接受几个有意思的参数：

- `system`：默认为当前系统

- `config`：默认为 null

- 其他……

**system** 参数，按照表达式中的注释所说，是为这些软件包进行构建的目标系统。例如，它允许在 amd64 机器上安装 i686 软件包。

**config** 参数是一个简单的属性集合。软件包可以读取其中的一些值，并改变某些 derivation 的行为。

## system 参数

你会在许多其他 .nix 表达式中（例如 release 表达式）看到这个参数。原因是，既然 pkgs 接受一个 system 参数，那么每当你想导入 pkgs 时，你也会想把 system 的值传递下去。例如：

`myrelease.nix`：

```nix
{ system ? builtins.currentSystem }:

let pkgs = import <nixpkgs> { inherit system; };
...
```

它为什么有用？有了这个参数，为特定系统选择一组软件包就非常容易了。例如：

```console
nix-build -A psmisc --argstr system i686-linux
```

这将为 i686-linux 而不是 x86_64-linux 构建 `psmisc` derivation。这个概念与 Debian 的 multi-arch 非常相似。

交叉编译的配置也在 `nixpkgs` 中，不过讨论它有点绕，而且我对它也不太了解。

## config 参数

我相信你一定在 wiki 或其他手册上读到过 `~/.config/nixpkgs/config.nix`（以前是 `~/.nixpkgs/config.nix`），也一定好奇过它是否硬编码在 nix 里。它不是，它在 [nixpkgs](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28) 中。

`all-packages.nix` 表达式接受 `config` 参数。如果它是 `null`，就会读取 `NIXPKGS_CONFIG` 环境变量。如果没有指定，`nixpkgs` 会选用 `$HOME/.config/nixpkgs/config.nix`。

确定 `config.nix` 之后，它会作为一个 nix 表达式被导入，其结果就是 `config` 的值（前提是它没有被作为参数传入 `<nixpkgs>` 的 import）。

`config` 在最终的仓库中是可用的：

```console
$ nix repl
nix-repl> pkgs = import <nixpkgs> {}
nix-repl> pkgs.config
{ }
nix-repl> pkgs = import <nixpkgs> { config = { foo = "bar"; }; }
nix-repl> pkgs.config
{ foo = "bar"; }
```

哪些属性放进 `config`，这是一个看便利性和约定的问题。

例如，`config.allowUnfree` 是一个属性，它默认禁止构建带有非自由（unfree）许可证的软件包。`config.pulseaudio` 设置则决定在适用的地方、且 derivation 遵循该设置时，是否构建带 `pulseaudio` 支持的软件包。

## 关于 .nix 函数

一个 `.nix` 文件包含一个 nix 表达式。因此它也可以是一个函数。提醒你一下，`nix-build` 期望表达式返回一个 derivation。所以直接从 `.nix` 文件返回一个 derivation 是很自然的。不过，`.nix` 文件接受一些参数以调整所返回的 derivation，同样是非常自然的。

在这种情况下，nix 玩了一个小把戏：

- 如果表达式是一个 derivation，就构建它。

- 如果表达式是一个函数，就调用它并构建得到的 derivation。

例如你可以对下面的 `.nix` 文件执行 nix-build：

```nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.psmisc
```

Nix 能够调用这个函数，因为 pkgs 参数有默认值。这允许你使用 `--arg` 选项为 pkgs 传入不同的值。

如果是一个返回「返回 derivation 的函数」的函数，还能工作吗？不能，Nix 只调用它遇到的那一次函数。

## 结论

我们揭开了 `<nixpkgs>` 仓库的面纱。它是一个接受一些参数、并返回所有软件包集合的函数。得益于惰性求值，只有被访问到的 derivation 才会被构建。

你可以像上一篇药丸中创建我们自己的仓库时所看到的那样，用这个仓库来构建你自己的软件包。

最近我有点忙于 NixOS 14.11 的发布和其他事情，同时我也在考虑从 blogger 迁移到一个更面向代码的博客平台。所以这篇药丸延迟了而且比较短，抱歉啦 :)

## 下一篇药丸

……我们将讨论在 `nixpkgs` 仓库中进行 override（覆盖）。如果你想修改某个库的一些选项，并让所有其他软件包使用新库，该怎么办？一种可能是像上面描述的那样，在适用时使用 `config` 参数。另一种可能是 override derivation。
