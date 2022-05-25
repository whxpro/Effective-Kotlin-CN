# 第17条：考虑使用具名参数

当阅读代码时，你并不是总能理解每个参数是做什么的，请看下面一个例子：

```kotlin
val text = (1..10).joinToString("|")
```

"|" 是什么？ 如果你了解 `joinToString` 这个函数的话，你就知道它表示 `separator`，尽管也可以表示 `prefix`，但这一点也不清晰。我们可以通过让那些没有明确表明其含义的参数更加清晰，以提高可读性，最好的方法就是使用具名参数：

```kotlin
val text = (1..10).joinToString(separator = "|")
```

我们可以通过给变量命名来达到类似的效果：

```kotlin
val separator = "|"
val text = (1..10).joinToString(separator)
```

虽然给参数命名很可靠，因为变量名表达了开发人员的意图，但这不一定是最正确的。如果开发人员犯了一些错误，把入参放在了错误的位置，该怎么办？ 如果函数的入参顺序改变了，又该怎么办呢？ 具名参数可以避免这种情况，而传入被命名的变量却不能。这就是为什么当我们有命名的变量时，使用具名参数依然是合理的：

```kotlin
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```

### 什么时候该使用命名参数？

显然，具名参数会让代码看起来比较长，但是它有两个重要的优点：

* 可以表示期望参数的名称
* 更安全，因为它们有顺序

参数名是非常重要的信息，不仅对于调用该函数的开发人员来说如此，对于阅读该函数的人亦是如此，来看下面这个调用：

```kotlin
sleep(100)
```

它会 `sleep` 多久呢？ 100ms？ 还是 100s？ 我们可以使用具名参数让代码清晰起来：

```kotlin
sleep(timeMillis = 100)
```

这种场景下，这样的方式并不是让代码清晰的唯一选择，在 Kotlin 这样的静态类型语言中，在传参时保护程序的第一个机制就是检查参数的类型，我们可以用该机制来确保时间单位的正确性：

```kotlin
sleep(Millis(100))
```

或者我们可以使用一个扩展属性来创造一个类似 DSL 的语法：

```kotlin
sleep(100.ms)
```

明确类型是传递此类消息的好方法，如果你关心其效率，可以使用内联类，如_第46条：给高阶函数使用 inline 修饰符_所述。它帮助我们实现参数安全，但这并不能解决所有与问题，有些参数可能还不清楚，一些参数可能会被放置在错误的地方上，这就是为什么我仍然建议考虑使用具名参数，特别是对以下这些参数：

* 有默认值的可选参数
* 与其它参数类型相同
* 是一个函数类型，而且它不是最后一个参数

### 有默认值的可选参数

当一个函数有默认的可选参数时，我们总是要使用具名参数来找到它。这样的可选参数比必选参数改动的更加频繁，我们不想错过那些改动。函数名通常指出其必选参数，而非可选参数，这就是为什么对可选参数具名更加安全，而且通常更简洁的原因。

### 与其它类型相同

如前面所述，当参数具有不同类型时，通常不会将参数放在错误的位置，但如果某些参数具有相同的类型，那情况就不同了：

```kotlin
fun sendEmail(to: String, message: String) { /*...*/ }
```

对于这样的函数，最好使用具名参数来使其明确：

```kotlin
sendEmail(
    to = "contact@kt.academy",
    message = "Hello, ..."
)
```

### 函数类型的参数

最后，我们应该特殊处理函数类型的参数。在 Kotlin 中，这些参数有一个特殊的位置：最后一个位置。有时函数名描述了函数类型的参数，例如，当我们看到 `repeat` 时，我们预期后面的 lambda 块就是我们需要重复的代码。当你看到 `thread` 时，很直观就能看到后面紧跟的代码块就是这个线程的主体。这些名称描述的是最后一位置使用的函数。

```kotlin
thread {
    // ...
}
```

而所有其他具有函数类型参数的函数都应该命名，因为很容易误解它们，例如，来看下简单的视图 DSL：

```kotlin
val view = linearLayout {
    text("Click below")
    button({ /* 1 */ }, { /* 2 */ })
}
```

对于 button 来说，哪个函数是 builder 的一部分？哪个函数是单击监听器？我们应该通过具名监听器，和将构建器移到参数之外来提高代码的可读性：

```kotlin
val view = linearLayout {
    text("Click below")
    button(onClick = { /* 1 */ }) {/* 2 */}
    }
```

函数类型的多个可选参数特别容易让人混淆：

```kotlin
fun call(before: ()->Unit = {}, after: ()->Unit = {}) {
    before()
    print("Middle")
    after()
}

call({ print("CALL") }) // CALLMiddle
call { print("CALL") } // MiddleCALL
```

为了防止这种情况，当有传入多个有不同意义的函数参数时，应该都为它们命名:

```kotlin
call(before = { print("CALL") }) // CALLMiddle
call(after = { print("CALL") }) // MiddleCALL
```

对于响应式库尤其如此，例如在 RxJava 中，当我们观察一个 `Observable` 时，可以设置一些被回调的函数：

```java
// Java
observable.getUsers()
          .subscribe((List<User> users) -> { // onNext
              // ...
          }, (Throwable throwable) -> { // onError
              // ...
          }, () -> { // onCompleted
              // ...
          });
```

在 kotlin 中，可以更进一步，使用命名参数：

```kotlin
observable.getUsers()
    .subscribeBy(
        onNext = { users: List<User> ->
             // ...
        },
        onError = { throwable: Throwable ->
             // ...
        },
        onCompleted = {
             // ...
        })
```

注意这里，我将函数名从 `subscribe` 改成了 `subscribeBy`。这是因为 RxJava 是用 Java 编写的，我们在调用 Java 函数时不能使用具名参数，这是因为 Java 不保留函数名的信息。为了能够使用具名参数，我们通常需要为这些函数创建 Kotlin 装饰器（（使用扩展函数替换这些函数）。

### 总结

具名参数不仅仅在使用可选参数时有用，对于阅读这些代码的开发人员来说也是很重要的，它可以提高我们代码的安全性。我们应该考虑使用它们，特别是当有更多相同类型（或者函数类型）的参数作为可选参数时。当有多个函数类型参数时，它们几乎总是应该被命名的。比较特殊的就是函数类型作为最后一个参数的时候，例如 DSL。
