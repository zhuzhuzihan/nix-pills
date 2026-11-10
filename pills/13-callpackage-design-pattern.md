# CallPackage 设计模式

欢迎来到第 13 篇 Nix 药丸。在上一篇[第 12 篇药丸](12-inputs-design-pattern.md)中，我们介绍了用于组织软件仓库的第一个基础设计模式。此外，我们还打包了 `graphviz`，从而有了两个软件包可以打包进一个示例仓库。

接下来要考察的设计模式叫做 `callPackage` 模式。这一技术在 [nixpkgs](https://github.com/NixOS/nixpkgs) 中被广泛使用，也是当前在仓库中导入软件包的事实标准。它的目的是减少软件包 derivation 的 inputs 与仓库中 derivation 之间标识符的重复。

## callPackage 带来的便利

在上一篇药丸中，我们展示了 `inputs` 模式如何把软件包与仓库解耦。这允许我们手动把 inputs 传给 derivation；derivation 声明它的 inputs，调用方传入参数。

然而，与常见的编程语言一样，这里存在一些重复劳动：我们声明参数名，然后再传参数，而且通常使用相同的名字。例如，如果我们用 `inputs` 模式定义一个软件包 derivation，形如：

```nix
{ input1, input2, ... }:
...
```

我们很可能想通过一个形如下面的 attribute set，把这个软件包 derivation 打包进仓库：

```nix
rec {
  lib1 = import package1.nix { inherit input1 input2; };
  program2 = import package2.nix { inherit inputX inputY lib1; };
}
```

有两点值得注意。第一，inputs 的名字往往与仓库本身中的属性同名。第二，（由于 `rec` 关键字的存在）软件包 derivation 的 inputs 可能是仓库本身中的其他软件包。

与其把 inputs 传两遍，我们更希望从仓库中自动传入这些 inputs，并允许手动 override（覆盖）默认值。

为此，我们将定义一个具有如下调用约定的 `callPackage` 函数：

```nix
{
  lib1 = callPackage package1.nix { };
  program2 = callPackage package2.nix { someoverride = overriddenDerivation; };
}
```

我们希望 `callPackage` 是一个接受两个参数的函数，行为如下：

- 导入第一个参数所指文件中包含的表达式，并返回一个函数。该函数返回一个使用 inputs 模式的软件包 derivation。

- 确定该函数的参数名（即软件包 derivation 的 inputs 名字）。

- 从仓库集合中传入默认参数，并允许我们在想定制软件包 derivation 时 override 这些参数。

## 实现 `callPackage`

在这一节中，我们将从零开始构建 `callPackage` 模式。首先，我们需要一种在运行时获取函数参数名的方法（在这里，函数指接收 "inputs" 并产出软件包 derivation 的函数）。这是因为我们希望自动传入这些参数。

Nix 提供了一个内置函数来完成这件事：

```console
nix-repl> add = { a ? 3, b }: a+b
nix-repl> builtins.functionArgs add
{ a = true; b = false; }
```

除了返回参数名之外，`functionArgs` 返回的 attribute set 还指示了参数是否有默认值。就我们的目的而言，我们只关心参数名；暂时不关心默认值。

下一步是让 `callPackage` 依据我们刚用 `functionArgs` 获得的参数名，自动把 inputs 传给我们的软件包 derivation。

为此，我们需要两样东西：

- 一个软件包仓库集合，其中包含的软件包 derivation 与我们获得的参数名相匹配

- 一种方法，用于得到一个自动填充的 attribute set，它把软件包仓库与 `functionArgs` 的返回值结合起来

前者很容易：我们只需把软件包 derivation 的 inputs 设为仓库（例如 `nixpkgs`）中的软件包名。至于后者，Nix 提供了另一个内置函数：

```console
nix-repl> values = { a = 3; b = 5; c = 10; }
nix-repl> builtins.intersectAttrs values (builtins.functionArgs add)
{ a = true; b = false; }
nix-repl> builtins.intersectAttrs (builtins.functionArgs add) values
{ a = 3; b = 5; }
```

`intersectAttrs` 返回一个 attribute set，其名字是两个参数的属性名的交集，属性值取自第二个参数。

这就是我们要做的全部工作：我们从一个函数获得了参数名，并用一组现成的属性填充了它们。下面是我们对 `callPackage` 的简单实现：

```console
nix-repl> callPackage = set: f: f (builtins.intersectAttrs (builtins.functionArgs f) set)
nix-repl> callPackage values add
8
nix-repl> with values; add { inherit a b; }
8
```

让我们剖析一下上面的代码片段：

- 我们定义了一个 `callPackage` 变量，它是一个函数。

- `callPackage` 函数的第一个参数是一个"名字-值"对的集合，这些名字可能出现在我们想"自动调用"（autocall）的函数的参数集合中。

- 第二个参数是要被"自动调用"的函数。

- 我们取函数的参数名，并与包含所有值的集合求交集。

- 最后，我们用求得的交集调用传入的函数 `f`。

在上面的代码片段中，我们还展示了 `callPackage` 调用等价于直接调用 `add a b`。

我们已经实现了大部分目标：给定一组可能的参数，自动调用函数。如果某个参数在我们调用函数所用的集合中找不到，就会收到错误（除非函数带有用 `...` 表示的可变参数，如[第 5 篇药丸](05-functions-and-imports.md)所述）。

最后缺少的一环是允许用户 override 部分参数。我们未必总是想用来自那个大集合的值来调用函数。因此，我们加入第三个参数，它接收一组 override：

```console
nix-repl> callPackage = set: f: overrides: f ((builtins.intersectAttrs (builtins.functionArgs f) set) // overrides)
nix-repl> callPackage values add { }
8
nix-repl> callPackage values add { b = 12; }
15
```

除了括号越来越多之外，应该很清楚：我们只是对默认参数和 override 集合做了一次集合并。

## 使用 callPackage 简化仓库

有了我们的 `callPackage`，就可以简化 `default.nix` 中的仓库表达式：

```nix
let
  nixpkgs = import <nixpkgs> { };
  allPkgs = nixpkgs // pkgs;
  callPackage =
    path: overrides:
    let
      f = import path;
    in
    f ((builtins.intersectAttrs (builtins.functionArgs f) allPkgs) // overrides);
  pkgs = with nixpkgs; {
    mkDerivation = import ./autotools.nix nixpkgs;
    hello = callPackage ./hello.nix { };
    graphviz = callPackage ./graphviz.nix { };
    graphvizCore = callPackage ./graphviz.nix { gdSupport = false; };
  };
in
pkgs
```

让我们详细审视一下：

- 上面的表达式定义了我们自己的软件包仓库，我们称之为 `pkgs`，其中包含 `hello` 以及我们的两个 `graphviz` 变体。

- 在 `let` 表达式中，我们导入了 `nixpkgs`。注意，之前我们用变量 `pkgs` 指代这个导入，但现在这个名字被我们自己正在创建的仓库占用了。

- 我们需要一种方式把 `pkgs` 以某种形式传给 `callPackage`。我们没有直接从 `default.nix` 返回软件包集合，而是先把它赋给一个 `let` 变量，再在 `callPackage` 中复用它。

- 为了方便，在 `callPackage` 中我们先导入文件，而不是直接调用它。否则我们就得为每个软件包写一遍 `import`。

- 由于我们的表达式使用了来自 `nixpkgs` 的软件包，`callPackage` 中我们使用 `allPkgs`，即 `nixpkgs` 与我们自己的软件包的并集。

- 我们把 `mkDerivation` 移进了 `pkgs` 本身，这样它也会被自动传入。

注意，对于不带 `gd` 的 `graphviz`，我们如此轻松地 override 了参数。另外，注意合并两个仓库 \-\- `nixpkgs` 与我们的 `pkgs` \-\- 是何等容易！

读者应当注意到一件神奇的事情正在发生。我们用 `callPackage` 来定义 `pkgs`，又用 `pkgs` 来定义 `callPackage`。这种魔法之所以可能，要归功于惰性求值：`builtins.intersectAttrs` 在执行交集运算时，并不需要知道 `allPkgs` 中的值，只需要那些不涉及 `callPackage` 求值的键。

## 结论

"`callPackage`" 模式大幅简化了我们的仓库。给定源自 `nixpkgs` 的所有软件包组成的集合，我们得以导入需要命名参数的软件包并自动调用它们。

我们还介绍了一些实用的内置函数，它们让我们能够对 Nix 函数进行内省并操作属性。这些内置函数通常不用于打包软件本身，而是充当打包的工具。它们的文档见 [Nix 手册](https://nix-doc.mczihan.link/language/builtins)。

用 Nix 编写仓库，是编写便捷函数来组合软件包这一过程的自然演进。本篇药丸展示了 Nix 如何成为一个构建和部署软件的通用工具，以及它多么适合用我们自己的约定来创建软件仓库。

## 下一篇药丸

在下一篇药丸中，我们将讨论 "`override`" 设计模式。`graphvizCore` 看起来很直白：它从 `graphviz.nix` 出发，构建一个不带 `gd` 的版本。在下一篇药丸中，我们将换一个视角：从 `pkgs.graphviz` 出发并禁用 `gd`，会怎样？
