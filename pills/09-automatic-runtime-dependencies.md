# 自动运行时依赖

欢迎来到第 9 篇 Nix 药丸。在上一篇[第 8 篇药丸](08-generic-builders.md)中，我们为 autotools 项目编写了一个通用 builder。我们传入构建依赖和一个源码压缩包，然后得到了一个 Nix derivation（派生）作为结果。

今天我们顺便造访 GNU `hello` 程序，分析它的构建依赖和运行时依赖，并且增强我们的 builder 以消除不必要的运行时依赖。

## 构建依赖

让我们开始分析 GNU `hello` 软件包的构建依赖：

```console
$ nix-instantiate hello.nix
/nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
$ nix-store -q --references /nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
/nix/store/0q6pfasdma4as22kyaknk4kwx4h58480-hello-2.10.tar.gz
/nix/store/1zcs1y4n27lqs0gw4v038i303pb89rw6-coreutils-8.21.drv
/nix/store/2h4b30hlfw4fhqx10wwi71mpim4wr877-gnused-4.2.2.drv
/nix/store/39bgdjissw9gyi4y5j9wanf4dbjpbl07-gnutar-1.27.1.drv
/nix/store/7qa70nay0if4x291rsjr7h9lfl6pl7b1-builder.sh
/nix/store/g6a0shr58qvx2vi6815acgp9lnfh9yy8-gnugrep-2.14.drv
/nix/store/jdggv3q1sb15140qdx0apvyrps41m4lr-bash-4.2-p45.drv
/nix/store/pglhiyp1zdbmax4cglkpz98nspfgbnwr-gnumake-3.82.drv
/nix/store/q9l257jn9lndbi3r9ksnvf4dr8cwxzk7-gawk-4.1.0.drv
/nix/store/rgyrqxz1ilv90r01zxl0sq5nq0cq7v3v-binutils-2.23.1.drv
/nix/store/qzxhby795niy6wlagfpbja27dgsz43xk-gcc-wrapper-4.8.3.drv
/nix/store/sk590g7fv53m3zp0ycnxsc41snc2kdhp-gzip-1.6.drv
```

它恰好就是 `derivation` 函数中引用的那些 derivation，不多也不少。当然，其中一些我们可能根本没用到。不过，考虑到我们的通用 `mkDerivation` 函数总是会拉取这些依赖（可以把它想象成 Debian 的 [build-essential](https://packages.debian.org/unstable/build-essential)），这些软件包已经存在于 nix store 中，可供今后任何需要它们的软件包使用。

为什么我们要查看 `.drv` 文件？因为 `hello.drv` 文件是构建 `hello` 输出路径这一构建动作的表示。因此，它包含了构建 `hello` 之前所需的输入 derivation。

## 关于 NAR 文件的题外话

`NAR` 格式即 "Nix ARchive"（Nix 归档）。之所以设计这种格式，是因为现有的归档格式（例如 `tar`）有所不足。Nix 受益于确定性的构建工具，但常用的归档工具缺乏这一特性：它们会添加填充、不排序文件、添加时间戳等等。这可能导致包含比特完全相同文件的目录被归档成比特不同的归档，进而产生不同的哈希。

因此，`NAR` 格式被开发成一种简单、确定性的归档格式。`NAR` 在 Nix 内部被大量使用，我们将在下文看到。

关于 `NAR` 背后的更多设计动机和实现细节，参见 [Dolstra 的博士论文](http://nixos.org/~eelco/pubs/phd-thesis.pdf)。

要从存储路径创建 NAR 归档，我们可以使用 `nix-store --dump` 和 `nix-store --restore`。

## 运行时依赖

现在我们注意到，一旦我们的 `derivation` 调用引用了构建依赖，Nix 就会自动识别它们，但我们从未指定过运行时依赖。

Nix 会为我们自动处理运行时依赖。它所使用的技术乍一看可能显得脆弱，但它工作得如此之好，以至于 NixOS 操作系统就是建立在它之上的。底层机制依赖于存储路径的哈希。它分三步进行：

1.  把 derivation 的输出转储（dump）为 NAR。回想一下，这是对 derivation 输出的序列化——因此无论输出是单个文件还是目录，它都能正常工作。

2.  对每个构建依赖 `.drv` 及其对应的输出路径，在 NAR 的内容中搜索这个输出路径。

3.  如果找到了该路径，那么它就是一个运行时依赖。

下面的代码片段展示了 `hello` 的依赖：

```console
$ nix-instantiate hello.nix
/nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
$ nix-store -r /nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
/nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
$ nix-store -q --references /nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19
/nix/store/8jm0wksask7cpf85miyakihyfch1y21q-gcc-4.8.3
/nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
```

我们看到 `glibc` 和 `gcc` 都是运行时依赖。凭直觉来说，`gcc` 不应该出现在这个列表里！显示 `hello` 二进制文件中的可打印字符串可以看到，`gcc` 的输出路径确实出现了：

```console
$ strings result/bin/hello|grep gcc
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib:/nix/store/8jm0wksask7cpf85miyakihyfch1y21q-gcc-4.8.3/lib64
```

这就是 Nix 把 `gcc` 加进来的原因。但为什么这个路径一开始就存在呢？答案是它是 [ld rpath](http://en.wikipedia.org/wiki/Rpath)：运行时可以找到库文件的目录列表。在其他发行版中，这个机制通常不会被如此使用。但在 Nix 中，我们必须引用特定版本的库，因此 rpath 扮演着重要角色。

构建过程添加了 `gcc` 的库路径，以为它在运行时可能有用，但这其实没有必要。为了解决这类问题，Nix 提供了一个名为 [patchelf](https://nixos.org/patchelf.html) 的工具，它会把 rpath 缩减为二进制文件实际使用的那些路径。

即便在缩减 rpath 之后，`hello` 二进制文件仍然会因为一些调试信息而依赖 `gcc`。这会不必要地增大我们运行时依赖的体积。下一节我们将探讨 `strip` 如何帮助我们解决这个问题。

## builder 中的又一个阶段

我们将为 autotools builder 添加一个新阶段。builder 已经有六个阶段了：

1.  "环境设置"阶段

2.  "解包阶段"：我们把源码解压到当前目录（记住，Nix 会先切换到一个临时目录）

3.  "切换目录"阶段，我们把源码根目录切换到解压出来的目录

4.  "configure" 阶段：`./configure`

5.  "构建阶段"：`make`

6.  "安装阶段"：`make install`

现在我们将在安装阶段之后添加一个新阶段，称之为 "fixup" 阶段。在 `builder.sh` 的末尾，我们追加：

```console
find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
```

也就是说，对每个文件我们运行 `patchelf --shrink-rpath` 和 `strip`。注意我们在这里用了两个新命令，`find` 和 `patchelf`。必须把它们加入我们的 derivation。

**练习：**把 `findutils` 和 `patchelf` 加入 `autotools.nix` 的 `baseInputs`。

现在，我们重新构建 `hello.nix`……

```console
$ nix-build hello.nix
[...]
$ nix-store -q --references result
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19
/nix/store/md4a3zv0ipqzsybhjb8ndjhhga1dj88x-hello
```

我们看到 `glibc` 是一个运行时依赖，而 `gcc` 已经不在其中了。这正是我们想要的。

这个软件包是自包含的。这意味着我们可以把它的闭包复制到另一台机器上，然后就能运行它。记住，`/nix/store` 下只需要极少数组件就能[运行 nix](02-install-on-your-running-system.md)。`hello` 二进制文件会使用它在二进制中引用的确切版本的 `glibc` 库和加载器，而不是系统的那个：

```console
$ ldd result/bin/hello
 linux-vdso.so.1 (0x00007fff11294000)
 libc.so.6 => /nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib/libc.so.6 (0x00007f7ab7362000)
 /nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib/ld-linux-x86-64.so.2 (0x00007f7ab770f000)
```

当然，只要一切都在 `/nix/store` 路径之下，这个可执行文件就能正常运行。

## 结论

我们了解了 Nix 提供的一些工具及其特性。特别地，我们看到了 Nix 如何自动计算运行时依赖。这不仅限于共享库，还可以引用可执行文件、脚本、Python 库等等。

以这种方式进行构建能让软件包自包含，从而确保（除了数据和配置之外）把运行时闭包复制到另一台机器上就足以运行程序。这使我们能够借助 `nix-shell` 无需安装即可运行程序，并为[云端可靠部署](https://github.com/NixOS/nixops)奠定了基础。

## 下一篇药丸

下一篇药丸将介绍 `nix-shell`。使用 `nix-build`，我们一直都是从头构建 derivation：源码被解压、配置、构建并安装。但对于大型软件包，这可能耗时很久。如果我们只想做一些小的修改并进行增量编译，同时又想保留一个类似 `nix-build` 的自包含环境，该怎么办？`nix-shell` 正是为此而生。
