---
description: Basics
---

# 第20条：不要重复实现常用算法

我经常看到开发人员一次又一次地实现相同的算法，这里指的是那些不特定于项目模式的算法，它们不包含任何业务逻辑，所以可以提取到单独的模块甚至库中。这些可能是数学操作、集合处理或者任何其它常见的行为。有时这些算法可能又长又复杂，比如优化的排序算法，还有很多简单的例子，例如限制给定数字在一个范围内：

```kotlin
val percent = when {
    numberFromUser > 100 -> 100
    numberFromUser < 0 -> 0
    else -> numberFromUser
}
```

注意，我们不需要再实现它了，因为它已经作为 `coerceIn` 扩展函数在 stdlib 库中实现了：

```kotlin
val percent = numberFromUser.coerceIn(0, 100)
```

提取即使是简短但重复的算法，优点是：

* **编程速度更快**，因为单个函数调用比算法（一系列步骤）更短
* **它们是被命名的，所以我们需要通过名称来了解概念，而不是通过阅读它的实现来了解它**。这对于那些熟悉概念的程序员来说比较容易，但对那些不熟悉概念的人来说这可能会比较困难，但学习这些可能会重复的算法名称是值得的，一旦他们学会了，就会从中收益
* **我们消除了杂质，所以更容易注意到一些非常规的东西**。在一个冗长的算法中，很容易漏掉隐藏的非常规逻辑。 想想 `sortedBy` 和 `sortedByDescent` 之间的区别，当我们调用这些函数时，排序方向是明确的，即使它们的主体几乎相同。如果我们每次都要实现这种逻辑，那么很容易混淆所实现的排序是升序还是降序， 即使这个算法前面带有注释也不是个很管用的办法，实践证明，开发人员确实会在不更新注释的情况下更改代码， 随着时间的推移，我们渐渐对这些注释失去了信任
* **一旦它进行优化，我们任何使用到这个函数的地方都能从中获益**

### 学习标准库

常见的算法几乎总是由别人定义的，大多数类库只是集合这些普通的算法，其中最特殊的就是 stdlib（标准库），它是一个庞大的实用工具集合，主要定义了扩展函数，学习 stdlib 函数成本可能很高，但这是值得的，如果没有它，开发人员就会一次又一次地重复工作。下面来看一个例子，看看来自开源项目的代码片段：

```kotlin
override fun saveCallResult(item: SourceResponse) {
    var sourceList = ArrayList<SourceEntity>()
    item.sources.forEach {
        var sourceEntity = SourceEntity()
        sourceEntity.id = it.id
        sourceEntity.category = it.category
        sourceEntity.country = it.country
        sourceEntity.description = it.description
        sourceList.add(sourceEntity)
    }
    db.insertSources(sourceList)
}
```

在这里使用 `forEach` 是没有用的，我看不出用它代替 for 循环有什么好处。我在这段代码中看到的是从一种类型到另一种类型的映射， 在这种情况下，我们可以使用 `map` 函数。 另外一件需要注意的事情是： `SourceEntity` 的设置方式并不好，这是一种已经过时的 JavaBean 处理模式，在 Koltin 中，我们可以使用工厂方法或主构造函数（_第五章：对象的创建_）。 如果处于某种原因需要保持这种方式，我们至少应该使用 `apply` 来隐式的设置单个对象的所有属性，这是我们优化之后的函数：

```kotlin
override fun saveCallResult(item: SourceResponse) {
    val sourceEntries = item.sources.map(::sourceToEntry)
    db.insertSources(sourceEntries)
}

private fun sourceToEntry(source: Source) = SourceEntity().apply {
    id = source.id
    category = source.category
    country = source.country
    description = source.description
}
```

### 实现你个人的工具类

在每个项目中，我们都需要一些标准库中没有的算法，例如，如果我们需要计算集合中所有整数的乘积怎么办？这是一个总所周知的数学上的概念，所以最好的将其定义为通用有效的函数：

```kotlin
fun Iterable<Int>.product() =
       fold(1) { acc, i -> acc * i }
```

你不需要等别人来多次使用它， 它是一个众人皆知的数学概念，开发人员能够顾名思义。也许其他开发人员将来有这个需求时，他们会很高兴看到这个概念已经被定义了。希望开发人员能够找到这个函数，使用重复的函数来实现相同的结果本身是一种不好的做法，我们应该意识到，不要定义我们不需要的函数，因此，在实现自己的函数之前，应该首先搜索下现有的函数。

请注意： 与 Kotlin 标准库中大多数的函数一样， product 是一个扩展函数，有很多方法可以提取通用函数，例如 顶级函数、属性委托，甚至使用一个类。 尽管扩展函数是一个好的选择：

* 函数不保持状态，所以它们很适合代表行为，特别是如果它们没有副作用的时候
* 相较于顶层函数，扩展函数更好，因为它们只适用于具有具体类型的对象
* 修改扩展函数接收者比修改参数更加直观
* 在与对象上的方法比较，扩展更容易在提示中找到，因为它们是出现在 IDE 的建议上，例如 "Text.isEmpty" 比 "TextUtils.isEmpty("Text")" 更加容易找。 这是因为当你在 "Text" 后面加一个点时，你会看到所有的可以应用与该对象的扩展函数的建议。 而若要使用 "TextUtils"，你要先找到它在哪里，你可以要从不同的库中搜寻可选的 util 对象
* 当我们调用一个方法时，很容易将顶层函数与来自其类或其超类的其它方法混淆，它们预期行为是非常不同的，顶级扩展函数则没有这个问题，因为它们需要在对象上调用

### 总结

不要重复实现常用的算法。 首先，stdlib 库中可能已经存在一个可以替换的函数了。 这就是为什么学习标准库是一件值得的事。 如果 stdlib 中没有已知的算法，或者你经常需要使用某个特定的算法，你可以将其定义在你的项目中。一个好的选择是将其定义为扩展函数。
