+++
title = "[译] Tagless Final 快速引言"
date = 2024-04-08T00:00:00Z
+++

> [原文地址](https://nrinaudo.github.io/articles/tagless_final.html)，由 [Eric Torreborre](https://github.coxm/etorreborre) 编写。

在过去的几周内，有人询问我关于 Tagless Final 的事情。其中一些描述非常令人困惑，这应该是 Scala 社区中一个非常着迷的议题。奇怪的是，这些人所得出的结论与我所理解的 Tagless Final 编码似乎只有隐约的联系。

这片文章的目的是为了让人们了解关于 Tagless Final 最低限度的知识，以了解这个概念是怎么回事。如果你想更深入的了解，我已经就这个问题做了一个[演讲](https://nrinaudo.github.io/talks/dsl_tagless_final.html)。或者你可以直接去阅读 Oleg Kiselyov 的[论文](https://okmij.org/ftp/tagless-final/course/lecture.pdf)。

## 动机

考虑构建一个非常简单的领域特定语言 (Domain Specific Language)，暂时允许我们表达：

- 整数
- 两个整数的加法计算

举个例子：`1 + (2 + 4)`

这个东西可能看起来没啥意思。但得宜于它的简单性，能使我们专注于真正重要的事情：如何对这个 DSL 进行编程实现。

我们还需要提供多种解释器 (Interpreter)。在这篇文章中，我们将主要讨论：

- Pretty-Printing：让表达式变成美观的和人类可读的。
- 求值 (Evaluation)：计算表达式的结果。

## 初始编码

### 简单实现

任何熟悉代数数据类型的人 (Algebraic Data Types) 对于这个模型都会有直觉 —— 和类型 (Sum Types)，其中每个分支代表一个句法 (Syntax) 元素。

```scala
enum Exp:
  case Lit(value: Int)
  case Add(lhs: Exp, rhs: Exp)
```

表达式可以是一个字面值 (即整数)，也可以是一个加法。而加法是递归定义的，它的运算数 (Operands) 可以是表达式本身。

`Add` 的递归性质非常重要。因为没有它，我们就不能写出这种嵌套表达式。

```scala
// 1 + (2 + 4)
Add(Lit(1), Add(Lit(2), Lit(4)))
```

### 解释器

这种实现使我们可以通过简单的自然递归 ([Natural Recursion](https://stackoverflow.com/questions/32260444/what-is-the-definition-of-natural-recursion)) 来编写解释器，例如 Pretty-Printing：

```scala
def print(exp: Exp): String = exp match
  case Lit(value)    => value.toString
  case Add(lhs, rhs) => s"(${print(lhs)} + ${print(rhs)})"
```

求值部分遵循完全相同的模式：

```scala
def eval(exp: Exp): Int = exp match
  case Lit(value)    => value
  case Add(lhs, rhs) => eval(lhs) + eval(rhs)
```

将我们的 DSL 使用 ADT 直接实现在这里称为「初始编码」(Initial Encoding)，它完美实现了我们的初始需求。

## 表达式问题

然而这种实现有一个缺陷，如果我们需要往 DSL 中添加新的元素则会破坏现有的解释器。比如说我们想增加乘法则需要修改 `Exp` 类：

```scala
enum Exp:
  case Lit(value: Int)
  case Add(lhs: Exp, rhs: Exp)
  case Mult(lhs: Exp, rhs: Exp)
```

现有的解释器并不知道 `Mult` 也不可能处理它，因为当解释器被编写的时候，`Mult` 还不存在。当执行关于乘法相关的表达式时，代码会直接崩溃。

举个例子，以下代码会引起运行时的 `MatchError` 异常：

```
// eval(1 * 2)
eval(Mult(Lit(1), Lit(2)))
```

这也被称为「[表达式问题](https://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt)」 (The Expression Problem)，为 DSL 找到一个静态检查的编码。以允许我们在不破坏任何东西的情况下增加句法 (比如这里的乘法) 和解释器 (比如 Pretty-Printing)，我在这里有简化。但你能理解大致意思。

我认为这个名字的起源在这句话中得到了表达：

> 一种编程语言是否能解决「表达式问题」是衡量其表达能力的一个重要指标。

换句话说，「表达式问题」就是测试编程语言表达性的考验。

我们的编码显然没有解决这个问题，因为增加句法会破坏现有的解释器。让我们看看我们是否能修复这个问题。

## 最终编码

### 用函数构建

「最终编码」的核心直觉是使用函数来编码表达式，而不是使用 ADT。我不完全确定这个想法是怎么产生的。但有一种方式可以引导我们理解，考虑向现有的 DSL 添加元素。这可以看作成将两个 DSL (旧的那个和一个包含我们想要添加所有句法的新 DSL) 复合成一个更丰富的 DSL，从这个角度看，ADT 并不是一个很好的工具。它们无法复合。另一方面，函数则是被广知易于被复合的。

因此，如果我们把我们的 DSL 表示为函数，可能会取得更大的进步。让我们试试吧。

```scala
def lit(value: Int)         = value
def add(lhs: Int, rhs: Int) = lhs + rhs
```

这样做效果很好，甚至用起来跟「初始编码」惊人的相似。

```scala
// 1 + (2 + 4)
add(lit(1), add(lit(2), lit(4)))
```

我们已将我们的第一个 DSL 用函数定义。现在我们来定义乘法操作。

```scala
def mult(lhs: Int, rhs: Int) = lhs * rhs
```

由于所有函数都直接使用表达式的求值结果，即 `Int`。它们可以毫无障碍地一起工作：

```scala
mult(lit(1), lit(2))
```

可以称为「最终编码」，它的一个特点是我们直接处理解释后的值。看看 `lit`，`mult`。它们都在接收和返回 `Int`，但这也是一个主要缺陷。

### 支持多个解释器

目前这个简单的「最终编码」的问题在于，在我们的创新中，似乎我们有些忘记了我们的目标。如果你对编写多个解释器不感兴趣，那么我们的工作就已经完成了。因为我们的函数会立即对表达式立即求值，所以我们只有一个解释器：求值。

我们正在尝试解决表达式问题，这意味着我们要能在不破坏任何东西的前提下编写多个解释器。我们目前的实现不能编写超过一个，无论是否设计破坏性变更。显然我们的实现存在缺陷，我们需要找到一种方向来编写多个解释器。

这里的直觉是，既然我们正在使用函数，那么告诉它们如何解释某些数据的唯一方式就是将解释器作为参数传递。

这样的解释器必须能够处理我们语言的每个声明，像这样：

```scala
trait ExpSym:
  def lit(i: Int): ???
  def add(lhs: ???, rhs: ???): ???
```

这个 `Sym` 后缀来自于人们常称其为「语义句法」(Symantic)，是一个令人讨厌的组合词。由 [Syntax](https://en.wikipedia.org/wiki/Syntax) (句法) 和 [Semantic](https://en.wikipedia.org/wiki/Semantics) (语义) 构成：类型描述了我们的句法，类型的值则表述了其语义。

问题在于 `???` 类型，它们应该被填入什么，这可能并不容易被看出来。

首先来看返回值：因为它们是对表达式解析后产生的结果。它们的类型应与解释器返回的类型一致 (在本文其他部分将此称为「解释类型」)，这需要参数化。Pretty-Printer 的求值结果为字符串，而一个求值器的求值结果为整数。我们显然需要将其可配置，这给我们带来了如下设计：

```scala
trait ExpSym[A]:
  def lit(i: Int): A
  def add(lhs: ???, rhs: ???): A
```

对于其余的 `???` 部分，记住我们在进行「最终编码」。在这种编码中，我们操纵的是解释后的值，而不是中间表示。所以 `add` 将要取嵌套表达式的结果，那就是 `A`。

```scala
trait ExpSym[A]:
  def lit(i: Int): A
  def add(lhs: A, rhs: A): A
```

如果不是很清晰的话，我们可以写一个表达式来帮助理解这些类型如何对应起来的。

```scala
// 1 + (2 + 4)
def exp[A](sym: ExpSym[A]): A =
import sym.*
add(lit(1), add(lit(2), lit(4)))
```

我将 `exp` 定义为方法而不是函数的原因是因为它是多态 (Polymorphic) 的。对于多态函数的支持要么不存在 (Scala 2)，要么是带有非常令人不幸的[语法](https://docs.scala-lang.org/scala3/reference/new-types/polymorphic-function-types.html)。(Dotty Scala / Scala 3)

看 `add(lit(1), ...)` 这部分，它有助于理解为什么我们的子表达式和我们解释的值会共享同一类型。

我们现在已经为我们的 DSL 编码并有了一个表达式。我们确认它是否有效的唯一方式就是编写一个实际的解释器，目前我们先实现 Pretty-Printing。将解释类型设为 String 然后填充空白：

```scala
val print = new ExpSym[String]:
  def lit(i: Int)                   = i.toString
  def add(lhs: String, rhs: String) = s"($lhs + $rhs)"
```

在完成所有操作之后，Pretty-Print 我们的表达式只是一个简单的方法调用：

```scala
exp(print)
// val res0: String = (1 + (2 + 4))
```

### 语法糖

我必须承认以上实现是非常繁琐的，但可以通过以下方式改进：

- 声明 `lit` 和 `add` 的辅助函数，这样我们就不用显式导入 `sym.*`。
- 使 `ExpSym` 成为隐式参数，避免我们显式传递它。

这基本上意味着我们将 `ExpSym` 变成一个有专有句法的类型类 (TypeClass)，让我们开始重构吧。

```scala
def lit[A](i: Int)(using sym: ExpSym[A]): A =
  sym.lit(i)

def add[A](lhs: A, rhs: A)(using sym: ExpSym[A]): A =
  sym.add(lhs, rhs)
```

这些辅助函数能使我们以一种令人愉悦的方式重写 `exp` 方法。

```scala
// 1 + (2 + 4)
def exp[A: ExpSym]: A =
  add(lit(1), add(lit(2), lit(4)))
```

首先，我们需要使我们的 Pretty-Printer 隐式化 (如果你想更加专业一点，你可以说把它变成了一个 `ExpSym` 类型类的一个实例)：

```scala
given ExpSym[String] with
  def lit(i: Int)                   = i.toString
  def add(lhs: String, rhs: String) = s"($lhs + $rhs)"
```

调用解释器因此稍微变得容易一些，我觉得这样写起来也更加愉快：

```scala
exp[String]
```

我们在不破坏任何现有代码的情况下为我们的 DSL 添加了句法，我希望这点能很明确。我们在不破坏任何东西的前提下添加了解释器，为这仅仅涉及到编写 `ExpSym` 和 `MultSym`。

### 操纵 DSL 的值

不过我们的实现有一个重大缺陷，它不允许我们操作 DSL 的值。我们能做的只是解释它们，例如我们不能把它们传递给其他函数，或者从函数中返回。这是因为我们把它们声明为方法，而方法不是一等公民。

理论上，这不应该是个大问题。因为编译器可以在需要的时候把方法转换成函数，而函数是一等公民。遗憾的是，由于我们的方法是多态的，所以情况不会太理想。

如果你正在使用 Scala 2，情况就不太好了。Scala 2 并不支持多态函数，你可以继续使用方法。这大大降低了最终编码的实用性 (表达式不是值意味着你不能从文本文件中解析它们，比如说)，或者你可以写大量的额外代码来模拟多态函数。后者可行，但并不有趣。也不是本篇文章的主题。

如果你正在使用 Scala 3，情况会好一些。因为 Scala 3 确实支持多态函数，我鼓励你自己去试试。比如说考虑将表达式编码成 Json 并反序列化为内存表示的方法。过程可能不是很愉快，但肯定会很有启发性。

## 高阶语言 (Higher-Order Languages)

我们谈到了初始编码和最终编码之间的区别。现在我们来解决无标签 (Tagless) 部分。为此，我们必须考虑一个高阶语言。在这种语言中，求值一个表达式可能会产生多个类型的结果。

为了实现这一点，我们使用上述提及现有的 DSL。并添加两个数字是否相等的的功能，例如：`(1 + 2) = 3`。

### 标签式的初始编码

正如我们所见，初始编码只是对使用 ADT 方案的困惑命名。这是新 DSL 的 ADT 定义：

```scala
enum Exp:
  case Lit(value: Int)
  case Add(lhs: Exp, rhs: Exp)
  case Eq(lhs: Exp, rhs: Exp)
```

到目前为止一切都很顺利，我们可以将 DSL 的表达式表示为值。还可以容易地为它们编写一个 Pretty-Printer 函数：

```scala
def print(exp: Exp): String = exp match
  case Lit(value)    => value.toString
  case Add(lhs, rhs) => s"(${print(lhs)} + ${print(rhs)})"
  case Eq(lhs, rhs)  => s"(${print(lhs)} = ${print(rhs)})"
```

然而求值而言存在问题。首先，我们如何为其指定类型？一个表达式可能会得出整数 (`Add` 和 `Lit`) 或一个布尔值 (`Eq`)。其次，在我们无法保证两个运算数都会得出整数的前提下我们如何实现 `Add`？

```scala
def eval(exp: Exp): ??? = exp match
  case Lit(value)    => value      // Returns Int
  case Add(lhs, rhs) => ???
  case Eq(lhs, rhs)  => lhs == rhs // returns Boolean
```

我们可以考虑使用 `Any` 和在运行时进行类型检查，但我不想这么做。首先我有自尊，其次，我们知道这不会解决我们的问题。如果我们必须依靠运行时类型检查，这意味着我们没有对其进行类型检查，而这是为了解决表达式问题所必须做的。

相反，我们使用 Scala 3 里的一个好东西，联合类型 (Union Type)。如果你还在使用 Scala 2，则可以使用和它一样有效的和类型。

联合类型允许我们表达一个类型可能是多个可选项之一；在我们的例子中，是 `Int` 和 `Boolean`。这些都是静态检查过的，也正是我们所期望的。

```scala
def eval(exp: Exp): Int | Boolean = exp match
  case Lit(value)    => value
  case Add(lhs, rhs) => (eval(lhs), eval(rhs)) match
                          case (left: Int, right: Int) => left + right
                          case _                       => sys.error("Type error")
  case Eq(lhs, rhs)  => eval(lhs) == eval(rhs)
```

这被称为标签式的初始编码，标签部分是因为我们使用 `Int | Boolean` 这样的类型标签来追踪我们正在处理的内容。这种编码形式并不是很令人愉快。

特别是 `Add`，它是可怕的。这种模式匹配 (Pattern Matching) 代码我在做噩梦的时候梦见过。敏锐的读者应该会注意到，事实上我已经使这个情况比应有的情况少了一些不愉快之处。通过让 `Eq` 的匹配在不应该起作用的场景中起作用，在那里原应有另一个尴尬的模式匹配。

这是一个更深层次问题的症状，我们无论是有意识地还是无意识地一直忽视着：我们的 ADT 允许我们编写类型不匹配的异常，例如 `(1 = 2) + 3`。解决这个问题的方法将是一个无标签式的初始编码。

### 无标签式的初始编码
