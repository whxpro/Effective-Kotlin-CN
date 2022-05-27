# 第43条： 考虑将 API 的非必要部分提取到扩展函数中

在类中定义 `final` 方法时，我们需要决定将其定义为成员函数，还是扩展函数。

```kotlin
// 定义方法为成员
class Workshop(/*...*/) {
    //...
    
    fun makeEvent(date: DateTime): Event = //...
    
    val permalink
        get() = "/workshop/$name"
}
```

```kotlin
// 定义方法为扩展
class Workshop(/*...*/) {
    //...
}

fun Workshop.makeEvent(date: DateTime): Event = //...

val Workshop.permalink
    get() = "/workshop/$name"
```

这两种方式在许多方面是相似的，它们的使用，甚至通过反射引用它们都是非常相似的：

```kotlin
fun useWorkshop(workshop: Workshop) {
    val event = workshop.makeEvent(date)
    val permalink = workshop.permalink
    
    val makeEventRef = Workshop::makeEvent
    val permalinkPropRef = Workshop::permalink
}
```

但它们也有显著的差异，有各自的优点和缺点，一种方式不会凌驾于另一种方式之上。这就是为什么我的建议是考虑这么做，而不是一定要这么做。关键是要明智地做出决定，为此，我们需要了解其中的差异。

在使用方面，成员和扩展之间最大的区别就是扩展需要单独导入。由于这个原因，它们可以位于不同的包中。当我们不能添加成员时，就会使用到这个机制。它也用于旨在分离数据和行为的项目。带有字段的属性需要位于类中，但方法只要访问的是这个类的公有api，它就能位于别的文件中。

由于需要导入扩展，我们可以在同一类型上有许多同名的扩展，这是很好的，因为不同的库可以提供额外的方法，而且不会产生冲突。另一方面，如果两个扩展名相同，但行为不同，这就是危险的。对于这种情况，我们可以通过创建成员函数来解决问题。编译器总是选择成员函数而不是扩展函数。

另一个重要的区别是扩展不能是抽象的，这意味着它不能在派生类中重新定义。要调用的扩展函数是在编译期间静态选择的。这与 Kotlin 中抽象成员行为不同。因此，我们不应该对那些为了继承而设计的元素使用扩展。

```kotlin
open class C
class D: C()
fun C.foo() = "c"
fun D.foo() = "d"

fun main() {
    val d = D()
    print(d.foo()) // d
    val c: C = d
    print(c.foo()) // c
    
    print(D().foo()) // d
    print((D() as C).foo()) // c
}
```

这个行为结果的原因是：扩展函数在底层被编译成普通函数，扩展的接收者被放置在第一个参数中：

```kotlin
fun foo(`this$receiver`: C) = "c"
fun foo(`this$receiver`: D) = "d"

fun main() {
    val d = D()
    print(foo(d)) // d
    val c: C =d
    print(foo(c)) // c
    
    print(foo(D())) // d
    print(foo(D() as C)) // c
 }
```

这一行为所体现出的另一个机制是：我们是在为类型定义扩展，而不是为类定义。这给了我们更多自由，例如，可以在为可空或具体类型的泛型上定义扩展：

```kotlin
inline fun CharSequence?.isNullOrBlank(): Boolean {
    contract {
        returns(false) implies (this@isNullOrBlank != null)
    }
    
    return this == null || this.isBlank()
}

public fun Iterable<Int>.sum(): Int {
    var sum: Int = 0
    for (element in this) {
        sum += element
    }
    return sum
}
```

最后一个重要的区别是扩展函数、扩展属性在类引用中不是作为成员列出的。这就是为什么注解处理器不考虑使用它们，以及为什么在使用注解处理类时，我们不能处理那些扩展函数。换个角度说，如果我们将非必须的元素提取到扩展中，我们就不需担心它们被那些处理器看到，我们不需要隐藏它们，因为它们不在类的任何的地方。

### 总结

成员和扩展之间最重要的区别是：

* 需要导入扩展
* 扩展不是抽象的
* 成员有更高的优先级
* 扩展是在一个类型上，而不是一个类上
* 类引用不会列出那些扩展的名称

总而言之，扩展给了我们更多的自由和灵活性。它们比较模棱两可，尽管它们不支持继承、注解处理，而且它们没有出现在类中会让人感到困惑。我们 API 的基本部分应该作为成员，但有更好的理由把那些非必要的部分提取到扩展中。
