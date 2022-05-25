# 第13条：避免返回或操作 Unit?

我的一好朋友在面试中被问到一个问题：“如果一个函数返回 `Unit?` ，它是要表达什么意思呢？”。好吧， `Unit?` 只有两个可能的值： `Unit` 或者 null。就像 `Boolean` 可以是 true 或者 false 一样。因此，这些类型都是同构的，它们可以互换使用。但我们为什么要用 `Unit` ，而不是 `Boolean`呢？例如下面代码：

```kotlin
fun keyIsCorrect(key: String): Boolean = //...

if(!keyIsCorrect(key)) return
```

我们也可以使用 `Unit?` 来做到：

```kotlin
fun verifyKey(key: String): Unit? = //...

verifyKey(key) ?: return
```

这似乎是预期的答案。但我朋友在这次面试中没有思考到一个更重要的问题：“但我们应该这样做吗？”，这个技巧在我们编写代码时候看起来很好，但是在读代码时就不一定了。使用 `Unit?` 来表示逻辑值具有误导性，可能导致难以检测的错误。我们已经讨论过这种表达方式会令人吃惊：

```kotlin
getData()?.let{ view.showData(it) } ?: view.showError()
```

当 `getData` 返回非空，而`showData` 返回 null 值，`showData()` 和 `showError` 都会被调用。 使用标准 `if..else` 可以更容易的理解这段代码：

```kotlin
if (person != null && person.isAdult) {
    view.showPerson(person)
} else {
    view.showError()
}
```

我们再来比较下面两种方式：

```kotlin
if(!keyIsCorrect(key)) return

verifyKey(key) ?: return
```

我从来没就没有发现过哪段代码使用 `Unit?` 能让代码更容易阅读，这种写法会产生误导和混乱。它几乎总是应该被 `Boolean` 值来替换，这就是为什么我们应该避免返回或操作 `Unit?` 类型。
