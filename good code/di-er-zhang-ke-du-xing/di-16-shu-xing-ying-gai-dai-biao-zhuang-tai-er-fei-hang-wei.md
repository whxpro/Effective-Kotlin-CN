# 第16：属性应该代表状态，而非行为

Kotlin 的属性（Properties）看似为 Java 的字段（Fields），但它们实际上是不同的概念：

```kotlin
// Kotlin proerties 属性
var name: String? = null

// Java fields 字段
String name = null;
```

尽管可以以同样的方式使用属性来存储数据，但是我们需要记住 Kotlin 属性还有很多功能，首先，它们总是可以自定义 getter 和 setter：

```kotlin
var name: String? = null // 这个是 备用字段(backing field)，即把 null 作为备用
    get() = field?.toUpperCase()
    set(value) {
        if(!value.isNullOrBlank()) {
            field = value
        }
    }
```

这段代码里，我们使用了 `filed` 来表示 `name`，它指向一个_备用字段（`Backing Field`）_，允许我们在此属性中存储数据。这些备用字段是默认生成的，因为 getter 和 setter 的默认实现使用了它们。我们还可以实现一个不使用它们的自定义访问器，在这种情况下，属性根本不用设置字段。例如 Kotlin 可以为只读 `val` 属性设置一个 getter ：

```kotlin
val fullName: String
   get() = "$name $surname"
```

对于可读写属性 `var`，我们可以通过自定义 getter 和 setter 来创建一个属性。这些属性被称为_派生属性(derived properties)_，因为它们很常见，所以 Kotlin 对所有属性都默认封装了 getter / setter。 想象一下，你必须在对象 `date` 中保存一个日期，并且数据源是 Java 标准库中的 `Date` 类型， 然后在某一时刻，出于某种原因，不能再使用 `Date` 来表示数据源了，而是要用 `Long`，可能是因为序列化问题，也可能是因为此对象要被提到公共模块中去，反正现在的问题是这个属性在整个项目中都被引用了，改动其类型可能会引起问题。而如果你使用了 Kotlin 的话，这将不会是一个问题，因为你可以将源数据提取到一个单独的属性 `millis（Long）` 中去保存，并修改 `date` 属性，它不再保存数据，而是包装/解包装这个 `millis` 属性。

```kotlin
/**
 * 实际的数据源是 millis, date 只是封装这个数据源，外界不直接引用 millis, 而是引用 date
 * 这是设计模式中的装饰模式,相较于 Java 来说，它实现更加简单
 */ 
var date: Date
  get() = Date(millis)
  set(value) {
      millis = value.time
  }
```

属性不需要字段，实际上，它们的本质其实是访问器（`val` 的 getter， `var` 的 getter 和 setter），这就是为什么我们可以在接口中去定义它们：

```kotlin
interface Person {
    val name: String
}
```

这意味着，我们的接口定义了一个访问器，用于访问 `name`， 我们还可以重写这个属性：

```kotlin
open class Supercomputer {
    open val theAnswer: Long = 42
}

class AppleComputer : Supercomputer() {
    override val theAnswer: Long = 1_800_275_2273
}
```

出于同样的原因，我们可以使用委托来修饰属性：

```kotlin
val db: Database by lazy { connectToDb() }
```

属性委托在_第21条：使用委托属性提取通用属性模式_中进行了详细描述，因为属性本质上是函数，所以我们也可以给类扩展属性：

```kotlin
val Context.preferences: SharedPreferences
   get() = PreferenceManager
       .getDefaultSharedPreferences(this)

val Context.inflater: LayoutInflater
   get() = getSystemService(
       Context.LAYOUT_INFLATER_SERVICE) as LayoutInflater

val Context.notificationManager: NotificationManager
   get() = getSystemService(Context.NOTIFICATION_SERVICE)
       as NotificationManager
```

如你所见，**属性代表的是访问器，而不是字段**，这可以使它们代替某些函数，但我们应该注意使用它们的目的，属性不应该被用来表示算法行为，就像下面的例子：

```kotlin
// 不要这样做！
val Tree<Int>.sum: Int
   get() = when (this) {
       is Leaf -> value
       is Node -> left.sum + right.sum
    }
```

这里 `sum` 属性用来遍历所有元素，所以它代表一个算法行为。因此这个属性很具有误导性：对于大型的 `Tree<Int>` 来说，要得到 `sum` 结果可能需要进行大量的递归计算，而 getter 不该承载这个行为，这里正确的做法是将算法搬到函数中去：

```kotlin
fun Tree<Int>.sum(): Int = when (this) {
    is Leaf -> value
    is Node -> left.sum() + right.sum()
}
```

一般的规则是：**我们应该只是用它们来表示或设置状态，不应该涉及其它逻辑**。决定某个东西是否应该是属性的一个有效方法是：如果一个东西不用给它定义 get / set 的前缀，它就不应该是一个属性。更详细点，下面是我们使用函数而非属性的典型情况：

* **操作的计算成本很高，或者计算复杂度高于 O(1)**，用户一般认为引用属性的成本会比使用函数要低，所以引用属性有昂贵的成本的话，使用函数会更好，这样用户就会尽可能节省使用函数，开发人员甚至考虑缓存其结果
* **它涉及到业务逻辑（应用程序如何运行）**， 当我们读代码时，我们希望属性仅仅实现一个简单的功能，而不是做更多的事情，例如日志记录、通知监听器或者更新观察者等等。
* **它是不确定的**，连续调用两次，可能会产生不同的结果
* **它是一个转化逻辑**，例如 `Int.toDouble()`，转化这个行为从习惯上来说，应该属于一个方法或者扩展函数
* **getter 不应该改变属性的状态**，我们希望可以自由地使用 getter ，而不担心改变了某些状态

例如，计算元素的和需要遍历所有的元素（这是一个行为，而不是状态），并且具有线性复杂性，因此它不应该是一个属性，而应该标准库的函数：

```kotlin
val s = (1..100).sum()
```

另一方面，为了获取和设置状态，我们在 Kotlin 中使用属性，除非有更好的方法，否则我们不应该使用函数。使用属性来表示和设置状态，如果以后需要修改它们，可以使用自定义 getter 和 setter：

```kotlin
// 不要这么做！
class UserIncorrect {
    private var name: String = ""
    
    fun getName() = name
    
    fun setName(name: String) {
        this.name = name
    }
}

class UserCorrect {
    var name: String = ""
}
```

一个简单的经验法则是：**属性描述和设置状态，而函数描述行为**。
