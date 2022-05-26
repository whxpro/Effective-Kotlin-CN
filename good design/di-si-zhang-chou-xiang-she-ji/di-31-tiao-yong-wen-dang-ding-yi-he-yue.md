# 第31条：用文档定义合约

再次来看看_第27条：使用抽象来保护代码不受更改_中的展示消息的函数。

```kotlin
fun Context.showMessage(
    message: String,
    length: MessageLength = MessageLength.LONG
) {
    val toastLength = when(length) {
        SHORT -> Toast.LENGTH_SHORT
        LONG -> Toast.LENGTH_LONG
        }
    Toast.makeText(this, message, toastLength).show()
}

enum class MessageLength { SHORT, LONG }
```

我们提取它是为了让自己能够自由地更改消息的展示方式。然而，它并没有很好的文档记录，另一个开发人员可能会阅读它的代码，并认为它总是显示 toast。这与我们“不建议以具体消息类型来命名的方式实现接口”的原则是相违背的。为了更清楚的说明这一点，最好添加一个有意义的 KDoc 注释，来解释这个函数。

```kotlin
/**
* Universal way for the project to display a short
* message to a user.
* @param message The text that should be shown to
* the user
* @param length How long to display the message.
*/
fun Context.showMessage(
    message: String,
    length: MessageLength = MessageLength.LONG
) {
    val toastLength = when(length) {
        SHORT -> Toast.LENGTH_SHORT
        LONG -> Toast.LENGTH_LONG
        }
    Toast.makeText(this, message, toastLength).show()
}

enum class MessageLength { SHORT, LONG }
```

在许多情况下，有些细节根本不能通过名字清楚地推断出来，例如 `powerset`，即使它是一个定义良好的数学概念，也需要一个解释，因为它鲜为人知，解释也不够清楚：

```kotlin
/**
* Powerset returns a set of all subsets of the receiver
* including itself and the empty set
*/
fun <T> Collection<T>.powerset(): Set<Set<T>> =
    if (isEmpty()) setOf(emptySet())
    else take(size - 1)
        .powerset()
        .let { it + it.map { it + last() } }
```

注意，这个描述给了我们一些自由，它没有指定这些元素的顺序，作为用户，我们不应该依赖于这些元素的排序方式，隐藏在这个抽象背后的实现可以在不改变函数表面的情况下进行优化：

```kotlin
/**
* Powerset returns a set of all subsets of the receiver
* including itself and empty set
*/
fun <T> Collection<T>.powerset(): Set<Set<T>> =
        powerset(this, setOf(setOf()))

private tailrec fun <T> powerset(
    left: Collection<T>,
    acc: Set<Set<T>>
): Set<Set<T>> = when {
    left.isEmpty() -> acc
    else -> {
        val head = left.first()
        val tail = left.drop(1)
        powerset(tail, acc + acc.map { it + head })
        }
}
```

一般的问题是，**当行为没有文档记录，元素名称不清楚时，开发人员将依赖于当前的实现，而不是我们打算建立的抽象**。我们通过描述可以预期的行为来解决这个问题。

### 合约

当我们描述某些行为时，用户会将其视为一种承诺，并基于此调整他们的期望。**我们将所有这些预期行为称为元素的合约**。就像在现实生活中，我们遵守法律一样，在这里，用户也希望我们一旦确立了合约的稳定性后就一直遵守它（_第28条：指定API的稳定性_）。

在这点上，定义合约听起来很可怕，但实际上，这对双方都是好事。 当合约被很好的指定时，创建者不需要担心类被如何使用，用户也不需要担心某些东西在底层是如何实现的。用户可以依赖这个合约，而不需要了解任何实际信息。对于创造者来说，合约赋予了他们只要能够满足要求，就能改变一切的自由。用户和创建者都依赖合约中定义的抽象，因此他们可以独立工作。只要合约有效，一切都会很顺利的。这对双方来说都是一种舒适和自由。

假如我们不建立这个合约会怎么样呢？如果用户不知道他们能做什么、不能做什么，他们将会依赖于实现细节。如果创建者不知道用户依赖了什么，他们开发就会被阻塞，或者可能破坏用户的实现，如你所见，明确合约是很重要的。

### 定义一个合约

我们该如何定义合约？ 有很多种方式，包括：

* **名称** —— 当一个名称与一些一般概念有关联时，我们希望这个元素与这个概念相一致。例如，当你看到 `sum` 方法时，你不需要读它的注释就知道它是做什么的。这是因为求和已经是数学上定义好的概念了。
* **注释和文档** —— 这是最强大的方式，因为它可以描述所有需要的东西
* **类型** —— 类型可以说明很多关于对象的信息，每种类型指定一组通常定义良好的方法，有些类型在其文档中还具备组织架构的职责，当我们看见一个函数时，返回信息和参数信息非常有意义

### 我们需要注释吗

纵观历史，我们会惊奇的发现社区的意见是如何波动起伏的。当 Java 还在早期时，有一个非常流行的编程概念：它建议在注释中说明一切。 十年后，我们可以听到对注释强烈的声讨声，认为我们应该忽略注释，转而专注于编写于可读的代码（我相信宣扬此论点最有影响力的书是 Robert C.Martin 的 《代码整洁之道》）

没有极端才是健康的。我完全同意我们应该首先专注于编写可读的代码，但需要理解的是元素（函数或类）上的注释可以在更高的层次上描述它，并制定它们的合约。**此外，注释现在经常用于自动生成文档，在项目中通常被视为真实信息的来源**。

当然了，我们通常是不需要注释的。例如，许多函数是自解释的，它们不需要任何特殊的描述，例如，我们可以假设 `product`（点积） 是程序员所知道的一个清晰的数学概念，而不加任何评论：

```kotlin
fun List<Int>.product() = fold(1) { acc, i -> acc * i }
```

显而易见这里加注释反而多此一举，只会让我们分心。不要写只描述函数名和参数所表达的内容的注释，下面的例子演示了不必要的注释，因为功能可以从方法名和参数类型推断出来：

```kotlin
// 让一个List中所有的数相乘
fun List<Int>.product() = fold(1) { acc, i -> acc * i }
```

我也同意，当我们只需要组织代码，而不是在实现中添加注释时，我们应该提取一个函数，看看下面这个例子：

```kotlin
fun update() {
    // 更新 users
    for (user in users) {
        user.update()
    }
    
    // 更新 books
    for (book in books) {
        updateBook(book)
    }
}
```

函数 `update` 显然是有可以提取的地方，并且注释建议可以分别来描述这部分内容。因此，最好将这些部分提取为单独的抽象，比如实例方法，并且让它们的名称足够清晰，可以解释它们的含义（就如_第26条：每个函数都应在单个抽象级别上编写_）。

```kotlin
fun update() {
    updateUsers()
    updateBooks()
}

private fun updateBooks() {
    for (book in books) {
        updateBook(book)
    }
}

private fun updateUsers() {
    for (user in users) {
        user.update()
    }
}
```

尽管注释通常是有用和重要的，要找到示例，请查看来自 Kotlin 标准库中的几乎所有公有函数，它们有定义明确的合约，给了我们很多自由，例如，看下函数 `listOf`:

```kotlin
/**
* Returns a new read-only list of given elements.
* The returned list is serializable (JVM).
* @sample samples.collections.Collections.Lists.
readOnlyList
*/
public fun <T> listOf(vararg elements: T): List<T> =
        if (elements.size > 0) elements.asList()
        else emptyList()
```

它所承诺的只是返回 JVM 上只读和可序列化的 `List`，列表不需要是不可变的，没有承诺具体的类。这个合约是极简的，但是满足大多数 Kotlin 开发人员的需求，你还可以看到它指向示例用法，这在我们学习如何使用元素时也很有用。

### KDOC 格式

当我们使用注释为函数编写文档时，表示注释的正式格式成为 KDoc，所有 KDoc 注释都以 /\*\* 开始，以 \*/ 结束，并在内部所有行通常以 \* 开始。 并使用 KDoc Markdown 编写内容的。

KDoc 注释的结构如下：

* 文档文本的第一段是对元素的描述总结
* 第二部分是详细的内容
* 每一行都已一个标签开始，这些标签用于引入一个元素来描述它。

下面是已支持的标签：

* `@params <name>` —— 记录函数的参数，或者类的类型、属性、函数
* `@return` —— 记录函数的返回值
* `@constructor` —— 记录类的主构造函数
* `@receiver` —— 记录扩展函数的接收方（指针）
* `@property <name>` —— 记录指定名称的类的属性，用于在主构造函数定义的属性
* `@throws <class>, @exception <class>` —— 记录该函数可能会抛出的异常
* `@sample <identifier>` —— 将带有指定限定名的函数嵌入到当前元素的文档中，以展示如何使用该元素
* `@see <identifier>` —— 添加到指定类或方法的链接
* `@author` —— 指定被标记元素的作者
* `@since` —— 指定被记录的元素被引入的软件版本
* `@suppress` —— 从生成的文档中排除该元素，可以用于不属于官方 API 的模块但仍然必须对外部可见的元素

无论是在注释，还是在描述标签的文本中，我们可以链接到元素、具体的类、方法、属性或参数，当我们希望可以换一种名称来表示链接元素（和元素名称不同）时，链接使用中括号或者双中括号表示：

```kotlin
/**
* This is an example descriptions linking to [element1],
* [com.package.SomeClass.element2] and
* [this element with custom description][element3]
*/
```

所有这些标记都被 Kotlin 文档生成工具所理解，其官方名称是 “Dokka”。它们生成可以在线发布并呈现给外部用户看的文档文件，下面是简单描述的示例文档：

```kotlin
/**
* Immutable tree data structure.
*
* Class represents immutable tree having from 1 to
* infinitive number of elements. In the tree we hold
* elements on each node and nodes can have left and
* right subtrees...
*
* @param T the type of elements this tree holds.
* @property value the value kept in this node of the tree.
* @property left the left subtree.
* @property right the right subtree.
*/
class Tree<T>(
    val value: T,
    val left: Tree<T>? = null,
    val right: Tree<T>? = null
) {
    /**
    * Creates a new tree based on the current but with
    * [element] added.
    * @return newly created tree with additional element.
    */
    operator fun plus(element: T): Tree { ... }
}
```

请注意，并不是所有的内容都需要描述，最好的文档应该短小精悍，准确描述那些可能不明确的内容。

### 类型系统和期望

类型层次结构是关于对象的一个重要信息源。接口不仅仅是我们承诺要实现的方法列表，类和接口也可以有一些期望。如果一个类承诺了一个期望，它的所有子类也应该保持这一点。这个原则被称为里氏替换原则，是面向对象编程中最重要的规则之一。它通常被解释为：“如果 S 是 T 的子类，那么类型 T 的对象可以被替换为类型 S 的对象，而不改变程序的任何属性。” 为什么它很重要的一个简单解释是：每个类都可以用作超类，因此如果它的行为不像我们所期望的那样，我们可能会遇到意料之外的异常。在编程中，子类应该遵守父类的约定。

该规则的一个重要含义是：我们应当适当的开放函数指定的合约。例如，回到我们之前比喻过的汽车，我们可以在代码中使用以下接口表示汽车：

```kotlin
interface Car {
    fun setWheelPosition(angle: Float)
    fun setBreakPedal(pressure: Double)
    fun setGasPedal(pressure: Double)
}

class GasolineCar: Car {
    // ...
}

class GasCar: Car {
    // ...
}

class ElectricCar: Car {
    // ...
}
```

这个接口留下了很多问题： `setWheelPosition` 中的角度是什么意思？ 使用什么单位？ 如果有人不清楚有油门和刹车踏板的作用该怎么办？ 使用 Car 类型的实例的人需要知道如何使用它们，所有品牌（子类）在作为 Car 使用时，都应该表现出类似的行为，我们可以通过文档来解决这个问题：

```kotlin
interface Car {
    /**
    * Changes car direction.
    *
    * @param angle Represents position of wheels in
    * radians relatively to car axis. 0 means driving
    * straight, pi/2 means driving maximally right,
    * -pi/2 maximally left.
    * Value needs to be in (-pi/2, pi/2)
    */
    fun setWheelPosition(angle: Float)

    /**
    * Decelerates vehicle speed until 0.
    *
    * @param pressure The percentage of brake pedal use.
    * Number from 0 to 1 where 0 means not using break
    * at all, and 1 means maximal pedal pedal use.
    */
    fun setBreakPedal(pressure: Double)
    
    /**
    * Accelerates vehicle speed until max speed possible
    * for user.
    *
    * @param pressure The percentage of gas pedal use.
    * Number from 0 to 1 where 0 means not using gas at
    * all, and 1 means maximal gas pedal use.
    */
    fun setGasPedal(pressure: Double)
}
```

现在所有的汽车都设定了一个标准，来描述它们应该如何表现。 stdlib 和主流的库中都对其子类有定义良好、描述良好的合约和期望，我们也应该为元素定义合约，这些合约将真正盘活接口。它们将给我们自由去使用那些遵守合约实现接口的类。

### 实现的泄漏

实现细节总是会泄漏。在一辆汽车中，不同种类的发动机表现略有不同，我们仍然能够驾驶汽车，但我们可以感觉到不同。这没有问题，因为合约中没有说明。

在编程语言中，实现细节也会泄露。例如，使用反射可以调用一些函数，但它比普通函数的调用要慢得多(除非编译器对它进行了优化)。我们将在关于性能优化的章节中看到更多的例子。尽管只要一种语言像它承诺的那样工作，一切都没问题，但是我们仍然需要记住并应用到好的做法去。

在我们的抽象中，实现也会泄漏，但我们仍要尽可能地保护它。我们通过封装来保护实现，封装可以描述为“你可以做我所允许的事，仅此而已”。封装的类和函数越多，在内部就有越多的自由，因为我们不需要考虑使用方如何依赖于我们的实现。

### 总结

当我们定义一个元素，特别是外部 API 的部分时，我们应该定义一个合约，我们通过名称、文档、注释和类型来达到目标。合约指定了对这些元素的期望，它还可以描述应该如何使用这些东西。合约让用户对元素现在和将来的行为建立了使用的信心，它还让创建者可以更自由的更改合约中没有指定的内容。合约是一种协议，只要双方都遵守它，它就有效。
