# 第12条：操作符的行为应该与其名称一致

运算符重载是一个强大的特性，与大多数强大的特性一样，它也是危险的，在编程中，能力越大，责任越大。作为一名培训师，我经常看到学生第一次看见操作符的重载时，他们是如何晕掉的。例如，下面是一个用于产生阶乘结果的实现：

```kotlin
fun Int.factorial(): Int = (1..this).product()

fun Iterable<Int>.product(): Int =
    fold(1) { acc, i -> acc * i }
```

该函数被定义为 Int 的扩展函数，因此它可以比较方便地使用：

```kotlin
print(10 * 6.factorial()) // 7200
```

数学家都知道有一种特殊符号能表示阶乘，那就是数字后面跟上一个感叹号：

```kotlin
10 * 6!
```

Kotlin不支持这样的操作符，但正如我的一个研讨会参与者所发现的那样：它可以用重载 `not` 操作符来实现:

```kotlin
operator fun Int.not() = factorial(

print(10 * !6) // 7200
```

我们确实可以这样做，但我们应该这样做吗？答案是否定的，你只要阅读了函数声明就可以注意到这个函数叫 `not()`，顾名思义，它不该被这样使用，它表示的是一个逻辑操作，而不是求数字的阶乘。这种用法会使人感到困惑和误导，在 Kotlin 中，所有的操作符都只是拥有具体名称的函数的语法糖， 如下图所示，每个操作符都可以作为函数，而不是操作语法来使用。那么这样的话，上面这个函数又会是什么样子呢？

```kotlin
print(10 * 6.not()) // 7200
```

![](<../../.gitbook/assets/image (3).png>)

在 Kotlin 中，每个运算符的含义始终保持不变。这是一个非常重要的设计决策。有些语言，例如 Scala，可以提供无限的运算符重载功能。 总所周知，一些开发者严重滥用了这种自由。第一次使用不熟悉的库时，即使它有一个有意义的函数名和类名，在阅读代码上依然会比较难。现在想象一下，你需要分别理解每个运算符，记住它在特定上下文的意思，然后全部记在心里，以便将各个部分连接起来，以理解整行语句。 在 Kotlin 中则没有这样的问题，因为这些运算符都有具体的含义，例如，当你看到下面的表达式：

x + y == z

你会知道它等同于

x.plus(y).equal(z)

如果加法定义了一个可空返回类型，那么代码也可以是下面这样：

(x.plus(y))?.equal(z) ?: (z == null)

这些函数都有一个具体的名称，我们希望所有函数都能做它们名称所表达的事情。这高度限制了每个操作符的用途。使用 `not()` 返回阶乘结果显然违反了这个约定，我们不应该允许这种事情发生。

### 不明确的例子

最大的问题是不清楚某些用法是否符合规定。举个例子，当我们用函数乘3表示什么呢？对一些人来说，很明显它是用来生成一个新的函数，这个函数的作用是被调用函数重复执行三次：

```kotlin
operator fun Int.times(operation: () -> Unit): ()->Unit ={ 
    repeat(this) { operation() } 
}

val tripledHello = 3 * { print("Hello") }

tripledHello() // Prints: HelloHelloHello
```

对于另一些人来说，可能意味着：要立即调用这个函数三次：

```kotlin
operator fun Int.times(operation: ()->Unit) {
    repeat(this) { operation() }
}

3 * { print("Hello") } // Prints: HelloHelloHello
```

当意义不明确时，最好使用带描述性的扩展函数。如果我们想让它们像运算操作符一样使用，可以使其变成中缀的形式

```kotlin
infix fun Int.timesRepeated(operation: ()->Unit) = {
    repeat(this) { operation() }
}

val tripledHello = 3 timesRepeated { print("Hello") }

tripledHello() // Prints: HelloHelloHello
```

有时使用顶层函数会更好，将函数重复执行3次的函数，已经在标准库中实现并发布：

```kotlin
repeat(3) { print("Hello") } // Prints: HelloHelloHello
```

### 什么时候可以打破这个规则？

有一种非常重要的场景，允许让我们去重载奇奇怪怪的运算操作符：当我们在设计领域特定语言（DSL）时。想想一个经典的 HTML DSL 例子：

```html
body {
    div {
        +"Some text"
    }
}
```

可以看到，要将文本添加到元素中，我们使用了 `String.unaryPlus`，这是可以接受的，因为它是 DSL 的一部分。在特定的上下文中，适用不同的规则对读者来说并不稀奇。

### 总结

我们应当切实地重载操作符。函数名应该与其行为保持一致，避免操作符含义不明确的情况。通过使用带有描述性名称的常规函数来使其含义清晰。如果你希望其用法类似于运算符（例如加减乘除），那么可以使用中缀修饰符或顶层函数。
