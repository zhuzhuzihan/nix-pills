# 第一个 derivation

欢迎来到第六篇 Nix 药丸。在上一篇[第五篇药丸](05-functions-and-imports.md)中我们介绍了函数与导入。函数与导入是非常简单的概念，却可以用来构建复杂的抽象和模块组合，从而搭建一个灵活的 Nix 系统。

在这篇文章中，我们终于要开始编写 derivation（派生）了。从文件系统的视角来看，derivation 是 Nix 系统的构建基石。Nix 语言就是用来描述这些 derivation 的。

再次提醒进入 Nix 环境的方法：`source ~/.nix-profile/etc/profile.d/nix.sh`

## derivation 函数

[derivation 内置函数](https://nix-doc.mczihan.link/language/derivations)用于创建 derivation。我建议你阅读 Nix 手册中关于 derivation 内置函数的链接。从 Nix 语言的视角来看，derivation 只是一个集合（set），带有若干属性。因此，你可以像对待其他任何东西一样，用变量来传递 derivation。

这正是它真正的威力所在。

`derivation` 函数的第一个参数是一个集合。这个集合至少需要以下三个属性：

- name：derivation 的名称。在 nix store 中格式是 hash-name，这就是那个 name。

- system：derivation 可以在其上构建的系统名称。例如 x86_64-linux。

- builder：构建 derivation 的二进制程序。

首先，在 nix 看来我们的系统名称是什么？

```console
nix-repl> builtins.currentSystem
"x86_64-linux"
```

让我们试着伪造一个系统名称：

```console
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> d
«derivation /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv»
```

哦哦，这是什么？它构建了 derivation 吗？没有，但它**确实创建了 .drv 文件**。`nix repl` 不会构建 derivation，除非你明确要求它这么做。

## 关于 .drv 文件的题外话

那个 `.drv` 文件是什么？它是如何构建这个 derivation 的规范描述，去掉了所有 Nix 语言的繁杂细节。

在继续之前，先与 C 语言做些类比：

- `.nix` 文件就像 `.c` 文件。

- `.drv` 文件是中间文件，就像 `.o` 文件。`.drv` 描述如何构建一个 derivation；它是最精简的信息。

- out path（输出路径）则是构建的产物。

如你所见，drv path 和 out path 都存储在 nix store 中。

那个 `.drv` 文件里有什么？你可以直接阅读它，但最好把它漂亮地打印出来：

<div class="info">

注意：如果你的 nix 版本没有 `nix derivation show`，请改用 `nix show-derivation`。

</div>

```console
$ nix derivation show /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
{
  "/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
      }
    },
    "inputSrcs": [],
    "inputDrvs": {},
    "platform": "mysystem",
    "builder": "mybuilder",
    "args": [],
    "env": {
      "builder": "mybuilder",
      "name": "myname",
      "out": "/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname",
      "system": "mysystem"
    }
  }
}
```

好，我们可以看到一个 out path，但它还不存在。我们从未告诉 Nix 去构建它，但我们事先就知道构建产物会在哪里。为什么？

想一想，如果 Nix 只因为我们在 Nix 中访问了 derivation 就去构建它，那么假如它是 Firefox，我们就得等上很长时间。这就是为什么 Nix 让我们事先知道路径并继续对 Nix 表达式求值，但它仍然是空的，因为从未进行过任何构建。

重要提示：在当前版本的 Nix 中，out path 的哈希仅基于输入的 derivation，而不是基于构建产物的内容。不过，也可以有[内容寻址](https://en.wikipedia.org/wiki/Content-addressable_storage)的 derivation，例如针对 tarball，我们稍后会看到。

那个 `.drv` 里有很多东西是空的，不过我会为你写一份 [.drv 格式](http://nixos.org/~eelco/pubs/phd-thesis.pdf)的摘要：

1.  输出路径（可以有多个）。默认情况下，nix 会创建一个名为 "out" 的 out path。

2.  输入 derivation 的列表。它是空的，因为我们没有引用任何其他 derivation。否则，这里会是一个由其他 .drv 文件组成的列表。

3.  系统和 builder 可执行文件（是的，它是个伪造的）。

4.  然后是传递给 builder 的环境变量列表。

就是这样，这些就是构建我们的 derivation 所需的最少信息。

重要说明：传递给 builder 的环境变量只是你在 .drv 中看到的那些，再加上一些其他与 Nix 相关的配置（核心数、临时目录等）。builder 不会从你正在运行的 shell 继承任何变量，否则构建会受到[不确定性](https://wiki.debian.org/ReproducibleBuilds)的影响。

回到我们的伪造 derivation。

让我们构建这个彻头彻尾伪造的 derivation：

```console
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> :b d
[...]
these derivations will be built:
  /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
building path(s) `/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname'
error: a `mysystem' is required to build `/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv', but I am a `x86_64-linux'
```

`:b` 是 `nix repl` 特有的命令，用于构建一个 derivation。你可以用 `:?` 查看更多命令。所以在输出中可以看到，它把 `.drv` 作为如何构建 derivation 的信息。然后它说它正试图产生我们的 out path。最后是我们一直等待的错误：这个 derivation 无法在我们的系统上构建。

我们是在 `nix repl` 里进行构建的，但如果我们不想用 `nix repl` 呢？你可以用以下命令**实现（realise）**一个 `.drv`：

```console
$ nix-store -r /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
```

你会得到与之前相同的输出。

让我们修正 system 属性：

```console
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = builtins.currentSystem; }
nix-repl> :b d
[...]
build error: invalid file name `mybuilder'
```

前进了一步：当然，那个 `mybuilder` 可执行文件并不真的存在。先停下来想一想。

## derivation 集合里有什么

先检视 derivation 函数的返回值是很有用的。在这个例子里，返回值是一个普通的集合：

```console
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> builtins.isAttrs d
true
nix-repl> builtins.attrNames d
[ "all" "builder" "drvAttrs" "drvPath" "name" "out" "outPath" "outputName" "system" "type" ]
```

你可以猜到 `builtins.isAttrs` 是干什么的：如果参数是一个集合，它就返回 true。而 `builtins.attrNames` 返回给定集合的键列表。你可能会说，这是一种反射。

从 drvAttrs 开始：

```console
nix-repl> d.drvAttrs
{ builder = "mybuilder"; name = "myname"; system = "mysystem"; }
```

这基本上就是我们传给 derivation 函数的输入。`d.name`、`d.system` 和 `d.builder` 属性也恰好就是我们给出的输入。

```console
nix-repl> (d == d.out)
true
```

所以 out 就是 derivation 本身，这看起来很奇怪，但原因是我们只有一个输出。这也是 `d.all` 是单元素列表的原因。我们稍后会看到多个输出的情况。

`d.drvPath` 是 `.drv` 文件的路径：`/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv`。

有趣的是 `type` 属性。它的值是 `"derivation"`。Nix 确实对带有 type derivation 的集合施加了一点魔法，但并不多。为了帮助你理解，你可以自己创建一个带该 type 的集合，它就是一个简单的集合：

```console
nix-repl> { type = "derivation"; }
«derivation ???»
```

当然，它没有其他任何信息，所以 Nix 不知道该说什么 :-) 但你明白了，`type = "derivation"` 只是一种约定，让 Nix 和我们都能明白这个集合是一个 derivation。

在编写软件包时，我们关心的是输出。其他元数据则是 Nix 用来知道如何创建 drv path 和 out path 所必需的。

`outPath` 属性是 nix store 中的构建路径：`/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname`。

## 引用其他 derivation

就像其他软件包管理器中的依赖一样，我们如何引用其他软件包？就磁盘上的文件而言，我们如何引用其他 derivation？我们使用 `outPath`。`outPath` 描述了该 derivation 的文件所在的位置。为了方便起见，Nix 能够把一个 derivation 集合转换成字符串。

```console
nix-repl> d.outPath
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
nix-repl> builtins.toString d
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
```

只要存在 `outPath` 属性，Nix 就会进行"集合到字符串的转换"（很像其他语言中的 toString 方法）：

```console
nix-repl> builtins.toString { outPath = "foo"; }
"foo"
nix-repl> builtins.toString { a = "b"; }
error: cannot coerce a set to a string, at (string):1:1
```

假设我们想使用 coreutils 中的二进制程序（先忽略 nixpkgs 等内容）：

```console
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> coreutils
«derivation /nix/store/1zcs1y4n27lqs0gw4v038i303pb89rw6-coreutils-8.21.drv»
nix-repl> builtins.toString coreutils
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21"
```

暂时不用操心 nixpkgs 的细节，只需知道我们往当前作用域里添加了一系列变量。其中之一就是 coreutils。它是你在其他 Linux 发行版上都见过的 coreutils 软件包的 derivation。它包含 GNU/Linux 系统的基础二进制程序（nix store 中可能同时存在多个 coreutils 的 derivation，不必担心）：

```console
$ ls /nix/store/*coreutils*/bin
[...]
```

提醒一下，可以在字符串中用 `${...}` 插值 Nix 表达式：

```console
nix-repl> "${d}"
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
nix-repl> "${coreutils}"
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21"
```

这非常方便，因为这样一来，我们就可以这样引用例如 bin/true 这个二进制程序：

```console
nix-repl> "${coreutils}/bin/true"
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21/bin/true"
```

## 一个几乎能工作的 derivation

在上一次尝试中，我们使用了一个伪造的 builder `mybuilder`，它显然并不存在。但我们可以使用例如 bin/true，它总是以 0（成功）退出。

```console
nix-repl> :l <nixpkgs>
nix-repl> d = derivation { name = "myname"; builder = "${coreutils}/bin/true"; system = builtins.currentSystem; }
nix-repl> :b d
[...]
builder for `/nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv' failed to produce output path `/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname'
```

又前进了一步，它执行了 builder（bin/true），但 builder 当然没有创建 out path，它只是以 0 退出了。

显而易见的说明：每当我们修改 derivation，就会生成一个新的哈希。

既然我们引用了另一个 derivation，现在来看看新的 `.drv`：

```console
$ nix derivation show /nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv
{
  "/nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname"
      }
    },
    "inputSrcs": [],
    "inputDrvs": {
      "/nix/store/hixdnzz2wp75x1jy65cysq06yl74vx7q-coreutils-8.29.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29/bin/true",
    "args": [],
    "env": {
      "builder": "/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29/bin/true",
      "name": "myname",
      "out": "/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname",
      "system": "x86_64-linux"
    }
  }
}
```

啊哈！Nix 给我们的 myname.drv 添加了一个依赖，它就是 coreutils.drv。在进行我们的构建之前，Nix 应当先构建 coreutils.drv。但由于 coreutils 已经在我们的 nix store 中，无需任何构建，它已经在那里了，out path 为 `/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29`。

## derivation 何时被构建

Nix 不会在 Nix 表达式**求值期间**构建 derivation。事实上，这就是为什么我们一开始要在 `nix repl` 中使用 ":b drv"，或者使用 nix-store -r。

Nix 中做了一个重要的区分：

- **实例化/求值阶段（Instantiate/Evaluation time）**：Nix 表达式被解析、解释并最终返回一个 derivation 集合。在求值期间，你可以引用其他 derivation，因为 Nix 会创建 .drv 文件，而且我们会事先知道 out path。这是通过 [nix-instantiate](https://nix-doc.mczihan.link/command-ref/nix-instantiate) 实现的。

- **实现/构建阶段（Realise/Build time）**：来自 derivation 集合的 .drv 被构建，首先构建其 .drv 输入（构建依赖）。这是通过 [nix-store -r](https://nix-doc.mczihan.link/command-ref/nix-store/realise) 实现的。

把它想象成 C/C++ 项目中的编译期和链接期。你先把所有源文件编译成目标文件，然后把目标文件链接成一个可执行文件。

在 Nix 中，先是 Nix 表达式（通常在 .nix 文件中）被"编译"成 .drv，然后每个 .drv 被构建，其产物被安装到相应的 out path 中。

## 结语

为 Nix 创建一个软件包真的有那么复杂吗？不，并不复杂。

我们正在梳理 Nix derivation 的基本原理，以理解它们如何工作、如何表示。Nix 中的打包当然会比这更简单，但在这篇文章里我们还没走到那一步。还需要更多 Nix 药丸。

借助 derivation 函数，我们提供了一组关于如何构建软件包的信息，并得到关于软件包被构建到哪里的信息。只要存在 `outPath`，Nix 就会把集合转换为字符串；这非常方便。有了它，引用其他 derivation 就很容易了。

当 Nix 构建一个 derivation 时，它先从 derivation 表达式创建一个 .drv 文件，再用它来构建输出。它对所有依赖（输入）递归地执行这一过程。它像一台机器一样"执行" .drv 文件。其实并没有太多魔法。

## 下一篇药丸

……我们将终于编写出第一个**能工作的** derivation。是的，这篇文章叫"第一个 derivation"，但我从没说过它能工作 ;)
