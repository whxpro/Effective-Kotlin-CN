# 第44条：避免在成员中定义扩展

当我们为某个类定义扩展函数时，它不会作为成员添加到这个类中。扩展函数是一种不同的函数，我们调用第一个参数，叫做接收者。在底层，扩展函数被编译为普通函数，接收者作为第一个参数，例如下面的函数：

```kotlin
fun String.isPhoneNumber(): Boolean =
    length == 7 && all { it.isDigit() }
```

在底层，它被编译为类似下面这段的函数：

```kotlin
fun isPhoneNumber(`$this`: String): Boolean =
    `$this`.length == 7 && `$this`.all { it.isDigit() }
```

了解这些后，我们可以定义成员扩展函数，甚至在接口中定义它：

```kotlin
interface PhoneBook {
    fun String.isPhoneNumber(): Boolean
}

class Fizz: PhoneBook {
    override fun String.isPhoneNumber(): Boolean =
        length == 7 && all { it.isDigit() }
}
```

即使这是可以做到的，但我们有很好的的理由去避免定义成员扩展函数/属性（DSL除外）。特别是，不要仅仅为了限制可见性而将扩展定义为成员。

```kotlin
// 不好的做法，千万不要这样做
class PhoneBookIncorrect {
    // ...
    
    fun String.isPhoneNumber() =
        length == 7 && all { it.isDigit() }
}
```

一个很大的原因是它并没有真正限制可见性，这只会让扩展函数变得更加复杂，因为用户需要同时提供扩展和分发接收者：

```kotlin
PhoneBookIncorrect().apply { "1234567890".test() }
```

你应该使用可见性修饰符来限制扩展的可见性，而不是将其作为成员。

```kotlin
// 这是我们如何给扩展函数限制可见性的
class PhoneBookCorrect {
    // ...
}

private fun String.isPhoneNumber() =
    length == 7 && all { it.isDigit() }
```

我们倾向避免扩展成员有几个很好的理由：

* 这样做不支持引用：

```kotlin
val ref = String::isPhoneNumber
val str = "1234567890"
val boundedRef = str::isPhoneNumber

val refX = PhoneBookIncorrect::isPhoneNumber // ERROR
val book = PhoneBookIncorrect()
val boundedRefX = book::isPhoneNumber // ERROR
```

* 对有两个接收者的隐式访问可能会令人困惑：

```kotlin
class A {
    val a = 10
}
class B {
    val a = 20
    val b = 30
    
    fun A.test() = a + b // 它是40还是50？
}
```

* 当我们期望一个扩展去修改或引用一个接收者时，我们不清楚这个扩展函数作用在哪个接收者上：

```kotlin
class A {
    //...
}
class B {
    //...
    fun A.update() = ... // 这个 update 函数是针对A的，还是针对B的？？？
}
```

* 对于经验较少的开发者来说，看到成员扩展可能会违反直觉或让人感到害怕

总而言之，如果有很好的理由去使用成员扩展，这还好。只是要意识到它所到来的缺点，能避免就尽量避免。要限制可见性，请使用可见性修饰符。仅仅在类中放置一个扩展并不会限制它在外部的使用。
