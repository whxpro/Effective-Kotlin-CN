# 第48条：消除过时的对象引用

习惯了具有自动管理内存的语言的程序员很少会考虑释放对象，例如在 Java 中，垃圾收集器（GC）可以完成所有工作。但是完全忘记内存管理会导致内存泄漏 —— 不必要的内存泄漏 —— 在某些情况下还会导致 OOM。最重要的一条规则是，我们不应该保留那些不再有用的对象的引用。特别是如果这样的对象很大，或者可能有很多这样的对象的实例。

在 Android 中，有一个初学者常见的错误是，开发者愿意从任何地方自由的访问一个 Activity（一个类似于桌面应用程序中的窗口的概念），将其存储在一个静态或伴生对象中（通常在一个静态字段中）：

```kotlin
class MainActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //...
        activity = this
    }
    
    //...
    companion object {
        // 千万不要这样做，会产生巨大的内存泄漏
        var activity: MainActivity? = null
    }
}
```

在伴生对象中，持有对 `Activity` 的引用不会让垃圾收集器在应用程序运行期间释放它， Activity 是一个很重的对象，因此会产生巨大的内存泄漏。有一些方法可以改进它，但最好根本不要静态地持有这些资源。请正确地管理依赖关系，而不是静态地存储它们。此外，请注意，当我们持有一个对象，它引用另一个对象时，我们可能会出现内存泄漏。向下面的例子一样，我们持有一个 lambda 函数，它捕获 MainActivity 的引用：

```kotlin
class MainActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //...
        // 要小心，这里的 this 引用被泄漏出去了
        logError = { Log.e(this::class.simpleName, it.message) }
    }
    
    //...
    
    companion object {
        // 千万不要这样做，会产生巨大的内存泄漏
        var logError: ((Throwable)->Unit)? = null
    }
}
```

不过存储方面的问题可能更加微妙，看看下面的栈实现：

```kotlin
class Stack {
    private var elements: Array<Any?> =
        arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
        
    private var size = 0 
    
    fun push(e: Any) {
        ensureCapacity()
        elements[size++] = e
    }
    
    fun pop(): Any? {
        if (size == 0) {
            throw EmptyStackException()
        }
        return elements[--size]
    }
    
    private fun ensureCapacity() {
        if (elements.size == size) {
            elements = elements.copyOf(2 * size + 1)
        }
    }
    companion object {
        private const val DEFAULT_INITIAL_CAPACITY = 16
    }
}
```

你能发现这段代码的问题吗？ 花一分钟想想。

这里的问题是，当我们调用 `pop` 时，我们只是减小了数组的大小， 而不是释放数组中的元素。假设栈中有 1000 个元素，我们一个接一个地取出了几乎所有的元素，现在数组大小等于 1， 我们应该只有一个元素，而且只能够访问这个元素。但实际上，我们的栈仍然保存着 1000 个元素，并且不允许 GC 去销毁它们。所有这些对象都在浪费我们的内存，这就是为什么这被称为内存泄漏。如果这个泄漏不断积累，我们可能会遇到 OOM。如何修复这个问题？ 一个非常简单的解决方案是，当一个对象不再使用时，在数组中设置 null：

```kotlin
fun pop(): Any? {
    if (size == 0)
        throw EmptyStackException()
    val elem = elements[--size]
    elements[size] = null
    return elem
}
```

这是一个很罕见的例子，但也是一个代价高昂的 Bug，我们的日常开发也可以从这个规则中获益。假设我们需要一个 `mutableLazy` 的属性委托。它应该像 `lazy` 一样工作，但它还应该允许属性状态变化，我们可以使用以下实现来定义它：

```kotlin
fun <T> mutableLazy(initializer: () -> T):
ReadWriteProperty<Any?, T> = MutableLazy(initializer)

private class MutableLazy<T>(
    val initializer: () -> T
) : ReadWriteProperty<Any?, T> {
    private var value: T? = null
    private var initialized = false
    
    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            if (!initialized) {
                value = initializer()
                initialized = true
            }
            return value as T
        }
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            initialized = true
        }
    }
}
```

Usage: 代码引用：

```kotlin
var game: Game? by mutableLazy { readGameFromSave() }

fun setUpActions() {
    startNewGameButton.setOnClickListener {
        game = makeNewGame()
        startGame()
    }
    resumeGameButton.setOnClickListener {
        startGame()
    }
}
```

上述的 `mutableLazy` 实现是正确的，但是会有一个缺陷， 在使用了 `initializer` 之后，`initializer` 没有被清理。这意味着只要 `MutableLazy` 实例的引用一直存在，它就会被保存，即使它不再有用了。以下是 `MutableLazy` 实现的优化：

```kotlin
fun <T> mutableLazy(initializer: () -> T):
ReadWriteProperty<Any?, T> = MutableLazy(initializer)
 
private class MutableLazy<T>(
    var initializer: (() -> T)?
) : ReadWriteProperty<Any?, T> {
    private var value: T? = null
    
    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            val initializer = initializer
            if (initializer != null) {
                value = initializer()
                this.initializer = null
            }
            return value as T
        }
    }
    
    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            this.initializer = null
        }
    }
}
```

当我们将 `initializer` 设置为 null 时，之前的值就能被 GC 所回收。

这种优化有多重要？对于很少使用的对象来说，问题没有那么重要。有一种说法是，过早的优化是万恶之源。不过，在不需要花费太多成本的情况下，最好将未使用的对象设置为 null。特别当它是一个包含许多变量的函数类型时，或者当它是一个未知的类（如 `Any` 或泛型时）。例如，上面的例子中 Stack 可能被某人用来存放很重的对象，这是一个通用的工具，我们不知道它将如何被使用。对于这样的工具，我们应该更多的关注性能方面的优化。在创建库时尤其如此。例如，在 Kotlin stdlib 的懒惰委托的三个实现中，我们可以看到 `initializer` 在使用后都会被置空：

```kotlin
private class SynchronizedLazyImpl<out T>(
    initializer: () -> T, lock: Any? = null
) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE
    
    private val lock = lock ?: this
    
    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }
            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }
    
    override fun isInitialized(): Boolean =
        _value !== UNINITIALIZED_VALUE
        
    override fun toString(): String =
        if (isInitialized()) value.toString()
        else "Lazy value not initialized yet."
    
    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```

一般的规则是，当我们持有状态时，我们应该在心中对其进行内存管理，在更改实现之前，我们应该始终考虑项目的最佳权衡，不仅要考虑内存和性能，还要考虑解决方案的可读性和可维护性。一般来说，可读性高的代码在性能或内存方面也会做得相当好，反之，不可读的代码更有可能隐藏内存泄漏或浪费 CPU 电源。虽然这两种价值观是对立的，但在大多数情况下，可读性更加重要，而当我们开发一个库时，性能和内存通常更重要。

我们需要讨论一些常见的内存泄漏来源。首先，缓存那些可能永远不会再使用的对象。这是缓存背后的设计思想，但当我们遇到内存不足的问题时，它并不会帮助我们。解决方案是使用软引用。如果需要内存，GC 仍然可以收集对象，但它们通常会存在并被使用。

最大的问题是：内存泄漏有时很难预测，并且可能直到应用程序崩溃才会显示出来。对于 Android 应用来说尤其如此，因为它对内存使用的限制要比桌面端等其他类型的客户端更加严格。这就是为什么我们需要使用特殊的工具来排查它们。最基本的工具就是堆分析器（heap profiler），还有一些库可以帮助我们检测内存泄漏，例如，Android 上一个很流行的库 LeakCanary，它会在检测到内存泄漏时展示一个通知。

重要的是要记住，手动释放对象的要求比较少见。在大多数情况下，由于作用域的关系，这些对象无论如何都会被释放的，或者当持有它们的对象被清理时也会被随之清理。避免内存混乱最重要的解决方法是在局部作用域中定义变量（_第2条：最小化变量的作用域_），并且在顶级属性或对象声明（包括伴生对象）中不去存储可能很重的数据。
