+++
title = "[译] 解析，而非验证"
date = 2024-05-12T00:00:00Z
+++

> [原文](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)由 Alexis King 发表。

一直以来，我都难以简洁明了地向他人解释什么是类型驱动设计（Type-Driven Design）。经常有人问我：「你是怎么想到这种方法的？」我发现我无法给他们一个令人满意的答案。这并不是一时的灵光一现 —— 针对其动机我有一个逐步的推理过程。而不是凭空从哪个地方蹦出来的。然而，我在向他们传达这个过程方面并不太成功。

然而，大概一个月前我在 Twitter 上[反思在静态和动态类型语言中解析 JSON 的差异](https://twitter.com/lexi_lambda/status/1182242561655746560)。最终，我终于找到了我要的东西。现在我有了一个简洁的口号，它概括了类型驱动设计对我来说的意义，更棒的是，它只有三个单词：

<p align="center"><b>Parse, don’t validate.（解析，而非验证）</b></p>

<br>

# 类型驱动设计的本质

好吧，我承认：除非你已经了解什么是类型驱动设计，否则这个引人注目的标语对你可能意义不大。幸运的是，这篇博客的剩余部分就是为此而写。我将详细解释我所指的含义 —— 但首先，我们需要引入一些设想。

## 探索可能性

静态类型系统的一个美妙之处在于，它们可以使回答诸如「这个函数能被实现么？」这类问题变得可能，有时甚至容易。以一个极端的例子为例，考虑下面 Haskell 代码的类型签名：

```haskell
foo :: Integer -> Void
```

能实现 `foo` 函数吗？答案是不行，`Void` 是一个不包含任何值的类型，因此任何函数都不可能产生类型为 `Void` 的值。这个例子可能有点无聊，但如果我们选择一个更现实的例子，这个问题就会变得更加有趣：

> 严格来讲，在 Haskell 此处忽略了 Bottom 值。这是一种理论上可以代表任何值的构造。但并非「真实」的值（不同于其他语言中的 `null`），它们类似于无限循环或抛出异常的计算。通常我们避免使用它，因此请假设它不存在，这并不影响上述的讨论。但不要仅凭我的话来判断，我推荐阅读 Danielsson 的 《[Fast and Loose Reasoning is Morally Correct](https://www.cs.ox.ac.uk/jeremy.gibbons/publications/fast+loose.pdf)》来理解这个问题。

```haskell
head :: [a] -> a
```

这个[函数]((https://hackage.haskell.org/package/hedgehog-1.4/docs/Hedgehog-Internal-Prelude.html#v:head)返)会返回列表中的第一个元素。能实现吗？它看起来并不复杂，但如果我们尝试去实现它，编译器将不会让我们这么做：

```haskell
head :: [a] -> a
head (x:_) = x
```

```text
warning: [-Wincomplete-patterns]
    Pattern match(es) are non-exhaustive
    In an equation for ‘head’: Patterns not matched: []
```

错误信息很有帮助地指出了我们的函数是偏函数（Partial Function），也就是说它并不适用于所有可能的输入。具体来说，这里没有定义当输入为 `[]` 的情况，即空列表。所以这里的报错是有道理的，因为列表为空我们就不可能返回列表的第一个元素 —— 没有元素可以返回！因此这个函数在定义上不是全函数（Total Function）。

## 将偏函数重构成全函数

对于只具有动态类型语言背景的人来说，这可能看起来有些复杂。如果我们有一个列表，我们可能确实想要获取它的第一个元素。「获取列表的第一个元素」的操作在 Haskell 是可以实现的，它只是需要一些额外的仪式。有两种不同的方法可以修正 `head` 函数，我们将从最简单的方法开始。

## 调整期望值

如前所属，`head` 是偏函数。因为如果列表为空时没有元素可以返回：我们做出了一个我们根本无法实现的承诺。幸运的是，有一个简单的解决方案可以解决这个困境：我们可以弱化我们的承诺。由于我们不能保证调用者能够从列表中得到一个元素，我们将尽可能返回一个元素，但我们保留完全不返回任何东西的权利。在 Haskell 中，我们用 `Maybe` 类型来表达这种可能性：

```haskell
head :: [a] -> Maybe a
```

这给了我们实现 `head` 所需的自由 —— 当我们发现无法返回 `a` 类型的值时，允许我们返回 `Nothing`。

```haskell
head :: [a] -> Maybe a
head (x:_) = Just x
head []    = Nothing
```

问题解决了，对么？目前来说确实，但这个解决方案有一个隐形成本。

当我们实现 `head` 时，返回 `Maybe` 无疑是最方便的。然而，当我们实际使用它时，它变得明显不那么方便了！由于 `head` 有可能返回 `Nothing`，调用者必须处理这种可能性，有时这种推锅可能会令人非常难过。为了理解为什么，考虑下面的代码：

```haskell
getConfigurationDirectories :: IO [FilePath]
getConfigurationDirectories = do
  configDirsString <- getEnv "CONFIG_DIRS"
  let configDirsList = split ',' configDirsString
  when (null configDirsList) $
    throwIO $ userError "CONFIG_DIRS cannot be empty"
  pure configDirsList

main :: IO ()
main = do
  configDirs <- getConfigurationDirectories
  case head configDirs of
    Just cacheDir -> initializeCache cacheDir
    Nothing -> error "should never happen; already checked configDirs is non-empty"
```

当 `getConfigurationDirectories` 从环境中检索文件路径列表时，它会主动检查列表是否非空。然而，当我们在 `main` 函数中使用 `head` 来获取列表的第一个元素时，返回的 `Maybe FilePath` 结果仍然要求我们处理一个我们知道永远不会发生的 `Nothing` ，这产生了几个方面的问题：

-  这写起来很蛋疼，我们已经检查过列表是非空的，为什么还要在代码中增加一个冗余检查呢？
-  它有潜在的性能成本。虽然在这个特殊的例子中，这种检查的成本微不足道，但我们可以想象在一个更复杂的场景中，这些冗余检查可能会累积，比如在一个紧密的循环中。
-  最糟糕的是，这段代码有一个潜在的错误！如果 `getConfigurationDirectories` 被修改为不再检查列表是否为空，无论是有意还是无意的，程序员可能忘了要同步更新 `main` 函数，瞬间「不可能」的错误不仅可能发生，而且很可能发生。

这种冗余检查的需求实际上迫使我们在类型系统中留下了一个漏洞。如果我们能编译期证明 `Nothing` 的情况是不可能发生的，那么一旦 `getConfigurationDirectories` 的修改导致其不再检查列表是否为空，这样的变更将会使得之前的证明失效，并触发一个编译时错误。然而，根据目前的编写方式，我们不得不依赖测试套件（Test Suite）或手动检查来发现这个错误。

## 前瞻性设计

在前面的部分中，我们尝试修改 `head` 函数的实现，但仍有不足之处。我们希望这个函数能更加智能：如果我们已经验证过列表非空，`head` 应该能直接返回列表的第一个元素，而不需要我们处理我们已知不可能发生的情况。那么，我们该如何做呢？

让我们再次回顾一下 `head` 函数的原始（偏函数版本）类型签名：

```haskell
head :: [a] -> a
```

上面已经说明，我们可以通过弱化返回类型中的承诺，将偏类型签名重构成全类型签名。然而，既然我们不想这么做，那么就只剩下一个可以改变的东西了：传入的参数类型（在本例中是 `[a]` ）。与其削弱返回类型，我们可以加强参数类型，从一开始就消除在空列表上调用 `head` 的可能性。

为此，我们需要一个代表非空列表的类型。幸运的是 `Data.List.NonEmpty` 中已有的 `NonEmpty` 类型正好符合这一需求。它的定义如下：

```haskell
data NonEmpty a = a :| [a]
```

注意，`NonEmpty a` 实际上只是一个元组（Tuple），包含一个元素 `a` 和一个可能为空的列表 `[a]`。这种设计巧妙地模拟了一个非空列表，通过将列表的第一个元素与列表的尾部分开存储：即使后面的 `[a]` 里面是空的，前面的元素 `a` 必须始终存在。这使得 `head` 的实现变得非常简单。

> 在标准库里 `Data.List.NonEmpty` 已经提供了一个相同类型签名的 `head` 函数，但为了便于说明，我们将自己重新实现它。

```haskell
head :: NonEmpty a -> a
head (x:|_) = x
```

与之前不同的是没有报错，GHC 对于我们的实现没有意见 —— 证明我们的实现是全函数，我们可以沿用新的实现更新之前的例子：

```haskell
getConfigurationDirectories :: IO (NonEmpty FilePath)
getConfigurationDirectories = do
  configDirsString <- getEnv "CONFIG_DIRS"
  let configDirsList = split ',' configDirsString
  case nonEmpty configDirsList of
    Just nonEmptyConfigDirsList -> pure nonEmptyConfigDirsList
    Nothing -> throwIO $ userError "CONFIG_DIRS cannot be empty"

main :: IO ()
main = do
  configDirs <- getConfigurationDirectories
  initializeCache (head configDirs)
```

注意 `main` 中的冗余检查已完全消失，取而代之的是，我们在 `getConfigurationDirectories` 中只进行一次检查。它使用 `Data.List.NonEmpty` 中的 `nonEmpty` 函数从 `[a]` 构造出一个 `NonEmpty a` ，其类型如下：

```haskell
nonEmpty :: [a] -> Maybe (NonEmpty a)
```

这里的 `Maybe` 仍然存在，但这次我们在程序的很早阶段就处理了 `Nothing`：即在我们原本进行输入验证的地方。一旦通过了这个检查，我们现在拥有一个 `NonEmpty FilePath` 类型的值，它在类型系统中保留了列表确实非空的信息。换句话说，你可以将 `NonEmpty a` 类型的值视为 `[a]` 类型的值，附加了一个这个列表绝不可能为空的「证明」。

通过加强参数类型而不是削弱返回类型，我们完全消除了上一节中的所有问题：

- 代码没有冗余检查，因此不会有任何额外的性能开销。
- 重要的，如果 `getConfigurationDirectories` 函数改变不再检查列表是否非空，其返回类型也必须改变。因此 `main` 函数将无法通过类型检查，这在我们实际运行程序之前就提醒了我们这个问题！


更妙的是，通过将 `head` 函数与 `nonEmpty` 函数组合使用，我们可以轻松地复原先前提到的 `head` 函数的全函数版本。（它们具有相同的类型签名）

```haskell
head' :: [a] -> Maybe a
head' = fmap head . nonEmpty
```

注意这里没办法逆向，即不能从旧版的 `head` 函数衍生出新版的 `head` 函数。总的来说，第二种方法在各个方面都表现更加优越。

## 解析的力量

您可能想知道上述例子与本博客文章标题有什么关系。毕竟，我们只是探讨了两种验证列表非空的不同方法 —— 并没有看到任何解析。这种解释并没有错，但我想提出另一种观点：在我看来验证（Vaildate）与解析（Parse）之间的区别几乎完全在于信息如何被保存。考虑下面两个函数：

```haskell
validateNonEmpty :: [a] -> IO ()
validateNonEmpty (_:_) = pure ()
validateNonEmpty [] = throwIO $ userError "list cannot be empty"

parseNonEmpty :: [a] -> IO (NonEmpty a)
parseNonEmpty (x:xs) = pure (x:|xs)
parseNonEmpty [] = throwIO $ userError "list cannot be empty"
```

这两个函数几乎相同：它们检查提供的列表是否为空，如果为空，则中断程序并显示错误消息。然而，它们之间有一个重要的区别：`validateNonEmpty` 始终返回 `()`，一个不携带任何信息的类型。而 `parseNonEmpty` 返回 `NonEmpty a`，这是基于输入类型 `[a]` 的一种改进，它在类型系统中保留了列表非空的信息。这两个函数检查的是同一件事，但 `parseNonEmpty` 允许调用者查看解析结果，而 `validateNonEmpty` 则把它丢了。

思考一下：什么是解析器（Parser）？实际上，解析器仅是一种函数，它处理结构较松散的输入，并输出结构更丰富的结果。由于其本质，解析器是一个偏函数 —— 即其定义域中的某些值在值域中无对应值，因此所有解析器必须具备处理失败的能力。虽然解析器常常处理的是文本数据，但这并非必然要求，例如 `parseNonEmpty` 就是一个极好的例证：它将普通列表解析成非空列表，一旦发现列表为空，则通过抛出错误信息终止程序，从而表明失败。

在这种灵活的定义下，解析器是一个非常强大的工具：它们允许在程序与外部世界的边界上提前进行输入检查，一旦完成这些检查，就不再需要再次检查！Haskellers 很清楚这种力量，他们经常在不同地方使用解析器：

- [aeson](https://hackage.haskell.org/package/aeson) 库提供了一个 Parser 类型，可用于将 JSON 数据解析为领域类型（Domain Types）。
- 同样，[optparse-applicative](https://hackage.haskell.org/package/optparse-applicative) 也提供了一套用于解析命令行参数的语法分析组合子（Parser Combinator）。
- 数据库的库（如 [persistent](https://hackage.haskell.org/package/persistent) 和 [postgresql-simple](https://hackage.haskell.org/package/postgresql-simple)）配备了一种机制，用于解析存储在外部数据存储中的值。
- [servant](https://hackage.haskell.org/package/servant) 生态系统围绕从路径组件、查询参数、HTTP 头部等解析 Haskell 数据类型构建而成。

这些库的共同点是，它们位于您 Haskell 应用程序与外部世界之间的界限上。外部世界不是用积类型（Product Types）以及和类型（Sum Types）进行数据交换，而是用字节流进行数据传输，因此不得不进行一些解析。在对数据进行操作之前先进行解析，可以在很大程度上避免许多类型的错误，其中一些错误甚至可能涉及安全漏洞。

这种预先解析所有内容的方法有一个缺点，那就是有时候需要在实际使用这些值很久之前就对它们进行解析。在动态类型语言中，保持解析与处理逻辑同步可能有些困难，除非有广泛的测试覆盖，而这些测试往往维护起来非常繁琐。然而，在静态类型系统中，这个问题变得非常简单，正如上面的 `NonEmpty` 示例所示：如果解析和处理逻辑不同步，程序甚至无法通过编译。

## 验证的风险

希望您至此已对解析优于验证的观点有所认同，但您可能还有些疑虑。如果类型系统最终还是会强制您进行必要的检查，验证真的有那么糟糕吗？也许抛出错误会不太好，但多做一些重复检查感觉也没什么大不了？

遗憾的是，事情并不那么简单。临时性的验证会导致一种[编程语言理论安全领域](https://langsec.org/)所称的「Shotgun Parsing」现象。在 2016 年的论文《[The Seven Turrets of Babel: A Taxonomy of LangSec Errors and How to Expunge Them](http://langsec.org/papers/langsec-cwes-secdev2016.pdf)》中，作者提供了以下定义：

> Shotgun Parsing 是一种编程负面模式，其特点是将解析和输入验证代码与业务处理代码混杂在一起，对输入进行大量的检查，希望借此捕捉到所有「坏情况」，而这种做法缺乏系统的理论支持。

他们接着解释了这种技术固有的问题：

> Shotgun Parsing 必然剥夺程序拒绝无效输入而非处理它的能力。在输入流中较晚发现的错误将导致一部分无效输入已被处理，其后果是程序状态难以准确预测。

说人话即是：一个没有从最开始就解析其所有输入的程序，有可能在处理一部分有效输入时，发现另一部分输入无效，突然需要回滚它已经执行的任何修改以维持一致性。有时这是可能的，比如在 RDBMS 中回滚事务 —— 但通常来说不是很行。

Shotgun Parsing 与验证之间的关系可能并不立即显而易见 —— 毕竟，如果您一开始就完成了所有验证，您就减轻了 Shotgun Parsing 的风险。问题在于，基于验证的方法使得极其困难或不可能确定是否事前真的进行了全部验证，或者某些所谓的「不可能」情况可能实际发生。整个程序必须假设在任何地方抛出异常不仅是可能的，而且是经常必要的。

解析通过将程序划分为两个阶段 —— 解析和执行来避免前述问题。在此模式中，无效输入引起的失败只会在第一阶段出现。与此相比，执行阶段出现问题的可能性较小，相对来说可以更细致地处理。

# 在实践中应用

迄今为止，这篇博文有点像是一场推销演讲。「亲爱的读者，你应该进行解析！」如果我做得不错的话，至少有部分读者会被说服。然而，即使你理解了「是什么」和「为什么」，你可能对「怎么做」还不是特别有信心。

我的建议是：关注数据类型。

假设你正在编写一个函数，这个函数接受一个代表键值对的元组列表，突然间你意识到如果列表中有重复的键应该怎么办。一个解决方案是编写一个函数，断言（Assert）列表中没有任何重复项：

```haskell
checkNoDuplicateKeys :: (MonadError AppError m, Eq k) => [(k, v)] -> m ()
```

然而，这种检查是脆弱的：它非常容易被忘记。由于它的返回值未被使用，这个检查可以随时被省略，而依赖它的代码仍然会通过类型检查。一个更好的解决方案是选择一种在构造时就不允许重复键的数据结构，如 `Map`。调整你的函数类型签名，使其接受一个 `Map` 而不是一个元组列表，并照常实现它。

一旦你做了这些，你的新函数的调用位置很可能会因为类型检查失败，因为它仍然被传递了一个元组列表。如果调用者是通过其中一个参数获得这个值的，或者是从其他某个函数的结果中接收到这个值，你可以继续将类型从列表更新为 `Map`，一直向上追溯调用链。最终，你要么会到达值被创建的位置，要么会找到一个实际上应该允许重复的地方。在那时，你可以插入一个调用到修改版的 `checkNoDuplicateKeys`：

```haskell
checkNoDuplicateKeys :: (MonadError AppError m, Eq k) => [(k, v)] -> m (Map k v)
```

现在这个检查不能被省略，因为它的结果对程序的继续进行实际上是必需的。

这个假设的情景强调了两个简单的想法：

- 使用一种数据结构来确保无法表示非法状态。尽可能使用最精确的数据结构来建模你的数据。如果你当前使用的实现难以排除某种可能性，考虑使用可以更容易表达你关心的属性的其他实现。不要害怕重构。
- 在设计和编程过程中，应该尽可能地将数据的校验和验证的责任放在系统或代码结构向调用链的上层推，但也不能过分。尽快让你的数据转变为你需要的最精确的表现形式。理想情况下，这个过程应该在你的系统的边界处进行，在任何对数据的操作之前就完成。

> 有时，在解析用户输入之前进行某种授权是必要的，以避免拒绝服务攻击，但这是可以的：授权应该只涉及较小的范围，并且不应对系统的状态造成任何重大修改。

如果某个特定的逻辑分支最终需要更精确的数据表现形式，一旦选择了该分支，就尽早将数据解析为更精确的表现形式。明智地使用和类型，让你的数据类型能够反映并适应控制流。

换言之，应当根据你期望的数据表现形式编写函数，而不仅仅是依赖于已有的数据形式。因此，设计过程转化为一种缩小差异的努力，通常是从两端同时着手，直至在某个中点汇合。在设计过程中，不要害怕迭代调整部分设计，因为在重构过程中，你可能会学到新的东西。

下面是一些额外的建议，排列顺序不分前后：

- 让你的数据类型（Datatypes）控制你的代码，不要让你的代码控制你的数据类型。避免因为当前正在编写的函数需要而随意在 Record 中添加 Bool 的诱惑。不要害怕重构代码以使用正确的数据表示 —— 类型系统将确保你涵盖了所有需要更改的地方，并且这样做很可能会在之后为你避免头痛的问题。

- 对返回 `m ()` 的函数持怀疑态度。有时这些确实是必需的，因为它们可能执行一个没有有意义结果的命令式效果（Imperative Effect），但如果该效果的主要目的是抛出异常，那么可能有更好的方法。

不要害怕进行多次数据解析。避免 Shotgun Parsing 只是意味着在数据完全解析之前不应该处理输入数据，并不意味着你不能使用部分输入数据来决定如何解析其他输入数据。许多有用的解析器都是上下文敏感的。

- 避免数据的非规范化表述（Denormalized Representations），尤其是在数据可变的情况下。在多处重复相同的数据会轻易地造成一种非法状态：数据不同步。应当努力追求单一的真实数据源。

  - 保持数据的非规范化表示在抽象界限之内。如果非规范化是绝对必要的，那么应使用封装技术，确保一个小而可信的模块独立负责维护这些表示的同步性。

- 始终发挥你的判断力。通常无需仅为了解决一个「不可能」出现的错误而重构整个应用程序 —— 只需要像对待有害物质一样小心处理这些情况，采取适当的防护措施。如果其他所有方法都不奏效，至少应添加注释来记录这些不变量，方便之后可能需要修改代码的人员查阅。

# 回顾、反思和延伸阅读

这就是全部了。希望这篇博客文章能证明，要充分利用 Haskell 类型系统并不需要拥有 PhD 学位，甚至也不需要使用 GHC 最新最好最炫酷的语言扩展 —— 尽管它们有时确实有帮助。有时候，充分利用 Haskell 的最大障碍仅仅是了解有哪些可用的选项，不幸的是，Haskell 社区过小的一个缺点是缺乏记录，导致它们只成为了属于小团体的设计模式和技术资源。

这篇博客中的所有想法都不是新的。事实上，核心理念 —— 「编写全函数」在概念上相当简单。尽管如此，我发现要传达我编写 Haskell 代码的方式的具体可行的细节非常有挑战性。很容易花很多时间讨论抽象概念 —— 其中许多非常有价值！而没有传达关于其过程的任何有用信息。我希望这是朝着这个方向迈出的一小步。

遗憾的是，关于这个主题，我不知道太多延伸阅读资源，但我确实知道一个：我推荐 Matt Parson 的《[Type Safety Back and Forth](https://www.parsonsmatt.org/2017/10/11/type_safety_back_and_forth.html)》。如果你想从另一个通俗易懂的角度了解这些观点，我强烈推荐阅读它。如果你想深入研究，我还可以推荐 Matt Noonan 于 2018 年发表的论文《[Ghosts of Departed Proofs](https://kataskeue.com/gdp.pdf)》，其中概述了一些在类型系统中捕获比我这里描述的更复杂不变式的技术。

作为结束语，我想说，在这篇博客中描述的那种重构并不总是容易的。我给出的例子很简单，但现实例子通常远非如此简单。即使对于那些在类型驱动设计中有经验的人来说，要在类型系统中捕获某些不变量也可能是真正困难的，所以如果你不能像你希望的那样解决问题，不要认为这是你个人的失败！将这篇博客文章中的原则视为应该追求的理想，而不是必须满足的严格要求。重要的是尝试。