# 第8条：妥善处理空值

`null` 代表值的缺失。对于属性来说，这可能意味着未设置值给它，或者已经将其值移除。当一个函数返回 `null` 时，它可能有不同的含义，这取决于函数，例如：

* 当 String 不能被正常解析为 Int 时， `String.toInOrNull` 会返回null
* `Iterable<T>.firstOrNull((T) -> Boolean)` 会在列表没有匹配谓词的元素时，返回null

**这些情况，或其他所有情况下，`null` 的含义应当尽可能的清楚**。这是因为外界是必须要处理空值的，而那些使用 API 的程序员需要决定如何去处理它。

```kotlin
val printer: Printer? = getPrinter()
printer.print()  // 编译错误

printer?.print()  // 安全调用
if (printer != null) print.print()  // 智能转化，printer不为空了
printer!!.print() // 非空断言
```

通常情况下，处理可空类型有三种方法，我们可以：

* 使用像 `?`、智能转换、`Elvis`操作符等，来安全地使用可空对象
* 抛出一个错误
* 重构该函数或性，使其不能为空 让我们来逐个详细介绍它们。

### 安全处理空值

如上所述，处理空值最安全和最流行的两种方法是使用**空安全调用**和**智能转型**：

```
printer?.print()  // 安全调用
if (printer != null) print.print()  // 智能转化
```

在这两种情况下，只有当 `printer` 不为空时，函数 `print()` 才会被调用。 从应用程序用户的角度来看，这是最安全的选择，对于程序员来说，这种写法也非常舒适，怪不得说这是处理可空值最流行的方法呢。

与其他语言相比，Kotlin 对处理可空变量的支持要广泛的多，一种主流的做法是使用 `Elvis` 操作符，它为可空类型提供默认值。它允许包含任何使用 `return` 和 `throw` 的表达式：

```
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?: throw Error("Printer must be named")
```

许多对象都有额外的支持。例如，通常要求一个空集合而不是null， 有像 `collection<T>?.orEmpty` 这样的扩展函数，用于返回一个不为空的只读列表 `List<T>`

Kotlin 的特性也支持智能转型，它允许我们在函数中进行智能类型转换：

```
println("What is your name?")
val name = readLine()
if (!name.isNullOrBlank()) {
    println("Hello ${name.toUpperCase}")
}

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) {
    news.forEach { notifyUser(it) }
}
```

Kotlin 开发人员应该了解所有这些选择，它们都提供了正确处理可空属性的有效方式。

#### 防御型和进攻型编程

以正确的方式处理所有的可能的情况 —— 比如之前的例子，当 `printer` 为空时，就不能使用 `print` 方法，这是防御型编程的一种实现。防御型编程是一个概括性的术语，它指的是代码投入生产后，用各种实践来使代码提高稳定性，通常是通过防御当前不可能出现的事情。最好的做法是有一种正确的方式来处理所有可能的发生的情况。

如果我们期望 `printer` 不能为空，并且应该被使用，这是有问题的，因为我们无法强迫其不为空。 在这种情况下，是不可能做到安全处理的。 我们应该使用一种被称为进攻型编程的技术。攻击型编程背后的原理是：在出现意外情况时，我们将其反馈给开发人员，并迫使他们来纠正错误。这个想法的直接实现是第5条中提出的：使用`require`、`check` 和 `assert` 指明你对参数和状态的期望。 重要的是要去理解，虽然这两种模式看起来是冲突的，但实际上相反，他们更像阴阳结合而成，**为了安全起见，我们的程序都需要这两种不同模式，我们要理解并适当的使用它们**。

### 抛出一个错误

安全处理的一个问题是：如果 `printer` 有时可能是空的，我们将不会感知到，我们只能在 `printer` 无法调用时才知道。 这样我们可能会隐藏掉了重要的信息。 如果我们希望 `printer` 永远不为空，那么当`print` 方法没有调用时，我们会觉得很意外，这可能会导致难以发现的错误。当我们很想捕获一些意外的情况，最好抛出一个错误来通知程序员意外的情况，这个时候可以使用 `throw` ，也可以使用非空断言 `!!.`， 还有 `requireNotNull`、`checkoutNotNull` 或其他抛出错误的函数:

```
fun process(user: User) {
    requireNotNull(user.name)
    val context = checkNotNull(context)
    val networkService = getNetworkService(context) ?: throw NoInternetConnection()
    networkService.getData { data, userData ->
        show(data!!, userData!!)
    }
}
```

### 使用非空断言 !!. 的问题

处理空值最简单的方法是使用非空断言 `!!.`，它在概念上类似于 Java 中的发生的事情 —— 我们认定某些东西不是空的，如果我们错了，就会抛出 NPE。 非空断言 `!!` 是懒人的选择，它短又简单，这也使它容易被滥用或误用。 **断言 `!!` 通常用于值可空但不期望为空的情况，但问题是，即使当前不为空，那也不代表未来不是**，而且这个操作符只是悄悄的隐藏了值得可空性。

一个非常简单的例子是在 4 个参数中寻找最大值的函数，假设我们决定通过将这些参数打包到一个列表中，然后用 `List<Int>.max()` 来找到最大值来实现它。问题是 `max` 函数返回一个可空值，因为当传入的集合为空时，它会返回 null，而开发人员知道这个列表不能为空，所以很可能会使用非空断言 `!!`:

```
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int = listOf(a, b, c, d).max()!!
```

正如这本书的评论员 Marton Braun 向我展示的那样，在这样一个简单的函数种使用非空断言，会导致非良性的思维。有些人可能需要重构这个函数，以接收任意数量的参数，但忽略了集合不能为空的情况：

```
fun largestOf(vararg nums: Int): Int = nums.max()!!largestOf() // NPE
```

关于可空性的信息已经被隐藏了，当它可能很重要时也很容易被忽略。 这个问题同样出现在变量上。假设你有一个变量需要延迟设置，它肯定会在首次使用前被设置，而将其设置为 null 并使用非空断言的做法，并不是一个好的选择：

```
class UserControllerTest {    private var dao: UserDao? = null    private var controller: UserController? = null       @BeforeEach        fun init() {        dao = mockk()        controller = UserController(dao!!)    }       @Test    fun test() {        controller!!.doSomething()    }}
```

这不仅让我们每次都需要解包这些属性，而且我们还阻止了这些属性在未来可以实际拥有一个有意义 null 的可能性。在本项目后面，我们将看到处理这种情况的正确方式是使用 `lateinit` 或 `Delegates.notNull`

没有人能够预测代码将来会如何演变，如果你使用非空断言或者显式的抛出错误，你应该假设它总有一天会抛出错误。异常应该在意外和不正确的情况下被抛出（_第7条：当返回结果可能缺失时，优先使 null 或 Failure_ ），然而，**显式错误一般比 NPE 更能说明问题，它们几乎总是首选的**。

一种会用到非空断言的罕见情况主要是：在与不能正确表达可空性的库进行交互。当你与那些 Kotlin 良好设计的 API 交互时，这不应该成为一种规范。

一般来说，我们应该避免使用非空断言，这个建议得到了社区的广泛认可，许多团队都有阻止使用它的规范，有些设置了 Detekt 静态代码扫描，若代码中有使用非空断言时，编译会抛出错误。我认为这种方式太极端了！ 但是我同意它确实是一种代码坏味道。看起来这个操作不是偶然，`!!` 似乎在向我们警告：“小心” 或 “这里有问题”。

### 避免使用无意义的空值

使用可空性需要付出代价，因为它需要被正确地处理，**我们希望在不必要时，避免检查值的可空性**。 Null 可能会传递一个重要的消息，但是我们应该避免它对其他开发人员造成无意义的情况。 不然他们就会忍不住使用不安全的非空断，或者被迫重复只会让代码混乱的通用安全处理。 我们应该避免产生对用户没有意义的可空值。 最重要的方法有：

* 类可以提供函数的变体，其中的结果是可预期的，并考虑值不足的情况下返回可空的结果或密封的结果类，简单的例子是 `List.get` 和 `List.getOrNull`，在第7条:当返回结果可能缺失时，优先使 null 或 Failure 中解释了这些
* 当一个值在使用之前设置，但不是在类创建时候设置时，使用 `lateinit` 或非空委托类来修饰
* **不用返回 null 来代替返回空集合**。当我们处理集合时，比如 `List<Int>?` 或 `List<Strin>`，null 与空集合有不同的含义。 空值意味着不存在集合，若是表明缺少元素，请使用空集合
* 可空枚举和非空枚举是两个不同的消息， null 是需要单独处理的特殊消息，但它不在枚举的定义中，因此可以将其添加任何你使用的地方

让我们讨论 `lateinit` 属性和非空代理，因为它们值得更深入的解释。

### lateinit 属性 和 非空委托类

在项目中，有些属性不能在类创建期间初始化，这并不罕见，但在第一次使用它之前肯定会初始化这些属性，一个典型的例子是：一个属性在一个比其他所有函数都先调用的函数中设置，就像 JUnit5 中的 `@BeforeEach` 那样：

```
class UserControllerTest {

    private var dao: UserDao? = null
    private var controller: UserController? = null

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao!!)
    }

    @Test
    fun test() {
        controller!!.doSomething()
    }

}
```

在我们需要时，将这些属性从可空转化为非空是非常不可取的，这也是没有意义的，因为我们期望这些值在使用之前就被设置，这个问题的正确解决方案是使用 lateinit 修饰符，我们延迟对其初始化：

```
class UserControllerTest {
    private lateinit var dao: UserDao
    private lateinit var controller: UserController
   
    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao)
    }
   
    @Test
    fun test() {
        controller.doSomething()
    }
}
```

使用 `lateinit` 的代价是，如果我们在试图初始化之前获取它的值，那会抛出异常，听起来很可怕，但它确实是我们想要的。只有当我们确信它会在第一次使用之前被初始化，就应该使用 `lateinit`， 如果错了，我们希望被告知。 使用`lateinit` ，它和 `nullable` 的主要区别是：

* 我们不用每次都对可空值进行“解包”
* 如果我们将来使用 null 表示一些有意义的东西，我们可以轻易的把它设置为空
* 一旦属性初始化，它就不能回到未初始的状态

当我们确信属性在第一次使用之前初始化时， lateinit 将是一个很好的实践。我们主要在类有生命周期时处理这种情况，在第一个被调用的方法中设置属性，以便在后面的方法中使用它。 例如，当我们在 Android Activity 的 `onCreate` 中设置对象， 在 iOS 的 `UIViewController` 中设置 `viewDidAppear`，或者在 React `React.Component` 中设置 `componentDidMount`（译者注:对异常路径的 `lateinit` 属性需要进行判断，如果没有初始化则不操作，例如 Android 中的 `Activity.onDestroy` 有可能在没有 `onCreate` 就会被调用）。

`lateinit` 不能使用的一种情况是：当我们需要在 JVM 上使用原语相关的类型（如 Int、 Long、Double 或 Boolean）初始化属性时。对于这种情况，我们必须使用 `Delegate.notNull`，它会稍微慢一些，但是支持这些类型：

```
class DoctorActivity: Activity() {
    private var doctorId: Int by Delegates.notNull()
    private var fromNotification: Boolean by Delegates.notNull()
   
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        doctorId = intent.extras.getInt(DOCTOR_ID_ARG)
        fromNotification = intent.extras.getBoolean(FROM_NOTIFICATION_ARG)
    }
}
```

这些情况通常被属性委托代替，就像上面的例子，若我们在 `onCreate` 中读取参数，就可以使用一个委托来惰性初始化这些属性：

```
class DoctorActivity: Activity() {
    private var doctorId: Int by arg(DOCTOR_ID_ARG)
    private var fromNotification: Boolean by arg(FROM_NOTIFICATION_ARG)
}
```

属性委托在第21条：使用属性委托提取通用属性模式中进行了详细描述，它如此受欢迎的原因是其有效帮助我们安全避免了可空性。
