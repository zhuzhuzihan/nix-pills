# 基础依赖与钩子

欢迎来到第 20 篇 Nix 药丸。在上一篇[第 19 篇](19-fundamentals-of-stdenv.md)药丸中，我们介绍了 Nixpkgs 的 stdenv，包括 `setup.sh` 脚本、`default-builder.sh` 辅助脚本，以及 `stdenv.mkDerivation` builder。我们重点讲解了 stdenv 是如何组装起来的、如何使用它，还简单了解了 `genericBuild` 的各个阶段。

这次，我们将关注用 `stdenv.mkDerivation` 构建的软件包之间的交互。当然，软件包需要相互依赖。为此我们有了 `buildInputs` 和 `propagatedBuildInputs` 属性。我们还发现，依赖有时需要以依赖方无法预测、也不应该预测的方式影响下游。为此我们有了 setup hook 和 env hook。这 4 个概念加在一起，支撑了几乎所有构建期的软件包交互。

<div class="info">

注意：为了支持交叉编译，依赖与钩子基础设施的复杂度随着时间推移不断增加。一旦掌握了核心概念，你就能够理解这些额外的复杂度。作为起点，你可以参考 nixpkgs 的提交 [6675f0a5](https://github.com/nixos/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25)，那是不含交叉编译复杂度的最后一个 stdenv 版本。

</div>

## `buildInputs` 属性

对于最简单的依赖——当前软件包直接需要另一个软件包——我们使用 `buildInputs` 属性。这正是[第 8 篇药丸](08-generic-builders.html)中我们的 builder 所用的模式。为了演示，我们先构建 GNU Hello，然后构建另一个软件包，它提供一个 shell 脚本来 `exec` Hello。

```nix
let

  nixpkgs = import <nixpkgs> { };

  inherit (nixpkgs) stdenv fetchurl which;

  actualHello = stdenv.mkDerivation {
    name = "hello-2.3";

    src = fetchurl {
      url = "mirror://gnu/hello/hello-2.3.tar.bz2";
      sha256 = "0c7vijq8y68bpr7g6dh1gny0bff8qq81vnp4ch8pjzvg56wb3js1";
    };
  };

  wrappedHello = stdenv.mkDerivation {
    name = "hello-wrapper";

    buildInputs = [
      actualHello
      which
    ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out/bin"
      echo "#! ${stdenv.shell}" >> "$out/bin/hello"
      echo "exec $(which hello)" >> "$out/bin/hello"
      chmod 0755 "$out/bin/hello"
    '';
  };
in
wrappedHello
```

注意，wrappedHello 这个 derivation（派生）是从 `PATH` 中找到 `hello` 二进制文件的。之所以可行，是因为 stdenv 中包含了类似这样的代码：

```sh
pkgs=""
for i in $buildInputs; do
    findInputs $i
done
```

其中 `findInputs` 的定义如下：

```sh
findInputs() {
    local pkg=$1

    ## 不需要重复处理已经处理过的软件包
    case $pkgs in
        *\ $pkg\ *)
            return 0
            ;;
    esac

    pkgs="$pkgs $pkg "

    ## 实际上这里还有更多代码，目前可以先忽略。
}
```

然后，在这段代码运行之后：

```sh
for i in $pkgs; do
    addToEnv $i
done
```

其中 `addToEnv` 的定义如下：

```sh
addToEnv() {
    local pkg=$1

    if test -d $1/bin; then
        addToSearchPath _PATH $1/bin
    fi

    ## 实际上这里还有更多代码，目前可以先忽略。
}
```

`addToSearchPath` 调用会在 `$1/bin` 存在时把它加入 `_PATH`（代码在[这里](https://github.com/NixOS/nixpkgs/blob/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/stdenv/generic/setup.sh#L60-L73)）。当 `buildInputs` 中的所有软件包都处理完毕后，`_PATH` 的内容会按如下方式加入 `PATH`：

```sh
PATH="${_PATH-}${_PATH:+${PATH:+:}}$PATH"
```

有了 `PATH` 中真正的 `hello`，`installPhase` 应该就说得通了。

## `propagatedBuildInputs` 属性

`buildInputs` 覆盖的是直接依赖，但间接依赖怎么办——一个软件包需要第二个软件包，而后者又需要第三个？Nix 本身处理得很好，它会理解前面构建中讲到的各种依赖闭包（closure）。但 `buildInputs` 提供的那些便利呢，也就是在 `pkgs` 环境变量中的累积，以及把 `«pkg»/bin` 目录加入 `PATH`？为此，stdenv 提供了 `propagatedBuildInputs`：

```nix
let

  nixpkgs = import <nixpkgs> { };

  inherit (nixpkgs) stdenv fetchurl which;

  actualHello = stdenv.mkDerivation {
    name = "hello-2.3";

    src = fetchurl {
      url = "mirror://gnu/hello/hello-2.3.tar.bz2";
      sha256 = "0c7vijq8y68bpr7g6dh1gny0bff8qq81vnp4ch8pjzvg56wb3js1";
    };
  };

  intermediary = stdenv.mkDerivation {
    name = "middle-man";

    propagatedBuildInputs = [ actualHello ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out"
    '';
  };

  wrappedHello = stdenv.mkDerivation {
    name = "hello-wrapper";

    buildInputs = [
      intermediary
      which
    ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out/bin"
      echo "#! ${stdenv.shell}" >> "$out/bin/hello"
      echo "exec $(which hello)" >> "$out/bin/hello"
      chmod 0755 "$out/bin/hello"
    '';
  };
in
wrappedHello
```

可以看到，中间软件包有一个 `propagatedBuildInputs` 依赖，而包装软件包只需要对中间层有一个 `buildInputs` 依赖。

这是怎么工作的？你可能以为我们在 Nix 里做了什么，但实际上这不是在求值期（eval time）完成的，而是在构建期由 bash 完成。让我们看看 stdenv 的 `fixupPhase` 中的一部分：

```sh
fixupPhase() {

    ## 此处省略

    if test -n "$propagatedBuildInputs"; then
        mkdir -p "$out/nix-support"
        echo "$propagatedBuildInputs" > "$out/nix-support/propagated-build-inputs"
    fi

    ## 此处省略

}
```

它把被传播的构建输入（propagated build inputs）转储到 `$out/nix-support/` 下一个同名文件中。然后，回到 `findInputs`，看看我们之前省略掉的底部的几行：

```sh
findInputs() {
    local pkg=$1

    ## 实际上这里还有更多代码，目前可以先忽略。

    if test -f $pkg/nix-support/propagated-build-inputs; then
        for i in $(cat $pkg/nix-support/propagated-build-inputs); do
            findInputs $i
        done
    fi
}
```

可以看到 `findInputs` 实际上是递归的：查看每个依赖的被传播构建输入，以及那些依赖的被传播构建输入，依此类推。

实际上，我们把之前的 `findInputs` 调用点做了简化；在现实中，`propagatedBuildInputs` 也会被循环处理：

```sh
pkgs=""
for i in $buildInputs $propagatedBuildInputs; do
    findInputs $i
done
```

这说明了一个重要的观点。仅就_当前_软件包而言，一个依赖是否被传播并不重要。它都会以同样的方式被处理：传入 `findInputs` 和 `addToEnv` 调用。（由 `findInputs` 发现、同样累积在 `pkgs` 中并传给 `addToEnv` 的那些软件包，在两种情况下也完全相同。）然而对下游而言，这当然很重要，因为只有被传播的直接依赖才会被写入 `$out/nix-support/propagated-build-inputs`。

## Setup Hooks

如前所述，有时依赖需要以不同于_仅仅作为依赖_的方式影响使用它们的软件包。[^1] `propagatedBuildInputs` 实际上可以看作这方面的一个例子：使用它的软件包实际上是在把这些依赖「注入」为其下游依赖者的额外 `buildInputs`。但总的来说，一个依赖可能以任意方式影响依赖它的软件包。_任意_是这里的关键词。我们可以教 `setup.sh` 认识诸如 `«pkg»/nix-support/propagated-build-inputs` 这样的上游软件包机制，但无法应对任意的交互。

setup hook 就是我们为此拥有的基本构件。在 nixpkgs 中，「hook」基本上就是一个 bash 回调，setup hook 也不例外。让我们看看 `findInputs` 中还没讲到的最后一部分：

```sh
findInputs() {
    local pkg=$1

    ## 实际上这里还有更多代码，目前可以先忽略。

    if test -f $pkg/nix-support/setup-hook; then
        source $pkg/nix-support/setup-hook
    fi

    ## 实际上这里还有更多代码，目前可以先忽略。

}
```

如果一个软件包包含路径 `«pkg»/nix-support/setup-hook`，那么任何把该软件包作为依赖的基于 stdenv 的构建都会 source 它。

这比本章介绍的其他任何机制都更通用。例如，试着编写一个 setup hook，让它达到与一条 _propagatedBuildInputs_ 条目相同的效果。几乎可以把它看作绕过 Nix 正常隔离保证、以及「依赖是不可变且惰性」这一原则的一个逃生舱口。我们并没有做任何不安全的事情，也没有修改依赖，但我们确实在允许任意的临时行为。因此，setup hook 只应作为最后手段使用。

## Environment Hooks

最后一个便利机制是 environment hook。回顾[第 12 篇药丸](12-inputs-design-pattern.md)，我们为 `-I` 标志创建了 `NIX_CFLAGS_COMPILE`，为 `-L` 标志创建了 `NIX_LDFLAGS`，方式与我们准备 `PATH` 时类似。其中一个难看之处在于它的反模块化。在通用 builder 中构建 `PATH` 是合理的，因为 `PATH` 是 shell 使用的，而通用 builder 本质上与 shell 绑定在一起。但 `-I` 和 `-L` 标志只与 C 编译器相关。stdenv 并不绑定于必须包含一个 C 编译器（尽管默认如此），而且还有其他编译器，它们可能采用完全不同的标志。

第一步，我们可以把那段逻辑移到 C 编译器的一个 setup hook 上；事实上，我们在 CC Wrapper 中正是这样做的。[^2] 但这个模式出现得相当频繁，于是有人决定添加一些辅助支持来减少样板代码。

`addToEnv` 的另一半是：

```sh
addToEnv() {
    local pkg=$1

    ## 实际上这里还有更多代码，目前可以先忽略。

    # 运行由 setup-hook 脚本设置的软件包专属钩子。
    for i in "${envHooks[@]}"; do
        $i $pkg
    done
}
```

列在 `envHooks` 中的函数会被应用到传入 `addToEnv` 的每一个软件包。我们可以编写这样的 setup hook：

```sh
anEnvHook() {
    local pkg=$1

    echo "I'm depending on \"$pkg\""
}

envHooks+=(anEnvHook)
```

如果某个依赖带有这个 setup hook，那么所有依赖都会被这样 `echo` 出来。允许依赖了解它的_同级_依赖，正是编译器所需要的。

## 下一篇药丸……

……我还不确定！我们可以谈谈交叉编译所必需的额外依赖类型和钩子，在我们这里所学的基础上覆盖今天实际运作的 stdenv。我们可以谈谈 nixpkgs 是如何引导（bootstrap）的。或者我们可以谈谈 `localSystem` 和 `crossSystem` 是如何被细化为每个引导阶段所接收的 `buildPlatform`、`hostPlatform` 和 `targetPlatform` 的。告诉我们你最感兴趣的是哪一个！

[^1]: 现在我们可以精确地定义：`addToEnv` 单独做的事情就是对依赖的最小处理；也就是说，一个_仅仅_是依赖的软件包，就_只_会对它应用 `addToEnv`。

[^2]: 在本篇药丸建议使用的 nixpkgs 版本中，它叫 [GCC Wrapper](https://github.com/NixOS/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/build-support/gcc-wrapper)；当时 Darwin 和 Clang 支持还没有促使它改名。
