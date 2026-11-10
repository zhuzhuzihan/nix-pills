# 通用 builder

欢迎来到第 8 篇 Nix 药丸。在上一篇[第 7 篇药丸](07-working-derivation.md)中，我们成功构建了一个 derivation（派生）。我们编写了一个 builder 脚本，它编译一个 C 文件并把二进制文件安装到 nix store 下。

在这篇文章中，我们将把 builder 脚本泛化，为 [GNU hello world](https://www.gnu.org/software/hello/) 编写一个 Nix 表达式，并围绕 derivation 内置函数创建一个包装器。

## 打包 GNU hello world

在上一篇药丸中，我们打包了一个简单的 .c 文件，它是通过直接调用 gcc 来编译的。这并不是一个项目的好例子。许多项目使用 autotools，既然我们要泛化 builder，最好就用最常用的构建系统来做。

[GNU hello world](https://www.gnu.org/software/hello/)，尽管名字如此，其实是一个简单而完整的项目，它使用 autotools。在这里获取最新的压缩包：<https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz>。

让我们为 GNU hello world 创建一个 builder 脚本，hello_builder.sh：

```sh
export PATH="$gnutar/bin:$gcc/bin:$gnumake/bin:$coreutils/bin:$gawk/bin:$gzip/bin:$gnugrep/bin:$gnused/bin:$bintools/bin"
tar -xzf $src
cd hello-2.12.1
./configure --prefix=$out
make
make install
```

以及 derivation hello.nix：

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "hello";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./hello_builder.sh ];
  inherit (pkgs)
    gnutar
    gzip
    gnumake
    gcc
    coreutils
    gawk
    gnused
    gnugrep
    ;
  bintools = pkgs.binutils.bintools;
  src = ./hello-2.12.1.tar.gz;
  system = builtins.currentSystem;
}
```

<div class="info">
<h4>darwin 上的 Nix</h4>

Darwin（即 macOS）的构建通常使用 `clang` 而不是 `gcc` 作为 C 编译器。我们可以使用下面这个修改版的 `hello.nix` 来让这个早期示例适配 darwin：

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "hello";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./hello_builder.sh ];
  inherit (pkgs)
    gnutar
    gzip
    gnumake
    coreutils
    gawk
    gnused
    gnugrep
    ;
  gcc = pkgs.clang;
  bintools = pkgs.clang.bintools.bintools_bin;
  src = ./hello-2.12.1.tar.gz;
  system = builtins.currentSystem;
}
```

稍后我们会展示 Nix 如何自动处理这些差异。现在请记住，在后文中可能需要进行与上面类似的修改。

</div>

现在用 `nix-build hello.nix` 构建它，然后就可以运行 `result/bin/hello` 了。再简单不过，但问题是我们必须为每个软件包都创建一个 builder.sh 吗？我们总是要把依赖传给 `derivation` 函数吗？

请注意我们在[上一篇药丸](07-working-derivation.md)中谈论过的 `--prefix=$out`。

## 一个通用的 builder

让我们为 autotools 项目创建一个通用的 `builder.sh`：

```sh
set -e
unset PATH
for p in $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
done

tar -xf $src

for d in *; do
    if [ -d "$d" ]; then
        cd "$d"
        break
    fi
done

./configure --prefix=$out
make
make install
```

我们在这里做了什么？

1.  使用 `set -e`，出错时退出构建。

2.  首先执行 `unset PATH`，因为 PATH 最初被设成了一个不存在的路径。

3.  我们会在下文详细讲解这一点：对 `$buildInputs` 中的每个路径，我们把它的 `bin` 追加到 `PATH` 上。

4.  解开源码。

5.  找到源码解压出来的目录，并 `cd` 进去。

6.  准备就绪后，编译并安装。

可以看到，builder 中已经没有任何对 "hello" 的引用了。它仍然做了若干假设，但确实更通用了。

现在让我们重写 `hello.nix`：

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "hello";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./builder.sh ];
  buildInputs = with pkgs; [
    gnutar
    gzip
    gnumake
    gcc
    coreutils
    gawk
    gnused
    gnugrep
    binutils.bintools
  ];
  src = ./hello-2.12.1.tar.gz;
  system = builtins.currentSystem;
}
```

一切都清楚了，除了那个 buildInputs。不过它比你现在正在想的任何黑魔法都要简单。

Nix 能够把列表转换成字符串。它先把各个元素转换为字符串，然后用空格分隔并把它们拼接起来：

```console
nix-repl> builtins.toString 123
"123"
nix-repl> builtins.toString [ 123 456 ]
"123 456"
```

回想一下，derivation 也可以被转换成字符串，因此：

```console
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> builtins.toString gnugrep
"/nix/store/g5gdylclfh6d224kqh9sja290pk186xd-gnugrep-2.14"
nix-repl> builtins.toString [ gnugrep gnused ]
"/nix/store/g5gdylclfh6d224kqh9sja290pk186xd-gnugrep-2.14 /nix/store/krgdc4sknzpw8iyk9p20lhqfd52kjmg0-gnused-4.2.2"
```

很简单！buildInputs 变量就是一个以空格分隔各输出路径的字符串，非常适合在 bash 的 for 循环中使用。

## 一个更方便的 derivation 函数

我们已经成功写出了一个可用于多个 autotools 项目的 builder。但在 hello.nix 表达式中，我们指定的那些工具是许多项目共有的；我们不想每次都传入它们。

一个自然的做法是创建一个函数，它接受一个属性集（类似于 derivation 函数所接受的那个），并把它与另一个包含许多项目共有值的属性集合并。

创建 `autotools.nix`：

```nix
pkgs: attrs:
let
  defaultAttrs = {
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./builder.sh ];
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
    ];
    buildInputs = [ ];
    system = builtins.currentSystem;
  };
in
derivation (defaultAttrs // attrs)
```

好了，现在我们得稍微回顾一下 [Nix 函数](05-functions-and-imports.md)。这个 `autotools.nix` 文件的整个 Nix 表达式会求值为一个函数。这个函数接受一个参数 `pkgs`，然后返回一个接受参数 `attrs` 的函数。

函数体很简单，但乍一看可能不太容易理解：

1.  首先把那个神奇的 `pkgs` 属性集放进作用域。

2.  在 let 表达式中我们定义一个辅助变量 `defaultAttrs`，它充当 derivation 中常用属性的一组默认值。

3.  最后我们用那个奇怪的表达式（`defaultAttrs // attrs`）创建 derivation。

[// 运算符](https://nix-doc.mczihan.link/language/operators#update)是作用于两个集合之间的运算符。结果是两个集合的并集。当属性名发生冲突时，右侧集合中的值优先。

所以我们把 `defaultAttrs` 用作基础集合，再添加（或 override，覆盖）来自 `attrs` 的属性。

几个例子就足以说明这个运算符的行为：

```console
nix-repl> { a = "b"; } // { c = "d"; }
{ a = "b"; c = "d"; }
nix-repl> { a = "b"; } // { a = "c"; }
{ a = "c"; }
```

**练习：**完善新的 `builder.sh`，在 `for` 循环中把 `$baseInputs` 与 `$buildInputs` 一起处理。如你所见，我们在 derivation 中传入了这个新变量。我们没有把 buildInputs 与基础输入合并，而是让调用者看到的 buildInputs 保持原样，所以把它们分开了。这只是取舍问题。

接着我们按如下方式重写 `hello.nix`：

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

终于！我们得到了一个非常简洁的软件包描述！下面是一些你可能在学习 Nix 语言的过程中会用到的备注：

- 我们把之前在表达式中用 "with" 引入的内容赋给了 pkgs。别担心，就是这么简单直接。

- mkDerivation 变量是部分应用（partial application）的一个很好的例子，可以把它看作 `(import ./autotools.nix) pkgs`。我们先导入表达式，然后应用 `pkgs` 参数。这会返回一个接受属性集 `attrs` 的函数。

- 我们创建 derivation 时只指定了 name 和 src。如果项目最终还需要其他依赖出现在 PATH 中，我们只需把它们加入 buildInputs（hello.nix 中没有指定，因为它是空的）。

注意我们没有使用任何其他库。在编译时可能需要特殊的 C 编译参数来找到其他库的头文件，链接时则可能需要 ld 参数。

## 结论

Nix 为我们提供了创建 derivation、搭建构建环境以及把结果存入 nix store 的最底层工具。

通过这篇药丸，我们成功地为 autotools 项目创建了一个通用 builder，还创建了一个 `mkDerivation` 函数，它默认组合了 autotools 项目常用的组件，而不必在我们编写的每个软件包中重复它们。

我们正在熟悉 Nix 体系的成长方式：用 Nix 语言创建并组合 derivation。

类比：在 C 中，你在堆上创建对象，然后把它们组合进新的对象里。指针被用来引用其他对象。

在 Nix 中，你创建存储在 nix store 中的 derivation，然后通过创建新的 derivation 来组合它们。存储路径被用来引用其他 derivation。

## 下一篇药丸

……我们将聊一聊运行时依赖。GNU hello world 软件包是自包含的吗？它的运行时依赖是什么？我们只在 "hello" derivation 中通过使用其他 derivation 指定了构建依赖。
