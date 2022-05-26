# 第33条：考虑使用工厂方法代替构造函数

在 Kotlin 中，类允许客户端获取实例的最常见方式是：提供一个主构造函数：

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
)

val list = MyLinkedList(1, MyLinkedList(2, null))
```

构造函数不是创建对象的唯一方法，有很多用于对象实例化的创建者设计模式，它们中的大多数都围绕着这样的思想：“函数可以为我们创建对象，而不用我们直接去创建对象”。例如，下面的顶级函数创建了 `MyLinkedList` 的实例：

```kotlin
fun <T> myLinkedListOf(
    vararg elements: T
): MyLinkedList<T>? {
    if(elements.isEmpty()) return null
    val head = elements.first()
    val elementsTail = elements
        .copyOfRange(1, elements.size)
    val tail = myLinkedListOf(*elementsTail)
    return MyLinkedList(head, tail)
}

val list = myLinkedListOf(1, 2)
```

作为构造函数的替代函数被称为“工厂函数”，因为它们产生一个对象，使用工厂函数而不是构造函数有很多优点，包括：

* **与构造函数不同，函数有名称**。名称解释了对象是如何创建的，以及需要的参数是什么。例如，假设你看到下面的代码： `ArrayList(3)`，你能猜出里面的3是什么意思吗？它应该是新创建的列表的第一个元素呢？还是这个列表的大小呢？这肯定不是不言自明的，在这种情况下，像 `ArrayList.withSize(3)` 这样的名称可以消除任何困惑。名称非常有用，它们解释了参数或对象创建的特征。使用名称的另外一个原因是：它解决了具有相同参数类型的构造函数之间的冲突
* **与构造函数不同，函数可以返回其类型的任意子类型的对象**。这可以用来为不同的情况提供更好的对象，尤其是我们想要将实际的对象隐藏在接口后面时，你可以参考来自标准库的 `listOf` 函数。它声明返回的是一个 `List`，这是一个接口，它真正返回的是什么呢？答案取决于我们使用的平台， 对于 Kotlin/JVM、Kotlin/JS 和 Kotlin/Native 来说都是不同的，因为它们使用不同的原生集合。这是 Kotlin 团队做的一个重要优化，他还为 Kotlin 的创建者提供了更多的自由，因为列表的实际类型可能会随时间改变而改变，但只要新对象仍然实现了接口 `List` 并以相同的方式工作，一切都是正常的
* **与构造函数不同，函数不需要在每次调用时创建一个新对象**。它很有帮助，在我们使用函数创建对象时，我们可以包装一个缓存机制来优化对象的创建，或确保在某些情况下（单例模式）让对象复用。还可以定义一个静态工厂函数，如果无法创建对象，该函数将返回 null，例如 `Connection.createOrNull()`：当由于某些原因不能创建连接时，返回 null
* **工厂函数可以提供可能还不存在的对象**，这个机制被那些注解处理器库的创建者大量使用，这样，程序员可以使用通过代理生成的对象进行操作，而无需构建项目。（例如 ServiceLoader）
* **当我们在对象外部定义工厂方法时，可以控制它的可见性**。例如，我们可以使用顶级工厂函数，只能被同一文件或同一模块的内容访问
* **工厂函数可以内联（inline），因为它们的类型参数可以具体化（reified）**
* **工厂函数可以简化那些复杂的构造函数**
* **构造函数需要立即调用超类的构造函数或主构造函数**，而使用工厂函数时，可以延迟构造函数的调用：

```kotlin
fun makeListView(config: Config) : ListView {
    val items = … // 我们在这里读 config 的内容到items中
    return ListView(items) // 再调用实际的构造函数
}
```

工厂函数的使用有一个限制：不能在子类构造函数中使用，这是因为在子类构造函数中，需要调用超类构造函数：

```kotlin
class IntLinkedList: MyLinkedList<Int>() {
// 假设 MyLinkedList 是 open的
    
    constructor(vararg ints: Int): myLinkedListOf(*ints)
// Error
}
```

这通常都不是问题，因为如果我们决定要使用工厂函数创建超类，为什么要为它的子类构造函数中使用呢？我们更应该考虑为这样的类实现工厂函数：

```kotlin
class MyLinkedIntList(head: Int, tail: MyLinkedIntList?): MyLinkedList<Int>(head, tail)

fun myLinkedIntListOf(vararg elements: Int): MyLinkedIntList? {
    if(elements.isEmpty()) return null
    val head = elements.first()
    val elementsTail = elements
        .copyOfRange(1, elements.size)
    val tail = myLinkedIntListOf(*elementsTail)
    return MyLinkedIntList(head, tail)
}
```

上面的函数比前面的构造函数更长，但它有更好的特征 —— 灵活性、类的独立性以及声明可空返回类型的能力。

工厂方法的功能背后有更强大的理由支撑，你需要理解的是，**它们并不与主构造函数所竞争**，工厂函数仍然需要在其函数体中使用构造函数，因此构造函数必须存在。如果我们真的想强制使用工厂函数创建，它可以是私有的，但我们很少这样做（_第34条：考虑一个带命名可选参数的主构造函数_）。**工厂函数主要与二级构造函数竞争**，在 Kotlin 项目中，它们通常会胜出，二级构造函数很少使用。因为工厂函数的功能多种多样，所以对它们来说是一种竞争。让我们讨论不同的 Kotlin 工厂函数：

1. 伴生对象工厂函数
2. 扩展工厂函数
3. 顶级工厂函数
4. 伪构造函数
5. 工厂类上的函数

### 伴生对象工厂函数

定义工厂函数的一个主流方法是在一个伴生对象中定义它：

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
) {
    
    companion object {
        fun <T> of(vararg elements: T): MyLinkedList<T>? {
            /*...*/
        }
    }
}

// Usage
val list = MyLinkedList.of(1, 2)
```

Java 开发人员应该非常熟悉这种方法，因为它直接等同于静态工厂方法，而且其它语言的开发人员可能也熟悉它。在一些语言，如 C++ 中，它被称为“命名的构造函数法”，因为它的用法类似于构造函数，但是拥有了一个名称。

在 Kotlin 中，这种方法也适用于接口：

```kotlin
class MyLinkedList<T>(
    val head: T,
    val tail: MyLinkedList<T>?
): MyList<T> {
// ...
}

interface MyList<T> {
    // ...
    
    companion object {
        fun <T> of(vararg elements: T): MyList<T>? {
            // ...
        }
    }
}

// Usage
val list = MyList.of(1, 2)
```

请注意：上面的函数的名称并不是具有描述性的，但是大多数开发人员应该可以理解它。原因是它来自一些 Java 的约定，多亏了这些约定，像 `of` 这样的简短词语就能足以让人理解参数的含义。以下是一些常见的名称及其解释：

* `from` —— 这是一个类型转换的函数，用于接受一个参数并返回一个相应的实例，例如：

```kotlin
val date: Date = Date.from(instant)
```

* `of` —— 这是一个聚合函数，用于接受多个参数，并返回一个合并了这些参数的对应类型的实例。例如：

```kotlin
val faceCards: Set<Rank> = EnumSet.of(JACK, QUEEN, KING)
```

* `valueOf` —— 一个比 `from` 和 `of` 更加冗长的选项，例如

```kotlin
val prime: BigInteger = BigInteger.valueOf(Integer.MAX_VALUE)
```

* `instance`、`getInstance` —— 在单例模式中用于获取唯一的实例。当参数化时，将返回一个由入参参数化的实例。通常当实参相同时，返回的实例总是相同的，例如：

```kotlin
val luke: StackWalker = StackWalker.getInstance(options)
```

* `createInstance`、`newInstance` —— 类似于 `getInstance`，但该函数保证每个调用返回一个新实例，例如：

```kotlin
val newArray = Array.newInstance(classObject, arrayLen)
```

* `get<Type>`、 —— 类似于 `getInstance`，但用于工厂函数在不同类中的情况。 `Type` 是工厂函数返回的对象类型，例如：

```kotlin
val fs: FileStore = Files.getFileStore(path)
val br: BufferedReader = Files.newBufferedReader(path)
```

缺乏经验的 Kotlin 开发人员会将伴生对象成员视为静态成员，然后将其分组在单个块中。然而，伴生对象实际上更加强大，例如，伴生对象可以实现接口和扩展类。因此，我们可以实现一般的伴生对象工厂函数，如下所示：

```kotlin
abstract class ActivityFactory {
    abstract fun getIntent(context: Context): Intent
    
    fun start(context: Context) {
        val intent = getIntent(context)
        context.startActivity(intent)
    }
    
    fun startForResult(activity: Activity, requestCode: Int) {
        val intent = getIntent(activity)
        activity.startActivityForResult(intent,requestCode)
    }
}

class MainActivity : AppCompatActivity() {
    //...
    
    companion object: ActivityFactory() {
        override fun getIntent(context: Context): Intent =
            Intent(context, MainActivity::class.java)
    }
}
    
// Usage
val intent = MainActivity.getIntent(context)
MainActivity.start(context)
MainActivity.startForResult(activity, requestCode)
```

请注意，这种抽象的伴生对象工厂可以保存值，因此它们可以实现缓存或支持伪创建以进行测试。在 Kotlin 编程社区中，伴生对象的优点没有得到很好的利用。尽管如此，如果你查看 Kotlin 团队产品的实现，你将看到大量使用伴生对象，例如在 Kotlin Coroutines 库中，几乎每个 coroutine 上下文的伴生对象都实现了一个接口 `CoroutineContext.Key` ，作为用来标识此上下文的键。

### 扩展工厂函数

有时，我们希望创建一个工厂函数，它的作用类似于现有的伴生对象函数，而我们在既不能修改这个伴生对象，又只想在单独的文件中定义一个新的函数，这种情况下，我们可以使用伴生对象的另一个优点：为它们定义扩展函数。

假设我们不能更改接口：

```kotlin
interface Tool {
    companion object { /*...*/ }
}
```

然而，我们可以在它的伴生对象上定义一个扩展函数：

```kotlin
fun Tool.Companion.createBigTool( /*...*/ ) : BigTool {
    //...
}
```

在调用的地方，我们可以这样写：

```kotlin
Tool.createBigTool()
```

这提供了一种强大的可能性，它允许我们使用自己的工厂方法扩展外部库。一个问题是，要扩展伴生对象，前提是这些对象原本就要有伴生对象（甚至可以是空的）：

```kotlin
interface Tool {
    companion object {}
}
```

### 顶级函数

创建对象的一种主流方式是使用顶级工厂函数。一些常见的例子是 `listOf`、`setOf` 和 `mapOf`。类似地，库设计人员指定用于创建对象的顶级函数。顶级工厂函数被广泛使用，例如，在 Android 中，我们有定义一个函数来创建 `Intent` 以启动一个 Activity 的传统， 在 Kotlin 中， `getIntent` 可以作为一个伴生对象函数来编写：

```kotlin
class MainActivity: Activity {
    
    companion object {
         fun getIntent(context: Context) =
            Intent(context, MainActivity::class.java)
    }
}
```

在 Kotlin Anko 库中，我们可以使用顶级函数 `intentFor` 来替代具体的类型：

```kotlin
intentFor<MainActivity>()
```

这个函数也可以传递参数：

```kotlin
intentFor<MainActivity>("page" to 2, "row" to 10)
```

使用顶级函数创建对象对于像 List 或 Map 这样的小对象来说是一个完美的选择，因为 `listOf(1,2,3)` 比 `List.of(1,2,3)` 更简单，可读性更强。然而，公共顶级函数需要谨慎使用。公共顶级函数有一个缺点：它们无处不在。开发人员的 IDE 技巧很容易被弄乱。当顶级函数像类方法一样命名并且与它们混淆时，问题就会变得更严重。这就是为什么顶级函数应该被明智地命名。

### 伪构造函数

Kotlin 中的构造函数与顶级函数的使用方式相同：

```kotlin
class A
val a = A()
```

它们的引用也与顶级函数相同（而构造函数引用实现了函数接口）：

```kotlin
val reference: () -> A = ::A
```

从使用的角度来看，大小写是构造函数和函数之间的区别。按照惯例，类以大写字母开头，函数则是小写字母。从技术上函数也可以以大写字母开头，而这个情况在不同的地方使用，例如，在 Kotlin 标准库中， `List` 和 `MutableList` 都是接口，它们不能有构造函数，但 Kotlin 开发人员希望允许以下的 `List` 构造方法：

```kotlin
List(4) { "User$it" } // [User0, User1, User2, User3]
```

这就是为什么 Kotlin stdlib 中包含了以下函数（从 Kotlin 1.1 开始）：

```kotlin
public inline fun <T> List(
    size: Int,
    init: (index: Int) -> T
): List<T> = MutableList(size, init)

public inline fun <T> MutableList(
    size: Int,
    init: (index: Int) -> T
): MutableList<T> {
    val list = ArrayList<T>(size)
    repeat(size) { index -> list.add(init(index)) }
    return list
}
```

这些顶级函数的外观和行为类似于构造函数，但它们具有工厂函数的所有优点。许多开发人员没有意识到，它们实际上是顶级函数。这就是为什么它们经常被称为“伪构造函数”。

开发者选择伪构造函数而不是真实构造函数的两个主要原因是：

* 为了给一个接口添加“构造函数”
* 拥有一个具体化（reified）类型的参数

除此之外，伪构造函数的行为应该与普通构造函数类似。它们看起来像构造函数，并且事实也应该去构造对象。如果你希望包含缓存、返回可空类型或返回类的子类，请考虑使用带名称的工厂函数，比如伴生对象工厂方法。

还有一种方法可以声明伪构造函数。使用带有 `invoke` 操作符的伴生对象也可以获得类似的结果，看看下面这段代码：

```kotlin
class Tree<T> {
    
    companion object {
        operator fun <T> invoke(size: Int, generator: 
            (Int)->T): Tree<T>{
                //...
            }
    }
}

// Usage
Tree(10) { "$it" }
```

然而，我们很少会使用在伴生对象中实现 `invoke` 来生成伪构造函数，我不建议这样做。首先，因为它打破了_第12条：操作符的行为应该与其名称一致_。调用伴生对象意味着什么？要知道名称是可以替代操作符的：

```kotlin
Tree.invoke(10) { "$it" }
```

调用(invoke)与对象的构造是不同的操作。以这种方式使用操作符会与其名称不一致。更重要的是，这种方法比顶级函数更为复杂，从它们的反射中可以看出这种复杂性。只需要比较反射在引用构造函数、伪构造函数和伴生对象的函数时的样子就知道了：

```kotlin
// 构造函数
val f: ()->Tree = ::Tree

// 伪构造函数
val f: ()->Tree = ::Tree

// 伴生对象使用 invoke
val f: ()->Tree = Tree.Companion::invoke
```

当你需要使用伪构造函数时，我建议使用标准的顶级函数。当我们不能在类本身中定义构造函数，或者需要添加构造函数本身所不支持的功能（比如具体化的类型参数）时，应该谨慎地使用它们，并且实现的像典型构造函数那样。

### 工厂类上的函数

有许多与工厂类相关的创建模式。例如，抽象工厂或原型模式。每种方式都有一些优点。

我们将看到这些方法的其中一些在 Kotlin 中使用是不太合理的，在下一条中，我们将学到重叠构造函数和建造者模式在 Kotlin 中几乎没有什么意义。

工厂类比工厂函数更有优势，因为类可以有一个状态。例如，下面一个非常简单的工厂类可以生成具有下一个 ID 数字的 Student：

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val surname: String
)

class StudentsFactory {
    var nextId = 0
    
    fun next(name: String, surname: String) = Student(nextId++, name, surname)
}

val factory = StudentsFactory()
val s1 = factory.next("Marcin", "Moskala")
println(s1) // Student(id=0, name=Marcin, Surname=Moskala)
val s2 = factory.next("Igor", "Wojda")
println(s2) // Student(id=1, name=Igor, Surname=Wojda)
```

工厂类可以有属性，这些属性可以用来优化对象的创建。当我们可以持有一个状态时，可以引入各式各样的优化或功能。例如，我们可以使用缓存，或者通过复制先前创建的对象来加速下一个对象的创建。

### 总结

如你所见，Kotlin 提供了多种指定工厂函数的方法，它们都有各自的用途。当我们设计对象的创建时，我们应该把它们记在心中。每一种方法都适用于不同的情况，其中一些应该谨慎使用：伪构造函数、顶级工厂函数和扩展工厂函数。定义工厂函数最通用的方法是使用 `Companion Object`。对于大多数开发人员来说，它是安全的，而且非常直观，因为它的使用非常类似于 Java 静态工厂方法，而且 Kotlin 也继承了这种风格并实践它。
