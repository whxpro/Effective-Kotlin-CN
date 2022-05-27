# 第46条：给高阶函数使用 inline 修饰符

你可能已经注意到了，几乎所有的 Kotlin stdlib 中的高阶函数都有一个 `inline`（内联）修饰符，你有没有思考过为什么要这样定义呢？例如，下面是 Kotlin 中 `repeat` 函数的实现：

```kotlin
inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    } 
}
```

这个 `inline` 修饰符的作用是：在编译期间，所有对这个函数的调用都替换成其函数体。所以，所有在 `repeat` 内部调用的函数参数都会被替换成它们的函数体。那么 `repeat` 的调用：

```kotlin
repeat(10) {
    print(it)
}
```

在编译期间，将会变成下面这样：

```kotlin
for (index in 0 until 10) {
    print(index)
}
```

与正常调用函数的方式相比，上面的方式则是一个显著的变化。在普通函数中，执行跳转到该函数体，调用所有语句，然后跳回调用该函数的位置。而用函数体嵌入来代替调用则是一种截然不同的行为。

这种行为有几个好处：

1. 类型参数可以具体化（`reified`）
2. 高阶函数在内联时速度更快
3. 允许非本地的返回

使用这个修饰符也有一些成本。让我们来学习下 `inline` 修饰符的优点和缺点。

### 类型参数可以被具体化（reified）

Java 的旧版本中没有泛型，它是在2004年J2SE 5.0版本中添加到 Java 编程语言中的。但是它们仍然没有出现在 JVM 字节码中。因此在编译的过程中，泛型被擦除了。例如 `List<Int>` 被编译成 `List`。这就是我们不能检查对象是否为 `List<Int>` 类型，只能检查它是否为 `List` 的原因。

```kotlin
any is List<Int> // Error
any is List<*> // OK
```

![](<../../.gitbook/assets/image (10) (1).png>)

出于同样的原因，不能对泛型进行操作。

```kotlin
fun <T> printTypeName() {
    print(T::class.simpleName) // ERROR
}
```

我们可以通过内联函数来克服这个限制。通过使用 `reified` 修饰符，用函数体来替代函数调用，所以这其实是用实参来代替形参:

```kotlin
inline fun <reified T> printTypeName() {
    print(T::class.simpleName)
}

printTypeName<Int>() // Int
printTypeName<Char>() // Char
printTypeName<String>() // String
```

在编译期间， `printTypeName` 的函数体替换了其引用，并且实参替换了形参：

```kotlin
print(Int::class.simpleName) // Int
print(Char::class.simpleName) // Char
print(String::class.simpleName) // String
```

`reified` 是一个很有用的修饰符。例如，它在标准库中 `filterIsInstance` 中使用，用来过滤特定类型的元素：

```kotlin
class Worker 
class Manager

val employees: List = listOf(Worker(), Manager(), Worker())
val workers: List = employees.filterIsInstance()
```

### 高阶函数在内联时速度更快

更具体地说，所有函数在内联时都会变得稍微快一些，不需要在执行时跳转，也不需要跟踪之后的栈。这就是 stdlib 中经常使用的小函数基本上都内联的原因：

```kotlin
inline fun print(message: Any?) {
    System.out.print(message)
}
```

但是，当函数没有任何实参时，使用这种方法和不使用的差异几乎是微不足道的。这就是 IntelliJ 给出这样一个警告的原因。

![](<../../.gitbook/assets/image (8) (1).png>)

为了理解其中的原因，我们首先要理解函数作为对象进行操作的问题是什么。这些类型的对象（使用函数字面量创建）需要以某种方式保存。在 Kotlin/JS 中，这很简单，因为 JavaScript 视函数为一等公民。因此在 Kotlin/JS，它要么是一个函数，要么是一个函数引用。在 Kotlin/JVM 上，需要用 JVM 匿名类或普通类来创建对象，因此，下面的 lambda 表达式：

```kotlin
val lambda: ()->Unit = {
    // code
}
```

将被编译成一个类，要么是 JVM 上的一个匿名类：

```java
Function0<Unit> lambda = new Function0<Unit>() {
    public Unit invoke() {
        // code
    } 
};
```

要么是被编译在一个单独文件中定义的普通类：

```java
// 单独文件上的类
public class Test$lambda implements Function0<Unit> {
    public Unit invoke() {
        // code
    }
}  

// 引用
Function0 lambda = new Test$lambda()
```

这两种做法之间没有显著差异。

注意，函数类型转化为 `Function0` 类型，这就是 JVM 中没有参数的函数类型被编译之后的类型。其它函数类型亦是如此：

* `()->Unit` 被编译为 `Function0<Unit>`&#x20;
* &#x20;`()->Int` 被编译为 `Function0<Int>`&#x20;
* &#x20;`(Int)->Int` 被编译为 `Function1<Int, Int>` &#x20;
* `(Int, Int)->Int` 被编译为 `Function2<Int, Int, Int>`

所有这些接口都是 Kotlin 编译器生成的，但是你不能在 Kotlin 中显式的使用它们，因为它们是按需生成。我们应该改用函数类型，知道函数类型只是一个接口可能会让你看到一些新花样：

```kotlin
class OnClickListener: ()->Unit {
    override fun invoke() {
        // ...
    }
}
```

正如_第45条：避免不必要的对象创建_所述，将函数包装到对象中会降低代码的运行速度，这就是为什么下面两个函数中，第一个函数会更快：

```kotlin
// 这里不会将 action 包装成一个对象
inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}

fun repeatNoinline(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

这种差异是显而易见的，虽然在现实看起来少有区别，但如果我们设计好了测试，你就可以很清楚地看到它们的差异：

```kotlin
@Benchmark
fun nothingInline(blackhole: Blackhole) {
    repeat(100_000_000) {
        blackhole.consume(it)
    }
}

@Benchmark
fun nothingNoninline(blackhole: Blackhole) {
    noinlineRepeat(100_000_000) {
        blackhole.consume(it)
    }
}
```

第一个程序在我的电脑上平均运行189ms，第二个则为447ms。这种差异来源于这么一个事实：在第一个函数中，我们只对数字进行迭代，并调用一个空的函数；在第二个函数中，我们在每次的数字迭代时，都会引用一个对象，而该对象调用一个空函数。所有这些区别都是因为我们使用了一个额外的对象（_第45条：避免必要的对象创建_）。

为了展示一个更典型的例子，假设我们有5000种产品，我们需要把已经购买的产品的价格累加起来，我们可以简单地这样做：

```kotlin
users.filter { it.bought }.sumByDouble { it.price }
```

在我的机器上，平均需要38ms来计算。如果 `filter` 和 `sumByDouble`函数没有内联，那会是多少？ 在我的机器上是42ms，这看起来并没有什么差异，但实际上你对集合进行处理时，使用内联每次都大约快10%左右。

当我们在函数中获取局部元素时，内联函数和非内联函数之间的更显著的区别就会凸显出来。获取的值也需要包装到某个对象中，并且无论如何使用它，都需要使用该对象来完成。例如，在以下代码中：

```kotlin
var l = 1L
noinlineRepeat(100_000_000) {
    l += it
}
```

局部变量不能在非内联 lambda 中直接使用，这就是为什么在编译时，它会被包装成一个引用对象：

```kotlin
val a = Ref.LongRef()
a.element = 1L
noinlineRepeat(100_000_000) {
    a.element = a.element + it
}
```

这是一个更重要的区别，因为这些对象通常会被多次使用：每次我们都要使用由 `fun` 关键字创建的函数。例如，在上面的例子中，一次循环里我们调用了两次函数（`a.elemtent`），因此，局部对象将被使用2\* 100,000,000次。为了了解这种差异，让我们对比以下函数：

```kotlin
@Benchmark
// 平均 30 ms
fun nothingInline(blackhole: Blackhole) {
    var l = 0L
    repeat(100_000_000) {
        l += it
    }
    blackhole.consume(l)
}

@Benchmark
// 平均 274 ms
fun nothingNoninline(blackhole: Blackhole) {
    var l = 0L
    noinlineRepeat(100_000_000) {
        l += it
    }
    blackhole.consume(l)
}
```

我的机器上第一个函数需要30ms，第二需要274ms。这是由于函数被编译为对象，并且需要包装局部变量这一事实的影响。这是一个显著的区别，因为在大多数情况下，我们不知道高阶函数如何使用，所以当我们定义高阶函数（例如用于集合处理）时，最好将其内联。这就是为什么 stdlib 中大多数高阶函数都是内联的。

### 允许非本地的返回

之前定义的 `repeatNoninlin` 看起来很像一个流程控制结构，只需将它与for循环进行比较：

```kotlin
if(value != null) {
    print(value)
}

for (i in 1..10) {
    print(i)
}

repeatNoninline(10) {
    print(it)
}
```

尽管一个显著的区别是，其内部是不允许返回的。

```kotlin
fun main() {
    repeatNoinline(10) {
        print(it)
        return // ERROR: Not allowed
    }
}
```

这是没有使用内联函数编译的结果。当代码位于另一个类中时，就不能直接从 `main` 函数返回。当函数内联时，就没有这种限制了，无论如何，代码都将位于 `main` 函数中：

```kotlin
fun main() {
    repeat(10) {
        print(it)
        return // OK
    }
}
```

正因如此，函数看起来和表现起来就更像流程控制了：

```kotlin
fun getSomeMoney(): Money? {
    repeat(100) {
        val money = searchForMoney()
        if(money != null) return money
    }
    return null
}
```

### inline 修饰符的开销

内联是一个有用的修饰符，但它不应该被到处使用。**内联函数不能是递归的**。否则它们会无限替换它们调用，周期性循环调用尤其危险，因为直到目前 IntelliJ 中都没有错误的提示：

```kotlin
inline fun a() { b() }
inline fun b() { c() }
inline fun c() { a() }
```

**内联函数不能使用具有更严格可见性的元素**。不能在公共内联函数中使用私有内部函数或属性。事实上，内联函数不能使用任何具有更严格可见性的函数：

```kotlin
internal inline fun read() {
    val reader = Reader() // Error
    // ...
}

private class Reader {
    // ...
}
```

所以它们不能用来隐藏实现，很少会在类中使用它们。

最后，它们很容易让代码膨胀。为了让你看到这个膨胀率，现在假设我很喜欢打印3，我定义了下面这个函数：

```kotlin
inline fun printThree() {
    print(3)
}
```

我喜欢调用它3次，所以我添加了这个函数：

```kotlin
inline fun threePrintThree() {
    printThree()
    printThree()
    printThree()
}
```

仍然不满意，我又定义了以下函数：

```kotlin
inline fun threeThreePrintThree() {
    threePrintThree()
    threePrintThree()
    threePrintThree()
}

inline fun threeThreeThreePrintThree() {
    threeThreePrintThree()
    threeThreePrintThree()
    threeThreePrintThree()
}
```

看看它们都被编译成了什么？ 前两个比较容易阅读：

```kotlin
inline fun printThree() {
    print(3)
}

inline fun threePrintThree() {
    print(3)
    print(3)
    print(3)
}
```

接下来的两个被编译成下面的函数：

```kotlin
inline fun threeThreePrintThree() {
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
}
inline fun threeThreeThreePrintThree() {
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
    print(3)
}
```

这是一个极端的例子，但是它展示出内联函数的一个很大问题：当我们过度使用它们时，代码增长的非常快。我在一个真实项目中遇到了这个问题。有太多内联函数相互调用是危险的，因为我们的代码可能开始呈指数膨胀。

### crossinlin 和 noinline

有些情况下，我们想内联一个函数，但由于某些原因，我们不能内联所有的函数类型参数。在这种情况下，我们可以使用一下修饰符：

* `crossinline` —— 意味着该函数应该内联，但不允许非本地的返回。当此函数在一个不允许非局部返回的作用域中使用时，我们使可以用它。例如在用在一个没有内联的 lambda 中
* `noinline` —— 意味着该参数根本不应该内联。它主要用作：内联函数调用另一个非内联函数时传入的参数

```kotlin
inline fun requestNewToken(
    hasToken: Boolean,
    crossinline onRefresh: ()->Unit,
    noinline onGenerate: ()->Unit
) {
    if (!hasToken) {
        // 我们必须使用 noline 函数来作为一个参数传给一个非内联的函数
        httpCall("get-token", onGenerate)
    } else {
        httpCall("refresh-token") {
            // 在不允许非局部返回的上下文中，必须给内联函数传入一个 crossinline 函数
            onRefresh() 
            
            onGenerate()
        }
    }
}

fun httpCall(url: String, callback: ()->Unit) {
/*...*/
}
```

了解这两个修饰符的含义是好的， **但我们可以不用记住它们，因为当需要它们时，IntelliJ IDEA 会有建议的提示**：

![](<../../.gitbook/assets/image (2).png>)

### 总结

使用内联函数的主要场景有：

* 经常要使用的功能，例如打印
* 需要指定类型作为参数传递的函数，例如 `filterIsInstance`
* 当定义高阶函数时，特别是工具函数，如集合处理函数（`map`、`filter`、`flatMap`、`joinToString`）、作用域函数（如 `also`、`apply`、`let`）或顶级使用函数（如`repeat`、`run`、`with`）

我们很少使用内联函数来定义接口，当一个内联函数调用其它内联函数时，我们应该小心，请记住，代码会不断膨胀。
