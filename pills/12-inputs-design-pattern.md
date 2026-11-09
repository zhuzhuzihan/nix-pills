# 软件包仓库与 Inputs 设计模式 {#inputs-design-pattern}

欢迎来到第 12 篇 Nix 药丸。在上一篇[第 11 篇药丸](11-garbage-collector.md)中，我们暂停了打包工作，用垃圾回收器清理了系统。

这一次，我们将重新开始打包，并改进其中的多个方面。我们还会演示如何创建一个包含多个软件包的仓库。

## Nix 中的仓库

Nix 中的软件包仓库源于组织软件包的自然需要。Nix 本身并没有规定预设的目录结构或打包策略；作为一种完备的函数式编程语言，Nix 足够强大，可以支持多种不同的仓库形式。

随着时间推移，`nixpkgs` 仓库演化出了一种特定的结构。这种结构既反映了 Nix 的历史，也反映了它的用户在构建和组织软件包过程中所采用的、被证明有用的设计模式。下面我们将详细考察其中一些模式。

## 单仓库模式

不同的操作系统发行版对软件包仓库应当如何组织各有不同的看法。Debian 这类系统把软件包分散在若干个小仓库中（这往往使追踪相互依赖的变更变得更加困难，也阻碍了对仓库的贡献），而 Gentoo 这类系统则把所有软件包描述放在单一仓库中。

Nix 遵循"单仓库"模式，把所有软件包的全部描述放入 [nixpkgs](https://github.com/NixOS/nixpkgs)。事实证明，这种方式对新贡献者来说自然且具有吸引力。

在本篇药丸的其余部分，我们将采用单仓库模式。它在 Nix 中的自然实现方式是：创建一个顶层的 Nix 表达式，然后为每个软件包各写一个表达式。顶层表达式导入并组合所有软件包表达式，形成一个把名字映射到软件包的 attribute set。

在某些编程语言中，这样的做法 \-- 即把所有可能的软件包描述放进单一数据结构 \-- 会难以成立，因为语言需要先把整个数据结构加载进内存才能对其进行操作。然而 Nix 是一门惰性语言，只对需要的内容求值。

## 打包 `graphviz`

我们已经打包过 GNU `hello`。接下来，我们将打包一个名为 `graphviz` 的绘图程序，以便创建一个包含多个软件包的仓库。之所以选择 `graphviz`，是因为它使用标准的 autotools 构建系统，无需打补丁。它还有可选的依赖，这给了我们一个展示如何针对特定情形配置构建的机会。

首先，我们从 [gitlab](https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/2.49.3/graphviz-2.49.3.tar.gz) 下载 `graphviz`。`graphviz.nix` 表达式非常简单：

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
}
```

如果我们用 `nix-build graphviz.nix` 构建该项目，就会在 `result/bin` 下得到可运行的二进制文件。注意我们复用了与 `hello.nix` 相同的 `autotools.nix`。

默认情况下，`graphviz` 编译出来的版本不具备生成 `png` 文件的能力。因此，上面的 derivation 将构建出仅支持原生输出格式的二进制文件，如下所示：

```console
$ echo 'graph test { a -- b }'|result/bin/dot -Tpng -o test.png
Format: "png" not recognized. Use one of: canon cmap [...]
```

如果我们想让 `graphviz` 生成 `png` 文件，就必须把它加入我们的 derivation。合适的位置是 `autotools.nix`，我们在其中创建了一个会被拼接进 `baseInputs` 的 `buildInputs` 变量。这个变量存在的意义正是如此：让 `autotools.nix` 的使用者能够从软件包表达式中添加额外的输入。

`graphviz` 2.49 版本有若干可以输出 `png` 的插件。为简单起见，我们使用 `libgd`。

## 通过环境变量向 `pkg-config` 传递库信息

`graphviz` 的配置脚本使用 `pkg-config` 来指定传递给编译器的标志。由于库并没有一个全局的存放位置，我们需要告诉 `pkg-config` 到哪里去找它的描述文件，这些描述文件会告诉配置脚本到哪里去找头文件和库。

在经典 POSIX 系统中，`pkg-config` 直接在 `/usr/lib/pkgconfig` 之类的系统目录中查找所有已安装库的 `.pc` 文件。然而，在 Nix 提供的隔离环境中并不存在这些文件。

作为替代，我们可以通过 `PKG_CONFIG_PATH` 环境变量告诉 `pkg-config` 库的位置。我们可以用之前对 `PATH` 使用的同一个技巧来填充这个环境变量：从 `buildInputs` 自动填充变量。这是 `setup.sh` 中的相关片段：

```sh
for p in $baseInputs $buildInputs; do
    if [ -d $p/bin ]; then
        export PATH="$p/bin${PATH:+:}$PATH"
    fi
    if [ -d $p/lib/pkgconfig ]; then
        export PKG_CONFIG_PATH="$p/lib/pkgconfig${PKG_CONFIG_PATH:+:}$PKG_CONFIG_PATH"
    fi
done
```

现在，如果我们往 `buildInputs` 中添加 derivation，它们的 `lib/pkgconfig` 和 `bin` 路径就会被自动添加进 `setup.sh`。

## 加上 `gd` 补全 graphviz

下面，我们完成带有 `gd` 支持的 `graphviz` 表达式。注意在 `buildInputs` 中使用 `with` 表达式来避免重复书写 `pkgs`：

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
  buildInputs = with pkgs; [
    pkg-config
    (pkgs.lib.getLib gd)
    (pkgs.lib.getDev gd)
  ];
}
```

我们把 `pkg-config` 加入 derivation，以便 configure 脚本可以使用这个工具。由于 `gd` 是一个具有[分离输出](https://nixos.org/manual/nixpkgs/stable/#sec-multiple-outputs-)的软件包，我们需要同时添加库输出和开发输出。

构建完成后，`graphviz` 现在可以生成 `png` 了。

## 仓库表达式

现在我们有了两个软件包，想把它们组合成一个仓库。为此，我们模仿 `nixpkgs` 的做法：创建一个包含 derivation 的单一 attribute set。这个 attribute set 可以被导入，然后通过访问顶层 attribute set 来选取 derivation。

利用这一技术，我们能够摆脱对文件名的依赖。不必再通过 `REPO/some/sub/dir/package.nix` 来指代一个软件包，这种技术允许我们以 `importedRepo.package`（在我们的示例中即 `pkgs.package`）的形式选取一个 derivation。

首先，在当前目录创建一个 default.nix：

```nix
{
  hello = import ./hello.nix;
  graphviz = import ./graphviz.nix;
}
```

这个文件可以直接配合 `nix repl` 使用：

```console
$ nix repl
nix-repl> :l default.nix
Added 2 variables.
nix-repl> hello
«derivation /nix/store/dkib02g54fpdqgpskswgp6m7bd7mgx89-hello.drv»
nix-repl> graphviz
«derivation /nix/store/zqv520v9mk13is0w980c91z7q1vkhhil-graphviz.drv»
```

使用 `nix-build` 时，我们可以传入 -A 选项来访问给定 `.nix` 表达式中 attribute set 的某个属性：

```console
$ nix-build default.nix -A hello
[...]
$ result/bin/hello
Hello, world!
```

`default.nix` 文件是特殊的。当某个目录中存在 `default.nix` 文件时，它会被用作该目录的隐式 Nix 表达式。例如，这使我们可以运行 `nix-build -A hello` 而无需显式指定 `default.nix`。

现在我们可以用 `nix-env` 把软件包安装进用户环境：

```console
$ nix-env -f . -iA graphviz
[...]
$ dot -V
```

仔细观察上面的命令，可以看到以下选项：

- -f 选项用于指定要使用的表达式。在这里，表达式就是当前目录的 `./default.nix`。

- -i 选项代表"安装"（installation）。

- -A 与上文 `nix-build` 中的用法相同。

我们复现了 `nixpkgs` 最基本的行为：把多个 derivation 组合成一个顶层的 attribute set。

## Inputs 模式

我们目前采用的做法存在几个问题：

- 首先，`hello.nix` 和 `graphviz.nix` 依赖 `nixpkgs`，并且直接导入它。更好的做法是把 `nixpkgs` 作为参数传入，就像我们在 `autotools.nix` 中做的那样。

- 其次，我们没有一种直接的方法来编译同一软件的不同变体，例如带或不带 `libgd` 支持的 `graphviz`。

- 第三，我们没有办法用特定版本的 `libgd` 来测试 `graphviz`。

到目前为止，我们解决上述问题的方式一直不够理想，需要修改 Nix 表达式才能满足需求。借助 `inputs` 模式，我们给出另一种答案：让用户改变表达式的 `inputs`。

当我们谈论"表达式的 inputs"时，指的是构建该表达式所需的一组 derivation。在本例中：

- 来自 `autotools` 的 `mkDerivation`。回忆一下，`mkDerivation` 对工具链有一个隐式依赖。

- `libgd` 及其依赖。

`./src` 目录也是一个 input，但我们不会让调用方去改动源码。在 `nixpkgs` 中，我们更倾向于为版本变更单独编写另一个表达式（例如因为需要补丁或不同的 inputs）。

我们的目标是让软件包表达式独立于仓库。为此，我们使用函数来为 derivation 声明 inputs。例如，对 `graphviz.nix` 做如下修改，使 derivation 独立于仓库并且可定制：

```nix
{ mkDerivation, lib, gdSupport ? true, gd, pkg-config }:

mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
  buildInputs =
    if gdSupport
      then [
        pkg-config
        (lib.getLib gd)
        (lib.getDev gd)
      ]
      else [];
}
```

回忆一下，"`{...}: ...`" 是定义接受一个 attribute set 作为参数的函数的语法；上面的代码片段只是定义了一个函数。

我们把 `gd` 及其依赖变成了可选项。如果 `gdSupport` 为 true（默认即为 true），我们会填充 `buildInputs`，`graphviz` 将带着 `gd` 支持构建。否则，如果传入的 attribute set 中带有 `gdSupport = false;`，构建将在没有 `gd` 支持的情况下完成。

回到 `default.nix`，我们修改表达式以利用 inputs 模式：

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
with pkgs;
{
  hello = import ./hello.nix { inherit mkDerivation; };
  graphviz = import ./graphviz.nix {
    inherit
      mkDerivation
      lib
      gd
      pkg-config
      ;
  };
  graphvizCore = import ./graphviz.nix {
    inherit
      mkDerivation
      lib
      gd
      pkg-config
      ;
    gdSupport = false;
  };
}
```

我们把 `nixpkgs` 和 `mkDerivation` 的导入做了因式提取，还添加了一个禁用 `gd` 支持的 `graphviz` 变体。其结果是，`hello.nix`（留给读者作为练习）和 `graphviz.nix` 都独立于仓库，并且可以通过传入特定的 inputs 来定制。

如果我们想用特定版本的 `gd` 构建 `graphviz`，只需传入 `gd = ...;` 即可。

如果我们想更换工具链，只需传入一个不同的 `mkDerivation` 函数。

让我们更仔细地看看这段代码并剖析其语法：

- `default.nix` 中的整个表达式返回一个 attribute set，其键为 `hello`、`graphviz` 和 `graphvizCore`。

- 通过 "`let`"，我们定义了一些局部变量。

- 在定义软件包集合时，我们把 `pkgs` 带入作用域。这省去了我们反复输入 `pkgs` 的麻烦。

- 我们导入 `hello.nix` 和 `graphviz.nix`，它们各自返回一个函数。我们用一组 inputs 调用这些函数，从而取回 derivation。

- "`inherit x`" 语法等价于 "`x = x`"。也就是说，这里的 "`inherit gd`" 与上面的 "`with pkgs;`" 相结合，等价于 "`gd = pkgs.gd`"。

本篇的完整仓库可以在[第 12 篇药丸](https://gist.github.com/tfc/ca800a444b029e85a14e530c25f8e872)的 gist 中找到。

## 结论

"`inputs`" 模式让我们的表达式可以通过一组参数轻松定制。这些参数可以是开关标志、derivation，或者 Nix 语言所允许的任何其他定制。我们的软件包表达式只是函数：没有任何额外的魔法。

"`inputs`" 模式还使表达式独立于仓库。鉴于我们通过参数传入所有需要的信息，这些表达式可以在任何其他上下文中使用。

## 下一篇药丸

在下一篇药丸中，我们将讨论 "`callPackage`" 设计模式。它消除了把 inputs 名字写两遍的繁琐：一遍写在顶层 `default.nix` 中，一遍写在软件包表达式中。借助 `callPackage`，我们将从顶层表达式隐式传入必要的 inputs。
