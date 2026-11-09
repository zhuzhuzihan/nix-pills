# Nix 存储路径

欢迎来到第 18 篇 Nix 药丸。在上一篇[第 17 篇](17-nixpkgs-overriding-packages.md)药丸中，我们初步了解了 `nixpkgs` 仓库的结构。它是一个软件包集合，并且可以对这些软件包进行 override（覆盖），使所有其他软件包都使用覆盖后的版本。

在阅读现有的 derivation（派生）之前，我想先谈谈存储路径（store path）以及它们是如何计算的。我们尤其关心那些依赖于完整性哈希（例如 sha256）的固定存储路径，这种哈希通常用于源码压缩包。

存储路径的计算方式有点绕，主要是历史原因造成的。我们的参考依据是 [Nix 源代码](https://github.com/NixOS/nix/blob/07f992a74b64f4376d5b415d0042babc924772f3/src/libstore/store-api.cc#L197)。

## 源码路径

从简单的开始。你知道 nix 允许使用相对路径，这样文件或目录会被存入 nix store，也就是说 `./myfile` 会被存储到 `/nix/store/.......` 中。我们想弄清楚，对于这样的文件，存储路径是如何生成的：

```console
$ echo mycontent > myfile
```

提醒一下，你能写出的最简单的 derivation 需要一个 `name`、一个 `builder` 和一个 `system`：

```console
$ nix repl
nix-repl> derivation { system = "x86_64-linux"; builder = ./myfile; name = "foo"; }
«derivation /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv»
```

现在检查这个 .drv，看看 `./myfile` 被存储到了哪里：

```console
$ nix derivation show /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv
{
  "/nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo"
      }
    },
    "inputSrcs": [
      "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile"
    ],
    "inputDrvs": {},
    "platform": "x86_64-linux",
    "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
    "args": [],
    "env": {
      "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
      "name": "foo",
      "out": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo",
      "system": "x86_64-linux"
    }
  }
}
```

很好，nix 是怎么决定使用 `xv2iccirbrvklck36f1g7vldn5v58vck` 的呢？继续看 nix 源码中的注释。

**注意：** 执行 `nix-store --add myfile` 会把该文件存入相同的存储路径。

### 第 1 步：计算文件的哈希

注释告诉我们，先计算文件 NAR 序列化形式的 sha256。可以用两种方式完成：

```console
$ nix-hash --type sha256 myfile
2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3
```

或者：

```console
$ nix-store --dump myfile|sha256sum
2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3
```

一般而言，Nix 理解两种内容：针对普通文件的 flat（平铺），以及针对 NAR 序列化（可以是任何东西）的 recursive（递归）。

### 第 2 步：构建字符串描述

然后 nix 使用一个特殊的字符串，其中包含哈希、路径类型和文件名。我们把它存进另一个文件：

```console
$ echo -n "source:sha256:2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3:/nix/store:myfile" > myfile.str
```

### 第 3 步：计算最终哈希

最后，注释告诉我们，计算上述字符串的 sha256 的前 160 位（截断）的 base-32 表示：

```console
$ nix-hash --type sha256 --truncate --base32 --flat myfile.str
xv2iccirbrvklck36f1g7vldn5v58vck
```

## 输出路径

输出路径通常是为 derivation 生成的。我们使用上面的例子，因为它比较简单。即使我们没有构建这个 derivation，nix 也已经知道 out 路径 `hs0yi5n5nw6micqhy8l1igkbhqdkzqa1`。这是因为 out 路径只取决于输入。

它的计算方式与源码路径类似，不同之处在于被哈希的是 .drv，并且 derivation 的类型是 `output:out`。在多输出的情况下，我们可能会有不同的 `output:<id>`。

当 nix 计算 out 路径时，.drv 中每个 out 路径的位置都是一个空字符串。所以我们要做的是取出我们的 .drv，并把 out 路径替换为空字符串：

```console
$ cp -f /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv myout.drv
$ sed -i 's,/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo,,g' myout.drv
```

这个 `myout.drv` 就是 nix 在为我们的 derivation 计算 out 路径时 .drv 所处的状态：

```console
$ sha256sum myout.drv
1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5  myout.drv
$ echo -n "output:out:sha256:1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5:/nix/store:foo" > myout.str
$ nix-hash --type sha256 --truncate --base32 --flat myout.str
hs0yi5n5nw6micqhy8l1igkbhqdkzqa1
```

然后 nix 把这个 out 路径放进 .drv，就是这样。

如果 .drv 有输入 derivation（即它引用了其他 .drv），那么这些 .drv 路径会被同一个算法替换成返回的哈希。

换句话说，你最终得到的 .drv 中，每个其他 .drv 的路径都被替换成了它的哈希。

## 固定输出路径

最后，另一种最常用的路径类型是：我们事先就知道某个文件的完整性哈希。这对源码压缩包来说很常见。

一个 derivation 可以接受三个特殊属性：`outputHashMode`、`outputHash` 和 `outputHashAlgo`，它们在 [nix 手册](https://nix.dev/manual/nix/stable/language/advanced-attributes)中有完善的文档。

builder 必须创建 out 路径，并确保其哈希与 `outputHash` 声明的哈希一致。

假设我们的 builder 应该创建一个内容为 `mycontent` 的文件：

```console
$ echo mycontent > myfile
$ sha256sum myfile
f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb  myfile
nix-repl> derivation { name = "bar"; system = "x86_64-linux"; builder = "none"; outputHashMode = "flat"; outputHashAlgo = "sha256"; outputHash = "f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb"; }
«derivation /nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv»
```

检查这个 .drv，与前面的例子相比，可以看到它还记录了这是一个使用 sha256 算法的固定输出（fixed-output）derivation：

```console
$ nix derivation show /nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv
{
  "/nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/a00d5f71k0vp5a6klkls0mvr1f7sx6ch-bar",
        "hashAlgo": "sha256",
        "hash": "f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb"
      }
    },
[...]
}
```

无论使用了哪些输入 derivation，最终的 out 路径都必须只取决于声明的哈希。

nix 的做法是为固定输出内容创建一个中间字符串表示：

```console
$ echo -n "fixed:out:sha256:f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb:" > mycontent.str
$ sha256sum mycontent.str
423e6fdef56d53251c5939359c375bf21ea07aaa8d89ca5798fb374dbcfd7639  myfile.str
```

然后像处理普通的 derivation 输出路径那样继续：

```console
$ echo -n "output:out:sha256:423e6fdef56d53251c5939359c375bf21ea07aaa8d89ca5798fb374dbcfd7639:/nix/store:bar" > myfile.str
$ nix-hash --type sha256 --truncate --base32 --flat myfile.str
a00d5f71k0vp5a6klkls0mvr1f7sx6ch
```

因此，存储路径只取决于声明的固定输出哈希。

## 结论

存储路径还有其他类型，但你应该已经明白其中的原理了。Nix 先对内容做哈希，然后创建一个字符串描述，最终的存储路径就是这个字符串的哈希。

此外我们还介绍了一些基础知识，特别是 Nix 事先就知道 derivation 的 out 路径这一事实，因为它只取决于输入。我们还介绍了固定输出 derivation，nixpkgs 仓库尤其常用它来下载并校验源码压缩包。

## 下一篇药丸

……我们将介绍 `stdenv`。在之前的药丸中，我们自己动手编写了包装内置 derivation 的 `mkDerivation` 便捷函数，而 `nixpkgs` 仓库也有它自己的便捷函数，用来处理 `autotools` 项目和其他构建系统。
