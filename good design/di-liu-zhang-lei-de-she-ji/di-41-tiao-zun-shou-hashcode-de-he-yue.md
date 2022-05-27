# 第41条：遵守 hashCode 的合约

`Any` 中另一个可以重写的方法是 `hashCode`。让我们首先解释一下为什么需要它。 `hashCode` 函数用于一种流行的数据结构 —— 哈希表。它在底层被用于各种不同的集合和算法中。

### 哈希表

让我们从哈希表被发明来解决的问题开始。假设我们需要一个能快速添加和查找元素的集合，这些集合比如 `set` 或 `map`，都是不允许重复的，所以每当我们添加一个元素时，我们就遍历集合寻找是否有相同的元素。

基于数组或链表的集合，在检索是否包含某个元素的速度还不够快，因为要进行检查，就必须要将该元素和集合中的每个元素逐一进行比较。假设你有一个包含百万条文本的数组，现在你需要检查它是否包含某个特定的文本，将你的文本与数以百万计的文本一一比较是非常耗时的。

这个问题的一个主流解决方案是哈希表，你所需要的是一个函数，它为每个元素都分配一个数字。这样的函数被称为哈希函数，对于相同的元素，它必须始终返回相同的值。另外，如果我们的哈希函数具备下面的特点，那将会很nice：

* 很快的
* 理想情况下，对于不相同的元素返回不同的值，或者至少有足够的变化将冲突限制到最小

该函数通过为每个元素分配一个数字，而分到不同的桶（bucket）中。更重要的是，根据我们对哈希函数的要求，所有彼此相等的元素将始终放在同一个桶中。这些存储单元保存在一个名为哈希表（hash table）的结构中，哈希表是一个大小等于存储单元数量的数组。每次我们添加一个元素，我们使用哈希函数来计算它应该放在哪里，然后把它添加到那里。注意，这个过程会非常快，因为计算哈希值一般都是很快的，然后我们只用将哈希函数的结果（哈希码）作为数组中的索引来查找桶。当我们搜索一个元素时，我们用同样的方法找到它所在的存储桶，然后我们只需检查它是否等于这个存储桶中的任何元素。我们无需检查其他桶，因为哈希函数必须对相等的元素返回相同的值。这种方式，以一种很低的成本查找元素，它所需的平均成本是容器元素总量除以桶的数量。例如我们有 1,000,000个元素和1,000个桶，那么搜索一个副本平均只需要比较 1,000 个元素，并且达到这种性能改进所使用的成本非常小。

为了呈现更直观的例子，假设我们有以下字符串和一个分成4个桶的哈希函数：

| 文本                                      | 通过哈希算法得到的哈希码 |
| --------------------------------------- | ------------ |
| "How much wood would a woodchuck chuck" | 3            |
| "Perter Piper picked a peck of pickled" | 2            |
| "Betty bough a bit of butter"           | 1            |
| "She sells seashells by the seashore"   | 2            |

基于这些数字，我们将构建以下哈希表：

| 索引 | 每个哈希桶上存放的文本                                                                       |
| -- | --------------------------------------------------------------------------------- |
| 0  | \[]                                                                               |
| 1  | \["Betty bough a bit of butter"]                                                  |
| 2  | \["Perter Piper picked a peck of pickled", "She sells seashells by the seashore"] |
| 3  | \["How much wood would a woodchuck chuck"]                                        |

现在，当我们检查一个新的文本是否在这个哈希表中，我们先计算它的哈希代码。如果它等于0，那么我们知道它不在这个表上，如果是 1 或 3，则需要将其与单个文本进行比较，如果是2，则需要将其与两个文本进行比较。

这个概念在技术领域非常流行。它被用于数据库、许多互联网协议以及许多语言的标准库集合中。在 Kotlin/JVM 中，默认集合（`LinkedHashSet`）和默认映射（`LinkedHashMap`）都使用它。为了生成一个哈希码，我们使用 `hashCode` 函数。

### 可变性的问题

注意，只有在添加元素时才会为元素计算哈希。元素发生变化时，并不会改变其位置。这就是 `LinkedHashSet` 和 `LinkedHashMap` 中一个对象被添加后如果发生了改变，这个集合就不能正常工作的原因。

```kotlin
data class FullName(
    var name: String,
    var surname: String
)

val person = FullName("Maja", "Markiewicz")
val s = mutableSetOf<FullName>()
s.add(person)
person.surname = "Moskała"
print(person) // FullName(name=Maja, surname=Moskała)
print(person in s) // false
print(s.first() == person) // true
```

这个问题已经在_第1条：限制可变性_中提到了：可变对象不能用于基于散列（哈希）的数据结构或可变属性组织的任何其他数据结构中。不应该将可变元素作为 `set` 的元素，或者 `map` 的键，或者至少不应该改变这些集合中的元素。这也是通常使用不可变对象的一个重要原因。

### hashCode 的合约

若想要知道我们需要 `hashCode` 来干嘛，就应该弄清楚它期望被拿来做什么。其正式合约如下（Kotlin 1.3.11）：

* 只要 `hashCode` 方法在同一个对象上被多次调用，它必须一致地返回相同的整数，前提是在对象上的 `equals` 函数中使用的信息没有被修改
* 如果两个对象使用 `equals` 比较相同，那么这两个对象调用 `hashCode` 产生的整数结果必须相同

请注意，第一个要求是我们需要 `hashCode` 保持一致，第二个是开发人员常常忘记的，需要重点知道： `hashCode` 总是要和 `equals` 保持一致，相同元素必须具有相同的哈希码。否则，元素将会丢失在那些使用哈希表的集合中：

```kotlin
class FullName(
    var name: String,
    var surname: String
) {
    override fun equals(other: Any?): Boolean =
        other is FullName
            && other.name == name 
            && other.surname == surname
}

val s = mutableSetOf<FullName>()
s.add(FullName("Marcin", "Moskała"))
val p = FullName("Marcin", "Moskała")
print(p in s) // false
print(p == s.first()) // true
```

这就是为什么 Kotlin 建议你当你自定义 `equals` 实现时应该重写 `hashCode`：

![](<../../.gitbook/assets/image (13).png>)

还有一个非必需的要求，但我们想让这个函数有用，这个要求非常重要。 `hashCode` 应该尽可能的分散元素，不同的元素尽可能有不同的哈希值。

Think about what happens when many different elements are placed in the same bucket - there is 想象一下，当许多不同的元素被放置在一个存储桶中时，会发生什么 —— 使用哈希表没有任何优势了！ 一个极端的例子是让 `hashCode` 总是返回相同的数字。 这样的函数总是将元素放入同一个桶中。它确实履行了合约，但是毫无用处，当 `hashCode` 总是返回相同的值时，使用哈希表没有任何好处。看看下面示例，你可以看到一个正确实现的 `hashCode`，以及一个总是返回 0 的 `hashCode`。对于每一个 `equals` 函数，我们添加了一个计数器来计算它被使用了多少次。你可以看到，当我们对这两种类型的值进行操作时，第二种类型 `Terrible` 将需要更多次比较：

```kotlin
class Proper(val name: String) {
    
    override fun equals(other: Any?): Boolean {
        equalsCounter++
        return other is Proper && name == other.name
    }
    
    override fun hashCode(): Int {
        return name.hashCode()
    }
    
    companion object {
        var equalsCounter = 0
    }
}

class Terrible(val name: String) {
    
    override fun equals(other: Any?): Boolean {
        equalsCounter++
        return other is Terrible && name == other.name
    }
    
    // 可怕的实现，千万不要这么做
    override fun hashCode() = 0
    
    companion object {
        var equalsCounter = 0
    }
}

val properSet = List(10000) { Proper("$it") }.toSet()
println(Proper.equalsCounter) // 0
val terribleSet = List(10000) { Terrible("$it") }.toSet()
println(Terrible.equalsCounter) // 50116683

Proper.equalsCounter = 0
println(Proper("9999") in properSet) // true
println(Proper.equalsCounter) // 1

Proper.equalsCounter = 0
println(Proper("A") in properSet) // false
println(Proper.equalsCounter) // 0

Terrible.equalsCounter = 0
println(Terrible("9999") in terribleSet) // true
println(Terrible.equalsCounter) // 4324

Terrible.equalsCounter = 0
println(Terrible("A") in terribleSet) // false
println(Terrible.equalsCounter) // 10001
```

### hashCode 的实现

在 Kotlin 中，实际上只有在自定义 `equals` 时才需要定义 `hashCode`。当我们使用 `data` 修饰符时，它生成了 `equals` 和一个一致的 `hashCode`。 当你没有自定义 `equals` 方法时，不要自定义 `hashCode`，除非你确信自己知道在做什么，并且有很好的理由。当你有一个自定义 `equals` 时，实现 `hashCode`，它总是为相同的元素返回相同的值。

如果你实现了检查重要属性是否相等的典型 `equals` 方法，那么应该使用这些属性的哈希码计算典型的 `hashCode`。我们如何从这么多的哈希码中得到一个哈希码呢？ 一种典型的方法是，我们把它们都累加成一个结果，每次我们把下一个加进来，就把结果乘以 31。它不需要恰好是 31，但是它的特性使它成为一个很好的数字。它经常以这种方法使用，现在我们可以把它当成一种惯例。有 `data` 修饰符生成的哈希码与此约定是一致的。下面是一个典型的 `hashCode` 及其 `equals` 的实现：

```kotlin
class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
    
    override fun equals(other: Any?): Boolean =
        other is DateTime 
            && other.millis == millis 
            && other.timeZone == timeZone

    override fun hashCode(): Int {
        var result = millis.hashCode()
        result = result * 31 + timeZone.hashCode()
        return result
    }
}
```

Kotlin/JVM 上一个有用的函数是 `Objects.hashCode`，它帮助我们计算哈希值：

```kotlin
override fun hashCode(): Int =
    Objects.hash(timeZone, millis)
```

在 Kotlin 标准库中没有这样的函数，但是如果你在其他平台上需要它，你可以自己实现它：

```kotlin
override fun hashCode(): Int =
     hashCodeFrom(timeZone, millis)

inline fun hashCodeOf(vararg values: Any?) =
    values.fold(0) { acc, value ->
        (acc * 31) + value.hashCode()
    }
```

这样的函数不在标准库中的原因是我们很少需要自己实现 `hashCode`。例如，在上面给出的 `DateTime` 类中，我们可以使用 `data` 修饰符，而不是自己实现 `equals` 和 `hashCode` ：

```kotlin
data class DateTime2(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
}
```

当你实现 `hashCode` 时，请记住最重要的规则是它总是需要与 `equals` 保持一致，并且对于相等的元素，它应该总是返回相同的值。
