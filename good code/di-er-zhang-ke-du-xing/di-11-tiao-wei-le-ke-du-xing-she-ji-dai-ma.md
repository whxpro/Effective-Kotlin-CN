# 第11条：为了可读性设计代码

在编程中，开发人员阅读代码的时间远远超过了其编写代码的时间，这是一个总所周知的现象。一个共识是，每花一分钟写代码，就要花十分钟读代码（这个比例是由 Robert C.Martin 在《代码整洁之道》一书中普及的）。如果这个比例看起来令人难以置信，那么请你想一想，当你试图定位 bug 时，你在阅读代码上花费了多少时间？我相信每个人在他们的职业生涯中都至少经历过一次这样的情况：他们花了数周的时间定位错误所在，只是为了通过修改一行代码来修复它。当我们学习如何使用一个新的 api 时，一般是通过阅读代码来学习的。我们通常阅读代码是为了理解其实现的逻辑或代码运作机制。**编程关键在于读而不是写**。我们应该清楚的知道，在编写代码时需要考虑到代码可读性。

### 减少认知负荷

代码可读性对于每个人来说都有不同的含义，然而，也有一些规则是基于经验或认知科学形成的，比较以下两种实现方式：

```
// 实现 A
if (person != null && person.isAdult) {
    view.showPerson(person)
} else {
    view.showError()
}

// 实现 B
person?.takeIf { it.isAdult }
   ?.let(view::showPerson)
   ?: view.showError()
```

实现A和实现B哪个更好？ 一个简单的判断是行数越少越好，但这并不是一个好的答案，我们也可以从第一个实现中删去换行符，但这并不会使其可读性更好。

这两个结构的可读性取决于我们理解它们的速度。反过来，这在很大程度上取决于我们的大脑对每个习语（结构、功能、模式）受过多少的训练。对于 Kotlin 的初学者来说，实现A当然更具有可读性，它使用一般的习语（`if..else...`、`&&`、函数调用）。 实现 B 则是具有典型的 Kotlin 习语（空安全调用 `?.`，`takeIf`、`Elvis`操作符 ?: ，函数引用`::showPerson`）。 当然，所有这些习语在 Kotlin 中都是常用的，所以大多数有经验的 Kotlin 开发人员都很熟悉它们，尽管如此，还是很难对它们进行比较。 Kotlin 并不是大多数开发人员的第一语言，我们在一些通用编程方面比在 Kotlin 编程上有更多的经验。我们不止为有经验的开发人员编写代码，很有可能，你雇佣的初级员工（在寻找高级员工几个月无果后）甚至不知道什么是 `let`、`takeIf` 和有界引用，而且他本人也从未见过 `Elvis` 操作符和其使用的方式。那个人可能会花一整天来思考这一段代码，此外，即使对于经验丰富的 Kotlin 开发人员， Kotlin 也不一定是他们唯一的编程语言。许多阅读你的代码的开发人员不会有使用 Kotlin 经验，大脑总是需要花一些时间来识别特定于 Kotlin 的习语，即使在使用 Kotlin 多年之后，我理解第一个实现所花的时间仍要比第二个少很多，每一个鲜为人知的习语都会引入一些复杂性，当我们在仅仅一行代码中分析好几个习语时，我们必须要同时理解所有习语，才能读懂这行代码，复杂性会迅速增加。

注意，实现 A 更容易修改，假设我们在 `if` 块上添加额外的操作，在实现 A 中很容易添加，在实现 B 中，我们就不能再使用函数引用了，在实现B中向 `else` 块添加一些东西就更难了——我们需要使用一个写函数来在 `Elvis` 操作符的右侧并涵盖不止一个表达式：

```
if (person != null && person.isAdult) {
    view.showPerson(person)
    view.hideProgressWithSuccess()
} else {
    view.showError()
    view.hideProgress()
}

person?.takeIf { it.isAdult }
   ?.let {
       view.showPerson(it)
       view.hideProgressWithSuccess()
   } ?: run {
       view.showError()
       view.hideProgress()
   }
```

调试实现A也简单得多，这不足为奇，调试工具都是为这样的基本结构而设计的。

一般的规则是，不太常见的“创造性”的结构通常不太灵活，也没有那么好的被支持。例如我们需要添加第三个分支来显示当 `person` 为 `null` 时的不同错误，以及当他或她不是 `adult` 时的不同错误，在实现 A 上，我们可以在 IntelliJ 重构时通过调整 `if..else` 语句，轻松调整流程控制。 在代码上相同的更改在 B 上就比较痛苦，这几行代码可能要被完全的重写。

你是否注意到了 A 和 B 它们甚至以不同的方式工作？ 你能看出区别吗，现在就去想想。

区别就在它从 `let` 返回一个 Lambda 表达式作为结果，这意味在最开始的 B 实现中，如果 `showPerson` 返回 null，那么 B 实现也将调用 `showError` ，这肯定不是显而易见的，它告诉我们，当我们使用不太熟悉的结构时，更容易出现意外的行为。

这里一般的规则是：我们要减少认知的负荷。我们的大脑能识别模式，并基于这些模式建立我们对程序如何工作的理解。当我们在考虑可读性时，大脑想要减少理解的成本。我们更喜欢代码更少，但也更常见的结构，当我们经常看到熟悉的模式时，就能马上认出它们，我们总是喜欢那些我们在其他领域中也熟悉的结构。

### 不要太极端

在前面的例子中， 我展示了 `let` 是如何被误用的，它是一种流行的习惯用法，可以在各种上下文中合理地使用，以实际地改善代码。 一个流行的例子是当我们有一个可为空的可变属性时，我们只能在它不为空时进行操作，智能转型不能使用，因为可变属性可以由另外一个线程修改，解决这个问题的一个好方法是使用安全调用 `?.let` :

```
class Person(val name: String)
var person: Person? = null
   
fun printName() {
    person?.let {
        print(it.name)
    }
}
```

这样的语句很流行，也被广泛认可。还有更多合理的使用 `let` 的案例，例如：

* 在计算完参数后，进行下一步操作
* 使用它以装饰器来包装一个对象

下面是这两个函数的示例（两者都是用了函数引用）：

```
students
     .filter { it.result >= 50 }
     .joinToString(separator = "\n") {
         "${it.name} ${it.surname}, ${it.result}"
     }
     .let(::print)

 var obj = FileInputStream("/file.gz")
     .let(::BufferedInputStream)
     .let(::ZipInputStream)
     .let(::ObjectInputStream)
     .readObject() as SomeObject
```

在所有这些情况下，使用它们都付出了代价——这些代码更难调试，也更难被经验较少的 Kotlin 开发人员所理解。但是我们为某些东西付出了代价，这看起来是公平的交易。主要的问题在于我们毫无理由的引入大量复杂性语句的场景。

有意义的事情和没有意义的事情总是会引发讨论。平衡是一门艺术，认识到不同的结构如何引入复杂性，或如何阐明事物是件好事。特别是当它们一起使用时，两个结构复合的复杂性通常比它们各自复杂性的总和要大很多。

### 一些惯例

我们知道不同的人对可读性有不同的看法，我们经常会因为函数名而争论，讨论应该使用什么命名习惯，等等。编程是一门表达艺术，尽管如此，还是有一些惯例和理解需要记住。

当我在旧金山的一个研讨会小组问我在 Kotlin 上遇到的最糟糕的事情是什么时，我给他们看了这个：

```
val abc = "A" { "B" } and "C"print(abc) // ABC
```

只需要下面的代码的帮助，就能让上面这种可怕的语法成为可能：

```
operator fun String.invoke(f: ()->String): String =
this + f()

infix fun String.and(s: String) = this + s
```

这段代码违反了很多规则：

* 它违反了操作符的意义—— `invoke()` 不应该被这样调用， 一个 String 不能被这样使用
* 这里使用了 “lambda 作为最后一个参数” 的约定令人困惑，在函数之后使用它是可以的，但是 `invoke` 操作符中使用它应该非常小心
* 显然 `and` 这个中缀操作符的命名不好，使用 `append` 可能会更好些
* 我们已经有字符串连接的特性了，我们应该直接使用那些特性，而不是重复造轮子

在这些建议的背后，有一个更通用的规则来保护 Kotlin 良好的代码风格，在本章中，我们将从第一条开始讨论最重要的操作符，即 `override` 操作符。