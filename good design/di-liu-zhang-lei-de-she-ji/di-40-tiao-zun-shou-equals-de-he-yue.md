# 第40条：遵守 equals 的合约

在 Kotlin，每个对象都继承了 `Any`，它具备一些合约方法，这些方法有：

* `equals`
* `hashCode`
* `toString`

它们的合约描述在注释中，并在官方文档中被详细阐述。正如我在_第32条：遵守抽象合约_中所描述的那样，一个类型的子类型都应该遵守这个合约。这些方法在 Kotlin 中具有重要地位，因为自从 Java 开始就定义了它们，因此许多对象和函数都依赖这些合约。违背它们的合约往往会导致一些功能不能正常运转。这就是为什么在当前和下一个条目中将讨论覆盖这些功能和它们的合约会发生什么。 让我们先从 `equals` 开始。

### 相等性

在 Kotlin 中，存在两种类别的相等：

* 结构上相等 —— 通过 `equals` 方法，或基于 `equals` 方法的 `==` 操作符（以及与之相反的 `!=`）来检查。当 a 不为空时，`a == b` 可以转化为 `a.equals(b)`，否则可以转化为 `a?.equals(b) ?: (b === null)`
* 引用相等 —— 由 `===` 操作符（以及与之相反的 `!==`） 检查，当两边都指向同一个对象时返回true

因为 `equals` 是在 `Any` 中实现的, `Any` 是每个类的超类，所以我们可以检查任意两个对象是否相等。尽管当它们不是同一个类型时，不允许使用操作符检查相等性:

```kotlin
open class Animal
class Book
Animal() == Book() // Error: Operator == cannot be
// applied to Animal and Book
Animal() === Book() // Error: Operator === cannot be
// applied to Animal and Book
```

对象要么需要相同的类型，要么需要是另一个对象的子类：

```kotlin
class Cat: Animal()
Animal() == Cat() // OK, because Cat is a subclass of Animal
Animal() === Cat() // OK, because Cat is a subclass of Animal
```

这是因为检查不同类型的两个对象是否相等没有意义。等我们解释 `equals` 的合约时，就会清楚了。

### 为什么我们需要 equals？

来自 `Any` 的 `equals` 的默认实现是检查另一个对象和它是否完全是同一个实例，就如同检查引用相等一样（===）。这意味着默认情况下每个对象都是唯一的：

```kotlin
class Name(val name: String)
val name1 = Name("Marcin")
val name2 = Name("Marcin")
val name1Ref = name1

name1 == name1 // true
name1 == name2 // false
name1 == name1Ref // true

name1 === name1 // true
name1 === name2 // false
name1 === name1Ref // true
```

这种行为对许多对象都是有用的。它非常适合激活的元素，比如数据库连接、存储库或线程。然而，有些对象需要用不同的方式表示相等。一个主流的检查相等的方案就是 data class 那样所做的：对比所有主构造函数属性是否相等:

```kotlin
data class FullName(val name: String, val surname: String)
val name1 = FullName("Marcin", "Moskała")
val name2 = FullName("Marcin", "Moskała")
val name3 = FullName("Maja", "Moskała")

name1 == name1 // true
name1 == name2 // true, 因为所有数据都是相同的
name1 == name3 // false

name1 === name1 // true
name1 === name2 // false
name1 === name3 // false
```

这种方式就非常适合那些持有数据的类，因此我们经常在数据模型类或其它数据持有者中使用 `data` 修饰符。

注意，当需要比较某些属性（而不是所有属性）时，data class 的相等性比较对我们也有帮助。例如当我们想要忽略掉缓存属性或其他冗余属性时。下面是一个表示时间对象的例子，它有两个属性 `asStringCache` 和 `changed`，在检查相等性时，不应该用这两个属性去比较：

```kotlin
class DateTime(
    /** The millis from 1970-01-01T00:00:00Z */
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
    
    override fun equals(other: Any?): Boolean =
        other is DateTime && other.millis == millis &&
other.timeZone == timeZone
    
    //...
}
```

使用 `data` 修饰符也能达到相同的效果：

```kotlin
data class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
    //...
}
```

请注意，在这种情况下， `copy` 不会复制那些不在主构造函数中声明的属性。只有当这些附加属性确实冗余时，这种行为才是正确的（即使它们丢失，对象也能正确的工作）

默认类和数据类的相等性，多亏了这两种可选的方法，我们很少需要在 Kotlin 中自行实现 `equals`。尽管有些情况下我们还是要去实现的。

另一个例子是用一个具体的属性决定两个对象是否相等。例如， User 类可能假设两个用户的 id 相等时，它们是相等的：

```kotlin
class User(
    val id: Int,
    val name: String,
    val surname: String
) {
    override fun equals(other: Any?): Boolean =
other is User && other.id == id

    override fun hashCode(): Int = id
}
```

如你所见，出现下面这些情况时，我们需要自行实现 `equals`：

* 我们需要这个相等性比较的逻辑和默认实现的不一样
* 我们只要需要比较属性的一个子集
* 我们不希望我们的对象是一个data类，或者需要比较的属性不在主构造函数中

### equals 的合约

这是 `equals` 注释中的描述（Kotlin 1.3.11）：

表示其他对象是否“等于”这个对象，实现必须满足如下要求：

* 自反性：对于任意非空值 x， `x.equals(x)` 应当返回 true
* 对称性：对于任意非空值 x、y， 若 `x.equals(y)` 返回 true，那么 `y.equals(x)` 也应当返回 true
* 传递性：对于任意非空值 x、y、z， 如果 `x.equals(y)` 返回 true 并且 `y.equals(z)` 返回 true，那么 `x.equals(z)` 应当返回 true
* 一致性：对于任意非空值 x、y， 在对象的 `equals` 中比较的信息没有改变的情况下，多次调用 `x.equals(y)` 应当一致返回 true 或 false
* 永远不要等于 null： 对于任意非空值 x， `x.equals(null)` 将返回 false

此外，我们希望 `equals`、`toString` 和 `hashCode` 是快速的。这虽然不是官方合约的一部分，但假如需要等上几秒钟才能知道相等性比较的结果，这会让人感到意外的。

所有这些要求都很重要，这个规则从一开始就被制定了出来，在 Java 中亦是如此，所以现在很多对象都依赖这些规则。如果它们看起来很混乱，不用担心，我将详细阐述它们。

* 对象的相等性应该是自反的，即 `x.equals(x)` 返回true。听起来这是显而易见的，但这是可以违背的，例如，某人可能想创建一个 `Time` 对象来表示当前的时间，并比较毫秒：

```kotlin
// 不要这样做！
class Time(
    val millisArg: Long = -1,
    val isNow: Boolean = false
) {
    val millis: Long get() =
        if (isNow) System.currentTimeMillis()
        else millisArg
    
    override fun equals(other: Any?): Boolean =
other is Time && millis == other.millis
}

val now = Time(isNow = true)
now == now // 有时候是 true，有时候是 false
List(100000) { now }.all { it == now }
// 大多都是 false
```

注意，这里的结果是不一致的，所以它也违反了最后一个原则。

当一个对象不等于其本身时，即使使用 `contains` 方法进行检查时，也可能无法在大部分集合中找到它。它在大多数单元测试断言中也不能正常工作。

```kotlin
val now1 = Time(isNow = true)
val now2 = Time(isNow = true)
assertEquals(now1, now2)
// 有时候可以通过，有时候不行
```

如果结果不是恒定的，我们就不能信任它。我们永远不能确定这个结果是否是正确的。我们应该如何改进它？简单地解决方案是单独检查对象是否表示当前的时间，如果不是，则它是否具有相同的时间戳。这是一个典型的标记类的例子，如_第39条：类层次结构优于标记类_所描述那样，使用类层次结构会更好：

```kotlin
sealed class Time
data class TimePoint(val millis: Long): Time()
object Now: Time()
```

* 对象的相等性是对称的，这意味着 x == y 和 y == x 的结果应该总是相同的。当我们在相等性中接收不同类型的对象时，很容易违反这一点。例如，假设我们实现了一个表示复数的类，并将其相等性设置为 Double 的相等：

```kotlin
class Complex(
    val real: Double,
    val imaginary: Double
) {
    // DO NOT DO THIS, violates symmetry
    override fun equals(other: Any?): Boolean {
        if (other is Double) {
            return imaginary == 0.0 && real == other
        }
        return other is Complex &&
real == other.real && imaginary == other.imaginary
    }
}
```

这里的问题是， Double 并不能接受与 `Complex` 相等。因此，相等性比较的结果取决于元素的位置：

```kotlin
Complex(1.0, 0.0).equals(1.0) // true
1.0.equals(Complex(1.0, 0.0)) // false
```

例如，缺乏对称性，意味着会在集合的 `contains` 或单元测试的断言上出现异常：

```kotlin
val list = listOf<Any>(Complex(1.0, 0.0))
list.contains(1.0) // 在 JVM 上是false， 
// 但是它依赖于集合的实现
// 并且不要相信结果永远会是一样的
```

**当相等性是不对称，并且使用另一个对象进行比较的时候，产生的结果是我们所不能信任，因为它取决于该对象是 x 来比较 y， 还是 y 来比较 x**。这个事实没有被记录，它并不是合约的一部分，因为类的创建者认为无论是哪种情况的结果都应该是一样的（他们认为相等是对称的）。这种比较方式也可以在任何时刻发生变化 —— 例如某些重构过程中，创建者可能会更改比较顺序。如果你的对象不是对称的，它可能会在你的实现中导致意想不到的并且很难调试的错误。这就为什么我们执行 `equals` 时，应该总是考虑到对称性。

一般的解决方法是我们不应该接受不同类型之间的相等性比较。我从来没有见过那样做是合理的。注意，在 Kotlin 中，相似的类并不相等。 1 不等于 1.0, 1.0 不等于 1.0f，这些都是不同的类型，它们甚至不能进行比较。同样，在 Kotlin 中，我们不能在两个除了 `Any` 以外没有公共超类的不同类型之间使用 `==` 操作符：

```kotlin
Complex(1.0, 0.0) == 1.0 // ERROR
```

* 对象的相等性是可以传递的，这意味着对于任意的非空引用 x、y、z如果 `x.equals(y)` 返回 true，并且 `y.equals(z)` 返回 true，那么 `x.equals(z)` 应该返回 true。传递性最大的问题是出现在，当我们实现不同相同性比较，它来检查属性的不同的子类型时，例如，让我们这样来定义 Date 和 DateTime：

```kotlin
open class Date(
    val year: Int,
    val month: Int,
    val day: Int
) {
    // 不要这么做，语义正确但是没有传递性
    override fun equals(o: Any?): Boolean = when (o) {
        is DateTime -> this == o.date
        is Date -> o.day == day && o.month == month &&
o.year == year
        else -> false
    }
    // ...
}

class DateTime(
    val date: Date,
    val hour: Int,
    val minute: Int,
    val second: Int
): Date(date.year, date.month, date.day) {

    // 不要这么做，语义正确但是没有传递性
    override fun equals(o: Any?): Boolean = when (o) {
        is DateTime -> o.date == date && o.hour == hour &&
o.minute == minute && o.second == second
        is Date -> date == o
        else -> false
    }
    // ...
}
```

上述实现的问题是，当比较两个 `DateTime` 时，相对与 `DateTime` 和 `Data` 的比较，要比较更多的属性。因此，两个日期相同但时间不同的 `DateTime` 不会相等，但它们都等于同一个 `Date`。因此，它们的关系是不可传递的：

```kotlin
val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1 == o2 // true
o2 == o3 // true
o1 == o3 // false <- 所以判断出，不具备传递性
```

注意，在这里，只比较相同类型的对象的限制是没有的，因为我们使用了继承。这种继承违反了里氏替换原则，不应该使用。在这种情况下，使用组合而不是继承（_第36条：组合优于继承_）。当你这样做时，不要比较两个不同类型的对象。这些类是保存数据的对象的完美例子，用下面方式表示它们相等是一个很好的选择：

```kotlin
data class Date(
    val year: Int,
    val month: Int,
    val day: Int
)

data class DateTime(
    val date: Date,
    val hour: Int,
    val minute: Int,
    val second: Int
)

val o1 = DateTime(Date(1992, 10, 20), 12, 30, 0)
val o2 = Date(1992, 10, 20)
val o3 = DateTime(Date(1992, 10, 20), 14, 45, 30)

o1.equals(o2) // false
o2.equals(o3) // false
o1 == o3 // false

o1.date.equals(o2) // true
o2.equals(o3.date) // true
o1.date == o3.date // true
```

* 相等性应该是一致的。这意味着对两个对象调用 `equals` 方法应该总是返回相同的结果，除非其中一个对象被修改。对于不可变对象，结果应该总是相同的，换句话说，我们希望 `equals` 是一个纯函数（不修改对象的状态），其结果总是只依赖于接收方的输入和状态。我们已经看到 Time 类违反了这个原则。总所周知， `java.net.URL.equals()` 也违反了这个原则
* 不等于 null：对于任何非空值 x，`x.equals(null)` 必须返回 false，这点很重要。因为 null 应该是唯一的，没有对象应该等于它

### URL 的 equals 问题

一个设计非常糟糕的 `equals` 示例来自于 `java.net.URL.equals`。两个 java.net.URL 的相等性比较取决于一个网络操作，如果两个主机名都可以解析为相同的 IP 地址，那么这两个主机就被认为是等价的。来看看下面这个例子:

```kotlin
import java.net.URL

fun main() {
    val enWiki = URL("https://en.wikipedia.org/")
    val wiki = URL("https://wikipedia.org/")
    println(enWiki == wiki)
}
```

结果并不是一致的，它应该打印 true，因为 `equals` 的实现，这两个地址被认为是相等的。但是如果你关闭了网络，它将打印 false。你可以自行检查一下，这是一个很大的错误！等价性不应该依赖于网络。

这种处理方式体现了几个很重要的问题：

* 这种行为是不一致的。例如，两个 URL 可以在网络可用时相等，而在网络不可用时不相等。此外，网络可能会发生变化，给定主机名的 IP 随时间和网络发生变化。两个 URL 可能在某些网络上相等，而在其它网络上不相等
* 网络可能很慢，我们希望 `equals` 和 `hashCode` 是很快的。一个典型的问题是当我们检查 URL 是否存在于列表中的时候。这样的操作需要对列表的每个元素进行网络调用。此外，在一些平台上， 如 Android，网络操作是禁止在主线程上进行的。因此，即使添加一个 url 到 `set` 上也需要在单独的线程上启动
* 虚拟主机（共享主机）会让这个 `equals` 行为不一致。 相同的 IP 地址并不意味着相同的内容，虚拟主机允许不相关的网站共享一个 IP 地址，此方法可能让两个在其他方面完全不相关的 url 相等，因为它们被托管于同一台服务器上

在 Android 中，这个问题在 Android 4.0（Ice Cream Sandwich）才得到解决。自改版本以来，只有主机名相等时，url 才相等。当我们在其他平台上使用 Kotlin/JVM 时，建议使用 `java.net.URI` 而不是 `Java.net.URL`

### 实现 equals

除非你有一个很好的理由，否则我不推荐你自行实现 `equals`，相反的，应该使用默认或 data 类的 equals。如果你确定需要实现自定义 `equals`，请始终考虑你的实现是否具有自反性、对称性、传递性和一致性。将这样的类设为final，否则注意子类不应该改变相等性行为。很难在自定义相等性的同时支持继承。有些人甚至说是不可能的。 data 类都是final的。
