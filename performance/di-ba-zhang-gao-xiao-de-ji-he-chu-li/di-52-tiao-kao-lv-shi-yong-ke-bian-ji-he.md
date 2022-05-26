# 第52条：考虑使用可变集合

使用可变集合而非不可变集合的最大优点是在性能方面更快。当向不可变集合添加元素时，需要创建一个新的集合并向其中添加所有元素。以下是目前在 Kotlin stdlib（Kotlin 1.2）中实现它的方式：

```kotlin
operator fun <T> Iterable<T>.plus(element: T): List<T> {
    if (this is Collection) return this.plus(element)
    val result = ArrayList<T>()
    result.addAll(this)
    result.add(element)
    return result
}
```

在处理更大的集合时，从以前的集合中添加所有元素是一个代价高昂的过程。这就是为什么使用可变集合（特别是在需要添加元素时）是一种性能优化。另一方面，_第1条：限制可变性_教会了我们在安全方面使用不可变集合的优点。但是请注意，这些参数很少应用于不需要同步或封装的局部变量。**这就是为什么对于本地处理，通常使用可变集合更有意义**。这一点可以在标准库中得到反映，所有的集合处理函数都是使用可变集合在内部实现的：

```kotlin
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
    val size = if (this is Collection<*>) this.size else 10
    val destination = ArrayList<R>(size)
    for (item in this)
        destination.add(transform(item))
    return destination
}
```

替代不可变集合的实现：

```kotlin
// 这不是 map 实际如何实现的
inline fun <T, R> Iterable<T>.map(
    transform: (T) -> R
): List<R> {
    var destination = listOf<R>()
    for (item in this)
        destination += transform(item)
    return destination
}
```

### 总结

添加到可变集合通常更快，但不可变集合让我们能够更好的控制如何更改它们。尽管在局部作用域中，我们通常不需要这种控制，所以可变集合应该是首选。特别是在 `utils` 工具类中，元素的插入操作可能会发生多次。
