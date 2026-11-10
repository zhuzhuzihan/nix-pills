# 垃圾回收器 {#garbage-collector}

欢迎来到第 11 篇 Nix 药丸。在上一篇[第 10 篇药丸](10-developing-with-nix-shell.md)中，我们对比了 `nix-build` 提供的隔离构建环境与 `nix-shell` 提供的隔离开发 shell。借助 `nix-shell`，我们可以在一个与 `nix-build` 提供的环境几乎一致的环境里调试、修改并手动构建软件。

今天，我们将不再聚焦于打包，而是关注 Nix 的一个关键组件：垃圾回收器。在使用 Nix 工具时，我们经常在构建 derivation（派生），这既包括 `.drv` 文件，也包括输出路径。这些产物会进入 Nix store，占用我们的存储空间。最终我们可能希望通过删除不再需要的 derivation 来释放一些空间。这正是第 11 篇药丸的主题。默认情况下，Nix 在自动判断哪些 derivation 是"需要的"时采取相对保守的策略。在本篇药丸中，我们还将看到一种执行更具破坏性的升级和删除操作的技术。

## 垃圾回收是如何工作的？

带有垃圾回收器的编程语言使用"垃圾回收器（GC）根"（GC roots）的概念来追踪"存活"的对象。GC 根是始终被视为"存活"的对象（除非被显式地移除出 GC 根）。垃圾回收过程从 GC 根出发，递归地将对象引用标记为"存活"。其余所有对象都可以被回收并删除。

Nix 的垃圾回收操作的不是对象，而是存储路径，[GC 根本身就是存储路径](https://nix-doc.mczihan.link/package-management/garbage-collector-roots)。与 `dpkg` 或 `rpm` 这类传统软件包管理器相比，这种方式要有原则得多——后者可能会留下无用的软件包或悬空的文件。

其实现对用户来说非常简单且透明。主要的 GC 根存放在 `/nix/var/nix/gcroots` 之下。如果该目录中存在指向某个存储路径的符号链接，那么被链接的存储路径就是一个 GC 根。

Nix 允许这个目录包含子目录：它会简单地递归遍历这些子目录，寻找指向存储路径的符号链接。当遇到一个符号链接时，其目标会被加入存活存储路径的列表。

总结一下：Nix 维护着一个 GC 根列表。这些根随后可用于计算所有存活存储路径的列表。其余的存储路径被视为死路径。接下来删除这些路径就很简单了。Nix 首先把死的存储路径移动到 `/nix/store/trash`，这是一个原子操作。之后，垃圾箱会被清空。

## 玩转垃圾回收器

开始之前，我们先运行一下 [Nix 垃圾回收器](https://nix-doc.mczihan.link/command-ref/nix-collect-garbage)，以便为后续实验准备一个干净的环境：

```console
$ nix-collect-garbage
finding garbage collector roots...
[...]
deleting unused links...
note: currently hard linking saves -0.00 MiB
1169 store paths deleted, 228.43 MiB freed
```

如果我们再次运行垃圾回收器，正如预期的那样，它不会再找到任何可删除的内容。运行垃圾回收器之后，Nix store 中只包含被 GC 根引用的路径。

现在我们安装一个新程序 `bsd-games`，查看它的存储路径，并检查它的 GC 根。`nix-store -q --roots` 命令用于查询引用某个给定 derivation 的 GC 根。在这里，我们当前的用户环境引用了 `bsd-games`：

```console
$ nix-env -iA nixpkgs.bsdgames
$ readlink -f `which fortune`
/nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17/bin/fortune
$ nix-store -q --roots `which fortune`
/nix/var/nix/profiles/default-9-link
$ nix-env --list-generations
[...]
   9   2014-08-20 12:44:14   (current)
```

现在我们卸载它并运行垃圾回收器，注意 `bsd-games` 仍然留在 Nix store 中：

```console
$ nix-env -e bsd-games
uninstalling `bsd-games-2.17'
$ nix-collect-garbage
[...]
$ ls /nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17
bin  share
```

旧的 generation 之所以仍留在 Nix store 中，是因为它是一个 GC 根。正如我们将在下文看到的，所有 profile 及其 generation 都会自动成为 GC 根。

移除一个 GC 根很简单。在我们的例子中，我们删除引用 `bsd-games` 的那个 generation，运行垃圾回收器，然后会发现 `bsd-games` 已不在 Nix store 中：

```console
$ rm /nix/var/nix/profiles/default-9-link
$ nix-env --list-generations
[...]
   8   2014-07-28 10:23:24
  10   2014-08-20 12:47:16   (current)
$ nix-collect-garbage
[...]
$ ls /nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17
ls: cannot access /nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17: No such file or directory
```

注意：`nix-env --list-generations` 并不依赖任何特定的元数据。它仅凭 profiles 目录下的文件名就能列出各个 generation。

注意，我们删除的是 `/nix/var/nix/profiles` 中的链接，而不是 `/nix/var/nix/gcroots` 中的链接。除了后者之外，Nix 还把 `/nix/var/nix/profiles` 也当作 GC 根。这很有用，因为这意味着任何 profile 及其 generation 都是 GC 根。其他路径同样被视为 GC 根；例如 NixOS 上的 `/run/booted-system`。命令 `nix-store --gc --print-roots` 会打印运行垃圾回收器时所有被视为 GC 根的路径。

## 间接根

回顾一下：用 `nix-build` 构建 GNU `hello` 软件包会在当前目录生成一个 `result` 符号链接。尽管做了上面那些垃圾回收，`hello` 程序仍然可以运行。因此，它并没有被垃圾回收。由于没有其他 derivation 依赖 GNU `hello` 软件包，它必定是一个 GC 根。

实际上，`nix-build` 会自动把 `result` 符号链接添加为 GC 根。注意，被添加的不是构建出的 derivation，而是这个符号链接本身。这些 GC 根被添加在 `/nix/var/nix/gcroots/auto` 之下。

```console
$ ls -l /nix/var/nix/gcroots/auto/
total 8
drwxr-xr-x 2 nix nix 4096 Aug 20 10:24 ./
drwxr-xr-x 3 nix nix 4096 Jul 24 10:38 ../
lrwxrwxrwx 1 nix nix   16 Jul 31 10:51 xlgz5x2ppa0m72z5qfc78b8wlciwvgiz -> /home/nix/result/
```

此时 GC 根符号链接的名字对我们来说并不重要。重要的是这样一个符号链接确实存在，并且指向 `/home/nix/result`。这被称为**间接 GC 根**（indirect GC root）。如果一个 GC 根的描述位于 `/nix/var/nix/gcroots` 之外，它就被视为间接根。在我们的场景中，这意味着 `result` 符号链接的目标不会被垃圾回收。

要移除一个被间接 GC 根视为"存活"的 derivation，有两种途径：

- 从 `/nix/var/nix/gcroots/auto` 中删除该间接 GC 根。

- 删除 `result` 符号链接。

在第一种情况下，该 derivation 会在垃圾回收过程中从 Nix store 中删除，而 `result` 会变成一个悬空符号链接。在第二种情况下，derivation 被删除，`/nix/var/nix/gcroots/auto` 中的间接根也会随之删除。

在删除 GC 根或间接 GC 根之后运行 `nix-collect-garbage`，就会把该 derivation 从 store 中移除。

## 清理一切

Nix store 中软件重复的主要来源是 GC 根，它们来自 `nix-build` 和 profile 的 generation。运行 `nix-build` 会为该次构建产生一个引用特定版本特定库（例如 `glibc`）的 GC 根。升级之后，如果我们希望垃圾回收器移除对应的 derivation，并清理旧的依赖，就必须删除之前的构建产物。

对 profile 来说也一样。操作 `nix-env` 的 profile 会创建更多的 generation。旧的 generation 引用旧的软件，因此在升级之后会增加 Nix store 中的重复。

其他系统通常在升级之后会"遗忘"关于其先前状态的一切。使用 Nix，我们也可以执行这种类型的升级（让 Nix 移除所有旧的 derivation，包括旧的 generation），但需要手动进行。具体分为四步：

- 第一步，下载新版本的 nixpkgs channel，它保存着所有软件的描述。通过 `nix-channel --update` 完成。

- 第二步，用 `nix-env -u` 升级已安装的软件包。这会把我们带进一个包含更新软件的新 generation。

- 第三步，删除 `nix-build` 生成的所有间接根：要当心，这会产生悬空符号链接。更聪明的策略是同时删除这些符号链接的目标。

- 最后，使用 `nix-collect-garbage` 的 `-d` 选项删除所有 profile 的旧 generation，然后进行垃圾回收。此后，你将失去回滚到任何先前 generation 的能力。运行这条命令之前，务必确认新 generation 工作正常。

这四步如下所示：

```console
$ nix-channel --update
$ nix-env -u --always
$ rm /nix/var/nix/gcroots/auto/*
$ nix-collect-garbage -d
```

## 结论

Nix 中的垃圾回收是一个清理系统的强大机制。借助 `nix-store` 命令，我们可以弄清某个 derivation 为什么存在于 Nix store 中，以及它是否符合垃圾回收的条件。我们还学习了如何执行更具破坏性的删除与升级操作。

## 下一篇药丸

在下一篇药丸中，我们将打包另一个项目，并介绍 "inputs" 设计模式。到目前为止我们只接触了单个 derivation；然而我们想开始组织一个小型的软件仓库。"inputs" 模式在 nixpkgs 中被广泛使用；它让我们可以把 derivation 与仓库本身解耦，并增加定制的可能性。
