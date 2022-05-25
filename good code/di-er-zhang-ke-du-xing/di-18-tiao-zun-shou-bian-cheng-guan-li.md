---
description: Basis
---

# 第18条：遵守编程惯例

Kotlin 已经建立了良好的代码约定，在文档中被称为 “编程惯例（Coding Conventions）”，**这些惯例并非对所有项目都是最好的，但对我们来说，作为一个社区，在所有项目中都遵守这些惯例是最理想的**。多亏这些惯例，能使得我们：

* 在不同项目之间切换更加容易
* 代码更具可读性，甚至对外部开发人员也是如此
* 更容易理解代码的工作原理
* 将代码 merge 到公共仓库，或将部分代码从一个项目迁移到另一个项目更容易

程序员应该熟悉文档中描述的那些约定，当它们发生变化时 —— 随着时间的推移可能会在某种程度上发生变化 —— 它们也应该被遵守。因为这很难做到，所以有两种工具能帮助你

* IntelliJ formatter 可以根据官方的编程风格设置代码自动格式化，为此，跳转到 Setting | Editor | Code Style | Kotlin, 点击右上角 "Setting from..."，并从菜单中选择 “Predefined style / Kotlin style guide”
* klint —— 主流的检查程序，它可以分析的你代码，并指出你所有违反编码惯例的地方

看看 Kotlin 项目，我发现它们很多都符合大部分的编程惯例，这可能是因为 Kotlin 主要遵循着 Java 编码惯例，而今天大多数 Kotlin 开发人员都是曾经的 Java 开发人员。我发现一个经常被违背的惯例是去如何格式化类和函数，按照惯例，具有短主构造函数的类可以在一行中定义：

```kotlin
class FullName(val name: String, val surname: String)
```

然而，有很多参数的类应该被格式化，使每个参数都放在其他行，并且第一行没有参数：

```kotlin
class Person(
    val id: Int = 0,
    val name: String = "",
    val surname: String = ""
) : Human(id, name) {
    // body
}
```

类似地，这是我们格式化长函数的方式：

```kotlin
public fun <T> Iterable<T>.joinToString(
    separator: CharSequence = ", ",
    prefix: CharSequence = "",
    postfix: CharSequence = "",
    limit: Int = -1,
    truncated: CharSequence = "...",
    transform: ((T) -> CharSequence)? = null
): String {
    // ...
}
```

请注意，上面这两个例子，都把第一个参数放在了第一行，然后其它参数的缩进与其相同， 这可能和其它语言上参数的惯例不同。

```kotlin
// 不要这么做！
class Person(val id: Int = 0,
             val name: String = "",
             val surname: String = "") : Human(id, name){
                 // body
}
```

上面这段代码有两个问题：

1. 每个类的参数都会根据其类名的长度，以不同的缩进开始。另外，当我们更改类名时，我们需要调整所有主构造函数参数的缩进
2. 以这种方式定义的类往往会太宽，宽度是 class关键字 + 类名 + 最长构造函数的参数， 可能还要加上其超类和接口

一些团队可能决定使用些许不同的惯例，这很好，但这些约定应该在给定的项目中被遵守使用。**每个项目应该看起来像是同一个人写的，而不是一群人相互争斗而写的**。开发人员通常对编码惯例不够重视，但是它们在实战中很重要，在最佳实战的书中，一定需要至少一个小的章节来专门介绍可读性的重要性。阅读它们，使用静态代码扫描器来帮助你遵守这些惯例，将它们运用到你的项目中，通过遵守编码约定，使 Kotlin 项目对我们所有人来说都是更好的。
