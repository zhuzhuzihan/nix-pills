# 语言基础 {#basics-of-language}

欢迎来到第四篇 Nix 药丸。在[上一篇文章](03-enter-environment.md)中，我们学习了 Nix 环境。我们以用户身份安装软件、管理用户的 profile、在各 generation 之间切换，并查询 Nix store。这些是使用 Nix 进行系统管理的最基础知识。

[Nix 语言](https://nix.dev/manual/nix/stable/language/)用于编写能够产生 derivation（派生）的表达式。[nix-build](https://nix.dev/manual/nix/stable/command-ref/nix-build) 工具用于从表达式构建 derivation。即使是一名希望定制安装的系统管理员，也有必要掌握 Nix。在工作中使用 Nix 意味着你可以免费获得前几篇文章中展示的那些特性。

Nix 的语法相当陌生，因此查看现有示例可能会让你以为其中有很多魔法在起作用。实际上，它主要是通过编写一些工具函数来让事情变得方便。

另一方面，同样的语法也非常适合描述软件包，因此学习语言本身在编写软件包表达式时会带来回报。

<div class="info">

重要：在 Nix 中，一切都是表达式，没有语句。这在函数式语言中很常见。

</div>

<div class="info">

重要：Nix 中的值是不可变的。

</div>

## 值类型

Nix 2.0 包含一个名为 `nix repl` 的命令，它是一个简单的命令行工具，可用于把玩 Nix 语言。事实上，Nix 是一门[纯函数式的惰性求值语言](https://nix.dev/manual/nix/stable/language/)，而不仅仅是一组管理 derivation 的工具。在给变量赋值时，`nix repl` 的语法与 Nix 语法略有不同，但只要你记住这一点，就不会造成困扰。在用更复杂的表达式充斥你的头脑之前，我更倾向于先从 `nix repl` 开始。

启动 `nix repl`。首先，Nix 支持基本的算术运算：`+`、`-`、`*` 和 `/`。（要退出 `nix repl`，使用命令 `:q`。可以通过 `:?` 命令获得帮助。）

```console
nix-repl> 1+3
4

nix-repl> 7-4
3

nix-repl> 3*2
6
```

在 Nix 中尝试进行除法可能会带来一些意外。

```console
nix-repl> 6/3
/home/nix/6/3
```

发生了什么？回想一下，Nix 不是一门通用语言，它是用于编写软件包的领域特定语言。在编写软件包表达式时，整数除法其实并不那么有用。Nix 把 `6/3` 解析成了相对于当前目录的路径。要让 Nix 执行除法，请在 `/` 后面留一个空格。另外，你也可以使用 `builtins.div`。

```console
nix-repl> 6/ 3
2

nix-repl> builtins.div 6 3
2
```

其他运算符包括用于布尔值的 `||`、`&&` 和 `!`，以及诸如 `!=`、`==`、`<`、`>`、`<=`、`>=` 之类的关系运算符。在 Nix 中，`<`、`>`、`<=` 和 `>=` 用得不多。本系列教程后面还会介绍其他运算符。

Nix 拥有整数、浮点数、字符串、路径、布尔和 null 这些[简单](https://nix.dev/manual/nix/stable/language/#overview)类型。此外还有列表、属性集和函数。这些类型足以构建一个操作系统。

Nix 是强类型的，但它不是静态类型的。也就是说，你不能把字符串和整数混用，必须先进行转换。

如上面所演示的，只要有一个后面不跟空格的斜杠，表达式就会被解析为路径。因此要指定当前目录，请使用 `./.`。此外，Nix 还会对 URL 进行特殊解析。

并非所有 URL 或路径都能以这种方式解析。如果出现语法错误，仍然可以回退为普通字符串。字面量的 URL 和路径为额外的安全性提供了便利。

## 标识符

这里没有太多可说的，只是标识符中允许使用连字符（`-`）。这很方便，因为很多软件包的名字里都带连字符。实际上：

```console
nix-repl> a-b
error: undefined variable `a-b' at (string):1:1
nix-repl> a - b
error: undefined variable `a' at (string):1:1
```

如你所见，`a-b` 被解析成了标识符，而不是减法运算。

## 字符串

理解字符串的语法非常重要。在学习阅读 Nix 表达式时，你可能会觉得美元符号（`$`）有些含糊，但它非常重要。字符串用双引号（`"`）或两个单引号（`''`）包裹。

```console
nix-repl> "foo"
"foo"
nix-repl> ''foo''
"foo"
```

在 Python 等其他语言中，你也可以用单引号表示字符串（例如 `'foo'`），但在 Nix 中不行。

可以使用 `${...}` 语法（且仅限该语法，不能是 `$foo`、`{$foo}` 或其他任何写法）在字符串内[插值](https://nix.dev/manual/nix/stable/language/string-interpolation)完整的 Nix 表达式。

```console
nix-repl> foo = "strval"
nix-repl> "$foo"
"$foo"
nix-repl> "${foo}"
"strval"
nix-repl> "${2+3}"
error: cannot coerce an integer to a string, at (string):1:2
```

注意：请忽略 `foo = "strval"` 这个赋值，那是 `nix repl` 的特殊语法。

如前所述，你不能把整数和字符串混用，需要显式进行转换。这一点我们稍后会看到：函数调用则是另一回事。

使用两个单引号的语法的好处是，可以在字符串中直接书写双引号而无需转义：

```console
nix-repl> ''test " test''
"test \" test"
nix-repl> ''${foo}''
"strval"
```

在双引号字符串中转义 `${...}` 要使用反斜杠；在两个单引号内，则使用 `''`：

```console
nix-repl> "\${foo}"
"${foo}"
nix-repl> ''test ''${foo} test''
"test ${foo} test"
```

## 列表

列表是由空格分隔（_而非_逗号）的一系列表达式：

```console
nix-repl> [ 2 "foo" true (2+3) ]
[ 2 "foo" true 5 ]
```

列表和 Nix 中的其他一切一样，都是不可变的。可以向列表添加或移除元素，但这会返回一个新列表。

## 属性集

属性集是字符串键与 Nix 值之间的关联。键只能是字符串。编写属性集时，你也可以使用不带引号的标识符作为键。

```console
nix-repl> s = { foo = "bar"; a-b = "baz"; "123" = "num"; }
nix-repl> s
{ "123" = "num"; a-b = "baz"; foo = "bar"; }
```

对于正在阅读 nixpkgs 中 Nix 表达式的读者：不要把属性集与函数中使用的参数集混淆。

访问属性集中的元素：

```console
nix-repl> s.a-b
"baz"
nix-repl> s."123"
"num"
```

是的，你可以用字符串来访问那些不是合法标识符的键。

在属性集内部，通常无法引用同一属性集中的元素：

```console
nix-repl> { a = 3; b = a+4; }
error: undefined variable `a' at (string):1:10
```

要这样做，请使用[递归属性集](https://nix.dev/manual/nix/stable/language/constructs#recursive-sets)：

```console
nix-repl> rec { a = 3; b = a+4; }
{ a = 3; b = 7; }
```

在定义软件包时这非常方便，因为软件包往往是递归属性集。

## If 表达式

这些是表达式，而不是语句。

```console
nix-repl> a = 3
nix-repl> b = 4
nix-repl> if a > b then "yes" else "no"
"no"
```

你不能只写 `then` 分支，还必须指定 `else` 分支，因为表达式在所有情况下都必须有值。

## Let 表达式

这种表达式用于为内部表达式定义局部变量。

```console
nix-repl> let a = "foo"; in a
"foo"
```

语法是：先定义变量，然后是 `in`，接着是一个可以使用这些已定义变量的表达式。整个 `let` 表达式的值就是 `in` 之后那个表达式的值。

```console
nix-repl> let a = 3; b = 4; in a + b
7
```

我们来写两个 `let` 表达式，一个嵌套在另一个里面：

```console
nix-repl> let a = 3; in let b = 4; in a + b
7
```

使用 `let` 时，不能对同一个变量赋值两次。不过，你可以遮蔽（shadow）外层变量：

```console
nix-repl> let a = 3; a = 8; in a
error: attribute `a' at (string):1:12 already defined at (string):1:5
nix-repl> let a = 3; in let a = 8; in a
8
```

你不能在 `let` 表达式之外引用其中的变量：

```console
nix-repl> let a = (let c = 3; in c); in c
error: undefined variable `c' at (string):1:31
```

在定义变量时，你可以引用 `let` 表达式中的其他变量，就像递归属性集那样：

```console
nix-repl> let a = 4; b = a + 5; in b
9
```

因此，当你想引用外层作用域中的某个变量，而它又在当前 let 表达式中被定义时，要格外小心。递归属性集也是同样的道理。

## With 表达式

这种表达式在其他语言中很少见。你可以把它想象成 C++ 中 `using` 的更细粒度版本，或者 Python 中的 `from module import *`。你可以按表达式来决定何时把符号引入作用域。

```console
nix-repl> longName = { a = 3; b = 4; }
nix-repl> longName.a + longName.b
7
nix-repl> with longName; a + b
7
```

就是这样：它接收一个属性集，并把其中的符号引入内层表达式的作用域。当然，只有集合的键中那些合法的标识符才会被引入。如果某个符号已存在于外层作用域，同时也会被 `with` 引入，那么它将_不会_被遮蔽。不过你仍然可以引用那个属性集：

```console
nix-repl> let a = 10; in with longName; a + b
14
nix-repl> let a = 10; in with longName; longName.a + b
7
```

## 惰性求值

Nix 只在需要时才对表达式求值。在处理软件包时这是一个非常好的特性。

```console
nix-repl> let a = builtins.div 4 0; b = 6; in b
6
```

由于 `a` 不需要用到，因此没有出现除以零的错误，因为这个表达式无需被求值。这就是为什么我们可以按需定义所有软件包，同时又能非常快速地访问特定软件包。

## 下一篇药丸

……我们将讨论函数与导入。在这一篇药丸里，我尽量避免使用函数调用，否则这篇文章会太长。
