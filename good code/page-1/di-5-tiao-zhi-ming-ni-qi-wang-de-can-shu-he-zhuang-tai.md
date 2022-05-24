# 第5条：指明你期望的参数和状态

当你对参数、状态有期望时，应该尽快地指明它们，我们在 Kotlin 主要使用：

* &#x20;`require` 代码块 —— 一个通用的方式来指定对参数的期望
* &#x20;`check` 代码块 —— 一个通用的方式来指定对状态的期望
* &#x20;`assert` 代码块 —— 一个通用的方法来检查某个东西是否为真， JVM 上的这种检查只会在测试模块下生效
* &#x20;`Elvis` 操作符 —— 要么返回、要么抛出

下面是使用它们的一个例子：

```
// Part of Stack<T>
fun <T> pop(num: Int = 1): List<T> {
    require(num <= size) {
        "Cannot remove more elements than current size"    
    }
    check(isOpen) {
        "Cannot pop from closed stack" 
    }    
    val ret = collection.take(num)    
    collection = collection.drop(num)    
    assert(ret.size == num)    
    return ret
}
```

以这种方式来指定参数期望，并不能使我们摆脱在文档中指定那些期望的必要性，但无论如何它确实很有帮助，这种声明式检查有很多优点：

* 即使是那些没阅读文档的程序员也可以看到期望
* 如果他们不满意，函数将抛出一个异常，而不是导致其他的异常行为。重要的是，这些异常是在修改状态前被抛出的，这样就不会出现只有一些修改被应用，而另一些没有被应用的情况。 这种情况是危险的，并且很难管理。得益于期望性检查，错误将更难以被忽视，使得我们的状态更加稳定
* 代码在某种程度上是自我检查的，当代码在这些情况中被检查时，就不太需要进行单元测试了
* 以上列出的所检查都是适用于类型的智能转型，所以多亏了它们，减少了一些必要的类型转换

让我们谈谈不同种类的检查，以及我们为什么需要他们，从最流行的检查开始：**参数检查**

### 参数期望

当你定义一个带参数的函数时，通常会对那些不能使用系统类型表示的参数有一些期望。让我们来看几个例子：

* 你计算一个数字的阶乘时，你可能要求这个数是一个正整数
* 当你要做搜索，可能需要一个不为空的点阵列表
* 当你向一个用户发电子邮件时，你可能要求该用户拥有电子邮件，并且该值必须是正确的电子邮件地址（假设用户会在使用此函数之前应该检查电子邮件的正确性）

**Kotlin 检查这些需求的最普遍、最常见的方式是使用 `require` 函数**，该函数检查期望，如果在不满足时就抛出一个异常：

```
fun factorial(n: Int): Long {
    require(n >= 0)    
    return if (n <= 1) 1 else factorial(n - 1) * n
}

fun findClusters(points: List<Point>): List<Cluster>{
    require(points.isNotEmpty())    
    // ..
}

fun sendEmail(user: User, message: String) {
    requireNotNull(user.email)    
    require(isValidEmail(user.email))
}
```

请注意，这些需求是非常明显的，因为它们在函数的一开始就声明了。这使得用户在阅读这些函数时候能够清楚地了解它们（尽管这些检查也应该在文档中说明，因为不是每个人都会去阅读函数内容）。

这些期望不能被忽略，因为当谓词不满足条件时，会抛出一个 `IllegalArgumentException` 异常。当这样的块被放置在函数开头时，我们知道如果参数不正确，函数将立即停止，使用者可以尽早的解决这个问题。相反的，如果你不尽早检查它，那这个潜在的问题可能会传播的很远，直到失败。换句话说，当我们在函数的开始适当地指定我们对参数的期望时，就可以假设这些期望会被满足，这使得使用者在阅读这些函数时能够清晰地了解它们（尽管这些检查也应该在文档中说明，因为不是每个人都会去阅读函数内容）。

我们也可以使用其附带的 Lmabda 表达式为这个异常指定一个惰性消息：

```
fun factorial(n: Int): Long {
    require(n >= 0) {
        "Cannot calculate factorial of $n because it is smaller than 0"
    }
    return if (n <= 1) 1 else factorial(n - 1) * n
}

```

`require` 函数在指定参数时要求使用。

另一种常见的情况是我们对当前的状态有期望，这种情况下我们可以使用 `check` 函数来代替。

### 状态期望

我们只允许在特定条件下使用某些函数，这是很常见的。 下面是几个常见的例子：

* 有些函数在使用时可能需要先初始化一个对象
* 只有当用户已经登录时，才允许接下来的操作
* 函数可能会要求一个状态打开了，才能使用

**检查状态是否满足期望的标准方法是使用 `check` 函数**：

```
fun speak(text: String) {
    check(isInitialized)
    // ...
}
fun getUserInfo(): UserINfo {
    checkNotNull(token)
    // ...
}
fun next(): T {
    check(isOpen)
}
```

`check` 函数的工作原理与 `require` 类似，**但是当其所声明的期望值没有满足时，会抛出一个 `IllegalSteException` 异常**，它检查状态是否正确。 这些异常信息可以使用惰性消息来自定义，像                            require 函数那样。 当期望在整个函数上时，**我们把它放在开头， 通常放在 require 块之后**。尽管有些期望是局部的，检查可以稍后使用。

特别是当我们怀疑使用者可能会违反我们的约定，在不该调用函数的时候调用了它的时候，我们会采取这样的检查。与其相信他们不会这么做，还不如进行检查并抛出适当的异常。当我们无法确保持有的状态是否达到函数可用的正确状态时，也可以使用它。**在这种情况下，检查主要是针对于测试我们自己的实现时，我们有另一个函数叫做 `assert`**。

### 断言

当函数被正确实现时，有一些已知的事实。例如，当一个函数被要求返回 10 个元素时，我们可能期望它将返回 10 个元素。我们期望它符合条件，但这并不意味着我们总是对的。我们都会犯错，也许我们在执行过程中犯了错误，也许有人改变了我们使用的函数，使函数不再正常工作了，因为它被重构了。**对于所有这些问题，最普遍的解决方案是编写单元测试，检查我们的期望是否符合现实**：

```
class StackTest {
    @Test
    fun `Stack pops correct number of elements`() {
        val stack = Stack(20) { it }
        val ret = stack.pop(10)
        assertEquals(10, ret.size)
    }
}
```

**单元测试应该是检查实现正确性的最基本方法**，但请注意，`pop` 出的列表的大小与期望的大小匹配是正常普遍的，在几乎所有的 `pop` 函数中加入这样的匹配检查都会很有用。但这样单一的、使用仅仅只进行一次检查，是不成熟的做法，因为可能会出现些边缘情况。更好的解决方法是在函数中加入断言：

```
fun pop(num: Int = 1): List<T> {
    // ...
    aassert(ret.size == num)
    return ret
}
```

这些条件只能在 Kotlin/JVM 上使用，并且需要使用 `-ea JVM` 选项启用它们，否则断言不会生效。我们应该把它们当做单元测试的一部分 —— 它们检查我们的代码是否按预期工作。默认情况下，它们不会在生产环境中抛出任何错误。它们仅在运行测试时默认启用。这通常是我们所希望的，因为如果我们犯了错误，可能更希望用户不会注意到。如果这是一个可能发生的严重错误，并可能产生重大的负面后果，则可以使用 `check` 来替代。在函数中而不是在单元测试中设置断言检查的主要优点是：

* 断言能使代码自我检查，并产生更有效的测试
* 对每一个实际的用例，而不是某些具体的案例，都会对预期进行检查
* 我们可以使用它们来检查某些东西的确切执行
* 可以让代码尽早失败，更接近实际问题。由于这一点，我们也可以很容易地找到意外行为开始的地方和时间

请记住，要使用它们，我们仍然需要编写单元测试，在标准应用程序运行中， `assert` 不会抛出任何异常。

这样的断言在 Python 中很常见， 在 Java 中就不是这样的。 在 Kotlin 中你可以随意地使用它们，使你的代码更可靠。

### 可空性和智能转型

`require` 和 `check` 都有 Kotlin 合约，该合约规定函数返回一个值时，要检查其谓词为真。

```
public inline fun require(value: Boolean): Unit {
    contract {
        returns() implies value
    }
    require(value) { "Failed requirement." }
}
```

在这些代码块中被检查的内容将在该函数之后被视为 `true`。 它使用了智能转换，效果很好，因为一旦我们检查了某件事为真，编译器同样会视其为真。在下面的例子中，我们要求一个人的服装是 `Dress` ，之后，假设服装属性是 `final` 的，它将会被智能转化为 `Dress` :

```
fun changeDress(person: Person) {
    require(person.outfit is Dress)
    val dress: Dress = person.outfit
    // ...
}

```

当我们检查某项为空时，这个特性特别有用：

```
class Person(val email: String?)

fun sendEmail(person: Person, message: String) {
    require(person.email != null)
    val email: String = person.email
    // ...
}
```

对于这种情况，我们甚至有特殊的函数 `requireNotNull` 和 `checkNotNull`，它们都有智能转换变量类型的能力，我们也可以作为表达式来“解包”它：

```
class Person(val email: String?)
fun validateEmail(email: String) { /*...*/ }

fun sendEmail(person: Person, text: String) {
    val email = requireNotNull(person.email)
    validateEmail(email)
    // ...
}

fun sendEmail(person: Person, text: String) {
    requireNotNull(person.email)
    validateEmail(person.email)
    // ...
}
```

对于可空性，通常也用 `Elvis` 操作符，在右侧加上 throw 或 return。这样的结构也具有很高的可读性，同时它让我们能更加灵活地决定我们想要实现的行为。 首先我们可以使用 return 轻松地停止函数，而不是抛出错误：

```
fun sendEmail(person: Person, text: String) {
    val email: String = person.email ?: return
    //...
}
```

如果我们一个属性执行多个操作时，且这个属性是不正确的空值，我们可以通过包装 return 和 throw 到 `run` 函数中来添加这些操作，如果我们需要记录函数停止的原因，这样的功能可能会很有用：

```
fun sendEmail(person: Person, text: String){
    val email: String = person.email ?: run {
        log("Email not sent,")
        return
    }
}
```

带 return 或 throw 的 `Elvis` 操作符一种流行且惯用的方式，用于指定在变量为空时应该发生什么，我们应该毫不犹豫的使用它，同样如果可能的话，在函数开头的时候就应该进行这些检查，使它们清晰可见。

### &#x20;总结

通过指明你的期望，可以做到：

* 让它们更显眼
* 保护应用程序的稳定性
* 保护代码的正确性
* 智能的转化变量

为此，我们主要使用的四种机制是：

* `require` 代码块， 一种通用的，对参数进行检查的方法
* `check` 代码块，一种通用的，指定对状态期望的方法
* `asert` 代码块，一种在测试模式下测试某物是否为真的通用方法
* `Elvis` 操作符， 要么 return，要么throw

\


你也可以在遇到错误时，使用 Throwable 抛出不同的错误。？（你也可以用error throw来抛出一个不同的错误。）
