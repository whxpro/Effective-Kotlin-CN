# 第14条： 在变量不清晰时指定其类型

Kotlin 有一个很棒的类型推断系统，它允许我们可以省略声明那些对开发人员来说一眼就能看出的类型：

```kotlin
val num = 10
val name = "Marcin"
val ids = listOf(12, 112, 554, 997)
```

在上下文比较混乱、冗余的时候，这种写法不仅帮助我们节省了开发时间，还提高了可读性。 但是，在类型不明确的情况下，它不应该过度使用：

```kotlin
val data = getSomeData()
```

我们为了可读性而设计代码，所以我们不应该隐藏对读者来说可能很重要的信息。如果你以为读者总是可以跳到函数体中去看具体实现，就认定返回类型可以省略，这是不合理的！或许随着用户阅读代码越来越深入，类型可以被推断出来，但是，用户可能会在 GitHub 或其他不支持跳到具体实现环境的地方去阅读代码，就算他们可以，我们在这上面的记忆力（去推断对象的类型）也是有限的，像这样浪费记忆力不是一个好主意。类型是重要的信息，如果不明确，就应该指定其类型：

```kotlin
val data: UserData = getSomeData()
```

对于明确类型，不仅仅有提高可读性这么一个好处，它也是出于安全而考虑的，如_第3条：尽快消除平台类型_，和_第4条：不要对外暴露需要推断的类型_。 **类型对于开发人员和编译器都可能是重要的信息，无论何时，不要觉得指定类型很费时间，做这个事情成本很低且收益很高。**