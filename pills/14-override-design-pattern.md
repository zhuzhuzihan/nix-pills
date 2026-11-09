# Override 设计模式

欢迎来到第 14 篇 Nix 药丸。在上一篇[第 13 篇](13-callpackage-design-pattern.md)药丸中，我们介绍了 `callPackage` 模式，并利用它简化了仓库中软件的组合方式。

下一个设计模式并非那么必要，但在很多情况下都很有用，而且是深入学习 Nix 的一个很好的练习。

## 关于可组合性

函数式语言以能够组合函数而闻名。特别是，这些语言通过那些把原始值变换为具有相同结构的新值的函数来获得表达力。这使我们能够组合多个函数来完成想要的修改。

在 Nix 中，我们主要讨论的是接收输入并返回 **derivation（派生）** 的**函数**。在我们的世界里，我们想要能够操作这些结构的工具函数。这些工具函数为原始值添加一些有用的属性，并且我们希望能够在结果之上继续应用更多工具函数。

例如，假设我们有一个初始 derivation `drv`，我们想把它变换成一个带有调试信息和自定义补丁的 `drv`：

```nix
debugVersion (applyPatches [ ./patch1.patch ./patch2.patch ] drv)
```

最终结果应该是经过一些修改的原始 derivation。这既有趣，又与其他打包方式非常不同，这是使用函数式语言描述软件包的结果。

在没有静态类型的函数式语言中设计这样的工具函数并非易事，因为很难理解什么可以组合、什么不可以组合。但我们尽力而为。

## Override 模式

在[第 12 篇药丸](12-inputs-design-pattern.md)中，我们介绍了输入设计模式。我们不再直接从仓库中挑选依赖来返回一个 derivation，而是声明输入，让调用者传入必要的参数。

在我们的仓库中，有一组属性会导入软件包的表达式并传入这些参数，从而得到一个 derivation。以 `graphviz` 属性为例：

```nix
graphviz = import ./graphviz.nix { inherit mkDerivation gd fontconfig libjpeg bzip2; };
```

如果我们想生成一个使用了自定义 `gd` 版本的 `graphviz` derivation，就必须重复上面的大部分内容，并另外指定一个替代的 `gd`：

```nix
{
  mygraphviz = import ./graphviz.nix {
    inherit
      mkDerivation
      fontconfig
      libjpeg
      bzip2
      ;
    gd = customgd;
  };
}
```

这样很难维护。使用 `callPackage` 会更容易：

```nix
mygraphviz = callPackage ./graphviz.nix { gd = customgd; };
```

但我们可能仍然会偏离仓库中原始的 graphviz。

我们希望避免再次书写 nix 表达式。相反，我们希望复用仓库中原始的 `graphviz` 属性，并像下面这样添加我们的 override（覆盖）：

```nix
mygraphviz = graphviz.override { gd = customgd; };
```

区别一目了然，这种方法的优点也是如此。

注意：`.override` 并不是你可能认为的面向对象意义上的「方法」。Nix 是一门函数式语言。`.override` 只是一个集合（set）中的一个属性。

## Override 的实现

回顾一下，仓库中的 `graphviz` 属性是由从 `graphviz.nix` 导入的函数返回的 derivation。我们想给返回的集合添加一个名为 "`override`" 的额外属性。

我们先来创建一个函数 "`makeOverridable`"。这个函数接收两个参数：一个函数（必须返回一个集合）和要传给该函数的原始参数集合。

我们会把这个函数放在一个 `lib.nix` 中：

```nix
{
  makeOverridable =
    f: origArgs:
    let
      origRes = f origArgs;
    in
    origRes // { override = newArgs: f (origArgs // newArgs); };
}
```

`makeOverridable` 接收一个函数和一组原始参数。它返回原始的返回集合，外加一个新的 `override` 属性。

这个 `override` 属性是一个函数，它接收一组新参数，并返回以「原始参数与新参数合并」后的参数调用原始函数的结果。诚然这有些令人费解，但下面的例子应该能把它讲清楚。

让我们用 `nix repl` 试一下：

```console
$ nix repl
nix-repl> :l lib.nix
Added 1 variables.
nix-repl> f = { a, b }: { result = a+b; }
nix-repl> f { a = 3; b = 5; }
{ result = 8; }
nix-repl> res = makeOverridable f { a = 3; b = 5; }
nix-repl> res
{ override = «lambda»; result = 8; }
nix-repl> res.override { a = 10; }
{ result = 15; }
```

注意，正如我们上面所指定的，函数 `f` 返回的并不是单纯求和的结果，而是一个把和绑定到名称 `result` 上的集合。

变量 `res` 包含了不带任何 override 的函数调用结果。从 `makeOverridable` 的定义中很容易看出来。此外，你可以看到新的 `override` 属性是一个函数。

用一个集合调用 `res.override` 会按预期以这些 override 调用原始函数。

这是一个良好的开端，但我们不能再次 override！这是因为返回的集合（其中 `result = 15`）自身并没有 `override` 属性。这很糟糕，它破坏了进一步的组合。

解决方案很简单：`.override` 函数应当让结果再次变得可 override：

```nix
rec {
  makeOverridable =
    f: origArgs:
    let
      origRes = f origArgs;
    in
    origRes // { override = newArgs: makeOverridable f (origArgs // newArgs); };
}
```

请注意 `rec` 关键字。它是必需的，这样我们才能在 `makeOverridable` 内部引用 `makeOverridable` 自身。

现在让我们尝试 override 两次：

```console
nix-repl> :l lib.nix
Added 1 variables.
nix-repl> f = { a, b }: { result = a+b; }
nix-repl> res = makeOverridable f { a = 3; b = 5; }
nix-repl> res2 = res.override { a = 10; }
nix-repl> res2
{ override = «lambda»; result = 15; }
nix-repl> res2.override { b = 20; }
{ override = «lambda»; result = 30; }
```

成功了！结果是 30（正如预期），因为 `a` 在第一次 override 中被覆盖为 10，而 `b` 在第二次 override 中被覆盖为 20。

现在，如果 `callPackage` 能让我们的 derivation 变得可 override，那就更好了。这留给读者作为练习。

## 结论

"`override`" 模式简化了从一组现有软件包出发进行定制的方式。这为使用像 `nixpkgs` 这样的中央仓库、并在本地机器上定义 override 而无需修改原始软件包，打开了一个充满可能性的世界。

我们可以畅想一个自定义的、隔离的 `nix-shell` 环境，用一个自定义的 `gd` 来测试 `graphviz`：

```nix
debugVersion (graphviz.override { gd = customgd; })
```

一旦被 override 的软件包在仓库中发布了新版本，定制后的软件包会自动使用它。

Nix 的关键在于找到既强大又简单的抽象，让用户能够借助预定义的可组合组件，以最高的一致性和最低的维护成本来定制自己的环境。

## 下一篇药丸

在下一篇药丸中，我们将讨论 Nix 搜索路径。所谓「搜索路径」，指的是文件系统中 Nix 查找表达式的位置。这将回答 `<nixpkgs>` 从何而来的问题。
