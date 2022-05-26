---
description: >-
  请注意！！！！！虽然在 Kotlin1.6中，已经废弃 inline class 的用法，而换成了更加强大的 value
  修饰符，但本文依然有参考价值，它阐述了内联类解决的问题，详细请看：https://kotlinlang.org/docs/inline-classes.html
---

# 第47条：考虑使用内联类

不仅函数可以内联，包含单个值的对象的引用，也可以用“直接使用该值”的方式来代替。这种可能性是 Kotlin 在 1.3 版本中实验性引入的，为了使其成为可能，我们需要将 `inline` 修饰符放在具有单个主构造属性的类之前：

```kotlin
inline class Name(private val value: String) {
    // ...
}
```

这样的类在使用时将被替换成为它所保存的值：

```kotlin
// 使用
val name: Name = Name("Marcin")

// 整个编译过程中，将会被下面这样的代码所替换
val name: String = "Marcin"
```

此类中的方法将被作为静态所使用方法：

```kotlin
inline class Name(private val value: String) {
    // ...
    fun greet() {
        print("Hello, I am $value")
    }
}

// 使用
val name: Name = Name("Marcin")
name.greet()

 // 整个编译过程中，将会被下面这样的代码所替换
 val name: String = "Marcin"
 Name.`greet-impl`(name)
```

我们在没有性能开销的情况下（_第45条：避免不必要的对象创建_），使用内联类对某些类型（如上面示例中的 `String`）进行包装。内联类两个特别流行的用法是：

* 表示测量单位
* 使用类型，以保护我们免遭滥用

让我们来分别讨论这两种用法。

### 表示测量单位

假设你需要使用一种方法来设置计时器：

```kotlin
interface Timer {
    fun callAfter(time: Int, callback: ()->Unit)
}
```

`time` 的单位是什么呢？ 可能是毫秒、秒、分钟... 现在还不清楚，很容易出错。 一个著名的例子是：某个火星气候轨道器错误地撞向了火星大气层，背后的原因是控制它的软件是由一个外包公司开发的，它产生的输出单位不是 NASA 所期望的。它所输出的结果单位是磅力秒（lbf·s），但是 NASA 期望的单位是牛顿秒（N·s）。这次任务的总成本是3.276亿美元，以完全失败告终。正如你所看到的，测量单位的混乱可能会产生非常严重的后果。

开发人员建议描述单位的一种常见方法是将其包含在参数名中：

```kotlin
interface Timer {
    fun callAfter(timeMillis: Int, callback: ()->Unit)
}
```

这是更好的，但仍然留有犯错的余地。在使用函数时，属性名通常是不可见的。另一个问题是，当作为返回类型时，以这种方式指示类型会比较困难。在下面的例子中，由 `decideAboutTime` 返回时间，它的度量单位根本没有说明。它可能返回以分钟为单位的时间，那么时间设置将无法正常工作。

```kotlin
interface User {
    fun decideAboutTime(): Int
    fun wakeUp()
}

interface Timer {
    fun callAfter(timeMillis: Int, callback: ()->Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time: Int = user.decideAboutTime()
    timer.callAfter(time) {
        user.wakeUp()
    }
}
```

我们可能会给这些返回的数据予以单位描述，比如 `decideAboutTimeMills`，但这样的解决方案是相当少见的，因为它让一个函数名非常长，而且它展示了我们不需要知道的一个低级的，非必要信息。

解决这个问题的一个更好的方案是引入更严格的类型，以防止我们误用更广泛的类型，我们可以使用内联类：

```kotlin
inline class Minutes(val minutes: Int) {
    fun toMillis(): Millis = Millis(minutes * 60 * 1000)
    // ...
}

inline class Millis(val milliseconds: Int) {
    // ...
}

interface User {
    fun decideAboutTime(): Minutes
    fun wakeUp()
}

interface Timer {
    fun callAfter(timeMillis: Millis, callback: ()->Unit)
}

fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time: Minutes = user.decideAboutTime()
    timer.callAfter(time) { // ERROR: Type mismatch
        user.wakeUp()
    }
}
```

这样就可以强迫我们去使用正确的类型：

```kotlin
fun setUpUserWakeUpUser(user: User, timer: Timer) {
    val time = user.decideAboutTime()
    timer.callAfter(time.toMillis()) {
        user.wakeUp()
    }
}
```

它对于公制单位非常有用，比如在前端，我们经常使用各种各样的单位，如像素、毫米、dp等。为了支持对象创建，我们可以定义类似 DSL 的扩展属性（你可以称它们为内联函数）：

```kotlin
inline val Int.min
    get() = Minutes(this)

inline val Int.ms
    get() = Millis(this)
    
val timeMin: Minutes = 10.min
```

### 保护我们免受类型滥用

在 SQL 数据库中，我们经常通过元素的 id 来识别元素，而这些 id 都只是数字，因此，假设你的系统中有一个学生成绩，它可能会引用学生、老师、学校等的id：

```kotlin
@Entity(tableName = "grades")
class Grades(
    @ColumnInfo(name = "studentId")
    val studentId: Int,
    @ColumnInfo(name = "teacherId")
    val teacherId: Int,
    @ColumnInfo(name = "schoolId")
    val schoolId: Int,
    // ...
)
```

这里的问题是，以后很容易误用所有这些 id，因为它们都是 Int 类型，解决方案是将所有这些整数包装到单独的内联类中：

```kotlin
inline class StudentId(val studentId: Int)
inline class TeacherId(val teacherId: Int)
inline class SchoolId(val studentId: Int)

@Entity(tableName = "grades")
class Grades(
    @ColumnInfo(name = "studentId")
    val studentId: StudentId,
    @ColumnInfo(name = "teacherId")
    val teacherId: TeacherId,
    @ColumnInfo(name = "schoolId")
    val schoolId: SchoolId,
    // ...
)
```

现在这些 id 的使用将会是安全的，同时，数据库也会正确的生成，因为在编译期间，所有这些类型将被替换成 Int。通过这种方式，内联类允许我们引入以前不允许的类型，因此，我们的代码更加安全，而且没有性能开销。

### 内联类和接口

与其它类一样，内联类也可以实现接口。这些接口可以传递我们想要的正确的度量单位。

```kotlin
interface TimeUnit {
    val millis: Long
}

inline class Minutes(val minutes: Long): TimeUnit {
    override val millis: Long get() = minutes * 60 * 1000 
    // ...
}

inline class Millis(val milliseconds: Long): TimeUnit {
    override val millis: Long get() = milliseconds
}

fun setUpTimer(time: TimeUnit) {
    val millis = time.millis
    //...
}

setUpTimer(Minutes(123))
setUpTimer(Millis(456789))
```

问题在于，当通过接口使用对象时，它不能够内联我们的类。因此，在上面的示例中，使用内联类没有任何优势，因为需要创建封装的对象，以便我们通过该接口拿到类型。**当我们通过接口呈现内联类时，这样的类不是内联的**。

### 别名

Kotlin `typealias` 允许我们为类型创建另个一名称：

```kotlin
typealias NewName = Int
val n: NewName = 10
```

命名类型是一种有用的功能，特别是处理长且可重复的类型时，例如，主流的做法是命名可重复的数据类型：

```kotlin
typealias ClickListener =
    (view: View, event: Event) -> Unit

class View {
    fun addClickListener(listener: ClickListener) {}
    fun removeClickListener(listener: ClickListener) {}
    //...
}
```

但是需要了解的是，**类型别名并不能以任何方式保护我们免受类型误用**，它们只是为类型添加了一个新名称。如果我们将 Int 同时命名为 MiLls 和 Second，我们就产生一种错觉，认为类型系统保护了我们，而实际上并没有：

```kotlin
typealias Seconds = Int
typealias Millis = Int

fun getTime(): Millis = 10
fun setUpTimer(time: Seconds) {}

fun main() {
    val seconds: Seconds = 10
    val millis: Millis = seconds // 不会编译报错
    setUpTimer(getTime())
}
```

在上面的例子中，如果没有类型别名，找到问题所在会更容易。这就是为什么不应该这样使用它们。要表示度量单位，可以使用参数名称。命名成本低，但使用类将更加安全。当我们使用内联类时，可以鱼和熊掌兼得 —— 成本低又安全。

### 总结

内联类允许我们封装类型而不增加性能开销。正因如此，我们提高了安全性，使类型系统保护我们免受滥用。如果使用的类型不明确，特别是具有不同度量单位的类型，请考虑使用内联类包装它们。
