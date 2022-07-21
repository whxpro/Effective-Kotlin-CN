# 第35条：考虑为复杂的对象创建定义 DSL

Kotlin 特性允许我们创建一个类似于特定领域语言（DSL）的配置。当我们需要定义更复杂的对象或对象的层次结构时，这种 DSL 非常有用。它们不容易定义，但是一旦定义了它们，就隐藏了样板代码和复杂性，开发人员还可以清楚地表达其意图。

例如，Kotlin DSL 是一个主流的表示 HTML 的方式：包括经典的 HTML 和 React HTML。 它看起来是这样的：

```kotlin
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            +"Main site"
        }
    }
    +"Some content"
}
```

![](<../../.gitbook/assets/image (7).png>)

其他平台上的视图也可以使用 DSL 来定义，下面是使用 Anko 库定义的一个简单的 Android 视图：

```kotlin
verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { toast("Hello, ${name.text}!") }
    }
}
```

![](<../../.gitbook/assets/image (14).png>)

桌面应用程序也是如此，以下是在 TornadoFX 中定义的一个视图，它是建立在 JavaFx 之上的：

```kotlin
class HelloWorld : View() {
    override val root = hbox {
        label("Hello world") {
            addClass(heading)
        }
        
        textfield {
            promptText = "Enter your name"
        }
    }
}
```

![](<../../.gitbook/assets/image (13) (1).png>)

DSL 还经常用于定义数据或配置。下面是 Ktor 中的API定义，也是一个 DSL：

```kotlin
fun Routing.api() {
    route("news") {
        get {
            val newsData = NewsUseCase.getAcceptedNews()
            call.respond(newsData)
        }
        get("propositions") {
            requireSecret()
            val newsData = NewsUseCase.getPropositions()
            call.respond(newsData)
        }
    }
    // ...
}
```

这里是在 Kotlin test 中定义的测试用例说明：

```kotlin
class MyTests : StringSpec({
    "length should return size of string" {
        "hello".length shouldBe 5
    }
    "startsWith should test for a prefix" {
        "world" should startWith("wor")
    }
})
```

我们甚至可以使用 Gradle DSL 来定义 Gradle 配置：

```kotlin
plugins {
    `java-library`
}

dependencies {
    api("junit:junit:4.12")
    implementation("junit:junit:4.12")
    testImplementation("junit:junit:4.12")
}

configurations {
    implementation {
        resolutionStrategy.failOnVersionConflict()
    }
}

sourceSets {
    main {
        java.srcDir("src/core/java")
    }
}

java {
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11
}
tasks {
    test {
        testLogging.showExceptions = true
    }
}
```

使用 DSL 可以使得创建复杂的分层数据结构变得更加容易。在这些 DSL 中，我们可以使用 Kotlin 提供的所有东西，并且我们有一些有用的提示，因为 Kotlin 中的 DSL 是完全类型安全的（不像 Groovy）。你可能已经使用了一些 Kotlin DSL，但是知道如何自己定义它们也很重要。

### 打造你专属的 DSL

要理解如何创建自己的 DSL，理解带有接收者（指针）的函数类型的概念是很重要的。但在此之前，我们将首先回顾函数类型本身的概念。函数类型是一种表示可作为函数使用的对象的类型。例如，在 `filter` 函数中，它表示一个谓词，决定是否可以接受一个元素。

```kotlin
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val list = arrayListOf<T>()
    for (elem in this) {
        if (predicate(elem)) {
            list.add(elem)
        }
    }
    return list
}
```

下面是一些函数类型的例子：

* `() -> Unit` —— 不带参数并返回 Unit 的函数
* `(Int) -> Unit` —— 接收整型并返回 Unit 的函数
* `(Int) -> Int` —— 接收整型并返回整型的函数
* `(Int, Int) -> Int` —— 接收两个整型参数，并返回整型的函数
* `(Int) -> () -> Unit` —— 接收整型，并返回另一个函数的函数， 这个函数没有参数，并返回 Unit
* `(() -> Unit) -> Unit` —— 接收另一个函数并返回 Unit 的函数。 这个函数没有参数，并返回 Unit

创建函数类型的示例方法如下：

* 使用 lambda 表达式
* 使用匿名函数
* 使用函数引用

例如，思考下面的函数：

```kotlin
fun plus(a: Int, b: Int) = a + b
```

通过类比，该函数还可以通过以下方式创建：

```kotlin
val plus1: (Int, Int)->Int = { a, b -> a + b }
val plus2: (Int, Int)->Int = fun(a, b) = a + b
val plus3: (Int, Int)->Int = ::plus
```

在上面的例子中，属性类型被指定，因此 lambda 表达式和匿名函数中的参数类型可以被推断。也可能是反过来的，如果指定参数类型，则可以推断函数类型。

```kotlin
val plus4 = { a: Int, b: Int -> a + b }
val plus5 = fun(a: Int, b: Int) = a + b
```

函数类型用来表示函数的对象，匿名函数看起来像是普通函数一样，只是没有名字，lamdba 表达式是匿名函数的一种更简短的表示方法。

如果我们有函数类型来表示函数，那么扩展函数呢？ 我们也能表示它吗？

```kotlin
fun Int.myPlus(other: Int) = this + other
```

前面提到过，我们以与普通函数相同的方式创建匿名函数，但是没有名称，因此匿名扩展函数的定义也是相同的：

```kotlin
val myPlus = fun Int.(other: Int) = this + other
```

myPlus 是什么类型的？ 答案是这是一种特殊的类型，用来表示扩展函数，它被称为\_带有接收者的函数类型\_。它看起来类似于普通的函数类型，但它在参数之前额外指定了接收方类型，它们之间用点来分割：

```kotlin
val myPlus: Int.(Int)->Int = fun Int.(other: Int) = this + other
```

这样的函数可以使用 lambda 表达式定义，特别是带有接收者的 lambda 表达式，因为在其作用域内 this 关键字引用的正是被扩展的接收者（在本例中是 Int 类型的示例）：

```kotlin
val myPlus: Int.(Int)->Int = { this + it }
```

使用匿名扩展函数或 lambda 表达式与接收者一起创建对象可以用3种方式调用：

* 像一个标准的对象，使用 `invoke` 方法调用
* 类似于非扩展函数
* 与普通的扩展函数相同

```kotlin
myPlus.invoke(1, 2)
myPlus(1, 2)
1.myPlus(2)
```

带有接收者的函数类型最重要的的特征是：它改变了 `this` 的含义，例如，在 `apply` 函数中使用它可以更容易地引用接收者对象的方法和属性：

```kotlin
inline fun <T> T.apply(block: T.() -> Unit): T {
    this.block()
    return this
}

class User {
    var name: String = ""
    var surname: String = ""
}

val user = User().apply {
    name = "Marcin"
    surname = "Moskała"
}
```

带有接收者的函数类型是 Kotlin DSL 中最基本的构建块，让我们创建一个非常简单的 DSL， 它允许我们创建下面的 HTML 表：

```kotlin
fun createTable(): TableDsl = table {
    tr {
        for (i in 1..2) {
            td {
                +"This is column $i"
            }
        }
    }
}
```

从 DSL 的开头开始，我们可以看到一个函数 `table`，我们处于顶层，没有任何接收器，所以它需要是一个顶级函数，尽管在它的函数参数中，可以看到我们使用 `tr`， `tr` 函数应该只允许在 `table` 定义中使用，这就是为什么 `table` 函数参数应该要带一个接收者。类似的， `tr` 函数参数需要一个包含 `td` 函数的接收者。

```kotlin
fun table(init: TableBuilder.()->Unit): TableBuilder {
    //...
}

class TableBuilder {
    fun tr(init: TrBuilder.() -> Unit) { /*...*/ }
}

class TrBuilder {
    fun td(init: TdBuilder.()->Unit) { /*...*/ }
}
class TdBuilder
```

那么如何处理这个代码呢：

```kotlin
+"This is row $i"
```

这是什么？ 这不过只是 String 上的 `plus` 操作符，它需要在 `TdBuilder` 中进行定义：

```kotlin
class TdBuilder {
    var text = ""

    operator fun String.unaryPlus() {
        text += this
    }
}
```

现在我们的 DSL 已经定义好了，为了使其正常工作，在每一步中，我们需要创建一个构建器，并使用一个来自参数的函数（下面示例中的 init）对其进行初始化，之后，构造器将包含 `init` 函数中指定的所有数据。这就是我们需要的数据，因此，我们可以返回该构建器，也可以生成另一个保存该数据的对象，在本例中，我们将只返回 builder。 下面是 table 函数的定义方式：

```kotlin
fun table(init: TableBuilder.()->Unit): TableBuilder {
    val tableBuilder = TableBuilder()
    init.invoke(tableBuilder)
    return tableBuilder
}
```

注意，我们可以使用 apply 函数，如下所示，来缩短这个函数：

```kotlin
fun table(init: TableBuilder.()->Unit) =
    TableBuilder().apply(init)
```

类似的，我们可以在 DSL 的其他部分使用它来更简洁：

```kotlin
class TableBuilder {
    var trs = listOf<TrBuilder>()
    
    fun tr(init: TrBuilder.()->Unit) {
        trs = trs + TrBuilder().apply(init)
    }
}

class TrBuilder {
    var tds = listOf<TdBuilder>()
   
    fun td(init: TdBuilder.()->Unit) {
    tds = tds + TdBuilder().apply(init)
    }
}
```

这是一个用于创建 HTML 表的全功能 DSL 构建器。可以使用\_第15条：考虑显式引用接收者\_中的阐述的 `DslMakrer` 来改进。

### 我们什么时候使用它？

DSL 为我们提供了一个定义信息的方法。它可以用来表示你想要的任何类型的信息，但是用户永远不会清楚这些信息以后将如何使用。在 Anko、TornadoFX 或 HTML DSL 中，我们相信视图会根据我们的定义正确地构建，但通常很难追踪到底是如何构建的，一些更复杂的用途可能很难发现，用法也会让不习惯的人感到困惑，更不要说维护了。它们的定义方式可能是一种成本 —— 对开发人员的困惑上和性能方面上。当我们可以使用其他更简单的特性时， DSL 就显得多余了。但是它在下面的场景中会非常有用：

* 复杂的数据结构
* 层次结构
* 海量数据

任何东西都可以在只使用构建器或构造器而不使用类似 DSL 的结构下表达出来。 **DSL 的作用是消除这类的样板结构**，当你看到重复的样板代码并且没有更简单的 Kotlin 特性可以提供帮助时，你应该考虑使用 DSL。

### 总结

DSL 是语言中的一种特殊语言，它可以非常简单地创建复杂的对象，甚至整个对象层次结构，如 HTML 代码或复杂的配置文件。另一方面，DSL实现可能会让新开发人员感到困惑。它们也很难定义，这就是为什么只有当它们提供真正价值时才应该使用它们。例如，用于创建一个非常复杂的对象，或者可能用于复杂的对象层次结构。这就是为什么最好在库中而不是在项目中定义它们。制作一个好的 DSL 并不容易，但是一个定义良好的 DSL 可以使我们项目更好。
