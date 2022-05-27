# 第37条：使用数据修饰符来表示一组数据

有时我们只需要传递一组数据，这就是我们使用 data class 的目的。这些是带有 `data` 修饰符的类，从我的经验来看，开发人员很快就会把它引入到他们的数据模型类中：

```kotlin
data class Player(
    val id: Int,
    val name: String,
    val points: Int
)

val player = Player(0, "Gecko", 9999)
```

当我们添加 `data` 修饰符时，它会生成一些有用的函数：

* `toString`
* `equals` 和 `hashCode`
* `copy`
* `componentN`（`component1`、`component2`等）

让我们根据数据类型依次讨论它们。

`toString` 显示类名和所有主构造函数属性的值及其名称。有助于日志的展示和调试：

```kotlin
print(player) // Player(id=0, name=Gecko, points=9999)
```

![](<../../.gitbook/assets/image (8).png>)

`equls` 检查是否所有的主构造函数属性都相等，并且 `hashCode` 与它是一致的（请看_第41条:遵守hashCode的合约_）。

```kotlin
player == Player(0, "Gecko", 9999) // true
player == Player(0, "Ross", 9999) // false
```

`copy` 对于不可变数据类型非常有用，它创建了一个新的对象，其中每个主构造函数属性在默认情况下都具有相同的值，但可以使用具名参数更改它们：

```kotlin
val newObj = player.copy(name = "Thor")
print(newObj) // Player(id=0, name=Thor, points=9999)
```

我们无法看到 `copy` 方法的实现，因为它是在底层生成的，就像通过 `data` 修饰符生成的其它方法一样，如果我们能看到它，大概就是下面生成 `Person` 的样子：

```kotlin
// This is how `copy` is generated under the hood by
// data modifier for `Person` class looks like
fun copy(
    id: Int = this.id,
    name: String = this.name,
    points: Int = this.points
) = Player(id, name, points)
```

注意，`copy` 方法对一个对象做了浅拷贝，但是当对象不可变的时候，这并不是问题 —— 对于不可变的对象，不需要深拷贝。

`componentN` 函数（`component1`、`component2` 等等）允许基于其位置的解构，就像下面这个例子：

```kotlin
val (id, name, pts) = player
```

Kotlin 中的解构直接转化为使用 `componentN` 函数中的变量来定义，因此上面的代码将被编译成下面这段代码：

```kotlin
// After compilation
val id: Int = player.component1()
val name: String = player.component2()
val pts: Int = player.component3()
```

基于位置的解构有优点和缺点。最大的优点是我们可以随意命名变量，我们也可以分解我们想要的一切，只要它提供 `componentN` 函数。 `List` 和 `Map.Entry` 都能体现它 ：

```kotlin
val visited = listOf("China", "Russia", "India")
val (first, second, third) = visited
println("$first $second $third") // China Russia India

val trip = mapOf(
    "China" to "Tianjin",
    "Russia" to "Petersburg",
    "India" to "Rishikesh"
)
for ((country, city) in trip) {
    println("We loved $city in $country")
    // We loved Tianjin in China
    // We loved Petersburg in Russia
    // We loved Rishikesh in India
}
```

另一方面，它是危险的，当类中的元素顺序发生变化时，我们需要调整每一个解构。它也很容易因混乱的顺序而错误的分解：

```kotlin
data class FullName(
    val firstName: String,
    val secondName: String,
    val lastName: String
)

val elon = FullName("Elon", "Reeve", "Musk")
val (name, surname) = elon
print("It is $name $surname!") // It is Elon Reeve!
```

我们需要小心地解构。使用和主构造函数中相同的属性的名称是很有用的，然后，不正确的顺序，IntelliJ / Android Studio 将会显示警告。甚至可以将警告升级为错误。

![](<../../.gitbook/assets/image (6).png>)

不要像下面的例子那样分解得到第一个值：

```kotlin
data class User(val name: String)
val (name) = User("John")
```

这可能会让读者感到困惑，特别是当你在 lambda 表达式中进行分解时：

```kotlin
data class User(val name: String)

fun main() {
    val user = User("John")
    user.let { a -> print(a) } // User(name=John)
    // 不要这样做
    user.let { (a) -> print(a) } // John
}
```

这是有问题的，因为在一些语言中，lambda 表达式中包住参数的括号是可选或必需的。

### 优先使用 data class 替代元组（tunples）

data class 提供的功能比元组更多。更具体的说，Kotlin 元组只是可序列化的通用数据类型，并且有一个自定义的 `toSring` 方法：

```kotlin
public data class Pair<out A, out B>(
    public val first: A,
    public val second: B
) : Serializable {
    
    public override fun toString(): String ="($first, $second)"
}

public data class Triple<out A, out B, out C>(
    public val first: A,
    public val second: B,
    public val third: C
) : Serializable {
    public override fun toString(): String = "($first, $second, $third)"
}
```

为什么我只展示了 `Pair` 和 `Triple`？ 这是因为它们是 Kotlin 最后剩下的元组类型。Kotlin 在 beta 版本就有无限元组了。我们可以通过括号和一组类型来定义类型，如: (Int, String, String, Long)。最后，我们所实现的行为与数据类相同，但可读性差很多。你能猜出这组类型代表什么吗？ 它可以是任何东西，使用元组很诱人，但使用 data class 几乎总是更好！这就是为什么元组被删除，只留下 `Pair` 和 `Triple`，它们被保留下是出于一些在小范围内使用的意图，例如：

* 当我们需要立即命名值时：

```kotlin
val (description, color) = when {
    degrees < 5 -> "cold" to Color.BLUE
    degrees < 23 -> "mild" to Color.YELLOW
    else -> "hot" to Color.RED
}
```

* 表示一个事先没有被定义的组合 —— 通常在标准库中使用：

```kotlin
val (odd, even) = numbers.partition { it % 2 == 1 }
val map = mapOf(1 to "San Francisco", 2 to "Amsterdam")
```

在其它情况下，我们更喜欢 data class。让我们来看一个例子，假设我们需要为一函数来将 fullname 解析为 name 和 surnname，可以将这个 name 和 surnname 表示为 `Pair<String, String>`:

```kotlin
fun String.parseName(): Pair<String, String>? {
    val indexOfLastSpace = this.trim().lastIndexOf(' ')
    if(indexOfLastSpace < 0) return null
    val firstName = this.take(indexOfLastSpace)
    val lastName = this.drop(indexOfLastSpace)
    return Pair(firstName, lastName)
}

// Usage
val fullName = "Marcin Moskała"
val (firstName, lastName) = fullName.parseName() ?: return
```

问题是，当有人阅读它时，他并不清楚 `Pair<String, String>` 里面的每个类型分别表示的是什么。更重要的是，这些值的顺序并不清楚，有人认为可能 surnname 会在前面：

```kotlin
val fullName = "Marcin Moskała"
val (lastName, firstName) = fullName.parseName() ?: return
print("His name is $firstName") // His name is Moskała
```

为了使引用更加安全、函数更加易读，我们应该使用数据类型：

```kotlin
data class FullName(
    val firstName: String,
    val lastName: String
)

fun String.parseName(): FullName? {
    val indexOfLastSpace = this.trim().lastIndexOf(' ')
    if(indexOfLastSpace < 0) return null
    val firstName = this.take(indexOfLastSpace)
    val lastName = this.drop(indexOfLastSpace)
    return FullName(firstName, lastName)
}

// Usage
val fullName = "Marcin Moskała"
val (firstName, lastName) = fullName.parseName() ?: return
```

它的成本几乎为0，并且显著改善了功能：

* 函数的返回类型是明确的
* 返回类型更短，更容易传递
* 如果用户解构的变量名不同，则会显示一个警告

如果不希望这个类在更大范围内使用时，可以限制其可见性，如果你只需要在单个文件或类中用某些本地函数处理使用它，它甚至可以是私有的。我们更值得去使用 data class 而不是元组。 Kotlin 的类成本很低，请不要害怕使用它们。
