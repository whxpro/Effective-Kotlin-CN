# 第15条：考虑显式引用接收者

有这么一个常见的场景：当我们想要凸显出一个函数或者属性是从某个接收者（指针）引出的，可能会选择一个较长的结构来显式表达这些意图，而不是使用局部或全局变量。在绝大多数情况下，这意味去引用该方法关联的类：

```kotlin
class User: Person() {
    private var beersDrunk: Int = 0

    fun drinkBeers(num: Int) {
        // ...
        this.beersDrunk += num  // 显式引用 beersDrunk 相关联的类，this 指代的就是 User，也就是接收者
        // ...
    }
}
```

同样，我们可以显式地引用扩展接收者（在扩展方法当中）让它其更加凸显。比较一下没有用接收者的快速排序实现：

```kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
    if (size < 2) return this
    val pivot = first()
    val (smaller, bigger) = drop(1)
        .partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

和使用了接收者的快速排序实现：

```kotlin
fun <T : Comparable<T>> List<T>.quickSort(): List<T> {
    if (this.size < 2) return this
    val pivot = this.first()
    val (smaller, bigger) = this.drop(1)
        .partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

这两个函数的作用是一样的：

```kotlin
listOf(3, 2, 5, 1, 6).quickSort() // [1, 2, 3, 5, 6]
listOf("C", "D", "A", "B").quickSort() // [A, B, C, D
```

### 多个接收者

当我们在多个接收者的作用域内时，显式使用接收者非常有帮助。 当使用 `apply`、`with`、`run` 函数时，就经常会遇到这种情况，这种情况相当危险，是我们应该避免的。而显式声明接收者，可以在使用对象时更加安全。要理解这个问题，请看如下代码：

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) = 
        create("$name.$childName")
            .apply { print("Created ${name}") }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child")
}
```

打印的结果是什么呢？现在停下来，花点时间来想想看这个问题。

你可能会想到的结果是 “Created parent.child”， 但实际上结果是：“Created parent”， 为什么呢？ 为了追究原因，可以在 `name` 之前显式声明接收者：

```kotlin
class Node(val name: String) {

    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created ${this.name}") } // Compilation error

    fun create(name: String): Node? = Node(name)
}
```

这里出现了问题： `apply` 里面应用的类型是 `Node?`，所以 `getName()` 不能被直接调用，我们需要对其解包，例如使用空安全调用，结果最终是 “Created parent.child”：

```kotlin
class Node(val name: String) {

    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created ${this?.name}") }

    fun create(name: String): Node? = Node(name)
}
```

当接收者不清晰时，我们要么避免它，要么就用显式引用接收者来展示它。当使用不带标签的接收者时，就表示我们想使用的是最近作用域的那个接收者。但当我们想使用外部接收者时，就需要使用标签，这种情况下，显式使用它尤其有用，下面的例子展示了两者的用法：

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) = 
        create("$name.$childName").apply { 
            print("Created ${this?.name} in " +
                 " ${this@Node.name}") 
 }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent")
    node.makeChild("child")
}
// Created parent.child in parent
```

这样的使用接收者就明确了我们要表达的意思。这可能是一个重要信息，不仅可以保护我们免受错误的困扰，还可以提高代码可读性。

### DSL marker

有这么一个上下文环境，能让我们可以在不同的嵌套作用域中使用不同的接收者进行操作，并且根本不需要显式使用接收者。它就是 Kotlin 的 DSL，不需要显式的使用接收者，因为 DSL 就是以这种方式设计的。然而，在 DSL 中，意外地使用外部作用域的函数是特别危险的，想象一个简单 HTML DSL，用它来创建一个 HTML 表：

```
table {
   tr {
       td { +"Column 1" }
       td { +"Column 2" }
   }
   tr {
       td { +"Value 1" }
       td { +"Value 2" }
   }
}
```

注意在默认情况下，每个作用域中都是允许使用来自外部作用域的接收者方法，我们可能就会因为这一机制而搅乱了 DSL：

```kotlin
table {
    tr {
        td { +"Column 1" }
        td { +"Column 2" }
        tr {  // 这里实际上调用了 table.tr 方法, 而 table 作为接收者在这个地方是隐式的
            td { +"Value 1" }
            td { +"Value 2" }
        }
    }
}
```

为了限制这种用法， 我们有一个特殊的元注解（用于注解的注解），它限制了隐式使用外部接收者，它就是 `@DslMarker`，当我们在一个注解上使用它，然后在作为 DSL 构建器的类上使用这个注解时，就无法在这个构建器中使用隐式接收者了。下面是一个如何使用 `@DslMarker` 的示例：

```kotlin
@DslMaker
annotation class HtmlDsl

fun table(f: TableDsl.() -> Unit) { /**..**/ }

@HtmlDsl
class TableDsl { /**..**/ }
```

有了它，就可以禁止使用外部接收者了：

```kotlin
table {
    tr {
        td { +"Column 1" }
        td { +"Column 2" }
        tr { // Compilation error
            td { +"Value 1" }
            td { +"Value 2" }
        }
    }
}
```

需要使用外部接收者的函数，需要显式的引用接收者：

```kotlin
table {
    tr {
        td { +"Column 1" }
        td { +"Column 2" }
        this@table.tr {
            td { +"Value 1" }
            td { +"Value 2" }
        }
    }
}
```

DSL 标记是一个非常重要的机制，我们可以使用它来强制使用最近的接收者，或者显式使用外部接收者。然而，无论如何，最好不要在 dsl 中使用显式接收者，尊重 DSL 的设计并规范地使用它。

### 总结

不要仅仅因为你可以指定接收者就去随意使用它。 如果有太多接收者都给我们可以使用的方法，随意使用反而可能会让代码混乱，让阅读代码的人感到困惑。显式引用通常更好，当我们想要改变接收者时，使用显式引用的方式可以提高代码可读性，因为它标明了函数的来源。当有多个接收者时，我们甚至可以用标签来说明函数是具体来自哪一个。 如果你希望：在使用外部的接收者时要强制的显式声明它，可以使用 `@DslMarker` 注解。
