# 第34条：考虑带命名默认参数的主构造函数

当我们定义一个对象并指定如何创建它时，最主流的选择是使用主构造函数：

```kotlin
class User(var name: String, var surname: String)
val user = User("Marcin", "Moskała")
```

主构造函数不仅非常方便，而且在大多数情况下，使用它们创建对象实际上是一种良好的实践。我们通常需要传递对象初始化需要的参数，如下面的例子所示，我们从最浅显的地方开始：代表数据类模型的 data class。对于这样的对象，我们一般会将整个状态传递，作为构造函数的初始数据，然后再将它们保存为属性：

```kotlin
data class Student(
    val name: String,
    val surname: String,
    val age: Int
)
```

这是另一个常见的例子，在这个例子中，我们创建了一个逻辑层（presenter），来显示一个带索引的quotes 序列，在这里我们使用主构造函数注入依赖：

```kotlin
class QuotationPresenter(
    private val view: QuotationView,
    private val repo: QuotationRepository
) {
    private var nextQuoteId = -1
    
    fun onStart() {
        onNext()
    }
    
    fun onNext() {
        nextQuoteId = (nextQuoteId + 1) % repo.quotesNumber
        val quote = repo.getQuote(nextQuoteId)
        view.showQuote(quote)
    }
}
```

注意， `QuotationPresenter` 除了主构造函数外，还有别的属性。在这里， `nextQuoteId` 每一次初始化的值总是 -1，这是非常好的，特别是使用默认值或主构造函数参数设置初始状态时。

为了更好地理解为什么主构造函数在大多数情况下都是一个很好的选择，我们必须首先思考那些在 Java 中常见的与构造函数相关的模式：

* 重叠构造函数模式
* 建造者模式

我们将学到它们实际解决了什么问题，以及 Kotlin 提供了什么更好的替代方案。

### 重叠构造函数模式

重叠构造函数模式不过是一组构造函数，用在不同的可能传入的参数集上：

```kotlin
class Pizza {
    val size: String
    val cheese: Int
    val olives: Int
    val bacon: Int
    
    constructor(size: String, cheese: Int, olives: Int,
bacon: Int) {
    this.size = size
    this.cheese = cheese
    this.olives = olives
    this.bacon = bacon
    }
    
    constructor(size: String, cheese: Int, olives: Int):
this(size, cheese, olives, 0)
    
    constructor(size: String, cheese: Int):
this(size, cheese, 0)
 
     constructor(size: String): this(size, 0)
}
```

好吧，这段代码在 Kotlin 中没有任何意义，因为我们可以使用默认参数：

```kotlin
class Pizza(
    val size: String,
    val cheese: Int = 0,
    val olives: Int = 0,
    val bacon: Int = 0
)
```

默认值不仅更简洁，而且它们在使用上也比重叠构造函数上更加强大，我们可以同时指定 size 和 olives：

```kotlin
val myFavorite = Pizza("L", olives = 3)
```

我们也可以在 olives 之前或之后添加另一个具名参数：

```kotlin
val myFavorite = Pizza("L", olives = 3, cheese = 1)
```

如你所见，默认实参比重叠构造函数更强大，因为：

* 我们可以使用默认参数设置任何参数子集
* 我们可以以任何顺序提供参数
* 我们可以显式地命名参数，以明确每个参数的含义

最后一个原因很重要，请思考下面的对象创建：

```kotlin
val villagePizza = Pizza("L", 1, 2, 3)
```

它很短，但是清晰吗？我敢打赌，即使是定义 `Pizza` 的人也不会记得 `bacon` 在什么位置， `cheese` 在什么位置。当然，在 IDE 中我们是可以看到的，但对于那些在 Github 上阅读代码的人呢？当参数不明确时，我们应该使用具名参数说明它们的名称：

```kotlin
val villagePizza = Pizza(
    size = "L",
    cheese = 1,
    olives = 2,
    bacon = 3
)
```

A如你所见，带有默认参数的构造函数超越了重叠构造函数模式，尽管在 Java 中还有更流行的构造模式，其中之一就建造者模式。

### 建造者模式

具名命名参数和默认参数在 Java 中是没有的，这就是 Java 开发人员主要使用建造者模式的原因，它允许他们：

* 参数有了命名
* 可以任意顺序指定参数
* 有默认值

下面是 Kotlin 定义的一个建造者模式示例：

```kotlin
class Pizza private constructor(
    val size: String,
    val cheese: Int,
    val olives: Int,
    val bacon: Int
) {
    
    class Builder(private val size: String) {
        private var cheese: Int = 0
        private var olives: Int = 0
        private var bacon: Int = 0
        
        fun setCheese(value: Int): Builder = apply {
            cheese = value
        }
        
        fun setOlives(value: Int): Builder = apply {
            olives = value
        }
        
        fun setBacon(value: Int): Builder = apply {
            bacon = value
        }
        
        fun build() = Pizza(size, cheese, olives, bacon)
    }
}
```

在建造者模式中，我们可以使用它们的名称来设置这些参数：

```kotlin
val myFavorite = Pizza.Builder("L").setOlives(3).build()

val villagePizza = Pizza.Builder("L")
    .setCheese(1)
    .setOlives(2)
    .setBacon(3)
    .build()
```

正如我们已经提到的， Kotlin 默认参数和具名参数完全满足了这两个优点：

```
val villagePizza = Pizza(
    size = "L",
    cheese = 1,
    olives = 2,
    bacon = 3
)
```

比较这两种简单的用法，你可以看到具名参数相对于建造者模式的优势：

* **它更简短** —— 带有默认参数的构造函数或工厂方法比构造器模式更容易实现。对于实现该代码的开发人员和阅读人员来说，这都是一个节省时间的方法。这是一个显著的区别，因为建造者模式的实现可能非常耗时。任何建造器的修改是非常困难的，例如，改变一个参数的名称，不仅需要更改用于设置它的函数的名称，还要修改这个函数体内的参数名、一些曾经引用它的变量名、私有构造器的参数名等等
* **它更干净** —— 当你想要看到一个对象是如何构造的，你所需要的是使用一个单一的方法，而不是分散在整个构造器类。如何持有对象？它们会交互吗？当我们创建一个大型的建造器时，这些问题就没那么容易回答了。另一方面，在工厂方法上类的创建通常是明确的
* **提供更简单的用法** —— 主构造器函数是一个内置的概念，而建造器模式则是一个人工概念，它需要一些相关的知识，例如，开发人员很容易忘记在最后调用 `build` 函数（或者其它情况下创建了对象）
* **并发没有问题** —— 这是一个比较罕见的情况，但在 Kotlin 中函数参数总是保持不变的，而大多数建造器中的属性是可变的。因此，很难为建造器实现线程安全的构建函数

这并不意味着我们应该总是使用构造函数而不是建造者。让我们看看这种模式展示其优点的例子。

建造者可以为一个名称要求一组值（`setPositiveButton` 、 `setNegativeButton` 和 `addRoute`），并允许我们聚合（addRoute）：

```kotlin
val dialog = AlertDialog.Builder(context)
    .setMessage(R.string.fire_missiles)
    .setPositiveButton(R.string.fire, { d, id ->
        // 点击确定
    })
    .setNegativeButton(R.string.cancel, { d, id ->
        // 点击取消 dialog
    })
    .create()
    
val router = Router.Builder()
    .addRoute(path = "/home", ::showHome)
    .addRoute(path = "/users", ::showUsers)
    .build()
```

如果你想在构造函数中实现类似的行为，则需要为单个参数引入特殊类型，以保存更多的数据：

```kotlin
val dialog = AlertDialog(context,
    message = R.string.fire_missiles,
    positiveButtonDescription = ButtonDescription(R.string.fire, { d, id ->
        // 点击确定
    }),
    negativeButtonDescription = ButtonDescription(R.string.cancel, { d, id ->
        // 点击取消 dialog
    })
)

val router = Router(
    routes = listOf(
        Route("/home", ::showHome),
        Route("/users", ::showUsers)
    )
)
```

这种表示法在 Kotlin 社区中通常不受欢迎，我们倾向与在这种情况下使用  DSL 构造器：

```kotlin
val dialog = context.alert(R.string.fire_missiles) {
    positiveButton(R.string.fire) {
        // FIRE ZE MISSILES!
    }
    negativeButton {
        // User cancelled the dialog
    }
}

val route = router {
    "/home" directsTo ::showHome
    "/users" directsTo ::showUsers
}
```

这种类型的 DSL 构造器通常比经典的构造器模式更受欢迎，因为他们提供了更多灵活性和更加清晰的表示法。确实，创建一个 DSL 更加困难，但另一方面，创造一个 Builder 类也很困难。如果我们决定投入更多的时间，为什么我不能选择表示更好的那个方法呢？作为回报，我们将拥有更多的灵活性和可读性，在下一章中，我们将更多地讨论如何使用 DSL 创建对象。

经典构造器模式的另一个优点是它可以用作工厂，它可能被部分填充并进一步往下传递，例如我们应用程序中的一个默认对话框：

```kotlin
fun Context.makeDefaultDialogBuilder() =
    AlertDialog.Builder(this)
        .setIcon(R.drawable.ic_dialog)
        .setTitle(R.string.dialog_title)
        .setOnCancelListener { it.cancel() }
```

为了在构造函数或工厂方法中达到类似的可能，我们需要柯里化（curring），这在 Kotlin 中是不支持的，或者，我们可以将对象配置保存在数据类中，并使用 copy 来修改现有的对象：

```kotlin
data class DialogConfig(
    val icon: Int = -1,
    val title: Int = -1,
    val onCancelListener: (() -> Unit)? = null
    //...
)

fun makeDefaultDialogConfig() = DialogConfig(
    icon = R.drawable.ic_dialog,
    title = R.string.dialog_title,
    onCancelListener = { it.cancel() }
)
```

尽管这两种选择很少被视为一种选择。比如说，如果我们想定义一个应用程序的默认对话框，我们可以使用一个函数来创建它，并将所有定制元素作为可选参数传递，这样的方法对对话框的创建有更多的控制。这就是为什么我认为建造者模式的这一优势微不足道的原因。

最后，建造者模式很少是 Kotlin 中的最佳选择，它有时被选择的原因是：

* 使代码与那些使用建造者模式的其它语言编写的库保持一致
* 当我们设计 API ，使其易于在其他不支持默认参数或 DSL 的语言中使用时

除此之外，我们更倾向于使用带默认参数的主构造函数或表达型 DSL。

### 总结

对于我们项目中绝大多数对象来说，使用主构造函数创建对象是最合适的方法。在 Kotlin 中，重叠构造函数模式被视为过时的产物。我建议使用默认值，因为它们更简洁、更灵活、更具表现力。建造器模式也很少是合理的，因为在简单的情况下，我们可以只使用带命名参数的主构造函数，当我们需要创建更复杂的对象时，可以使用定义的 DSL 来实现。
