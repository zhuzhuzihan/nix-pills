# 在 Nixpkgs 中覆盖软件包

欢迎来到第 17 篇 Nix 药丸。在上一篇[第 16 篇](16-nixpkgs-parameters.md)药丸中，我们开始深入 [nixpkgs 仓库](http://github.com/NixOS/nixpkgs)。`Nixpkgs` 是一个函数，我们已经看过一些参数，比如 `system` 和 `config`。

今天我们要讨论一个特殊的属性：`config.packageOverrides`。用不动点（fixed point）来覆盖（override）一个集合中的软件包，可以视为 nixpkgs 中的另一种设计模式。

## 覆盖一个软件包

回顾[第 14 篇药丸](14-override-design-pattern.md)中的 override 设计模式：我们不直接带着参数调用函数，而是让这次调用（函数 + 参数）可以被覆盖。

我们把 override 函数放进原始函数调用所返回的属性集合中。

以 `graphviz` 为例。它有一个输入参数 `xorg`。如果它是 null，那么 `graphviz` 将在不支持 X 的情况下构建。

```console
$ nix repl
nix-repl> :l <nixpkgs>
Added 4360 variables.
nix-repl> :b graphviz.override { withXorg = false; }
```

这会构建一个不带 X 支持的 `graphviz`，就是这么简单。

然而，假设有一个软件包 `P` 依赖 `graphviz`，我们如何让 `P` 依赖这个新的不带 X 支持的 `graphviz` 呢？

## 在命令式世界里……

……你可能会这样做：

```nix
pkgs = import <nixpkgs> {};
pkgs.graphviz = pkgs.graphviz.override { withXorg = false; };
build(pkgs.P)
```

如果 `pkgs.P` 依赖 `pkgs.graphviz`，那么用替换后的 `graphviz` 构建 `P` 就很容易。但在纯函数式语言里就没那么简单了，因为变量只能赋值一次。

## 不动点

在 Nix 这样的语言中，配合惰性求值的不动点虽然令人抓狂，但几乎是必需的。它让我们能够实现与命令式做法类似的效果。

下面是 [nixpkgs](https://github.com/NixOS/nixpkgs/blob/f224a4f1b32b3e813783d22de54e231cd8ea2448/lib/fixed-points.nix#L19) 中不动点的定义：

```nix
{
  # 接收一个函数，并用它自己返回的值来对它求值。
  fix =
    f:
    let
      result = f result;
    in
    result;
}
```

这是一个接受函数 `f` 的函数，它对刚由 `f result` 返回的结果再调用 `f result`，然后返回它。换句话说，它就是 `f(f(f(....`。

乍一看，这是一个无限循环。但在惰性求值下并不是，因为调用只在需要时才发生。

```console
nix-repl> fix = f: let result = f result; in result
nix-repl> pkgs = self: { a = 3; b = 4; c = self.a+self.b; }
nix-repl> fix pkgs
{ a = 3; b = 4; c = 7; }
```

不借助 `rec` 关键字，我们也能在同一个集合中引用 `a` 和 `b`。

- 首先 `pkgs` 被一个未求值的 thunk 调用 `(pkgs(pkgs(...)`

- 然后为了求出 `c` 的值，`self.a` 和 `self.b` 被求值。

- `pkgs` 函数被再次调用以取得 `a` 和 `b` 的值。

诀窍在于，内层调用并不需要求值 `c`，因此不会陷入无限循环。

这里就不再深入解释了。关于不动点与 Nix 的一篇好文章可以[在这里找到](http://r6.ca/blog/20140422T142911Z.html)。

### 用不动点覆盖一个集合

由于 `self.a` 和 `self.b` 引用的是传入的集合，而不是函数中的字面集合，我们能够覆盖 `a` 和 `b`，并得到 `c` 的新值：

```console
nix-repl> overrides = { a = 1; b = 2; }
nix-repl> let newpkgs = pkgs (newpkgs // overrides); in newpkgs
{ a = 3; b = 4; c = 3; }
nix-repl> let newpkgs = pkgs (newpkgs // overrides); in newpkgs // overrides
{ a = 1; b = 2; c = 3; }
```

第一种情况下我们计算了带 override 的 pkgs，第二种情况下我们还把被覆盖的属性包含进了结果中。

## 覆盖 nixpkgs 软件包

我们已经看到如何覆盖集合中的属性，使得依赖它们的属性会递归地取到覆盖后的值。这种方法同样适用于 derivation（派生），毕竟 `nixpkgs` 就是一个巨大的、彼此相互依赖的属性集合。

为此，`nixpkgs` 提供了 `config.packageOverrides`。也就是说，`nixpkgs` 返回的是软件包集合的不动点，而 `packageOverrides` 用来注入 override。

在某个地方创建一个 `config.nix` 文件，内容如下：

```nix
{
    packageOverrides = pkgs: {
    graphviz = pkgs.graphviz.override {
      # 禁用 xorg 支持
      withXorg = false;
    };
  };
}
```

现在我们可以构建例如 `asciidoc-full`，它会自动使用被覆盖的 `graphviz`：

```console
nix-repl> pkgs = import <nixpkgs> { config = import ./config.nix; }
nix-repl> :b pkgs.asciidoc-full
```

注意我们在导入 `nixpkgs` 时如何传入带有 `packageOverrides` 的 `config`。然后 `pkgs.asciidoc-full` 是一个以 `graphviz` 为输入的 derivation（`pkgs.asciidoc` 是更轻量的版本，完全不使用 `graphviz`）。

由于二进制缓存中没有「`graphviz` 不带 X 支持」版本的 `asciidoc`，Nix 会为你重新编译所需的内容。

## \~/.config/nixpkgs/config.nix 文件

在上一篇药丸中我们已经讨论过这个文件。我们刚写的上述 `config.nix` 可以作为 `~/.config/nixpkgs/config.nix`（或已弃用的位置 `~/.nixpkgs/config.nix`）的内容。

这样就不必在每次导入 `nixpkgs` 时显式传入它，它会[由 nixpkgs 自动导入](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28)。

## 结论

我们学习了一种新的设计模式：使用不动点来覆盖软件包集合中的软件包。

在命令式环境（比如其他软件包管理器）中，安装库时会替换旧版本，应用程序会使用新库；而在 Nix 中，事情并非如此直接简单，但它更精确。

Nix 应用程序会依赖特定版本的库，这就是为什么我们必须重新编译 `asciidoc` 才能使用新的 `graphviz` 库。

新构建的 `asciidoc` 会依赖新的 `graphviz`，而旧的 `asciidoc` 会继续不受干扰地使用旧的 `graphviz`。

## 下一篇药丸

……我们将暂停对 `nixpkgs` 的研究，谈谈存储路径（store path）。Nix 是如何计算出在 Nix store 中放置构建结果的路径的？如何把一个我们拥有完整性哈希的文件添加进 Nix store？
