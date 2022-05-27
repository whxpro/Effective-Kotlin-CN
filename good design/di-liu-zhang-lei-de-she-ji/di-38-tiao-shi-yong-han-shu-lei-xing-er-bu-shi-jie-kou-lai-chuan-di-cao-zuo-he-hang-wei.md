# 第38条：使用函数类型而不是接口来传递操作和行为

很多语言没有函数类型的概念，取而代之的是使用带有单一方法的接口，这种接口被称为 SAM（单一抽象方法）。下面是一个 SAM 用来传递当视图被点击时的回调信息的例子：

```kotlin
interface OnClick {
    fun clicked(view: View)
}
```

当一个函数接收一个 SAM 时，我们就必须传递一个实现了这个接口的对象实例：

```kotlin
fun setOnClickListener(listener: OnClick) {
    //...
}

setOnClickListener(object : OnClick {
    override fun clicked(view: View) {
        // ...
    }
})
```

然而，请注意用函数类型声明的参数会给我们更多的自由：

```kotlin
fun setOnClickListener(listener: (View) -> Unit) {
    //...
}
```

现在，我们可以传递这样的参数：

* 一个 lambda 表达式或匿名函数

```kotlin
setOnClickListener { /*...*/ }
setOnClickListener(fun(view) { /*...*/ })
```

* 一个函数引用或有界函数引用

```
setOnClickListener(::println)
setOnClickListener(this::showUsers)
```

* 一个实现了这个被声明的函数类型的对象

```kotlin
class ClickListener: (View)->Unit {
    override fun invoke(view: View) {
        // ...
    }
}

setOnClickListener(ClickListener())
```

这些选项可以覆盖更广泛的用例范围。另一方面，有人可能会说 SAM 的优点在于它和它的参数都是命名的，请注意，我们也可以使用类型别名来命名函数类型：

```kotlin
typealias OnClick = (View) -> Unit
```

参数也可以被命名，命名它们的好处是：IDE 可以默认的建议展示出这些名称。

```kotlin
fun setOnClickListener(listener: OnClick) { /*...*/ }
typealias OnClick = (view: View)->Unit
```

![](<../../.gitbook/assets/image (12).png>)

注意，在使用 lambda 表达式时，也可以对参数进行解构，总之，这使得**函数类型通常比 SAM 更好**。

当我们要为监听器设置许多观察者时，使用函数类型是尤其正确的。典型的 Java 做法通常是在单个监听器接口中去收拢它们：

```kotlin
class CalendarView {
    var listener: Listener? = null
    
    interface Listener {
        fun onDateClicked(date: Date)
        fun onPageChanged(date: Date)
    }
}
```

我认为这很大程度是一种懒惰的做法，从 API 使用者的角度来看，最好将它们设置为包含函数类型的独立属性：

```kotlin
class CalendarView {
    var onDateClicked: ((date: Date) -> Unit)? = null
    var onPageChanged: ((date: Date) -> Unit)? = null
}
```

这样，`onDataClicked` 和 `onPageChanged` 实现就不需要在绑定在一个接口中，这些函数可以独立改变。

如果你没有定义接口的理由，那么最好使用函数类型，它们得到了很好的支持，并且经常被 Kotlin 开发人员使用。

### 我们什么时候需要 SAM？

当我们去设计一个从其它语言而不是 Kotlin 中使用的类时，这种情况下我们会更倾向使用 SAM。接口对于 Java 客户端来说更加干净，他们不能看到类型别名或名称建议。最后，在某些语言（特别是 Java）中使用 Kotlin 函数类型时，需要函数显式的返回 Unit：

```kotlin
// Kotlin
class CalendarView() {
    var onDateClicked: ((date: Date) -> Unit)? = null
    var onPageChanged: OnDateClicked? = null
}

interface OnDateClicked {
    fun onClick(date: Date)
}

// Java
CalendarView c = new CalendarView();
c.setOnDateClicked(date -> Unit.INSTANCE);
c.setOnPageChanged(date -> {});
```

这就是为什么当我们设计用于 Java 的 Api 时，使用 SAM 而不是函数类型是更合理的。但在其它情况下，首选函数类型。
