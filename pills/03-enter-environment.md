# 进入环境 {#enter-environment}

欢迎来到第三篇 Nix 药丸。在[第二篇药丸](02-install-on-your-running-system.md)中，我们在现有系统上安装了 Nix。现在终于可以稍微把玩一下它了，这些内容同样适用于 NixOS 用户。

## 进入环境

**如果你在使用 NixOS，可以跳到[下一步](#install-something)。**

在上一篇文章中我们创建了一个 Nix 用户，那就先用 `su - nix` 切换到该用户吧。如果你的 `~/.profile` 已经被执行过，那么现在你应该可以运行 `nix-env` 和 `nix-store` 之类的命令了。

如果不是这样：

```console
$ source ~/.nix-profile/etc/profile.d/nix.sh
```

提醒一下，`~/.nix-profile/etc` 指向的是 `nix-2.1.3` derivation。此时，我们已经处于自己的 Nix 用户 profile 中。

## 安装点东西 {#install-something}

终于来点实际的了！把软件安装到 Nix 环境中是一个有趣的过程。我们来安装 `hello`，这是一个简单的命令行工具，会打印 `Hello world`，主要用于测试编译器和软件包安装。

回到安装环节：

```console
$ nix-env -i hello
installing 'hello-2.10'
[...]
building '/nix/store/0vqw0ssmh6y5zj48yg34gc6macr883xk-user-environment.drv'...
created 36 symlinks in user environment
```

现在你可以运行 `hello` 了。需要注意的几点：

- 我们是以用户身份安装软件的，而且只针对 Nix 用户。

- 它创建了一个新的用户环境（user environment）。这是我们 Nix 用户 profile 的一个新代（generation）。

- [nix-env](https://nix-doc.mczihan.link/command-ref/nix-env) 工具负责管理环境、profile 及其各代。

- 我们是按去掉版本号的 derivation 名称安装 `hello` 的。我再说一遍：我们指定的是**derivation 名称**（去掉版本号）来安装它。

我们可以不遍历 `/nix` 层级就直接列出各代：

```console
$ nix-env --list-generations
    1   2014-07-24 09:23:30
    2   2014-07-25 08:45:01   (current)
```

列出已安装的 derivation：

```console
$ nix-env -q
nix-2.1.3
hello-2.10
```

那么，`hello` 到底被装到了哪里？`which hello` 的结果是 `~/.nix-profile/bin/hello`，它指向 store。我们还可以用 `nix-env -q --out-path` 列出 derivation 路径。原来这些 derivation 路径的名字就叫：构建的**输出（output）**。

## 路径合并

此时你大概想运行 `man` 查看一些文档了。即使你在 Nix 环境之外已经系统级安装了 man，你也可以用 `nix-env -i man-db` 在 Nix 内安装并使用它。照例会创建一个新代，`~/.nix-profile` 会指向它。

让我们检视一下这个 [profile](https://nix-doc.mczihan.link/package-management/profiles)：

```console
$ ls -l ~/.nix-profile/
dr-xr-xr-x 2 nix nix 4096 Jan  1  1970 bin
lrwxrwxrwx 1 nix nix   55 Jan  1  1970 etc -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/etc
[...]
```

有意思。当只安装了 `nix-2.1.3` 时，`bin` 是指向 `nix-2.1.3` 的符号链接。而当我们真正安装了一些东西（`man`、`hello`）之后，它变成了一个真实的目录，不再是符号链接。

```console
$ ls -l ~/.nix-profile/bin/
[...]
man -> /nix/store/83cn9ing5sc6644h50dqzzfxcs07r2jn-man-1.6g/bin/man
[...]
nix-env -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
[...]
hello -> /nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10/bin/hello
[...]
```

好，现在更清楚了。`nix-env` 把来自已安装 derivation 的路径合并了。`which man` 指向的是 Nix profile，而不是系统的 `man`，因为 `~/.nix-profile/bin` 位于 `$PATH` 的最前面。

## 回滚与切换代

刚才的命令安装了 `man`。我们应该处于第 3 代，除非你中途改动过什么。假设我们想回滚到旧一代：

```console
$ nix-env --rollback
switching from generation 3 to 2
```

现在 `nix-env -q` 不再列出 `man` 了。`` ls -l `which man` `` 现在应该指向你系统里的那份拷贝。

回滚够了，让我们回到最新一代：

```console
$ nix-env -G 3
switching from generation 2 to 3
```

我建议你读一读 `nix-env` 的 manpage。`nix-env` 需要指定一个要执行的操作（operation），然后是所有操作通用的选项，以及每个操作各自的专属选项。

当然，你也可以[卸载](https://nix-doc.mczihan.link/command-ref/nix-env/uninstall)和[升级](https://nix-doc.mczihan.link/command-ref/nix-env/upgrade)软件包。

## 查询 store

到目前为止，我们学习了如何查询和操作 environment。但环境中的所有组件都指向 store。

查询和操作 store 用的是 `nix-store` 命令。我们能做一些有趣的事情，但现在先只看一些查询操作。

要显示 `hello` 的直接运行时依赖：

```console
$ nix-store -q --references `which hello`
/nix/store/fg4yq8i8wd08xg3fy58l6q73cjy8hjr2-glibc-2.27
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
```

`nix-store` 的参数可以是任何东西，只要它指向 Nix store 即可。它会自动解析符号链接。

现在你可能觉得没什么意义，但我们来打印一下 `hello` 的反向依赖（referrers）：

```console
$ nix-store -q --referrers `which hello`
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
/nix/store/fhvy2550cpmjgcjcx5rzz328i0kfv3z3-env-manifest.nix
/nix/store/yzdk0xvr0b8dcwhi2nns6d75k2ha5208-env-manifest.nix
/nix/store/mp987abm20c70pl8p31ljw1r5by4xwfw-user-environment
/nix/store/ppr3qbq7fk2m2pa49i2z3i32cvfhsv7p-user-environment
```

和你预期的一样吗？原来我们的环境依赖于 `hello`。是的，这意味着环境本身就在 store 中，而且由于它们包含指向 `hello` 的符号链接，所以环境依赖于 `hello`。

列表里出现了两个环境，第 2 代和第 3 代，因为只有这两代安装了 `hello`。

`manifest.nix` 文件包含关于环境的元数据，例如安装了哪些 derivation，这样 `nix-env` 才能列出、升级或移除它们。同样，当前的 `manifest.nix` 可以在 `~/.nix-profile/manifest.nix` 找到。

## 闭包（closure）

一个 derivation 的闭包（closure）是它所有依赖的递归列表，包含使用该 derivation 所需的一切东西。

```console
$ nix-store -qR `which man`
[...]
```

把这些 derivation 全部复制到另一台机器的 Nix store 中，就能让 `man` 在那台机器上开箱即用。这就是使用 Nix 进行部署的基础，你已经可以预见到在云中部署软件的潜力了（提示：`nix-copy-closures` 和 `nix-store --export`）。

闭包的一种更直观的视图：

```console
$ nix-store -q --tree `which man`
[...]
```

使用上面的命令，你可以确切地找出某个给定 derivation 为什么存在某个_运行时_依赖，无论是直接的还是间接的。

同样的操作也适用于环境。作为练习，运行 `nix-store -q --tree ~/.nix-profile`，你会看到第一层子节点是用户环境的直接依赖：已安装的 derivation，以及 `manifest.nix`。

## 依赖解析

这里没有像 `apt` 那样通过求解 SAT 问题来满足带版本上下界约束的依赖的东西。也没有这个必要，因为所有依赖都是静态的：如果 derivation X 依赖于 derivation Y，那么它永远依赖于它。某个依赖 Z 的 X 版本会是另一个不同的 derivation。

## 硬核自救

```console
$ nix-env -e '*'
uninstalling 'hello-2.10'
uninstalling 'nix-2.1.3'
[...]
```

糟糕，这一下把环境里的所有 derivation 都卸载了，包括 Nix 本身。这意味着我们连 `nix-env` 都运行不了了，怎么办？

之前我们是从环境中获取 `nix-env` 的。环境只是给用户提供的便利，但 Nix 依然在 store 里！

首先，挑一个 `nix-2.1.3` derivation：`ls /nix/store/*nix-2.1.3`，比如 `/nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3`。

第一种方案是回滚：

```console
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env --rollback
```

第二种方案是安装 Nix，从而创建一个新代：

```console
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env -i /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
```

## channel

那么我们是从哪里获取软件包的呢？这在[第二篇文章](02-install-on-your-running-system.md)中已经提到过。我们有一份 channel 列表可以从中获取软件包，不过通常只用一个 channel。管理 channel 的工具是 [nix-channel](https://nix-doc.mczihan.link/command-ref/nix-channel)。

```console
$ nix-channel --list
nixpkgs http://nixos.org/channels/nixpkgs-unstable
```

如果你在使用 NixOS，上面的命令可能没有任何输出（如果你用的是默认配置），或者你会看到一个名称以 "nixos-" 而非 "nixpkgs" 开头的 channel。

这基本上就是 `~/.nix-channels` 的内容。

<div class="info">

注意：`~/.nix-channels` 并不是指向 Nix store 的符号链接！

</div>

要更新 channel，运行 `nix-channel --update`。它会下载新的 Nix 表达式（软件包描述），为 channels profile 创建一个新代，并将其解包到 `~/.nix-defexpr/channels` 下。

这和 `apt-get update` 很相似。（参见[这张表](https://wiki.nixos.org/wiki/Cheatsheet)了解 Ubuntu 与 NixOS 软件包管理之间的大致对应关系。）

## 结论

我们学习了如何查询用户环境，以及通过安装和卸载软件来操作它。升级软件也很简单，你可以在[手册](https://nix-doc.mczihan.link/command-ref/nix-env/upgrade)中读到（`nix-env -u` 会升级环境中的所有软件包）。

每当我们改变环境，就会创建一个新代。在代之间切换既简单又即时。

接着我们学习了如何查询 store。我们检视了存储路径的依赖和反向依赖。

我们看到了如何用符号链接来组合来自 Nix store 的路径，这是一个很实用的技巧。

用一个编程语言的类比来快速说明：你有一个存放所有对象的堆（heap），它对应 Nix store；你有指向其他对象的对象，它们对应 derivation。这是一个耐人寻味的比喻，但它会是一条正确的道路吗？

## 下一篇药丸

……我们将学习 Nix 语言的基础。Nix 语言用于描述如何构建 derivation，它是一切其他内容的基础，包括 NixOS。因此，理解这门语言的语法和语义都非常重要。
