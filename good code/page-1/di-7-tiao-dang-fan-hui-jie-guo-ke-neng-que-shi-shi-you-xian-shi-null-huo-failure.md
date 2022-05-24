# 第7条：当返回结果可能缺失时，优先使 null 或 Failure

有时候，一个方法可能不会产出我们想要的结果，一些常见的例子是：

* 试图从服务器获取一些数据，但是网络连接出现了问题
* 试图从列表中获取符合某些条件的第一个元素，但是遍历完后发现没有这样的元素
* 试图从文本中解析一个对象，但文本格式不正确

有两种主要的措施，来处理这种问题

* 返回一个 null 或密封类，表示失败（通常表示 `Failure`）
* 抛出一个异常

这两者之间有一个重要的区别。**异常不应该用作信息传递的标准方式，所有异常都应该表示不正确的特殊情况**。我们应该只在异常情况下使用 Exception（这是 Joshua Bloch 在 《Effective Java》中提出的观点）。主要原因有：

* 对于大多数程序员来说，传递异常将会让代码可读性变差，并且很容易在代码中被忽略
* 在 Kotlin 中，所有异常都是未经检查的，用户不会被强迫甚至鼓励去使用这些异常。它们通常没有很好的文档来记录，当我们使用 API 时，它们实际是不可见的
* 因为 Exception 是为异常行为情况设计的，所以 JVM 实现者几乎没有动力让它们像测试代码一样那么快被抛出
* 将代码放在 `try-catch` 代码块中，可能会阻止编译器进行某些优化

另一方面，**`null` 和 `Failure` 都非常适合表示预期的错误，它们是显式的、有效的，并且可以按照习惯的编程方式来处理，这就是为什么我们应该在出现意料之内的错误时， 返回 null 和 Failure，而在出现意料之外的错误时候抛出异常**。这里有些例子：

```
inline fun <reified T> String.readObjectOrNull(): T? {
    //...
    if(incorrectSign) {
        return null
    }
        //...
    return result
}

inline fun <reified T> String.readObject(): Result<T> {
    //...
    if(incorrectSign) {
        return Failure(JsonParsingException())
    }
    //...
    return Success(result)
}
sealed class Result<out T>
class Success<out T>(val result: T): Result<T>()
class Failure(val throwable: Throwable): Result<Nothing>()

class JsonParsingException: Exception()
```

以这种方式表示的错误将更容易被外部处理，而且更难以被忽略。当我们使用 null 时，处理此类值的客户端可以从各种空安全支持特性中进行选择，例如使用 判空处理 或 `Elvis` 操作符：

```
val age = userText.readObjectOrNull<Person>()?.age ?: -1
```

当选择返回类似 `Result` 这种密封类型时，用户可以使用 `when` 表达式来处理：

```
val personResult = userText.readObject<Person>()
val age = when(personResult) {
    is Success -> personResult.value.age
    is Failure -> -1
}
```

使用这种错误处理，不仅比 `try-catch` 块更加有效，而且通常更容易使用，也难以忽略。异常可能会被遗漏，并且会终止整个应用程序。而 null 值或一个密封的结果类需要显式的处理，但它不会中断应用程序的流程。

**对比可空的结果和密封的结果类，当需要在函数失败，并且需要传递失败信息的时候，我们应该优先选择后者，其次才为空**。请记住， Failure 可以存储你所需要的任何信息。

函数有两种常见的形态 —— 一种是预期可能会发生某些异常，另一种是认为异常是意外情况。\`List\` 则是一个很好的体现了这两种形态的例子：

* `get` 函数，我们希望能够获取某个位置的素，如果这个位置没有元素，函数会抛出 `IndexOutOfBoundsException`
* `getOrNull` 函数，我们预期如果某位置拿不到元素的话，函数将应该返回 null 给我们

它还支持其他选项，例如 `getOrDefault`，这些选项在某些情况下很有用。但通常情况下容易被替换为 `getOrNull` 和 `Elvis ?:` 操作符配合使用。

这是一个很好的实践，因为如果开发人员知道他们获取的是一个安全的元素，他们就不用被迫去处理这个可空的值。同时，如果他们对这个结果有任何疑问，它们可以使用 `getOrNull()` 来适当的处理这个可能缺失的值。
