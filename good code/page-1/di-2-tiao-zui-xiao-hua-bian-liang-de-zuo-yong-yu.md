# 第2条：最小化变量的作用域

当我们定义一个状态时，我们倾向于通过以下方式来收紧变量和属性的范围：

* 使用局部变量，代替属性
* 在尽可能小的范围使用变量，例如，如果一个变量仅仅在一个循环中使用到，那么就在这个循环里定义它

元素的作用域是计算机程序中元素可见的区域。**在 Kotlin 中，作用域几乎总是由花括号创建的，我们通常可以从外部作用域去访问元素**。来看下面这个例子：

```kotlin
val a = 1
fun fizz() {
    val b = 2
    print(a + b)
}

val buzz = {
    val c = 3
    print(a + c)
}

// 在这个地方，我们使用a，但是无法使用 b 和 c， b、c对我们来说是“看不见的”
```

在上面的例子中，在 `fizz` 和 `buzz` 函数的作用域中，我们可以访问外部作用域的变量（a），然而，在外部作用域中，我们无法访问这些在函数里定义的变量。下面是一个限制变量作用域的示例：

```kotlin
// 不好的写法
var user: User
for (i in users.indices) {
    user = users[i]
    print("User at $i is $user")
}

// 好的写法
for (i in users.indices) {
    val user = users[i]
    print("User at $i is $user")
}

// 相同变量作用域下，更好的语法使用
for ((i, user) in users.withIndex()) {
    print("User at $i is $user")
}
```

在第一个示例中，`user` 变量不仅可以在 for 循环的范围内访问，也可以在 for 循环之外访问。在第二个和第三个示例中，我们将 `user` 变量的作用域具体限制在 for 循环的作用域内。

类似的，作用域内可能有许多作用域（比如 Lambda 表达式里面又有一层 Lambda 表达式），**我们最好在尽可能窄的作用域内定义变量**。

我们喜欢使用这种技巧的原因有很多，但最主要的原因是：我们收紧变量的作用域时，我们的程序将更加易于追踪和管理。当我们分析代码时，我们需要考虑此时存在哪些元素。要处理的元素越多，编程就越难进行。应用程序越简单，崩溃的可能性就越小。这与我们为什么更喜欢不可变属性而不是可变属性的原因类似。

如果考虑可变属性，当它们只能在较小的范围内修改时，跟踪它们的变化更容易。对它们进行推断并改变它们的行为更容易。

另一个问题是，**范围较广的变量可能会被另一个开发者过度使用**。举个例子，如果使用一个变量被用于指定迭代中的下一个元素，那么在循环完成后，列表中的最后一个元素应该保留在该变量中。这样的推断可能会导致严重的滥用，比如在迭代之后使用这个变量来处理最后一个元素，这是非常糟糕的，因为另一个试图理解程序的开发人员需要理解整个推断过程，这将是一个不必要的让程序复杂化的行为。

无论一个变量是只读的还是可读写的，我们总是希望在定义变量时就对其进行初始化。不要强迫其开发人员寻找它定义之处。这可以通过流程控制语句来支持，比如 `if`，`when`，`try-catch`，或者作为表达式使用的 Elvis 操作符：

```kotlin
// 不好的写法
val user: User
if (hasValue) {
    user = getValue()
} else {
    user = User("bbb")
}

// 好的写法
val user: User = if (hasValue) {
    getValue()
} else {
    User("bbb")
}
```

**如果我们需要设置多个属性值，可以使用解构函数**：

```kotlin
// 不好的写法
fun updateWeather(degrees: Int) {
    val description: String
    val color: Color
    if (degrees < 5) {
        description = "cold"
        color = Color.BLUE
    } else if (degrees < 23) {
        description = "mid"
        color = Color.YELLOW
    } else {
        description = "hot"
        color = Color.RED
    }
}

// 好的写法
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "mid" to Color.YELLOW
        else -> "hot" to Color.RED
    }
}
```

最后，过于广泛的变量范围可能是危险的，让我来描述其中一个常见的危险。

### 提取

当我在教授 Kotlin 协程时，我会布置的一个练习是：使用_埃拉托斯特尼筛法_来寻找质数，该算法其实原理很简单：

1. 我们从2开始构建一个数字列表；
2. 我们取出第一个，它是一个质数；
3. 从剩下的数中，我们移除掉第一个，然后过滤掉所有能被这个质数整除的数。

下面是这个算法的一个简单实现：

```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()
while (numbers.isNotEmpty()) {
    val prime = numbers.first()
    primes.add(prime)
    numbers = numbers.filter { it % prime != 0 }
}
print(primes) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 
// 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```

尽管在几乎每个团队中都有人试图去“优化”它，例如不会在每一个循环中创建变量，而是提取质数作为可变变量：

```kotlin
val primes: Sequence<Int> = sequence {
    var numbers = generateSequence(2) { it + 1 }
    var prime: Int
    while (true) {
        prime = numbers.first()
        yield(prime)
        numbers = numbers
            .drop(1)
            .filter { it % prime != 0 }
    }
}
print(primes.take(10).toList())
// [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

现在停下来，来尝试解释下这个错误的结果。

之所以会有这样的结果，是因为我们提取了这个 `prime` 作为可变变量存储每次计算的质数。因为使用的是一个 `sequence`，所以过滤是惰性计算的的。在每一步中，我们会添加越来越多的过滤器。在 “优化” 代码中，我们会去为 `可变属性prime` 添加过滤器，因此我们总是过滤 `prime` 的最后一个值。这就是这个 `filter()` 不能正常工作的原因，只有 `drop()` 方法正常工作，所以我们得到了连续的数字（除了4，当 `prime` 设置为2时，4就被过滤掉了）

我们应该意识到日常开发中无意识提取变量的问题，因为这种情况会时常发生。为了防止这种情况，**我们应该避免元素可变性，并选择使用更窄范围的变量作用域**。

### 总结

出于人性化的考虑，我们应该尽可能接近作用域的去定义变量。此外，对于局部变量，我们更喜欢使用 `val` 而不是 `var`，我们应该始终注意到这样一个事实：变量是在 lambda 表达式中提取的，这样一个简单的规则可以为我们省去诸多麻烦
