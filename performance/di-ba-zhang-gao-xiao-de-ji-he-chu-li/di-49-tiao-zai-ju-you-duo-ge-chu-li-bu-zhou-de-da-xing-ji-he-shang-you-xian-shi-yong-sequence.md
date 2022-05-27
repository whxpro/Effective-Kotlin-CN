# 第49条：在具有多个处理步骤的大型集合上，优先使用 Sequence

人们经常忽略 `Iterable`（集合） 和 `Sequence`（序列） 之间的区别。这是可以理解的，因为它们的定义几乎相同。

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

可以说它们之间的唯一的正式区别是名称，尽管 `Iterable` 和 `Sequence` 与完全不同的用法相关联（具有不同的合约），所以几乎所有它们的处理函数都以不同的方式工作。 `Sequence` 是惰性的，因此用于处理序列的中间函数不做任何计算，相反，它们返回一个新的、用new操作符来修饰的 `Sequence`。 所有这些计算都是在终止操作（如 `toList` 或 `count`）中计算的。而另外一方面， `Iterable` 每一步处理都会返回一个类似 `List` 的集合。

```kotlin
public inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}

public fun <T> Sequence<T>.filter(
    predicate: (T) -> Boolean
): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}
```

因此，一旦集合使用了处理操作，就会立刻执行这些操作。 而 `Sequence` 处理函数直到终止操作（返回序列以外的其他内容的操作）才会被调用。例如，对于 `Sequence`，`filter` 是一个中间操作，因此它不做任何操作，而使用新的处理步骤来包装序列。计算是在像 `toList` 这样的终止操作中完成的。

![](<../../.gitbook/assets/image (3) (1).png>)

```kotlin
val seq = sequenceOf(1,2,3)
val filtered = seq.filter { print("f$it "); it % 2 == 1 }
println(filtered) // FilteringSequence@...

val asList = filtered.toList()
// f1 f2 f3
println(asList) // [1, 3]

val list = listOf(1,2,3)
val listFiltered = list
    .filter { print("f$it "); it % 2 == 1 }
// f1 f2 f3
println(listFiltered) // [1, 3]
```

在 Kotlin 中，`Sequnece` 是惰性的，这一事实有几个重要的优点：

* 它们依然保持着工作的自然秩序
* 它们只做最少的操作
* 它们可以是无限的
* 它们不需要在每一步都创建集合

让我们逐一讨论这一优势：

### 顺序是重要的

由于 `iterable` 和 `sequence` 的实现方式不同，它们的操作顺序是不同的。在序列中，我们取第一个元素并应用所有操作，然后取下一个元素做同样的操作，以此类推。我们称它为**逐元素次序**或**惰性次序**。而在 `iterable` 处理中，我们将第一个操作运用于整个集合，然后继续运用下一个操作，以此类推，我们称其为**循序渐进**或**迫切次序**。

```kotlin
sequenceOf(1,2,3)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .forEach { print("E$it, ") }
// 打印: F1, M1, E2, F2, F3, M3, E6,

listOf(1,2,3)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .forEach { print("E$it, ") }
// 打印: F1, F2, F3, M1, M3, E2, E6,
```

![](<../../.gitbook/assets/image (6) (1) (1) (1).png>)

请注意，如果我们不使用任何集合处理函数来实现这些操作，而是使用经典的循环和条件控制，就会像序列处理那样按逐元素次序的顺序执行：

```kotlin
for (e in listOf(1,2,3)) {
    print("F$e, ")
    if(e % 2 == 1) {
        print("M$e, ")
        val mapped = e * 2
        print("E$mapped, ")
    }
}
// Prints: F1, M1, E2, F2, F3, M3, E6,
```

因此，在序列处理中，逐元素次序的顺序更加自然。它也为底层编译器优化打开了大门 —— 序列处理可以优化为基本的循环和条件控制。也许在未来，它会是这样的。

### 序列执行最少的操作

通常我们不需要在每个步骤中处理整个集合来产生结果。假设我们有一个包含数百万个元素的集合，在处理后，我们只需要取提前10个元素，为什么要处理所有其他元素？ `Iterable` 操作没有中间操作的概念，因此整个集合被处理，就好像每个操作都返回它一样。`Sequence` 不需要这样做，因此它们将执行获得结果所需的最少操作：

![](<../../.gitbook/assets/image (7) (1) (1) (1).png>)

看一下这个例子，我们有几个处理步骤，最终用 `find` 来结束操作：

```kotlin
(1..10).asSequence()
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .find { it > 5 }
// 打印: F1, M1, F2, F3, M3,

(1..10)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .find { it > 5 }
// 打印: F1, F2, F3, F4, F5, F6, F7, F8, F9, F10,
// M1, M3, M5, M7, M9,
```

由于这个原因，当我们有一些中间处理步骤，并且我们的终止操作不一定需要遍历所有元素时，使用序列很可能会提高处理的性能。所有这些看起来几乎与标准的集合处理相同。此类的操作的示例有 `first`、`find`、`take`、`any`、`all`、`none` 或 `indexOf`。

### 序列可以是无限的

得益于序列的按需处理，这样我们就可以有无限的序列了。创建无限序列的一个典型方式是使用序列生成器，如 `generateSeuence` 或 `sequence`，第一个函数需要第一个元素和指定如何计算下一个元素的函数：

```kotlin
generateSequence(1) { it + 1 }
    .map { it * 2 }
    .take(10)
    .forEach { print("$it, ") }
// Prints: 2, 4, 6, 8, 10, 12, 14, 16, 18, 20,
```

第二种提到的序列生成器 —— `sequence` —— 使用了挂起函数（协程）根据需要产生下一个数字。每当我们请求下一个数字时，序列生成器就会运行，直到 `yield` 生成一个值，运行就会停止，直到我们要求下一个数字时。下面是一个斐波那契数列的无限列表：

```kotlin
val fibonacci = sequence {
    yield(1)
    var current = 1
    var prev = 1
    while (true) {
        yield(current)
        val temp = prev
        prev = current
        current += temp
    }
}

print(fibonacci.take(10).toList())
// [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

请注意，无限序列某种意义上是需要一个限制的值，我们不能让迭代无限次进行下去：

```kotlin
print(fibonacci.toList()) // 一直运行，不会停下来
```

因此，我们要么需要使用像 `take` 这样的操作来限制它们，要么使用终止操作组织来规避无限元素，比如 `first`、 `find`、`any`、`all`、`none` 或 `indexOf`。基本上，对于这些相同的操作，序列的效率更高，因为它们不需要处理所有元素。不过请注意，对于这些操作中的大部分来说，很容易陷入无限循环， `any` 只能返回 true 或永远运行。类似地，对于无限集合， `all` 和 `none` 只能返回 false。因此，我们通常要么通过 `take` 限制元素的数量，要么使用 `first` 来请求序列中第一个值。

### 序列不会在每个处理步骤中创建集合

标准集合处理函数在每一步都返回一个新的集合。通常它是一个 `List`。这可能是一个优点 —— 在每一个操作后，我们都有一些东西可以使用或存储。但这是有代价的。这些集合需要在每个步骤中创建并填充数据。

```kotlin
numbers
    .filter { it % 10 == 0 } // 这里创建了一个集合
    .map { it * 2 } // 这里创建了一个集合
    .sum()
// 在底层总共有 2 个集合被创建

numbers
    .asSequence()
    .filter { it % 10 == 0 }
    .map { it * 2 }
    .sum()
 // 没有集合被创建
```

特别是当我们处理大型或大量的集合时，这将会成为一个问题。让我们从一个极端但常见的情况开始：文件读取。文件的大小可以达到 GB 级，在每个处理步骤中，都给集合的所有数据分配内存，将对内存造成巨大浪费。这就是默认情况下我们使用序列来处理文件的原因。

举个例子，让我们来分析一下芝加哥的犯罪记录。这座城市和其他许多城市一样，在互联网上分享了自 2001 年以来发生在这里的所有犯罪数据。今天，这个数据集的大小已经超过了 1.53 GB。假设我们的任务是找到它们的描述中有多少犯罪是有关大麻的，下面就是一个使用集合来处理的简单方案（`readLines` 返回一个 `List`）

```kotlin
// 不好的解决方案，不要使用集合来处理可能很大的文件
File("ChicagoCrimes.csv").readLines()
    .drop(1) // 删除掉列的描述
    .mapNotNull { it.split(",").getOrNull(6) }
    // 过滤描述
    .filter { "CANNABIS" in it }
    .count()
    .let(::println)
```

我的电脑上运行的结果是产生 `OutOfMemoryError`:

> Exception in thread “main” java.lang.OutOfMemoryError: Java heap space

不足为奇。我们创建一个集合，然后有3个中间处理步骤，总共会产生4个集合。其中3个包含大部分数据，每个占用1.53GB， 所以它们总的加起来超过4.59GB。这是对内存的巨大浪费。正确的实现应该将它们包在序列中，并且使用 `useLines` 来实现，它总是在单行上操作：

```kotlin
File("ChicagoCrimes.csv").useLines { lines ->
// lines 的类型是： Sequence<String>
    lines
        .drop(1) // 删除掉列的描述
        .mapNotNull { it.split(",").getOrNull(6) }
        // 过滤描述
        .filter { "CANNABIS" in it }
        .count()
        .let { println(it) } // 318185
}
```

在我电脑上只花了8.3s。为了比较两种方法的效率，我做了另一个实验，通过删除不需要的列来减少数据集大小。这样我就得到了犯罪记录相同的 `CrimeData.csv` 文件，而大小只有728MB。然后我们又做了相同的处理。在第一个实验中，使用集合处理，处理大约需要13s，而第二个实验，使用序列，处理大约需要4.5s。如你所见，对较大的文件使用序列处理不仅仅是为了内存，也是为了性能。

尽管一个集合不需要很重，但实际上，在创建一个新集合的每一步中，它本身也是一种成本，当处理包含大量元素的集合时，这种成本就会明显体现出来。不同方式差别不大的主要原因是：许多步骤的结果所创建的集合可以初始化预期的大小，所以当添加元素时，我们只是将它们放在下一个位置上而已。但这种差异和序列相比仍然很大，这也是为什么我们应该对具有多个处理步骤的大型集合优先使用 `Sequence`。

我所说的“大型集合”指的是那些有着许多元素并且很占内存的集合。它可以是一个包含数万个元素的整数列表，也可以是只包含几个字符串，但每个字符串都是很长且需要很多兆字节的列表。这些情况并不常见，但时有发生。

我所说的“具有多个处理步骤”，指的是使用多个用于处理集合的函数。如果你比较下面这两个函数：

```kotlin
fun singleStepListProcessing(): List<Product> {
    return productsList.filter { it.bought }
}

fun singleStepSequenceProcessing(): List<Product> {
    return productsList.asSequence()
        .filter { it.bought }
        .toList()
}
```

你会注意到，它们在性能上几乎没有什么不同（实际上，在简单的处理上 `List` 会更快，因为 `filter` 是内联的）。尽管当你比较具有多个处理步骤的函数时，如下面的函数，它们先使用 `filter` 然后再 `map`，对于更大的集合来说，差异是明显的。为了展示区别，让我们来对比5000个产品数据下，2个和3个加工步骤的差异：

```kotlin
fun twoStepListProcessing(): List<Double> {
    return productsList
        .filter { it.bought }
        .map { it.price }
}

fun twoStepSequenceProcessing(): List<Double> {
    return productsList.asSequence()
        .filter { it.bought }
        .map { it.price }
        .toList()
}

fun threeStepListProcessing(): Double {
    return productsList
        .filter { it.bought }
        .map { it.price }
        .average()
}

fun threeStepSequenceProcessing(): Double {
    return productsList.asSequence()
        .filter { it.bought }
        .map { it.price }
        .average()
}
```

下面你能看到在 MackBook Pro（Retina, 15英寸，2013年底）上处理 `productsList` 的5000个产品的平均结果：

```
twoStepListProcessing                   81 095 ns
twoStepSequenceProcessing               55 685 ns
twoStepListProcessingAndAcumulate       83 307 ns
twoStepSequenceProcessingAndAcumulate    6 928 ns
```

很难准确预测这样能够提高多少性能，根据我的经验，在一个具有多个步骤的典型集合处理中，对于至少数千个元素，我们可以估计有 20\~40% 的性能提升。

### 什么时候 sequence 才不是更快的那个呢？

有一些操作是无法在使用 `sequence` 中获益的，比如必须对整个集合进行操作。 `sorted` 就是 Kotlin sdlib 的一个示例（目前也是唯一的示例）。`sorted` 使用了最优的实现：它将 `Sequence` 堆积到了 `List` 中，然后调用 Java stdlib 的 `storted` 函数。缺点是，相较于集合上的同样处理，这个堆积的过程需要一些额外的时间（尽管 `Iterable` 不是一个集合或数组，然后配置又是2.6GHZ Intel Core i7处理器，内存16gb 1600MHZ DDR3 ，但这些差异并不重要，因为它同样还是要堆积的过程）。

序列是否应该集成像 `sorted` 这样的方法是有争议的，因为如果序列的方法要求所有的元素参与计算，那么这个序列只能是部分懒惰的（需要在获得第一个元素时进行评估），并且它不能用于无限列表。添加这个函数是因为它是一个主流的功能，而且这种方式更容易使用。Kotlin 开发人员应该记住其缺陷，尤其是它不能用于无限序列。

```kotlin
generateSequence(0) { it + 1 }.take(10).sorted().toList()
// [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
generateSequence(0) { it + 1 }.sorted().take(10).toList()
// 会跑无限时间，不会返回任何东西
```

`sorted` 是一个罕见的处理步骤的例子，它在 `Collection` 上会比在 `Sequence` 上更快。尽管如此，当我们执行几个处理步骤和一个排序函数（或其他需要处理整个集合的函数）时，我们可以期望使用序列处理来提高性能：

```kotlin
// 基准测试结果: 150 482 ns
fun productsSortAndProcessingList(): Double {
    return productsList
    .sortedBy { it.price }
    .filter { it.bought }
    .map { it.price }
    .average()
}

// 基准测试结果: 96 811 ns
fun productsSortAndProcessingSequence(): Double {
    return productsList.asSequence()
    .sortedBy { it.price }
    .filter { it.bought }
    .map { it.price }
    .average()
}
```

### 那么 Java stream 呢？

Java 8 引入了流（streams）来支持集合处理，它们的行为和外观与 Kotlin 的序列一致：

```kotlin
productsList.asSequence()
    .filter { it.bought }
    .map { it.price }
    .average()

productsList.stream()
    .filter { it.bought }
    .mapToDouble { it.price }
    .average()
    .orElse(0.0)
```

Java 8 的流是惰性的，在终止操作中进行集合处理。 Java 流和 Kotlin 的序列主要有以下三个区别：

* Kotlin 的序列有更多处理功能（因为它们都被定义为扩展函数），它们通常更容易使用（这是由于 Kotlin 序列在被设计时 Java 的流已经可以被使用了 —— 例如我们可以让集合使用 `toList`，而不是让集合使用 `Collectors.toList()`）
* Java 流处理可以使用并行功能，以多线程模式启动。当我们有一台经常未使用（现在很常见）的多核机器时，这可以给我们带来巨大的性能提升，但是要谨慎使用这个功能，因为这个功能有已知的缺陷
* Kotlin 序列可以用于通用模块， Kotlin/JVM， Koltin/JS 和 Kotlin/Native 模块，Java流仅在 Kotlin/JVM中，且要求的 JVM 版本至少为8

通常的，当我们不使用并行模式时，很难给出一个简单的答案，到底是 Java 流还是 Kotlin 序列更加高效呢。我的建议是尽量少使用 Java 流，只在计算量大的处理中使用，这样可以从并行模式中获益。否则，可以使用 Kotlin stdlib 函数来获得可在不同平台上或公共模块上使用的同类且干净的代码。

### Kotlin 序列的调试

Kotlin Sequence 和 Java Stream 都可以帮助我们在每个步骤中调试元素。对于 Java 流，它需要一个名为“Java Stream Debugger” 的插件，Kotlin Sequence 则需要一个名为 “Kotlin Sequence Debugger” 的插件，尽管现在这个功能已经集成到 Kotlin 插件中了。 下面展示调试了每一步的序列调试：

![](<../../.gitbook/assets/image (8) (1) (1) (1).png>)

### 总结

集合处理和序列处理非常相似，它们都支持几乎相同的处理方法，然而，两者之间却有重要的区别。序列处理更加困难，因为我们通常将元素保存在集合中，因此需要将集合转化为序列，而且通常也需要转化回所需的集合。序列是惰性的，这给我们带来了一些重要的优点：

* 它们依然保持着工作的自然秩序
* 它们只做最少的操作
* 它们可以是无限的
* 它们不需要在每一步都创建集合

因此，它们更适合处理大型对象或具有多个处理步骤的大型集合。Kotlin Sequence Debugger 通过可视化界面的方式来帮助我们对 `Sequence` 处理进行调试。你应该只在有充分理由时才使用它们，这样你将获得显著的性能优化。
