# 一个能工作的 derivation

## 简介

欢迎来到第七篇 Nix 药丸。在上一篇[第六篇药丸](06-our-first-derivation.md)中，我们介绍了 Nix 语言中 derivation（派生）的概念——如何定义一个原始的 derivation，以及如何（尝试）构建它。

在这篇文章中，我们沿着这条路继续前进，创建一个真正能构建出东西的 derivation。然后，我们尝试打包一个真实的程序：给定一套现成的工具链，我们编译一个简单的 C 文件，并据此创建一个 derivation。

再次提醒进入 Nix 环境的方法：`source ~/.nix-profile/etc/profile.d/nix.sh`

## 用脚本作为 builder

运行一串命令来构建东西，最简单的方式是什么？一个 bash 脚本。我们将编写一个自定义的 bash 脚本，并把它用作我们的 builder。给定一个 `builder.sh`，我们希望 derivation 运行 `bash builder.sh`。

我们不会在 `builder.sh` 中使用 [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))，因为在编写它的时候，我们并不知道 nix store 中 bash 的路径。是的，连 bash 也在 nix store 里，所有东西都在那里。

我们也不会使用 `/usr/bin/env`，因为那样就会失去 Nix 无状态（stateless）这一酷炫特性。更不用提构建时 `PATH` 会被清空，反正也找不到 `bash`。

总结一下：我们希望 builder 是 `bash`，并把 `builder.sh` 作为参数传给它。事实证明，"derivation" 函数接受一个可选的 `args` 属性，用于向 builder 可执行文件传递参数。

首先，在当前目录写下我们的 `builder.sh`：

```sh
declare -xp
echo foo > $out
```

命令 `declare -xp` 列出已导出的变量（`declare` 是一个 bash 内置函数）。正如上一篇药丸中所讲的，Nix 会计算 derivation 的输出路径。生成的 `.drv` 文件包含一份传递给 builder 的环境变量列表，其中之一就是 `$out`。

我们要做的，就是在 `$out` 这个路径下创建点什么，无论是一个文件还是一个目录。这里我们创建的是一个文件。

另外，我们会在构建过程中打印出环境变量。我们不能用 env 来做这件事，因为 env 属于 coreutils，而我们还没有对它的依赖。目前我们只有 bash。

就像上一篇药丸中的 coreutils 一样，我们从神奇的 nixpkgs 那里免费获得一个现成的 bash：

```console
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> "${bash}"
"/nix/store/ihmkc7z2wqk3bbipfnlh0yjrlfkkgnv6-bash-4.2-p45"
```

于是用惯用的技巧，我们就可以引用 bin/bash 并创建我们的 derivation：

```console
nix-repl> d = derivation { name = "foo"; builder = "${bash}/bin/bash"; args = [ ./builder.sh ]; system = builtins.currentSystem; }
nix-repl> :b d
[1 built, 0.0 MiB DL]

this derivation produced the following outputs:
  out -> /nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo
```

我们做到了！`/nix/store/w024zci0x1hh1wj6gjq0jagkc1sgrf5r-foo` 的内容确实是 foo。我们构建了第一个 derivation。

注意，我们使用的是 `./builder.sh` 而不是 `"./builder.sh"`。这样它会被解析为一个路径，Nix 会施一些我们稍后会讲到的魔法。试试用字符串版本，你会发现它找不到 `builder.sh`。这是因为它会相对于临时构建目录去寻找它。

## builder 的环境

我们可以用 `nix-store --read-log` 来查看 builder 产生的日志：

```console
$ nix-store --read-log /nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo
declare -x HOME="/homeless-shelter"
declare -x NIX_BUILD_CORES="4"
declare -x NIX_BUILD_TOP="/tmp/nix-build-foo.drv-0"
declare -x NIX_LOG_FD="2"
declare -x NIX_STORE="/nix/store"
declare -x OLDPWD
declare -x PATH="/path-not-set"
declare -x PWD="/tmp/nix-build-foo.drv-0"
declare -x SHLVL="1"
declare -x TEMP="/tmp/nix-build-foo.drv-0"
declare -x TEMPDIR="/tmp/nix-build-foo.drv-0"
declare -x TMP="/tmp/nix-build-foo.drv-0"
declare -x TMPDIR="/tmp/nix-build-foo.drv-0"
declare -x builder="/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash"
declare -x name="foo"
declare -x out="/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo"
declare -x system="x86_64-linux"
```

让我们检视一下构建过程中打印出的这些环境变量。

- `$HOME` 不是你的 home 目录，而且 `/homeless-shelter` 完全不存在。我们强迫软件包在构建过程中不依赖 `$HOME`。

- `$PATH` 和 `$HOME` 玩的是同一套把戏。

- `$NIX_BUILD_CORES` 和 `$NIX_STORE` 是 [nix 配置选项](https://nix.dev/manual/nix/stable/command-ref/conf-file)。

- `$PWD` 和 `$TMP` 清楚地表明 nix 创建了一个临时构建目录。

- 然后 `$builder`、`$name`、`$out` 和 `$system` 是由 .drv 文件的内容设置的变量。

这就是我们能在 derivation 中使用 `$out` 并往里面放东西的原因。这就像是 Nix 在 nix store 中为我们预留了一个槽位，而我们必须把它填上。

用 autotools 的术语来说，`$out` 就是 `--prefix` 路径。是的，不是 make 的 `DESTDIR`，而是 `--prefix`。这就是无状态打包的精髓。你不是把软件包安装到 `/` 下的某个全局公共路径，而是把它安装到你在 nix store 中那个槽位下的本地隔离路径里。

## .drv 的内容

这一次我们给 derivation 添加了别的东西：args 属性。让我们看看与上一篇药丸相比，这给 .drv 带来了什么变化：

```console
$ nix derivation show /nix/store/i76pr1cz0za3i9r6xq518bqqvd2raspw-foo.drv
{
  "/nix/store/i76pr1cz0za3i9r6xq518bqqvd2raspw-foo.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo"
      }
    },
    "inputSrcs": [
      "/nix/store/lb0n38r2b20r8rl1k45a7s4pj6ny22f7-builder.sh"
    ],
    "inputDrvs": {
      "/nix/store/hcgwbx42mcxr7ksnv0i1fg7kw6jvxshb-bash-4.4-p19.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
    "args": [
      "/nix/store/lb0n38r2b20r8rl1k45a7s4pj6ny22f7-builder.sh"
    ],
    "env": {
      "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
      "name": "foo",
      "out": "/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo",
      "system": "x86_64-linux"
    }
  }
}
```

与通常的 .drv 几乎一样，只是其中多了一份传递给 builder（bash）的参数列表，里面有 `builder.sh`……在 nix store 里..？Nix 会自动把构建所需的文件或目录复制进 store，以确保它们在构建过程中不会被改动，并确保部署是无状态的、与构建机器无关的。`builder.sh` 不仅出现在传给 builder 的参数中，还出现在输入源（input sources）里。

鉴于 `builder.sh` 是一个普通文件，它没有与之关联的 .drv。其存储路径是根据文件名和其内容的哈希计算出来的。存储路径将在[后面某篇药丸](18-nix-store-paths.md)中详细讲解。

## 打包一个简单的 C 程序

先写一个名为 `simple.c` 的简单 C 程序：

```c
#include <stdio.h>

void main() {
    puts("Simple!");
}
```

以及它的 `simple_builder.sh`：

```sh
export PATH="$coreutils/bin:$gcc/bin"
mkdir $out
gcc -o $out/simple $src
```

暂时不用太操心这些变量从哪里来；先写下 derivation 并构建它：

```console
nix-repl> :l <nixpkgs>
nix-repl> simple = derivation { name = "simple"; builder = "${bash}/bin/bash"; args = [ ./simple_builder.sh ]; gcc = gcc; coreutils = coreutils; src = ./simple.c; system = builtins.currentSystem; }
nix-repl> :b simple
this derivation produced the following outputs:

  out -> /nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple
```

现在你可以在 shell 中运行 `/nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple/simple` 了。

## 解析

我们在 derivation 调用中新增了两个属性：`gcc` 和 `coreutils`。在 `gcc = gcc;` 中，左边的名字是 derivation 集合中的属性名，右边的名字引用的是来自 nixpkgs 的 gcc derivation。coreutils 同理。

我们还添加了 `src` 属性，没有什么魔法——它只是一个名字，被赋值为路径 `./simple.c`。与 `simple-builder.sh` 一样，`simple.c` 也会被添加进 store。

诀窍在于：传给 `derivation` 的集合中的每个属性都会被转换成字符串，并作为环境变量传递给 builder。builder 正是这样获得对 coreutils 和 gcc 的访问能力的：当被转换为字符串时，这些 derivation 求值结果就是它们的输出路径，再在后面拼上 `/bin`，就找到了它们的二进制程序。

`src` 变量也是一样。`$src` 是 `simple.c` 在 nix store 中的路径。作为练习，你可以把 .drv 文件漂亮地打印出来。你会看到 `simple_builder.sh` 和 `simple.c` 出现在输入源中，还有 bash、gcc 和 coreutils 的 .drv 文件。上述新增的环境变量也会一并出现。

在 `simple_builder.sh` 中，我们为 gcc 和 coreutils 的二进制程序设置了 `PATH`，这样我们的构建脚本才能找到 mkdir 和 gcc 等必要的工具。

然后我们把 `$out` 创建为一个目录，并把二进制文件放进去。注意，gcc 是通过 `PATH` 环境变量找到的，但同样可以显式地用 `$gcc/bin/gcc` 来引用。

## 告别 `nix repl`

退出 nix repl，写一个文件 `simple.nix`：

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "simple";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./simple_builder.sh ];
  gcc = pkgs.gcc;
  coreutils = pkgs.coreutils;
  src = ./simple.c;
  system = builtins.currentSystem;
}
```

现在你可以用 `nix-build simple.nix` 来构建它。这会在当前目录创建一个符号链接 `result`，指向该 derivation 的 out path。

nix-build 做两件事：

- [nix-instantiate](https://nix.dev/manual/nix/stable/command-ref/nix-instantiate)：解析并求值 `simple.nix`，返回与解析出的 derivation 集合对应的 .drv 文件。

- [`nix-store -r`](https://nix.dev/manual/nix/stable/command-ref/nix-store/realise)：实现（realise）该 .drv 文件，也就是真正构建它。

最后，它会创建那个符号链接。

在 `simple.nix` 的第二行，有一个 `import` 函数调用。回顾一下：`import` 接受一个参数，即要加载的 nix 文件。在这个例子里，该文件的内容求值结果是一个函数。

随后，我们用空集合调用这个函数。我们在[第五篇药丸](05-functions-and-imports.md)中已经见过这种写法。再强调一次：`import <nixpkgs> {}` 是在调用两个函数，而不是一个。把它读作 `(import <nixpkgs>) {}` 会更清楚。

nixpkgs 函数返回的值是一个集合；更具体地说，是一个由 derivation 组成的集合。把 `import <nixpkgs> {}` 放进 `let` 表达式会创建局部变量 `pkgs` 并把它引入作用域。它的效果类似于我们在 nix repl 中使用的 `:l <nixpkgs>`，让我们可以方便地访问 `bash`、`gcc` 和 `coreutils` 等 derivation，只是这些 derivation 必须显式地作为 `pkgs` 集合的成员来引用（例如 `pkgs.bash`，而不是只写 `bash`）。

下面是使用 `inherit` 关键字改写后的 `simple.nix`：

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "simple";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./simple_builder.sh ];
  inherit (pkgs) gcc coreutils;
  src = ./simple.c;
  system = builtins.currentSystem;
}
```

这里我们也顺便介绍一下 [`inherit` 关键字](https://nix.dev/manual/nix/stable/language/constructs#inheriting-attributes)。`inherit foo;` 等价于 `foo = foo;`。类似地，`inherit gcc coreutils;` 等价于 `gcc = gcc; coreutils = coreutils;`。最后，`inherit (pkgs) gcc coreutils;` 等价于 `gcc = pkgs.gcc; coreutils = pkgs.coreutils;`。

这种语法只在集合内有意义。这里没有任何魔法，只是一种便利写法，避免为属性名和作用域中的值重复书写同一个名字。

## 下一篇药丸

我们将把 builder 泛化。你可能已经注意到，这篇文章里我们写了两个各自独立的 `builder.sh` 脚本。我们更希望有一个通用的构建脚本，尤其是因为每个构建脚本都会进入 nix store：有点浪费。

_在 Nix 中打包东西真的有那么难吗？不难_，我们正在学习的是 Nix 的基本原理。
