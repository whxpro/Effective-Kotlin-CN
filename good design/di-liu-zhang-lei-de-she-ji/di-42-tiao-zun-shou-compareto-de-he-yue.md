# 第42条：遵守 compareTo 的合约

`compareTo` 方法不在 `Any` 类中。它是 `Kotlin` 的一个运算符，可以换成数学中不等式的符号：

```kotlin
obj1 > obj2 // Translates to obj1.compareTo(obj2) > 0
obj1 < obj2 // Translates to obj1.compareTo(obj2) < 0
obj1 >= obj2 // Translates to obj1.compareTo(obj2) >= 0
obj1 <= obj2 // Translates to obj1.compareTo(obj2) <= 0
```

它也位于 `Compareable<T>` 接口中。当对象实现此接口，或者实现具有名为 `compareTo` 的操作符函数时，这意味着对象是有顺序的。具备顺序性需要遵循下面的规则：

* 反对称性， 即如果 a >= b 且 b >= a，则 a == b。 因此比较与相等性是有关系的，它们需要相互一致
* 传递性， 即如果 a >= b 且 b >= c，则 a >= c。 类似的，如果 a > b 且 b > c，则 a > c。这个特性很重要，因为如果没有了它，元素的排序可能会花费很长的时间
* 连通性，意味着任意两个元素间必定有一个关系。所以要么是 a >= b，要么是 b >= a 。在 Kotlin 中， `compareTo` 的类型系统保证了这一点，因为它返回 Int 值，每个 Int 要么是正的，要么是负的，要么是0。这很重要，因为如果两个元素之间没有联系，就不能使用快速排序或插入排序等经典算法。相反，我们需要使用一种特殊的算法来处理偏序，比如拓扑排序。

### 我们需要 `compareTo` 吗？

在 Kotlin 中，我们很少实现 `compareTo`。我们可以根据具体情况来指定顺序，而不是指定死整个的排序规则。例如，我们可以使用 `sortedBy` 对集合进行排序，并提供一个可以比较的键。所以在下面的例子中，我们按照姓氏对用户进行排序：

```kotlin
class User(val name: String, val surname: String)
val names = listOf<User>(/*...*/)

val sorted = names.sortedBy { it.surname }
```

如果我们需要一个比对比键更加复杂的比较呢？为此，可以使用 `sortedWith` 函数，该函数使用比较器对元素进行排序。这个比较器可以用 `compareBy` 来生成。在下面的例子中，我们首先按姓氏对用户进行排序，对于姓氏相同的，则按照名字来比较：

```kotlin
val sorted = names
    .sortedWith(compareBy({ it.surname }, { it.name }))
```

当然，我们可以让 `User` 实现 `Comparable<User>`，但它应该选择什么样的顺序呢？ 我们可能根据属性对它们进行排序，但当这一点不完全清楚时，最好不要让这些对象具有可比性。

`String` 有一个自然的顺序，即字母和数字的顺序，因此它是实现了 `Compare<String>`。这个事实非常有用，因为我们经常需要按字母数字顺序对文本进行排序。然而，它也有缺点，例如，我们可以使用不等号比较两个字符串，这看起来非常不直观。大多数人看到两个字符串比较时使用不等号会感到困惑。

```kotlin
// 不要这样做！
print("Kotlin" > "Java") // true
```

当然，有些对象是具有清晰的自然秩序。度量单位、日期、时间都是很好的例子。如果你不确定你的对象是否具有自然顺序，最好使用比较器。如果你经常使用它们中的一些，你可以把它们放在你的类的伴生对象中：

```kotlin
class User(val name: String, val surname: String) {
    // ...
    
    companion object {
        val DISPLAY_ORDER =
            compareBy(User::surname, User::name)
    }
}

val sorted = names.sortedWith(User.DISPLAY_ORDER)
```

### 实现 compareTo

当我们需要实现 `compareTo` 时，有顶级函数可以帮助我们。如果你只需要比较两个值，你可以使用 `compareValues` 函数：

```kotlin
class User(
    val name: String,
    val surname: String
): Comparable<User> {
    override fun compareTo(other: User): Int =
        compareValues(surname, other.surname)
}
```

如果你需要使用更多的值比较，或者你需要使用选择器来比较他们，你可以使用 `compareValuesBy`：

```kotlin
class User(
    val name: String,
    val surname: String
): Comparable<User> {
    override fun compareTo(other: User): Int =
        compareValuesBy(this, other, { it.surname }, { it.name })
}
```

这个函数帮助我们创建绝大多数能满足需求的比较器，如果你需要实现一些特殊的逻辑，记住它们应该返回：

* 如果接收方和另一方相等，返回 0
* 如果接收方大于另一方，返回正数
* 如果接收方小于另一方，返回负数

一旦你这样做了，不要忘了验证你的比较是反对称的、传递的和具有连接性的。
