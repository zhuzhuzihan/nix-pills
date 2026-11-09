# 函数与导入

欢迎来到第五篇 Nix 药丸。在上一篇[第四篇药丸](04-basics-of-language.md)中，我们初步接触了 Nix 语言。我们介绍了 Nix 语言的基本类型和值，以及诸如 `if`、`with` 和 `let` 之类的基本表达式。我建议你重读这些表达式的相关内容，并在 repl 中把玩它们。

函数有助于在像 [nixpkgs](https://github.com/NixOS/nixpkgs/) 这样的大型仓库中构建可复用的组件。Nix 手册中有一份[关于函数的出色讲解](https://nix.dev/manual/nix/stable/language/constructs#functions)。开始吧：一手拿着药丸，一手翻着 Nix 手册。

提醒一下进入 Nix 环境的方法：`source ~/.nix-profile/etc/profile.d/nix.sh`

## 匿名函数与单参数

函数是匿名的（lambda），并且只有一个参数。语法极其简单：输入参数名，然后是 "`:`"，接着是函数体。

```console
nix-repl> x: x*2
«lambda»
```

这里我们定义了一个函数，它接收参数 `x` 并返回 `x*2`。问题是我们无法以任何方式使用它，因为它没有名字……开个玩笑！

我们可以把函数存储在变量中。

```console
nix-repl> double = x: x*2
nix-repl> double
«lambda»
nix-repl> double 3
6
```

照旧，请忽略 `nix repl` 中用于赋值的特殊语法。我们定义了一个函数 `x: x*2`，它接收一个参数 `x` 并返回 `x*2`。这个函数随后被赋值给变量 `double`。最后我们完成了第一次函数调用：`double 3`。

特别提醒：这不像许多其他编程语言那样写成 `double(3)`，而确实是 `double 3`。

总结一下：调用函数时，先写变量名，然后是空格，接着是实参。没有更多可说的，就是这么简单。

## 多于一个参数

如何创建一个接收多个参数的函数？对于不习惯函数式编程的人来说，这可能需要一段时间才能领会。我们一步一步来。

```console
nix-repl> mul = a: (b: a*b)
nix-repl> mul
«lambda»
nix-repl> mul 3
«lambda»
nix-repl> (mul 3) 4
12
```

我们定义了一个接收参数 `a` 的函数，其函数体返回另一个函数。这另一个函数接收参数 `b` 并返回 `a*b`。因此，调用 `mul 3` 返回的是这种函数：`b: 3*b`。接着，我们用 `4` 调用这个返回的函数，得到期望的结果。

你完全可以不用括号，Nix 在解析代码时有合理的优先级：

```console
nix-repl> mul = a: b: a*b
nix-repl> mul
«lambda»
nix-repl> mul 3
«lambda»
nix-repl> mul 3 4
12
nix-repl> mul (6+7) (8+9)
221
```

这样可读性大大提高，你甚至察觉不到函数每次只接收一个实参。由于实参用空格分隔，要传入更复杂的表达式时就需要括号。在其他常见语言中，你会写成 `mul(6+7, 8+9)`。

鉴于函数只有一个参数，使用**部分应用**（partial application）就非常直接：

```console
nix-repl> foo = mul 3
nix-repl> foo 4
12
nix-repl> foo 5
15
```

我们把 `mul 3` 返回的函数存储到变量 foo 中，然后复用它。

## 参数集

现在这是 Nix 一个非常酷的特性。可以在参数上对一个集合进行模式匹配。我们先写出 `mul = a: b: a*b` 的另一种版本：先以集合作为实参，再使用模式匹配。

```console
nix-repl> mul = s: s.a*s.b
nix-repl> mul { a = 3; b = 4; }
12
nix-repl> mul = { a, b }: a*b
nix-repl> mul { a = 3; b = 4; }
12
```

第一种情况中，我们定义了一个接收单个参数的函数，然后从传入的集合中访问属性 `a` 和 `b`。注意，函数调用不带括号的语法在这种场景下非常优雅，而在其他语言中你得写成 `mul({ a=3; b=4; })`。

第二种情况中，我们定义了一个参数集（argument set）。它就像定义一个集合，只是没有值。我们要求传入的集合包含键 `a` 和 `b`。然后就可以在函数体中直接使用这些 `a` 和 `b`。

```console
nix-repl> mul = { a, b }: a*b
nix-repl> mul { a = 3; b = 4; c = 6; }
error: anonymous function at (string):1:2 called with unexpected argument `c', at (string):1:1
nix-repl> mul { a = 3; }
error: anonymous function at (string):1:2 called without required argument `b', at (string):1:1
```

只有恰好包含函数所需属性的集合才会被接受，不能多，也不能少。

## 默认属性与可变属性

可以在参数集中为属性指定**默认值**：

```console
nix-repl> mul = { a, b ? 2 }: a*b
nix-repl> mul { a = 3; }
6
nix-repl> mul { a = 3; b = 4; }
12
```

你还可以允许传入比预期更多的属性（**可变参数**，variadic）：

```console
nix-repl> mul = { a, b, ... }: a*b
nix-repl> mul { a = 3; b = 4; c = 2; }
```

不过，在函数体中你无法访问 "c" 这个属性。解决办法是使用 **@-pattern** 给传入的集合起一个名字：

```console
nix-repl> mul = s@{ a, b, ... }: a*b*s.c
nix-repl> mul { a = 3; b = 4; c = 2; }
24
```

就是这样：在集合模式之前用 name@ 给整个参数起一个名字。

使用参数集的优点：

- 带名字的无序参数：你不必记住参数的顺序。

- 你可以传入集合，这带来了全新层面的灵活性和便利性。

缺点：

- 部分应用（partial application）不能用于参数集。你必须指定完整的属性集，而不是其中一部分。

你可能会觉得它与 [Python 的 \*\*kwargs](https://docs.python.org/3/faq/programming.html#how-can-i-pass-optional-or-keyword-parameters-from-one-function-to-another) 有相似之处。

## 导入

`import` 函数是内置的，它提供了解析 `.nix` 文件的方式。自然的做法是：把每个组件定义在一个 `.nix` 文件中，然后通过导入这些文件来进行组合。

我们从最基础的开始。

`a.nix`：

```nix
3
```

`b.nix`：

```nix
4
```

`mul.nix`：

```nix
a: b: a*b
```

```console
nix-repl> a = import ./a.nix
nix-repl> b = import ./b.nix
nix-repl> mul = import ./mul.nix
nix-repl> mul a b
12
```

是的，就是这么简单。你导入一个文件，它就被解析为一个表达式。注意，被导入文件的作用域不会继承导入者的作用域。

`test.nix`：

```nix
x
```

```console
nix-repl> let x = 5; in import ./test.nix
error: undefined variable `x' at /home/lethal/test.nix:1:1
```

那么如何向模块传递信息呢？使用函数，就像我们在 `mul.nix` 中做的那样。一个更复杂的例子：

`test.nix`：

```nix
{ a, b ? 3, trueMsg ? "yes", falseMsg ? "no" }:
if a > b
  then builtins.trace trueMsg true
  else builtins.trace falseMsg false
```

```console
nix-repl> import ./test.nix { a = 5; trueMsg = "ok"; }
trace: ok
true
```

解释一下：

- 在 `test.nix` 中，我们返回一个函数。它接收一个集合，其中属性 `b`、`trueMsg` 和 `falseMsg` 有默认值。

- `builtins.trace` 是一个[内置函数](https://nix.dev/manual/nix/stable/language/builtins)，它接收两个参数：第一个是要显示的消息，第二个是要返回的值。它通常用于调试目的。

- 然后我们导入 `test.nix`，并用那个集合调用该函数。

那么消息什么时候显示？只有在它需要被求值的时候。

## 下一篇药丸

……我们终于要编写我们的第一个 derivation 了。
