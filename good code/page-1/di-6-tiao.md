# 第 6 条： 优先使用标准错误，而不是自定错误

`require`、`check`、`assert` 代码块涵盖了 Kotlin 最常见的错误。但是可能还有其他类型的异常情况需要抛出。例如，正当你使用一个库来解析 JSON 格式数据，而将被解析的 JSON 文件格式不正确时，会抛出 `JsonParsingException` 异常，这是合理的：

```kotlin
inline fun <reified T> String.readObject(): T {
    // ...
    if (incorrectString) {
        throw JsonParsingException()
    }
    // ...
    return result
}
```

这里我们使用了一个自定义的错误，因为标准库中没有合适的错误来表达这种情况。**你应该尽可能的使用来自标准库中的异常，而不是自定义的异常**。因为像这些标准异常是开发人员已知的，所以也会去重用它们。重用已知元素和完善合约可以使你的 API 更容易学习和理解，下面是一些你可以使用的最常见的异常：

* `IllegalArgumentException` 和 `IllegalStateException`：在我们使用 `require` 、 `check` 函数时被抛出，如_第5条：指明你对参数和状态的期望_所述
* `IndexOutOfBoundsException`： 索引参数超出了范围，特别是在集合和数组这种结构上会使用，比如可能会在使用 `ArrayList.get(Int)` 时抛出
* `ConcurrentModificationException`：表示禁止并发修改，在侦查到这样的操作时，会抛出异常
* `UnsupportedOperationException`：声明的方法不被对象支持，应该避免这种情况，当一个方法不受支持时，它不应该出现在这个类中
* `NoSuchElementException`：表示被请求的元素不存在，例如，当我们使用 `Iterable` 时，若已没有元素，继续调用 `next` 将会抛出这个异常。
