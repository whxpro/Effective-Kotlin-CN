---
description: Edu
---

# 第22条：当实现公共算法时使用泛型

我们能将值作为参数传递，同样地，我们也能将类型做为参数传递。一个能够接受类型参数的函数被称为泛型函数，一个已知的例子就是 stdlib 库中的 `filter` 函数，它接受类型参数 `T` ：

```kotlin
inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val destination = ArrayList<T>()
    
    for (element in this) {
        if (predicate(element)) {
            destination.add(element)
        }
    }
    return destination
}
```

类型参数对编译很有用，因为它允许编译器进一步检查和正确推断类型，这使我们的程序更加安全，对开发来说更有保障。例如，当我们在使用 `filter` 函数，编写 lambda 内容时，编译器知道实参的类型要与集合中元素的类型相同，因此它可以防止我们使用不合法的东西，IDE 还提供给我们有用的建议。

![](<../../.gitbook/assets/image (8).png>)

泛型主要被用于类和接口中，为了允许只创建具体类型的集合，例如 `List<String>` 和 `Set<User>`，这些类型在编译过程中会被擦除，但是在开发中，编译器会强制我们只传递正确的类型。例如，我们只能向 `MutableList<Int>` 添加 Int 类型的参数。 同样地，得益于这个机制，编译器知道我们从 `Set<User>` 中获取一个元素时，返回的类型是 `User`。通过这种方式，类型参数在静态类型语言中对我们有很大的帮助。 Kotlin 对难以被理解的泛型提供了强大的支持，据我所了解，即使是经验最丰富的 Kotlin 开发人员在他们的知识方面也存在差距，特别是关于型变。 因此接下来和_第24条：使用泛型时考虑型变_ 中，我将讨论 Kotlin 泛型最重要的方面。

### 泛型约束

类型参数的一个重要特征是：它们可以被约束为具体类型的子类型，我们可以通过在冒号后面放置超类来设置约束。该类型可以包含前面的类型参数：

```kotlin
fun <T : Comparable<T>> Iterable<T>.sorted(): List<T> {
    /*...*/
}

fun <T, C : MutableCollection<in T>> Iterable<T>.toCollection(destination: C): C {
    /*...*/
}

class ListAdapter<T: ItemAdaper>(/*...*/) {
    /*...*/ 
}
```

使用约束的最重要成果是：该类型的实例可以使用该约束类型提供的所有方法。这样，当 `T` 被限制为 `Iterable<Int>` 的子类型时，我们就知道可以遍历类型为 `T` 的实例，并且迭代器返回的类型为 `T`。当我们限制为 `Comparable<T>` 时，我们就知道该类型可以自行排序。约束的另一个主流选择是 Any，这意味着类型可以是任意的非空类型：

```kotlin
inline fun <T, R : Any> Iterable<T>.mapNotNull(
    transform: (T) -> R?
): List<R> {
    return mapNotNullTo(ArrayList<R>(), transform)
}
```

在极少数情况下，我们可以能需要设置多个上界，我们可以使用 `where` 来设置更多的约束：

```kotlin
fun <T: Animal> pet(animal: T) where T: GoodTempered {
    /*...*/
}

// OR

fun <T> pet(animal: T) where T: Animal, T: GoodTempered {
    /*...*/
}
```

### 总结

类型参数是 Kotlin 类型系统的一个重要组成部分，我们使用它们来实现类型安全的泛型算法或泛型对象。类型参数可以被约束为具体类型的子类型，当使用它们时，我们可以安全地使用这种类型提供的方法。
