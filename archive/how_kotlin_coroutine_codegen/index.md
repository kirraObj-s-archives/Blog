+++
title = "[译] Kotlin 协程代码 Codegen"
date = 2024-04-08T00:00:00Z
+++

> 在了解如何用 [call/cc](https://en.wikipedia.org/wiki/Call-with-current-continuation) 实现无栈 Coroutine 的细节时，这个[文档](https://github.com/JetBrains/kotlin/blob/master/compiler/backend/src/org/jetbrains/kotlin/codegen/coroutines/coroutines-codegen.md)出现在我的 Google 搜索结果中。鉴于我一直在用 Kotlin 要饭，我决定翻译一下。

这个文档旨在解释关于 Kotlin 协程代码 Codegen，以便程序员无需深入编译器代码或查看编译器生成的字节码，便能理解编译器为何以这种方式工作（或更确切地说「应当如何工作」）。希望能帮助从事编译器开发的人员及高级 Kotlin 程序员理解 Kotlin 某些特定设计决策的原因。

## 可挂起的 Lambda (Suspend Lambda)

让我们从介绍它开始。Suspend Lambda 是协程的一个示例，编译器会将 lambda 内部的顺序执行的代码转化成「可挂起」的代码。下面是一个简单的例子：

```kotlin
suspend fun dummy() {}

suspend fun main() {
    val lambda: suspend () -> Unit = {
       dummy()
       println(1)
       dummy()
       println(2)
    }
    lambda()
}
```

执行以上代码会如预期打印 1 和 2，有点平平无奇。

我们只能从 Suspend Lambda 中调用「可挂起」的函数，但是也可以调用普通的、不可挂起的函数。比如，这里的 `dummy` 和 `println` 都只能在 `lambda` 里被调用。由于我们不能从普通函数中调用「可挂起」的函数，我们可以想象两个世界：「可挂起」的世界和普通的世界。或者，我们可以把它们视为两种不同的颜色，通过 `suspend` 关键字来着色。

> 译者著：这里的「颜色」涉及到著名的函数染色问题。如果你对此感兴趣，可以查看海螺翻译的这篇[文章](https://izzel.io/2022/09/04/what-color-is-your-function/)。

