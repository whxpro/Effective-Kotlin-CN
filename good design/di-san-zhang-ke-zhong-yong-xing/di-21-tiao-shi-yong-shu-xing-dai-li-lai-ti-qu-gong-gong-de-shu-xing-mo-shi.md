---
description: Edu
---

# 第21条 使用属性代理来提取公共的属性模式

Kotlin 引入的一个支持代码重用的新特性是属性委托。它为我们提供了一种提取共有属性行为的通用方法。一个重要的例子就是 lazy 属性 —— 在第一次使用属性时才会被初始化。 这种模式非常流畅，在不支持提取属性的语言（如 Java 或者 JavaScript）中，每次都要去实现它，而在 Kotlin 中，可以通过属性委托轻松做到。在 Kotlin 标准库中，我们可以找到 `lazy` 函数，它返回一个实现 `lazy` 属性模式的属性代理：

```kotlin
val value by lazy { createValue() }
```

这并不是唯一被重复使用的属性模式。另一个重要的例子是 `observable` 属性 —— 当它被更改时，它就会做一些事情。例如，假设你有一个列表适配器用于绘制一个列表，每当其中的数据发生变化时，我们就需要重新绘制已更改的项。或者你可能需要记录属性的所有更改，这两种情况都可以使用 stdlib 里的 observable 实现：

```kotlin
var items: List<Item> by
    Delegates.observable(listOf()) { _, _, _ ->
        notifyDataSetChanged()
    }

var key: String? by
    Delegates.observable(null) { _, old, new ->
        Log.e("key changed from $old to $new") 9 
    }
```

从语言的角度来说，lazy 和 observable 委托属性并不是特别的，它们之所以可以被提取出来，得归功于一种更通用的属性委托机制，这种机制也可以用来提取许多其他模式。 一个很好的例子是视图和资源的绑定、依赖注入（正式的服务定位）或者数据绑定。其中许多模式需要在 Java 中使用注解来处理，但是 Kotlin 允许你使用简单、类型安全的属性委托机制来替代它们。

```kotlin
// View and resource binding example in Android
private val button: Button by bindView(R.id.button)
private val textSize by bindDimension(R.dimen.font_size)
private val doctor: Doctor by argExtra(DOCTOR_ARG)

// Dependency Injection using Koin
private val presenter: MainPresenter by inject()
private val repository: NetworkRepository by inject()
private val vm: MainViewModel by viewModel()

// Data binding
private val port by bindConfiguration("port")
private val token: String by preferences.bind(TOKEN_KEY)
```

为了理解这是如何实现的，以及如何使用委托属性提取其它常见行为，让我们从一个非常简单的属性委托开始。假设我们需要跟踪一些属性是如何使用的，为此，我们定义了自定义的 getter 和 setter 来记录它们的变化：

```kotlin
var token: String? = null
    get() {
        print("token returned value $field") 
        return field
    }
    set(value) {
        print("token changed from $field to $value") 
        field = value
    }

var attempts: Int = 0
    get() {
        print("attempts returned value $field")
        return field
    }
    set(value) {
        print("attempts changed from $field to $value")
        field = value
    }
```

尽管它们的类型不同，但这两个属性的行为几乎相同。 这似乎是一个可重复的模式，在我们项目中可能会有许多地方需要它，因此可以使用属性委托来提取此行为。 委托基于这样一种思想：属性是有它的访问器定义的 —— `val` 中的 getter 和 `var` 中的 getter 和 setter —— 这些方法可以委托给另一个对象的方法。 getter 将被委托给 `getValue` 函数， 而 `setter` 将被委托给 `setValue` 函数。然后我们将这样的对象放在 by 关键字的右侧，为了和上面的例子保持完全相同的属性行为，我们可以创建下面的委托：

```kotlin
var token: String? by LoggingProperty(null)
var attempts: Int by LoggingProperty(0) 

private class LoggingProperty<T>(var value: T) {
    
    operator fun getValue(
        thisRef: Any?,
        prop: KProperty<*>
    ): T {
        print("${prop.name} returned value $value")
        return value
    }

    operator fun setValue(
        thisRef: Any?,
        prop: KProperty<*>,
        newValue: T
    ) {
        val name = prop.name
        print("$name changed from $value to $newValue")
        value = newValue
    }
}
```

要充分理解属性委托的工作原理，请查看被编译为什么。上面的 `token` 属性将被编译成类似于下面的代码：

```kotlin
@JvmField
private val `token$delegate` = LoggingProperty<String?>(null) 

var token: String?
    get() = `token$delegate`.getValue(this, ::token)
    set(value) {
        `token$delegate`.setValue(this, ::token, value)
    }
```

正如你所见到的，`getValue` 和 `setValue` 方法不仅针对值进行操作， 而且它们还接收对属性的有界引用和上下文（this）。对属性的引用通常用于获取其名称，有时也获取其注解信息。 Context 为我们提供了函数被使用时的上下文信息。

当我们有多个 getValue 和 setValue 方法，但有不同的上下文类型时，在不同的情况下会选择不同的方法。这一事实可以用更聪明的方法加以利用。例如，我们可能需要一个可以在不同类型的视图中使用的委托，但对于每一个委托，它的行为都应该根据上下文提供不同的内容：

```kotlin
class SwipeRefreshBinderDelegate(val id: Int) {
    private var cache: SwipeRefreshLayout? = null
    
    operator fun getValue(
        activity: Activity,
        prop: KProperty<*>
    ): SwipeRefreshLayout {
        return cache ?: activity
            .findViewById<SwipeRefreshLayout>(id)
            .also { cache = it }
        }
    
    operator fun getValue(
        fragment: Fragment,
        prop: KProperty<*>
    ): SwipeRefreshLayout {
        return cache ?: fragment.view
                .findViewById<SwipeRefreshLayout>(id)
                .also { cache = it }
            }
}
```

为了使对象可以用作属性委托，它只需要 `val` 的 getValue 操作符， `var` 的 getValue 和 setValue 操作符。 这些操作符可以是成员函数，但也可以是扩展函数， 例如 `Map<String, *>` 可以用作属性委托。

```kotlin
val map: Map<String, Any> = mapOf(
    "name" to "Marcin", 
    "kotlinProgrammer" to true
) 

val name by map
print(name) // Marcin
```

这是可以的，因为在 Kotlin stdlib 中有以下扩展函数：

```kotlin
inline operator fun <V, V1 : V> 
    Map<in String, V>.getValue(thisRef: Any?, property: KProperty<*>): V1 = 
        getOrImplicitDefault(property.name) as V1
```

Kotlin 标准库中有一些我们应该知道的委托属性，例如：

* `lazy`
* `Delegates.observable`
* `Delegates.vetoable`
* `Delegates.notNull`

了解它们是有意义的，如果你注意到项目中有地方会围绕属性的一个常见模式，请记住，你可以自己创建自己的属性委托。

### 总结

属性委托完全控制属性， 并拥有属性关于上下文几乎所有的信息。 这个特性实际上可以用来提取仍和属性行为， lazy 、 observable 只是标准库中的两个例子。 属性委托是一个通用的提取属性模式的方法， 实践表明，有各种各样的属性模式。 它是一个强大的特性，每个 Kotlin 的开发工具中应该有它， 当我们知道这一点时，我们就有了更多的通用模式提取或定义更好的 Api 选项。
