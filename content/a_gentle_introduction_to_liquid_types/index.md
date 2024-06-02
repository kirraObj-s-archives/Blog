+++
title = "[译] 简要介绍 Liquid Types"
date = 2024-06-02T00:00:00Z
+++

> [原文](https://goto.ucsd.edu/~ucsdpl-blog/liquidtypes/2015/09/19/liquid-types/)由 UCSD The Programming Systems Group 发表。


类型系统已成功用于编译时期捕捉到潜在的错误，比如我们无法用 Boolean 除以 Int。尽管如此，良好类型化（Well-Typed）的代码仍然可能出现错误，比如将 Int 除以零通常会抛出一个运行时异常。

Liquid Types 通过引入逻辑谓词（Logical Predicates）扩展现有的类型系统，使我们能够指定并自动验证代码的语义属性（Semantic Properties）。

# 结构 vs. 语义属性

大多数类型系统依赖于其值的结构进行推理。Int 和 Boolean 具有不同的结构，通常在内部表示方式上有所不同，并且可以使用不同的操作集：即 Int 可以进行除法运算，而 Boolean 可以进行合取运算（Conjuncted）。

结构之外，值还具有语义属性。Liquid Types 通过逻辑谓词扩展了现有的类型系统，使我们能够指定值的语义属性。例如，Int 类型可以通过逻辑谓词来描述非零的整数：

```haskell
{v : Int | v /= 0}
```

或者用来描述自然数（Natural Numbers）：

```haskell
{v : Int | v >= 0}
```

由于 Liquid Types 能够描述值的语义，使我们可以静态地捕捉语义类型错误，例如上面阐述的除零案例。

# 通过类型规范防止除零错误

一个除法操作符的类型签名，给定两个 Int，得到另一个 Int。

```haskell
div :: Int -> Int -> Int
```

该类型签名有效地规定了操作符的结构。如果我们的程序可以编译，所有传给 `div` 的参数都是 Int，而不可能是 Boolean。

使用 Liquid Types，我们可以精炼（Refine）类型签名来描述操作符的语义属性，特别是除数不应该为零。

```haskell
div :: Int -> {v : Int | v /= 0} -> Int
```

上述类型为 `div` 操作符进行了规范。如果程序通过了上述类型检查，编译器在编译时就会验证 `div` 的第二个参数是否不同于零，从而防止除零运行时异常。接下来我们看看这个验证是如何进行的。

# 规范验证

验证是给定代码和一些规范（这里是以类型签名的形式）的过程，决定代码是否满足这些规范。

考虑 `div` 操作符的三种求值情况：`good`、`bad` 和 `impr (Imprecise)`：

```haskell
good = div 42 one  -- OK
bad  = div 42 zero -- 类型错误
impr = div 42 nat  -- 类型错误

one  :: { v : Int | 0 < v  }
one  = 1

zero :: { v : Int | 0 == v }
zero = 0

nat  :: { v : Int | 0 <= v }
nat  = 42
```

验证将决定只有 `good` 是正确求值了 `div`，过程如下。

首先，它触发子类型查询。`t1` 是 `t2` 的子类型，如果每个具有 `t1` 类型的表达式也具有 `t2` 类型，则记做 `t1 <: t2`。

在我们的例子中，子类型查询将检查在每次求值 `div` 时，除数的类型是否是规范中第二个参数的子类型，即 `{v : Int | v /= 0}`。例如，`good` 将触发以下子类型查询：

```haskell
{v : Int | 0 < v} <: {v : Int | 0 /= v}
```

然后，通过蕴涵检查（Implication checking）解决子类型查询。如果 `p ⟹ q` 则 `{v : Int | p } <: {v : Int | q }` 成立，例如：

```haskell
{v : Int | 0 < v} <: {v : Int | 0 /= v}
```
等价于

```haskell
∀ v. 0 < v ⟹ 0 ≠ v
```

因此 `good` 的验证成功，因为 `0 < v ⟹ 0 ≠ v`，但 `bad` 和 `impr` 的验证失败，因为 `0 = v ⟹ 0 ≠ v` 和 `0 ≤ v ⟹ 0 ≠ v` 都不成立。

请注意，类型为你的程序提供了一种抽象。当我们定义 `nat` 时，我们将值 42 抽象为一个自然数，即 `nat :: {v : Int | 0 <= v}`。这对于 42 来说是一个正确的规范，但不够精确，因为它忽略了 42 不是 0 的信息，因此 42 是一个好的除数。简而言之，我们的分析并不完全，因为它可能会产生并非真正错误的类型错误。好的一面是，我们的分析是可靠的，如果它说 OK，就一定不会违反规范。

# 延伸阅读

2008 年，Rondon、Kawaguchi 和 Jhala 在 UCSD 的 ProgSys 小组[引入了](https://goto.ucsd.edu/~rjhala/liquid/liquid_types.pdf) Liquid Types（Logically Qualified Data Types）。从那时起，它们已被用于规范和验证 ML、C 和 Haskell。

Liquid Types 是依值类型（Dependent Types），即类型依赖于任意表达式，类似于我们在 Coq、Agda 等中看到的类型。

Liquid Types 是精炼类型（Refinement Types），即在基本类型的基础上，通过逻辑谓词进一步限定的类型。这些逻辑谓词不能是任意的表达式（如依值类型），而是从一个[子语言](https://en.wikipedia.org/wiki/Sublanguage) （Sub-Language）中抽取的表达式。精化类型系统的例子包括 DML、Sage 和 F* 等。

Liquid Types 是一种受约束的精炼类型，其中的逻辑谓词来自一个具有可判定性的子语言，即一个可以进行蕴含检查的逻辑语言。例子包括无量词理论，如整数线性算术（Linear Arithmetic over Integers）、数组理论（Theory of Arrays）和集合论（Sets Theory）等。这些理论的一个共同特点是它们允许我们通过算法来确定逻辑表达式的真值，从而实现自动化的类型验证。

由于我们使用的约束是具有可判定性的（Decidable），我们可以可靠地使用 SMT-Solvers 来进行蕴含检查。这意味着，类型检查可以简化为蕴含检查，从而使得 Liquid Types 允许可判定性的类型检查和类型推断。

另一方面，可判定性的约束在语法上对规范语言产生了限制。使用可判定性逻辑究竟能表示哪些规范，这仍然是一个开放的问题。然而，通过使用抽象（Abstract）或有界精炼类型（Bounded Refinement Types）等技巧，我们可以在仅使用可判定性逻辑的情况下表达复杂的语义属性。

# 总而言之

Liquid Types 提供了一种通过受约束的依值类型来规范和自动验证代码语义属性的方法。它们使用一种具有可判定性的逻辑子语言来描述这些约束条件，从而实现了一个高度自动化的类型系统，并且减少了对显式类型注释的需求。