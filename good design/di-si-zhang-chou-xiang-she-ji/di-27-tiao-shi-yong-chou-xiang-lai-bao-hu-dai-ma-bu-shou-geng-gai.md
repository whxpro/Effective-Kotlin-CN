---
description: Not Kotlin-specific、Basics
---

# 第27条：使用抽象来保护代码不受更改

> 在水上行走和根据规范开发软件是很容易的，如果两者都是冻结的状态的话。– Edward V Berard ; Essays on object-oriented software engineering, p. 46

当我们将实际的代码隐藏在函数或类的抽象后时，我们不仅保护了用户不受这些细节影响，而且还给了自己修改这些代码的自由。例如，当你将排序算法提取到一个函数中时，你以后可以在不改变其使用方式的情况下优化其性能。

回想一下前面提到的汽车的比喻，汽车制造商和机械师傅可以改变汽车引擎盖下的一切，只要操作保持不变，用户就不会注意到。这给汽车制造商提供了更多便利，去让车子变得更环保或增加更多的传感器。

在本条例中，我们将看到不同种类的抽象如何在保护我们代码不受各种变化影响的同时，给予我们编程自由。我们将从三个实际示例开始，最后讨论如何从一些抽象概念中找到平衡。让我们从最简单的抽象开始：常量值

### 常量

常量值很少是不言自明的，当它们在我们的代码中重复时，问题就更大了。将值赋到常量属性中，不仅给值赋予了一个有意义的名称，而且还帮助我们更好地管理它们（以后需要更改时）。让我们来看一个密码验证的简单示例：

```kotlin
fun isPasswordValid(text: String): Boolean {
    if(text.length < 7) return false
    //...
}
```

数字 7 可以根据上下文来理解，但如果将它提取为一个常量，就会更容易理解：

```kotlin
const val MIN_PASSWORD_LENGTH = 7

fun isPasswordValid(text: String): Boolean {
    if(text.length < MIN_PASSWORD_LENGTH) return false
    //...
}
```

这样，要修改最小密码的大小就更加容易了。我们不需要理解验证逻辑，就能更改这个常量。 这就是为什么提取多次使用的值特别重要。例如，可以同时连接到数据库的最大线程数：

```
val MAX_THREADS = 10
```

一旦提取了它，你就可以在需要时轻松的更改它。想象一下，如果这个数字分布在整个项目中，要改变它会有多么困难。

如你所见。提取常量可以:

* 为其命名
* 帮助我们在以后能够更改它

### 函数

假设你正在开发一个应用程序，并且主要到经常需要向用户显示一个 Toast 信息，下面是你实现它的代码:

```kotlin
Toast.makeText(this, message, Toast.LENGTH_LONG).show()
```

![](<../../.gitbook/assets/image (1).png>)

我们可以将这个通用算法提取成一个简单的扩展函数来展示 Toast：

```kotlin
fun Context.toast(
    message: String,
    duration: Int = Toast.LENGTH_LONG
) {
    Toast.makeText(this, message, duration).show()
}

// 引用
context.toast(message)

// 在 Activity 或者 Context 的子类中引用
toast(message)
```

这个改变帮助我们提取了一个通用算法，这样我们就不需要每次都记住如何使用 toast 进行展示了。一般来说，如果 `Toast` 展示的方式有所改变（通常来说不太可能），这样的写法也会有所帮助，虽然有些变化我们还没有准备好。

如果我们必须改变向用户展示信息的方式，从 toast 到 snackbars （一种不同的消息展示方式）， 该怎么做呢？ 一个简单的方案是：提取这个功能后，我们只需改变该函数的内部实现并重命名它：

```kotlin
fun Context.snackbar(
    message: String,
    length: Int = Toast.LENGTH_LONG
) {
    //...
}
```

![](<../../.gitbook/assets/image (6) (1) (1).png>)

这个解决方案一点都不好。 首先，重命名这个功能可能会很危险，即使它只是在自己的模块中被使用，更别说被其他模块引用这个函数的情况了。 另一个问题是：参数不能如此轻易的改动，这里仍然需要 toast API 来声明消息持续的时间，这问题很大，当我们展示一个 snakebars 时，我们不应该依赖 `Toast` 的一个字段，另一方面，即使替换成 `Snackbar` 的枚举值，也会有问题：

```kotlin
fun Context.snackbar(
    message: String,
    duration: Int = Snackbar.LENGTH_LONG
) {
    //...
}
```

当我们知道消息的展示方式可能会改变时，我们就知道真正重要的不是该消息的显示方式，而是我们想将消息显示给用户的事实。我们需要的是使用一个更抽象的方法来显示消息，考虑到这一点，程序员可以将 toast 的展示逻辑隐藏在一个高级函数 `showMessage` 后面，它将独立于 toast 的概念：

```kotlin
fun Context.showMessage(
    message: String,
    duration: MessageLength = MessageLength.LONG
) {
    val toastDuration = when(duration) {
        SHORT -> Toast.LENGTH_SHORT
        LONG -> Toast.LENGTH_LONG
    }
        
    Toast.makeText(this, message, toastDuration).show()
}

enum class MessageLength { SHORT, LONG }
```

这里最大的变化是名称。有些开发人员可能忽略了这个更改的重要性，认为名称只是一个标签，不怎么重要。 从编译器的角度来看，这个观点是正确的。但从程序员的角度来看则不然，一个函数代表一个抽象，这个函数的签名告诉我们这个抽象是什么，一个有意义的名字非常重要。

函数是一个非常简单的抽象，但也非常有限。一个函数是没有状态的，函数签名的更改通常会影响到所有引用到的地方，抽象实现更强的方法是使用类。

### 类

下面是将消息显示到抽象到类中的方法：

```
class MessageDisplay(val context: Context) {
    
    fun show(
        message: String,
        duration: MessageLength = MessageLength.LONG
    ) {
        val toastDuration = when(duration) {
            SHORT -> Toast.LENGTH_SHORT
            LONG -> Toast.LENGTH_LONG
        }
        Toast.makeText(context, message, toastDuration).show()
    }
}

enum class MessageLength { SHORT, LONG }

// 使用
val messageDisplay = MessageDisplay(context)
messageDisplay.show("Message")
```

抽象到类中比到函数中更强大的关键原因是它们可以保存状态，并提供多个公有函数（类成员函数成为方法），在本例中，我们的类有一个上下文 context，它通过构造函数注入，使用依赖注入框架，我们可以委托类的创建：

```kotlin
@Inject lateinit var messageDisplay: MessageDisplay
```

此外我们可以 mock 这个类，以测试这个类的功能。这是可以做到的，因为我们出于测试的目的来模拟这个类：

```kotlin
val messageDisplay: MessageDisplay = mockk()
```

此外，还可以添加更多方法来设置消息展示：

```kotlin
messageDisplay.setChristmasMode(true)
```

正如你所看到的，这个类给了我们很多自由。 但仍然有局限性。 例如，当一个类是 final 类时，我们可以清楚的知道这个类型实现的是什么逻辑。 对于一个开放类，它有更多的自由，因为可以提供一个子类代替。 为了获得更多的自由，我们可以使它更加抽象，并将这个类隐藏在接口后面。

### 接口

阅读 Kotlin 标准库，你可以能会注意到几乎所有的内容都表示为接口。让我们来看几个例子：

* `listOf` 函数返回 List，这是一个接口，这类似于其它工厂方法（我们将在`第33条：考虑使用工厂方法而不是构造函数` 中解释这些）：
* 集合处理函数都是 `Iterable` 或 `Collection` 的扩展函数，并返回 `List`、 `Map` 等，这些都是接口
* 属性委托都隐藏在只读属性或读写属性后面，它们也是接口， 实际的类通常是私有的， lazy 函数还声明了 `lazy` 接口作为它的返回类型。

库创建者通常会限制内部类的可见性，并从接口后面公开它们，这样做是有很好的理由。通过这种方式，库创建者相信不会用户不会直接使用这个类，所以只要接口保持不变，它们就可以毫无顾虑地更改实现。这正是这个条目背后的思想 —— 通过将对象隐藏在接口后面，我们抽象出了任何实际的实现，并迫使用户依赖于这个实现，这样我们就减少了耦合。

在 Kotlin 中，返回接口而不返回类型还有另一个原因 —— Kotlin 是一种多平台语言，相同的 `listOf` 可以为 Kotlin/JVM、Kotlin/JS 和 Kotlin/Native 返回不同的列表实现。这是一种优化 —— Koltin 通常使用特定于平台的原生集合，这是很好的，因为它们都遵循 `List` 接口的合约。

让我们看看如何将这个想法引用到消息的展示中去。当我们将类隐藏在接口后面时，它看起来就是这样的：

```kotlin
interface MessageDisplay {
    fun show(
        message: String,
        duration: MessageLength = LONG
    )
}

class ToastDisplay(val context: Context): MessageDisplay {
    
    override fun show(
        message: String,
        duration: MessageLength
    ) {
        val toastDuration = when(duration) {
            SHORT -> Toast.LENGTH_SHORT
            LONG -> Toast.LENGTH_LONG
        }
        Toast.makeText(context, message, toastDuration).show()
    }
}

enum class MessageLength { SHORT, LONG }
```

这样我们得到了更多的自由。例如，可以注入在平板电脑上显示 Toast 和在手机上显示 Snakbar。还可以在 Android、iOS 和 Web 之间共享的公共模块中使用 `MessageDisplay`。然后我们可以为每个平台提供不同的实现，例如，在 iOS 和 Web 中，它可以显示 `alert`。

另一个好处是用于测试的接口比类更加容易模拟，而且不需要模拟任何库：

```kotlin
val messageDisplay: MessageDisplay = TestMessageDisplay()
```

最后，声明和引用解耦了，因此我们在更改实际实现类（如 `ToastDisplay`）时更加自由，另一方面，如果我们希望更改它的使用方式，则需要更改 `MessageDisplay` 接口和实现它的所有类。

### 下一个 ID

我们再来讨论一个例子，假设我们在项目中需要一个唯一的 ID，一个非常简单的方式是用一个顶层属性来保存下一个ID，当我们需要一个新的 ID 时就增加它：

```kotlin
var nextId: Int = 0

// Usage

val newId = nextId++
```

看到这样的使用在我们的代码中扩散，需要小心。如果我们想要改变 id 的创建方式会怎么样呢？ 说实话，这种的方式一点儿都不好，因为：

* 每次冷启动，都要从 0 开始
* 它不是线程安全的

假设现在要基于该代码来出一个方案，我们可以将 ID 创建提取到一个函数中来保护其不受更改：

```kotlin
private var nextId: Int = 0
fun getNextId(): Int = nextId++

// Usage
val newId = getNextId()
```

但是请注意，该解决方案仅保护我们不受 ID 创建更改的影响，但仍然容易发生许多变化。最大的变化就是 ID 类型的变化，如果有一天我们需要将 ID 保存为 String 呢？ 还要注意，如果有人看到 ID 表示为 Int，可能会使用一些类型相关的操作，例如，使用比较来检查哪个 ID 比较旧。 这种假设可能会导致严重的问题，为了防止这种情况发生，也为了我们将来可以轻松地更改 ID 类型，我们可以将 ID 提取为一个类：

```kotlin
data class Id(private val id: Int)

private var nextId: Int = 0
fun getNextId(): Id = Id(nextId++)
```

很明显，更多的抽象给了我们更多的自由，但也使定义和用法更难声明和理解。

### 抽象带来自由

我们已经介绍了几种引入抽象的常见方法：

* 提取常量
* 将行为包装到函数中
* 将函数包装到一个类中
* 将类隐藏在接口后面
* 将通用对象包装成专用对象

我们已经看到了每一种都给了我们不同的自由。请注意，还有更多可用的工具，举几个例子：

* 使用泛型类型参数
* 提取内部类
* 限制创建，例如强制通过工厂方法创建对象

另一方面，抽象也有其阴暗面，它们给了我们自由和分割的代码，但往往会使代码更难理解和修改，让我们来讨探讨如何解决这个问题。

### 抽象的问题

添加新的抽象要求代码的读者学习或熟悉特定的概念。每当我们定义一个抽象，我们的项目就要多理解一件事。当然，当我们限制抽象可见性（_第30条：最小化元素可见性_）或定义仅用于具体任务的抽象时，这就不是什么问题了。这就是模块化在大型项目中如此重要的原因。我们需要理解定义抽象是具有代价的，我们不应该默认抽象所有东西。

我们可以无限地提取抽象概念，但很快就会弊大于利， `FizzBuzz Enterprise Edition` 项目就专门恶搞了这一点，作者们展示了即使对于 `FizzBuzz` 这样简单的问题，人们也可以提炼出大量荒谬的抽象概念，使得问题难以理解。在写这本书时，这个项目有 61 个类 和 26 个接口，所有这些都是为了解决一个通常只需要不到 10 行代码就能解决的问题。当然，在任何级别上应用更改都很容易，但另方面，理解这些代码是做什么的，以及它是如何做到这一点的确实及其困难的。

<img src="../../.gitbook/assets/image (1) (1).png" alt="" data-size="original">

抽象可以隐藏很多东西，一方面，当需要考虑的东西比较少时，开发就更容易，另一方面，当我们使用太多抽象时，就很难理解我们行为会产生什么后果。使用 `showMessage` 函数时，我们可能会认为它仍然展示 Toast，而它展示 Snackbar 时，我们可能会感到惊讶。一旦看到出现了不是用 `toast` 来展示消息的情景，开发者可能会去使用 `Toast.makeText`，并且心存疑惑，因为它是使用 `showMessage` 展示的。 有太多的抽象让我们更难理解代码，当我们不确定自己行为的后果时，它也会让我们焦虑。

要理解抽象，示例（examples）非常有用。说明如何使用元素的单元测试或文档中的示例，使抽象对我们来说更加亲近。出于同样的原因，我在这本书中为我提出的大多数想法提供了具体的例子，抽象的描述很难理解，人们也很容易误解它们。

### 平衡点在哪里？

经验法则是：每一级的复杂性都给了我们更多的自由并组织我们的代码，但也使我们更难理解项目中真正发生了什么。两个极端都不好，最好的解决方案总是介于两者之间，它具体取决于许多因素，如：

* 团队规模
* 团队经验
* 项目规模
* 特性集
* 领域知识

我们致力于在每个项目中寻找平衡点，找到平衡点几乎是一门艺术，因为它需要要么数千要么数万小时的直觉来构建和编写项目，以下是我可以给出的一些建议：

* 在有更多开发人员的大型项目中，后期改变对象的创建和使用是非常困难的，所以我们更喜欢更抽象的解决方案，此外，模块或部分之间的分离也特别有用。
* 当我们使用依赖注入框架时，我们不太关心创建有多困难，因为我们可能只需要使用一次这个创建
* 测试或制作不同的应用程序变体可能需要我们使用一些抽象
* 当你的项目很小且处于实验阶段时，你可以自由地直接进行修改，而无需处理抽象问题。不过当它逐渐变的更正式了，那么需要尽快改变这种做法

我们需要经常思考另一件事是：什么是可能会改变的，并且其改变概率有多少。例如， 用于展示 Toast 的 API 发生变化的可能性很小，但是我们很有可能会改变消息展示的方式。“我们有没有可能要 mock 这个api？”，“也许某天我们需要一个更通用的api？”，“或者一种依赖于平台的api？”，这些事发生的概率都不是0，它们有多大？ 唯有持续观察事物多年的变化，才能给我们带来更好的判断依据。

### 总结

抽象不仅仅是为了消除冗余和组织代码。当我们需要更改代码时，它们也会帮助我们，尽管使用抽象比较困难。它是我们需要去学习和理解的东西，当使用抽象结构时，也更难理解其后果，我们需要理解使用抽象的正确性和风险，并且我们需要在每个项目中寻求平衡，抽象太多或太少都是不理想的情况。
