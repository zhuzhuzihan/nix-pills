# Nix 搜索路径

欢迎来到第 15 篇 Nix 药丸。在上一篇[第 14 篇](14-override-design-pattern.md)药丸中，我们介绍了 "override" 模式，它通过传入不同的输入来编写 derivation 的变体，十分有用。

假设你跟上了之前的文章，我希望你现在已经准备好理解 `nixpkgs` 了。但我们首先得在系统中找到 `nixpkgs`！所以这一步就是：介绍 nix 工具使用的一些选项和环境变量。

## NIX_PATH

[NIX_PATH 环境变量](https://nix-doc.mczihan.link/command-ref/env-common?highlight=NIX_PATH)非常重要。它非常类似于 `PATH` 环境变量。语法也很相似，多个路径之间用冒号 `:` 分隔。Nix 随后会从左到右在这些路径中搜索某个东西。

谁会使用 `NIX_PATH`？是 nix 表达式！是的，`NIX_PATH` 对 nix 工具本身来说用处不大，它主要是在编写 nix 表达式时使用的。

例如在 shell 中，当你执行命令 `ping` 时，它会在 `PATH` 的目录中被搜索，找到的第一个就是要使用的那个。

在 nix 中完全一样，只是语法不同。你不用只输入 `ping`，而必须输入 `<ping>`。是的，我知道……你可能已经想到了 `<nixpkgs>`。不过，别在这里停下来，我们继续往下看。

`NIX_PATH` 有什么用？Nix 表达式可以引用一个「抽象」路径，比如 `<nixpkgs>`，并且可以从命令行覆盖它。

为了方便，我们将使用 `nix-instantiate --eval` 来做测试。提醒一下，[nix-instantiate](https://nix-doc.mczihan.link/command-ref/nix-instantiate) 用于求值 nix 表达式并生成 .drv 文件。这里我们对构建 derivation 不感兴趣，所以求值就够了。它可以用于一次性的表达式。

## 稍微伪造一下

从 nix 的角度看这没什么用，但我想它有助于你的理解。让我们把 `PATH` 本身当作 `NIX_PATH`，尝试定位 `ping`（如果你没有这个命令，就用别的二进制文件）。

```console
$ nix-instantiate --eval -E '<ping>'
error: file `ping' was not found in the Nix search path (add it using $NIX_PATH or -I)
$ NIX_PATH=$PATH nix-instantiate --eval -E '<ping>'
/bin/ping
$ nix-instantiate -I /bin --eval -E '<ping>'
/bin/ping
```

很好。第一次尝试时 nix 显然说在搜索路径的任何地方都找不到它。注意 `-I` 选项接受单个目录。通过 `-I` 添加的路径优先于 `NIX_PATH`。

`NIX_PATH` 还接受另一种非常便利的语法："`somename=somepath`"。也就是说，我们不再在某个目录中搜索某个名称，而是直接指定该名称的确切值。

```console
$ NIX_PATH="ping=/bin/ping" nix-instantiate --eval -E '<ping>'
/bin/ping
$ NIX_PATH="ping=/bin/foo" nix-instantiate --eval -E '<ping>'
error: file `ping' was not found in the Nix search path (add it using $N
```

注意第二种情况中 Nix 如何检查该路径是否存在。

## 通向仓库的路径

你是出于好奇心吧？

```console
$ nix-instantiate --eval -E '<nixpkgs>'
/home/nix/.nix-defexpr/channels/nixpkgs
$ echo $NIX_PATH
nixpkgs=/home/nix/.nix-defexpr/channels/nixpkgs
```

根据你添加 channel 的方式等，你可能有不同的路径。无论如何，这就是全部要点。我们在 nix 表达式中使用的 `<nixpkgs>` 这个陌生家伙，指的是由 `NIX_PATH` 指定的文件系统中的一个路径。

你可以列出那个目录，并发现它只是 nixpkgs 仓库在某个特定提交处的检出（提示：`.version-suffix`）。

`NIX_PATH` 变量由 `nix.sh` 导出，这也是为什么我一直要求你在每篇文章的开头[source nix.sh](https://nix-doc.mczihan.link/installation/env-variables)。

你可能会想：那我是否也可以把 [nixpkgs](https://github.com/NixOS/nixpkgs) 路径指定为别的东西，例如 `nixpkgs` 的一个 `git checkout`？是的，可以，而且我鼓励你这样做。我们将在下一篇药丸中讨论这个。

那么，为我们的仓库定义一个路径吧！假设所有的 `default.nix`、`graphviz.nix` 等文件都在 `/home/nix/mypkgs` 下：

```console
$ export NIX_PATH=mypkgs=/home/nix/mypkgs:$NIX_PATH
$ nix-instantiate --eval '<mypkgs>'
{ graphviz = <code>; graphvizCore = <code>; hello = <code>; mkDerivation = <code>; }
```

是的，`nix-build` 也接受尖括号形式的路径。我们首先对整个仓库（`default.nix`）求值，然后选取 `graphviz` 属性。

## 关于 nix-env 的重要说明

[nix-env](https://nix-doc.mczihan.link/command-ref/nix-env) 命令与 `nix-instantiate` 和 `nix-build` 略有不同。`nix-instantiate` 和 `nix-build` 需要一个起始的 nix 表达式，而 `nix-env` 不需要。

刚开始时你可能会被这个概念难住，你可能以为 `nix-env` 使用 `NIX_PATH` 来查找 `nixpkgs` 仓库。但并不是这样。

`nix-env` 命令使用的是 `~/.nix-defexpr`，它默认也是 `NIX_PATH` 的一部分，但这只是一个巧合。如果你清空 `NIX_PATH`，`nix-env` 仍然能够找到 derivation，因为还有 `~/.nix-defexpr`。

所以如果你在自己的仓库中运行 `nix-env -i graphviz`，它安装的将是 nixpkgs 的那个。即使你把 `NIX_PATH` 设置为指向你的仓库，结果也一样。

为了指定 `~/.nix-defexpr` 的替代者，可以使用 -f 选项：

```console
$ nix-env -f '<mypkgs>' -i graphviz
warning: there are multiple derivations named `graphviz'; using the first one
replacing old `graphviz'
installing `graphviz'
```

咦，为什么它说还有另一个名为 `graphviz` 的 derivation？因为我们仓库中的 `graphviz` 和 `graphvizCore` 两个属性的 derivation 名称都叫 "graphviz"：

```console
$ nix-env -f '<mypkgs>' -qaP
graphviz      graphviz
graphvizCore  graphviz
hello         hello
```

默认情况下，`nix-env` 会解析所有 derivation，并使用 derivation 名称来解释命令行。所以在这个例子中，"graphviz" 匹配到了两个 derivation。或者，像 `nix-build` 那样，可以使用 -A 来指定属性名而不是 derivation 名：

```console
$ nix-env -f '<mypkgs>' -i -A graphviz
replacing old `graphviz'
installing `graphviz'
```

这种形式不仅更精确，还更快，因为 `nix-env` 不必解析所有的 derivation。

补充一点：你必须用 -A 来安装 `graphvizCore`，因为不加 -A 开关的话它是有歧义的。

总结一下：在玩 nix 时，可能会出现 `nix-env` 挑选的 derivation 与 `nix-build` 不同的情况。这时你多半指定了 `NIX_PATH`，而 `nix-env` 却在查看 `~/.nix-defexpr`。

为什么 `nix-env` 会有这种不同的行为？我自己也不确切知道，但可能的答案是：

- `nix-env` 试图保持通用，因此它不在 `NIX_PATH` 中查找 `nixpkgs`，而是在 `~/.nix-defexpr` 中查找。

- `nix-env` 能够通过查看所有可能的 derivation，把多个树合并到 `~/.nix-defexpr` 中。

**你还可能在安装时无法匹配到某个 derivation 名称**，原因就是上面描述的 derivation 名称与 -A 开关的问题。也许在这种情况下，`nix-env` 想对默认的用户配置更友好一些。

这对你来说可能有道理，也可能没有，或者只是历史原因如此，但目前的运作方式就是这样，除非有人想出更好的主意。

## 结论

`NIX_PATH` 变量是 nix 在使用尖括号语法时所使用的搜索路径。可以在 nix 表达式内部引用「抽象」路径，并通过 `NIX_PATH` 或 nix 工具常用的 `-I` 标志来定义「具体」路径。

我们还解释了一些让新手感到困惑的 `nix-env` 行为。`nix-env` 工具并不是用 `NIX_PATH` 来搜索软件包的，而是用 `~/.nix-defexpr`。当心这一点！

总体而言，不要滥用 `NIX_PATH`，在编写自己的 nix 表达式时应尽可能使用相对路径。当然，对于仓库中的 `<nixpkgs>` 来说，那是 `NIX_PATH` 的绝佳用法。而在仓库内部，则应使用 `./hello.nix` 这样的相对路径来引用表达式。

## 下一篇药丸

……我们将终于深入 `nixpkgs`。我们在这个系列中开发的许多技术，比如 `mkDerivation`、`callPackage`、`override` 等，都已经存在于 `nixpkgs` 中了，当然是以更好的形式。随着时间推移，这些基础工具会被社区不断增强，添加更多特性，以越来越通用的方式处理越来越多的用例。
