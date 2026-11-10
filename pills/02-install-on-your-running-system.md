# 在现有系统上安装

欢迎来到第二篇 Nix 药丸。在[第一篇](01-why-you-should-give-it-a-try.md)药丸中，我们简要地介绍了 Nix。

现在我们将在现有系统上安装 Nix，并了解安装之后系统发生了哪些变化。**如果你在使用 NixOS，Nix 已经安装好了；你可以直接跳到[下一篇](03-enter-environment.md)药丸。**

安装说明请参阅 Nix 参考手册中的 [Installing Nix](https://nix-doc.mczihan.link/installation/installing-binary) 章节。

## 安装

这一系列文章_不是_关于如何_使用_ Nix 的教程。我们要做的是走一遍 Nix 系统的内部，来理解它的基本原理。

首先要注意的一点是：Nix store 中的 derivation 引用的其他 derivation 本身也在 Nix store 中。它们不使用来自我们系统或其他任何地方的 `libc`。这是一个自成一体的存储仓库，包含了我们引导到任何特定软件包所需的全部软件。

<div class="info">

注意：在多用户安装中（例如 NixOS 所采用的安装方式），store 由 root 所有，多个用户可以通过 Nix 守护进程安装和构建软件。你可以在[这里](https://nix-doc.mczihan.link/installation/installing-binary#multi-user-installation)阅读更多关于多用户安装的内容。

</div>

## Nix store 的起点

让我们从安装命令的输出看起：

```
copying Nix to /nix/store..........................
```

这就是第一篇文章中谈到的 `/nix/store`。我们正在复制引导一个 Nix 系统所必需的软件。你可以看到 bash、coreutils、C 编译器工具链、perl 库、sqlite，以及 Nix 本身连同它自己的工具和 libnix。

你可能已经注意到，`/nix/store` 中不仅可以包含目录，还可以包含文件，但它们始终都采用 «hash-name» 的形式。

## Nix 数据库

复制完 store 之后，安装过程紧接着初始化一个数据库：

```
initialising Nix database...
```

是的，Nix 也有一个数据库。它存放在 `/nix/var/nix/db` 下。这是一个 sqlite 数据库，用于记录 derivation 之间的依赖关系。

它的模式非常简单：有一张有效路径（valid paths）表，把自增整数映射到存储路径。

然后还有一条从路径到其依赖路径的依赖关系记录。

你可以安装 sqlite（`nix-env -iA sqlite -f '<nixpkgs>'`），然后运行 `sqlite3 /nix/var/nix/db/db.sqlite` 来查看这个数据库。

<div class="info">

注意：如果这是你初次安装后第一次使用 Nix，记得先关闭并重新打开你的终端，这样你的 shell 环境才会被更新。

</div>

<div class="warning">

重要：绝对不要手动修改 `/nix/store`。如果你这么做了，它就会与 sqlite 数据库不再同步，除非你_真的_知道自己在做什么。

</div>

## 第一个 profile

安装的下一步，我们遇到了 [profile](https://nix-doc.mczihan.link/package-management/profiles) 的概念：

<pre><code class="hljs">creating /home/nix/.nix-profile
installing 'nix-2.1.3'
building path(s) `/nix/store/a7p1w3z2h8pl00ywvw6icr3g5l9vm5r7-<b>user-environment</b>'
created 7 symlinks in user environment
</code></pre>

Nix 中的 profile 是一个实现回滚（rollback）的通用且便利的概念。profile 用于把分散在多个路径下的组件组合到一个新的统一路径下。不仅如此，profile 还由多个「代（generation）」组成：它们是带版本管理的。每当你修改一个 profile，就会创建一个新代。

代之间可以原子化地切换和回滚，这使得它们非常适合管理对系统的更改。

让我们仔细看看我们的 profile：

<pre><code class="hljs">$ ls -l ~/.nix-profile/
bin -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-<b>nix-2.1.3</b>/bin
[...]
manifest.nix -> /nix/store/q8b5238akq07lj9gfb3qb5ycq4dxxiwm-<b>env-manifest.nix</b>
[...]
share -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-<b>nix-2.1.3</b>/share
</code></pre>

Nix store 中的那个 `nix-2.1.3` derivation 就是 Nix 本身，连同它的二进制文件和库。把这个 derivation「安装」到 profile 中的过程，本质上是通过符号链接在 profile 中复现 `nix-2.1.3` 存储 derivation 的目录层级。

这个 profile 的内容比较特殊，因为我们的 profile 中只安装了一个程序，因此例如 `bin` 目录指向的是唯一被安装的那个程序（Nix 本身）。

但这只是 profile 最新一代的内容。实际上，`~/.nix-profile` 本身是一个指向 `/nix/var/nix/profiles/default` 的符号链接。

而这个又是指向同一目录下 `default-1-link` 的符号链接。是的，这意味着它是 `default` profile 的第一代。

最后，`default-1-link` 是一个指向 Nix store 中那个「user-environment」derivation 的符号链接，也就是你在安装过程中看到的打印输出。

我们将在下一篇文章中进一步讨论 `manifest.nix`。

## Nixpkgs 表达式

安装器的更多输出：

```
downloading Nix expressions from `http://releases.nixos.org/nixpkgs/nixpkgs-14.10pre46060.a1a2851/nixexprs.tar.xz'...
unpacking channels...
created 2 symlinks in user environment
modifying /home/nix/.profile...
```

Nix 表达式使用 [Nix 语言](https://nix-doc.mczihan.link/language/)编写，用于描述软件包以及如何构建它们。[Nixpkgs](https://nixos.org/nixpkgs/) 就是包含所有这些表达式的仓库：<https://github.com/NixOS/nixpkgs>。

安装器从提交 `a1a2851` 下载了软件包描述。

我们发现的第二个 profile 是 channels profile。`~/.nix-defexpr/channels` 指向 `/nix/var/nix/profiles/per-user/nix/channels`，后者指向 `channels-1-link`，再指向一个包含下载下来的 Nix 表达式的 Nix store 目录。

channel 是一组可供下载的软件包和表达式。类似于 Debian 的 stable 和 unstable，这里有 stable 和 unstable 两种 channel。在这套安装中，我们跟踪的是 `nixpkgs-unstable`。

暂时不用担心 Nix 表达式，我们稍后会讲到。

最后，为了方便起见，安装器修改了 `~/.profile`，以便自动进入 Nix environment（环境）。`~/.nix-profile/etc/profile.d/nix.sh` 真正做的事情很简单：把 `~/.nix-profile/bin` 加到 `PATH`，把 `~/.nix-defexpr/channels/nixpkgs` 加到 `NIX_PATH`。我们稍后会讨论 `NIX_PATH`。

读一读 `nix.sh`，它很短。

## FAQ：能把 /nix 换成别的路径吗？

可以，但坚持使用 `/nix` 而不是其他目录是有一个很好的理由的。所有 derivation 都通过绝对路径引用其他 derivation。我们在第一篇文章中看到，bash 引用的是 `/nix/store` 下某个特定绝对路径上的 `glibc`。

你可以亲自验证，看到多个 bash derivation 也不用担心：

```console
$ ldd /nix/store/*bash*/bin/bash
[...]
```

把 store 放在 `/nix` 意味着我们可以直接使用来自 nixos.org 的二进制缓存（就像你从 debian 镜像获取软件包一样），否则的话：

- `glibc` 会被安装在 `/foo/store` 下

- 于是 bash 就得指向 `/foo/store` 下的 `glibc`，而不是 `/nix/store` 下的

- 这样二进制缓存就派不上用场了，因为我们需要一个_不同的_ bash，结果我们只好自己重新编译一切。

总之，`/nix` 是放置 store 的一个合理位置。

## 结论

我们在系统上安装了 Nix，它完全隔离，并归 `nix` 用户所有，因为我们还在逐步熟悉这个新系统。

我们学到了一些新概念，比如 profile 和 channel。特别是，借助 profile 我们能够管理软件包组合的多个代，而借助 channel 我们能够从 `nixos.org` 下载二进制文件。

安装过程把所有东西都放在了 `/nix` 下，并在 Nix 用户的主目录里放了一些符号链接。这是因为每个用户都可以在自己的环境中安装和使用软件。

我希望我没有遗漏什么，以至于让你觉得幕后有什么魔法在运作。一切不过是把组件放进 store，再用符号链接把这些组件连在一起。

## 下一篇药丸……

……我们将进入 Nix 环境，学习如何与 store 交互。
