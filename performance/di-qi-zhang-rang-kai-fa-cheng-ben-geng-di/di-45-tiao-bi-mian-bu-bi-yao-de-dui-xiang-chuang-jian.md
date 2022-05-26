# 第45条：避免不必要的对象创建

对象的创建有时候会很昂贵，而且总是要付出一些代价。这就是为什么避免不必要的对象创建是一项重要的优化。这可以在很多层面上实现。例如，在 JVM 中，就可以保证相同面量的字符串会被放到字符串常量池的同一引用中，方便被其它代码所重用。

```kotlin
val str1 = "Lorem ipsum dolor sit amet"
val str2 = "Lorem ipsum dolor sit amet"
print(str1 == str2) // true
print(str1 === str2) // true
```

当装箱的原语（Integer、Long）都很小的时候，它们也可以在 JVM 中重用（默认的 Integer Cache（整型的缓存）保存的数字范围从 -128 到 127）。

```kotlin
val i1: Int? = 1
val i2: Int? = 1
print(i1 == i2) // true
print(i1 === i2) // true, 因为 i2 是从缓存中获取的
```

引用相等表明它们是同一个对象，如果我们使用的 number 的值小于-128或大于127 ，则会产生不同的对象：

```kotlin
val j1: Int? = 1234
val j2: Int? = 1234
print(j1 == j2) // true
print(j1 === j2) // false
```

Notice that a nullable type is used to force Integer instead of int under the hood. When we use Int, i注意，如果使用了一个可空类型，在底层它会强制使用 `Integer` 类型而不是 `Int`。当使用整型时，通常将其编译为原语 `int` 类型，但是如果它是可空值，或者它作为参数被传递出去时，则使用 `Integer`。这是因为原语（primitive）不能为空，也不能用作参数。知道了语言中引入的这些机制，你就能理解它们有多么重要。那对象的创建是否是昂贵的么？

### 对象创建是昂贵的么？

包装使用一个对象，会在三个方面有所开销：

* **对象占有额外的空间**。在现代64位 JDK 中，对象有一个12字节的头部，被8字节的倍数填充，因此对象最小的大小为16字节。对于32位JVM，开销为8字节。此外，对象引用也很占用空间。通常，在32位平台上引用需要4字节，64位平台上则需要-Xmx32G，在32GB以上为8字节（-Xmx32G）。这些是相对较小的数字，但它们堆堆积起来将会产生巨大的成本。当我们考虑像 Int 这样小的元素时，它们是有区别的。 Int 作为原语可以容纳4个字节，而在我们今天主流的64位 JDK 上作为封装类型时，它需要16个字节（它可以容纳头部的后4个字节） + 引用的4或8个字节，最终，它会占用本身大小5到6倍的空间
* **当元素被封装时，访问需要额外的函数调用**。由于函数的使用非常快，所以这只会产生一个很小的消耗，但当我们需要对一个巨大的对象池进行操作时，它就会累加。不要限制封装，避免创建不必要的对象，特别是在对性能比较敏感的地方
* **需要创建对象**，一个对象需要被创建，并在内存中被分配，还需要创建引用等等。这也是非常小的成本，但是也会聚沙成塔

```kotlin
class A
private val a = A()

// Benchmark 结果: 2.698 ns/op
fun accessA(blackhole: Blackhole) {
    blackhole.consume(a)
}

// Benchmark 结果: 3.814 ns/op
fun createA(blackhole: Blackhole) {
    blackhole.consume(A())
}
// Benchmark 结果: 3828.540 ns/op
fun createListAccessA(blackhole: Blackhole) {
    blackhole.consume(List(1000) { a })
}
// Benchmark 结果: 5322.857 ns/op
fun createListCreateA(blackhole: Blackhole) {
    blackhole.consume(List(1000) { A() })
}
```

通过消除对象，我们可以避免这三种开销。通过重用对象，我们可以消除第一个和第三个开销。知道了这一点，你可能会开始考虑限制代码中不必要对象的数量。让我们看看有什么方法可以做到。

### object 声明

重用对象而不是每次都创建对象的一个非常简单的做法是使用 `object` 声明（单例）。举个例子，假设你需要实现一个链表，链表可以是空的，也可以是一个节点并带上指向剩余节点的指针。以下是它的实现方式：

```kotlin
sealed class LinkedList<T>

class Node<T>(
    val head: T,
    val tail: LinkedList<T>
): LinkedList<T>()

class Empty<T>: LinkedList<T>()

// Usage
val list: LinkedList<Int> =
    Node(1, Node(2, Node(3, Empty())))
val list2: LinkedList<String> =
    Node("A", Node("B", Empty()))
```

这种实现的一个问题是，每次创建链表时，都要创建一个 Empty 实例。反而，我们应该只有一个 `Empty` 的实例，并普遍地使用它。唯一的问题是泛型类型，我们应该使用什么泛型类型呢？ 我们希望这个空链表是所有列表的子类型，我们不能使用所有类型，但也不需要拿来做什么。一个解决方案是我们可以把它看成一个“什么都没有”的链表。 `Nothing` 是所有类型的子类型，所以一旦这个链表是协变的（`out` 修饰符）， `LinkedList<Nothing>` 将是每个 `LinkedList` 的子类型。使类型参数协变在这里有真正的意义，因为链表是不可变的，并且这种类型只在 `out` 的位置使用（_第24条：在使用泛型时考虑型变_）。下面是改进后的代码：

```kotlin
sealed class LinkedList<out T>

class Node<out T>(
    val head: T,
    val tail: LinkedList<T>
) : LinkedList<T>()

object Empty : LinkedList<Nothing>()

// Usage
val list: LinkedList<Int> =
    Node(1, Node(2, Node(3, Empty)))

val list2: LinkedList<String> =
    Node("A", Node("B", Empty))
```

这是一个非常有用的技巧，特别是在定义不可变密封类的时候。因为在使用可变对象的时候可能会导致：共享状态管理时出现微妙且难以检测的bug。一般规则是不应该缓存可变对象（_第1条：限制可变性_）。除了 object 声明之外，还有更多复用对象的方法。另一个方法是带缓存的工厂方法。

### 带缓存的工厂方法

每次使用构造函数时，都会有一个新的对象产生。但当你使用工厂方法，情况就不一定如此了。工厂方法可以实现缓存，最简单的情况是工厂方法总是返回相同的对象。例如， 来看下 stdlib 中的 `emptyList` 的实现：

```kotlin
fun <T> emptyList(): List<T> = EmptyList
```

有时我们有一组对象，我们只返回其中一个。例如，当我们在 Kotlin 协程库中使用默认分发器 `Dispatchers.Default` 时，它有一个线程池，如果我们使用它来启动任何事务，它就会从线程池中去选择并开启一个未使用的线程。 类似的，我们可能也会有一个数据库连接池。当要创建的对象比较重，并且我们可能需要同时使用一些可变对象时，拥有一个对象池是一个很好的方案（享元模式）。

缓存也可以用于参数化的工厂方法，在这种情况下，我们可以将对象保存到一个 `map` 中：

```kotlin
private val connections: MutableMap<String, Connection> =
    mutableMapOf<String, Connection>()

fun getConnection(host: String) =
    connections.getOrPut(host) { createConnection(host) }
```

缓存可以用于所有的纯函数，在这种情况下，我们称之为“记忆化”（memorization）。例如，下面是一个计算某个位置斐波那契的函数：

```kotlin
private val FIB_CACHE: MutableMap<Int, BigInteger> =
    mutableMapOf<Int, BigInteger>()

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
    if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}
```

现在我们的方法在第一次运行时几乎和原始的线性解法一模一样，但是之后再传入相同的参数，它会立即给出结果。下表给出了示例机器上记忆化实现与经典线性斐波那契实现的比较，还给出了经典实现的代码：



|             | n = 100 | n = 200 | n = 300 | n = 400 |
| ----------- | ------- | ------- | ------- | ------- |
| fibIter     | 1997ns  | 5243ns  | 7008ns  | 9727ns  |
| fib(first)  | 4413ns  | 9815ns  | 15485ns | 22205ns |
| fib(second) | 8ns     | 8ns     | 8ns     | 8ns     |

```
fun fibIter(n: Int): BigInteger {
    if(n <= 1) return BigInteger.ONE
    var p = BigInteger.ONE
    var pp = BigInteger.ONE

    for (i in 2..n) {
        val temp = p + pp
        pp = p
        p = temp
    }
    return p
}
```

你可以看到，第一次使用这个函数比经典函数还要慢，因为检查值是否在缓存中并将其添加到缓存中需要额外的开销。但一旦键值对被添加，检索几乎是瞬时的。

但是它有一个显著的缺点：我们需要使用更多的内存，因为 `Map` 需要存储在内存中。如果这事能在某个时候解决，那么一切都会好起来的。但是要考虑到，对于垃圾收集器（GC），缓存可能和将来需要的任何其它静态字段没有区别。即使我们不再使用 `fib` 函数，它也会尽可能长的保存这些数据。其中一种方式是使用软引用，在需要空出内存时，可以由 GC 将其删除。它不应该与弱引用混淆，简单来说，它们的区别是：

* 弱引用不会阻止 GC 清除该值，因此，一旦没有其它引用（变量）指向它，该值将被清除
* 软引用不能保证值会不会被 GC 清除，但在大多数 JVM 的实现中，除非需要空出内存，否则这些值是不会清理的。在实现缓存时，软引用是完美的

下面这个例子中，使用属性委托（详细请看_第21条：使用属性委托提取公共属性模式_）创建一个 `map` 并让我们使用它。但这并不能阻止 GC 在需要内存时，对 `map` 进行回收（完整的实现应该包括线程同步）。

```kotlin
private val FIB_CACHE: MutableMap<Int, BigInteger> by
    SoftReferenceDelegate { mutableMapOf<Int, BigInteger>() }

fun fib(n: Int): BigInteger = FIB_CACHE.getOrPut(n) {
    if (n <= 1) BigInteger.ONE else fib(n - 1) + fib(n - 2)
}

class SoftReferenceDelegate<T: Any>(
    val initialization: ()->T
) {
    
    private var reference: SoftReference<T>? = null
    
    operator fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        val stored = reference?.get()
        if (stored != null) return stored
        val new = initialization()
        reference = SoftReference(new)
        return new
    }
}
```

设计好缓存并不容易，最终，缓存总是内存性能中的一个权衡。记住这一点，明智地使用缓存，没有人希望话题从性能不足转移到内存不足上。

### 将重的对象提到外部

对于性能来说，一个非常有用的技巧是将较重的对象提到外部作用域去。当然如果可以的话，我们应该将所有繁重的集合处理操作提到一个通用的处理范围域中。比如在下面这个函数中，我们需要数出 `Iterable` 中最大的元素有多少个：

```kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int =
    count { it == this.max() }
```

一个更好的解决方案是提取最大元素到 `countMax` 的函数级：

```kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int {
    val max = this.max()
    return count { it == max }
}
```

这种解决方案的性能更好，因为我们不需要在每次迭代中都要找一遍 `Iterable` 上的最大元素。请注意，它还提高了可读性，因为扩展接收者上调用 `max` 是可见的，因此在所有迭代中都是相同的。

提取一个值的计算到一个外部范围去，这是一个重要的实践。这听起来很简单，但实际并不总是那么清晰的。看看下面这个函数，我们使用 `regex` 来验证字符串是否包含有效的 IP 地址：

```
fun String.isValidIpAddress(): Boolean {
    return this.matches("\\A(?:(?:25[0-5]|2[0-4][0-9]
|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]
?[0-9][0-9]?)\\z".toRegex())
}

// Usage
print("5.173.80.254".isValidIpAddress()) // true
```

这个函数的问题是每次使用它时都要创建 `Regex` 对象。这是一个致命的缺陷，因为 `regex` 模式匹配是很复杂的操作。这就是这个函数不适合性能敏感的代码中重复使用的原因。尽管我们可以通过将 `regex` 提到外部来优化它:

```kotlin
private val IS_VALID_EMAIL_REGEX = "\\A(?:(?:25[0-5]
|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4]
[0-9]|[01]?[0-9][0-9]?)\\z".toRegex()

fun String.isValidIpAddress(): Boolean =
   matches(IS_VALID_EMAIL_REGEX)
```

如果这个函数和其他一些函数在一个文件中，并且我们不想在没有使用它的时候创建这个对象，我们甚至可以延迟初始化它：

```kotlin
private val IS_VALID_EMAIL_REGEX by lazy {
"\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.)
{3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()
}
```

在处理类时，将属性设置为惰性也很有用。

### 延迟初始化

通常，当我们需要创建一个很重的类时，最好选择惰性创建它。例如，假设 A 类需要初始化重量级类 B、C 和 D 的实例，如果我们只是在类创建过程中去创建它们，那么 A 类的创建将非常繁重，因为它需要同时创建 B、C 和 D 类，然后再是主体的其它部分，创建对象的开销只会增加。

```kotlin
class A {
val b = B()
val c = D()
val d = D()

//...
}
```

不过以一种解决方案，我们可以使用延迟加载来初始化那些重的对象：

```kotlin
class A {
val b by lazy { B() }
val c by lazy { C() }
val d by lazy { D() }

//...
}
```

每个对象在第一次使用之前才初始化，创建这些对象的成本将被分摊而不是累积。

记住，这是一把双刃剑。你可能会遇到这样的情况：对象的创建可能很繁重，但你需要方法尽可能的快。假设 A 是后端应用程序中响应 HTTP 请求的控制器，它启动很快，但第一次调用需要所有重对象初始化。因此，第一次调用需要等待响应的时间要长得多，并且它不关心我们客户端 App 等待多长时间。这不是理想的行为，同时也可能会干扰我们的性能测试。

### 使用原语

在 JVM 中，我们有一种特殊的内置类型来表示数字或字符等基本元素。它们被称为原语，并由 Kotlin/JVM 编译器在可以使用的情况下尽可能的使用。尽管某些情况下需要使用装箱类来代替，主要有两种情况：

1. 在可空类型上操作时（原语不能为空）
2. 当我们将类型用作泛型时

简而言之是这样的：

| Kotlin      | Java type       |
| ----------- | --------------- |
| `Int`       | `int`           |
| `Int?`      | `Integer`       |
| `List<Int>` | `List<Integer>` |

知道了这一点，你就可以优化代码了，就是使用原语而不是封装的类型。这种优化主要在 Kotlin/JVM 和一些种类的 Kotlin/Native 上有意义，在 Kotlin/JS 上完全没有。还需要注意的一点是，只有在对一个数字进行多次操作时才有意义。与其他操作相比，当我们处理真正大的集合时（我们将在_第51条：考虑在性能关键处使用原语数组_）或批量操作对象时，基本类型和封装类型的访问都相对非常快。还要记住，强制更改可能导致代码可读性降低。**这就是为什么我们建议只对代码和库中性能敏感的地方进行这种优化**。你可以使用 profiler 找出哪些地方是性能是敏感的。

举个例子，假设你为 Kotlin 实现了一个标准库，并且希望引入一个函数，该函数返回一个集合中的最大元素，如果 `iterable` 是空的，则返回 null。 你不希望 `iterable` 进行多次迭代，这不是一个简单的问题，但可以用下面函数来解决：

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    var max: Int? = null
    
    for (i in this) {
        max = if(i > (max ?: Int.MIN_VALUE)) i else max
    }
    return max
}
```

这个实现有严重的缺点：

1. 我们需要在每一步都使用一次 `Elvis` 操作符
2. 我们使用一个可空值，因此在 JVM 内部，将使用一个 `Integer` 而不是 `int`

解决这两个问题需要我们使用 while 循环来实现迭代：

```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
    val iterator = iterator()
    if (!iterator.hasNext()) return null
    
    var max: Int = iterator.next()
    while (iterator.hasNext()) {
        val e = iterator.next()
        if (max < e) max = e
    }
    return max
}
```

对于一个1到1000万个元素的集合，在我的计算机上，优化后的函数实现用了289ms，而没优化之前则是518ms，这几乎快了两倍。但是请记住，这只是展示了一个比较极端的情况。这种优化在性能不重要的代码中很少是合理的。但是，如果你实现了一个 Kotlin 标准库，那么一切性能相关的问题都是至关重要的。这就是为什么在这里，我将选择下面这种实现：

```kotlin
/**
* 返回最大的元素，如果没有则返回 null
*/
public fun <T : Comparable<T>> Iterable<T>.max(): T? {
    val iterator = iterator()
    if (!iterator.hasNext()) return null

    var max = iterator.next()
    while (iterator.hasNext()) {
        val e = iterator.next()
        if (max < e) max = e
    }
    return max
}
```

在本文中，你已经看到了避免创建多余对象的不同方法。就可读性而言，其中一些成本会很低：它们应该自由随意的使用。例如，将重的对象从循环或函数中提取出来通常是一个好的主意。这对于性能来说是一个很好的实践，也多亏了这种提取，我们可以命名这个对象，从而使函数更容易阅读。那些难度较大或需要较大更改的优化可能会被跳过。我们应该避免过早的优化，除非我们的项目有这样的指导方针，或者我们开发的库可能会以不知道的方式被使用。我们还学习了一些可以用于代码中性能关键部分的优化手段。
