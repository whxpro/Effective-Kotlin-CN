---
description: Edu
---

# 第24条：在使用泛型时考虑型变

假设我们有下面的泛型：

```kotlin
class Cup<T>
```

上面声明的类型参数 `T` 没有任何的型变修饰符（`out` 或者 `in`），默认情况下，它是不变的（invariant），它意味着这个泛型类所生成的任何两个类是没有联系的。例如 `Cup<Int>` 和 `Cup<Number>`、`Cup<Any>` 和 `Cup<Nothing>` 之间没有关系。

```kotlin
fun main() {
    val anys: Cup<Any> = Cup<Int>() // Error: Type mismatch
    val nothings: Cup<Nothing> = Cup<Int>() // Error
}
```

如果我们需要这样的关系，那么我们应该使用型变: `out` 或 `in`， out 是协变（covariant），这意味着当 A 是 B 的子类且 `Cup` 为协变（被 `out` 修饰）的，那么 `Cup<A>` 就是 `Cup<B>` 的子类：

```kotlin
class Cup<out T>
open class Dog
class Puppy: Dog()

fun main(args: Array<String>) {
    val b: Cup<Dog> = Cup<Puppy>() // OK
    val a: Cup<Puppy> = Cup<Dog>() // Error
    
    val anys: Cup<Any> = Cup<Int>() // OK
    val nothings: Cup<Nothing> = Cup<Int>() // Error
}
```

使用 `in` 修饰可以达到相反的效果，它使类型参数逆变（contravariant）。 这意味着当 A 是 B 的子类且 `Cup` 为逆变的，则 `Cup<A>` 是 `Cup<B>` 的超类：

```kotlin
class Cup<in T>
open class Dog
class Puppy(): Dog()

fun main(args: Array<String>) {
    val b: Cup<Dog> = Cup<Puppy>() // Error
    val a: Cup<Puppy> = Cup<Dog>() // OK
    
    val anys: Cup<Any> = Cup<Int>() // Error
    val nothings: Cup<Nothing> = Cup<Int>() // OK
}
```

这些型变修饰符的关系如下图所示：

![](<../../.gitbook/assets/image (2) (1).png>)

### 函数类型

在函数类型中（详见_第35条：考虑为复杂的对象创建定义 DSL_），函数类型之间存在不同预期参数类型或返回类型的关系。 为了更贴合实际的去理解它，想象有这样一个函数：它期望一个接受 `Int` 并返回 `Any` 的函数做为参数：

```kotlin
fun printProcessedNumber(transition: (Int)->Any) {
    print(transition(42))
}
```

基于其定义，这样的函数可以接受类型为 `(Int) -> Any` 的函数，但它也适用于 `(Int)->Numebr`、`(Number)->Any`、`(Number)->Numebr`、`(Number)->Int`，等等。

```kotlin
val intToDouble: (Int) -> Number = { it.toDouble() }
val numberAsText: (Number) -> Any = { it.toShort() }
val identity: (Number) -> Number = { it }
val numberToInt: (Number) -> Int = { it.toInt() }
val numberHash: (Any) -> Number = { it.hashCode() }
printProcessedNumber(intToDouble)
printProcessedNumber(numberAsText)
printProcessedNumber(identity)
printProcessedNumber(numberToInt)
printProcessedNumber(numberHash)
```

这是因为这些存在如下关系:

![](<../../.gitbook/assets/image (1) (1) (1).png>)

请注意，当我们在这个层次结构中从上往下走时，参数类型：向类型系统层次结构更高的类型移动，而返回类型：向更低的类型移动。

![](<../../.gitbook/assets/image (4) (1).png>)

这并非巧合，**Kotlin 函数类型中所有参数类型都是逆变的**，就跟型变修饰符的名称 `in` 所表明的那样。 **Kotlin 函数类型中的所有返回类型都是协变的**，正如 `out` 这个型变修饰符的名称所表明的那样。

![](<../../.gitbook/assets/image (7) (1) (1) (1).png>)

当我们使用函数类型时，这一特性为我们提供了支持，但它不是唯一带有型变修饰符的主流 Kotlin 类型，一个更流行的是 `List`，它在 Kotlin 是协变的（使用了 `out` 修饰符）。不像 `MutableList`，它是不变的（没有型变修饰符）。下面我们需要了解型变修饰符的安全性。

### 型变修饰符的安全性

在 Java 中，数组是协变的。许多消息来源称，这一决定背后的原因是希望可以为每一种类型的数组都创建出像排序这样泛型操作的函数。但这一决定存在一个大问题，为了理解它，让我们先来分析下面这个合法的代码，它们不会产生编译错误，但是会抛出运行时错误：

```
// Java
Integer[] numbers = {1, 4, 2, 1};
Object[] objects = numbers;
objects[2] = "B"; // Runtime error: ArrayStoreException
```

如你所见，将数字数组转化成 `Object[]` 并没有改变其内部结构，它使用的类型仍然是 `Integer`，因此当我们试图将 String 类型的值赋给这个数组时，就会产生错误。这显然是一个 Java 的缺陷， Kotlin 通过使 `Array`（以及 `IntArray`、`CharArray`等）不型变（因此无法将 `Array<Int>` 推到 `Array<Any>`）来保护我们。

为了理解这里出了什么问题，我们应该首先认识到，当期望一个参数类型时，我们也可以传递该类型的任意子类型。因此，当我们传递一个参数时，可以进行隐式上推：

```kotlin
open class Dog
class Puppy: Dog()
class Hound: Dog()

fun takeDog(dog: Dog) {}

takeDog(Dog())
takeDog(Puppy())
takeDog(Hound())
```

这和协变是不相容的： 如果一个协变类型参数（`out`） 出现在 `in` 所修饰的位置（例如函数的入参），通过协变和向上转化，我们可以传递任何想要的类型，显然这是不安全的，因为值的类型是非常具体的类型，所以当它的类型是 `Dog` 时，它不能去存一个 `String`

```kotlin
class Box<out T> {
    private var value: T? = null
    
    // Illegal in Kotlin
    fun set(value: T) {
        this.value = value
    }
    
    fun get(): T = value ?: error("Value not set")
}

val puppyBox = Box<Puppy>()
val dogBox: Box<Dog> = puppyBox
dogBox.set(Hound()) // But I have a place for a Puppy

val dogHouse = Box<Dog>()
val box: Box<Any> = dogHouse
box.set("Some string") // But I have a place for a Dog
box.set(42) // But I have a place for a Dog
```

这种情况是不安全的，因为在类型转化之后，实际的对象还是不变的，只是类型系统对其可以进行不同的处理。 我们试图将 `Int` 设置进来，但是我们只能设置一个 `Dog`，如果这可以通过编译的话，我们肯定会出现错误。 这就是为什么 Kotlin 通过禁止在一个 public 的 `in` 修饰的地方使用有协变修饰符的参数来防止这种情况出现。

```kotlin
class Box<out T> {
    var value: T? = null // Error
    
    fun set(value: T) { // Error
        this.value = value
    }
    
    fun get(): T = value ?: error("Value not set")
}
```

当我们将其可见性修饰为 private 时， 这是合法的，因为在对象内部我们是无法使用协变对其向上转型的：

```kotlin
class Box<out T> {
    private var value: T? = null
    
    private set(value: T) {
        this.value = value
    }
    
    fun get(): T = value ?: error("Value not set")
}
```

在 public `out`修饰的位置下使用协变（`out`修饰符）是完全安全的，所以它们不受限制。这就是为什么我们对那些返回类型或者公开的类型使用协变（`out`修饰符），它通常是生产者或不可变数据持有者。

一个很好的例子是 `List<T>`，其中 `T` 在 Kotlin 中是协变的，多亏了这一点，当函数期望 `List<Any?>` 时，我们可以提供任意子类型的列表而用做任何转换。在 `MutableList<T>` 中， `T` 则是不能变的，因为它是在 `in` 修饰的地方，它并不安全：

```kotlin
fun append(list: MutableList<Any>) {
    list.add(42)
}

val strs = mutableListOf<String>("A", "B", "C")
append(strs) // Illegal in Kotlin
val str: String = strs[3]
print(str)
```

另一个很好的例子是 `Response`，使用协变后收益很多。你可以在下面的代码块中看到如何使用它们，多亏了型变修饰符，以下期望的事情得以实现：

* 当我们期望 `Response<T>` 时，任何 `T` 的子类型的 `Response`都能被接受。例如当期望 `Response<Any>` 时，我们接受 `Response<Int>` 或者 `Response<String>`
* 当我们期望接受 `Response<T1, T2>` 时，任何 T1 和 T2 的子类型的 Response 都能被接受
* 当我们期望 `Failure<T>` 时， `Failure` 含住 T 的任意子类型都将被接受，例如，当期望 `Failure<Number>` 时，接受 `Failure<Int>` 或 `Failure<Double>`。当预期 `Failure<Any>` 时，我们接受 `Failure<Int>` 和 `Failure<String>`
* `Success` 不需要指定潜在错误的类型， Failure 不需要指定潜在成功值的类型，我们可以使用协变和 `Nothing` 类型来达到目标

```kotlin
sealed class Response<out R, out E>
class Success<out R>(val value: R): Response<R, Nothing>()
class Failure<out E>(val error: E): Response<Nothing, E>()
```

当我们在一个 public 的 `out` 修饰的位置（函数的返回类型或属性类型） 上使用逆变类型参数时，也会出现像在 `in` 修饰位置使用协变类型时出现的问题。`out`修饰的位置也允许隐式的向上转型。

```kotlin
open class Car
interface Boat
class Amphibious: Car(), Boat

fun getAmphibious(): Amphibious = Amphibious()

val car: Car = getAmphibious()
val boat: Boat = getAmphibious()
```
