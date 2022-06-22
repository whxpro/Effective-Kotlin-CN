# 第28条：指定 Api 的稳定性

如果每辆汽车开起来都完全不同，那生活就会困难的多，汽车中有一些元素不是通用的，比如我们打开广播电台的方式，我经常看到车主在使用他们时遇到麻烦，我们太懒了，不去学习那些无意义的、临时的接口。我们更喜欢稳定的、普遍的东西。

在编程中也一样，我们更喜欢稳定而且最好是标准应用程序编程接口（Application programming interface API），主要原因有：

1. 当 Api 发生变化，开发人员拿到更新的接口后，他们将需要手动更新代码。当许多元素依赖此 API 时，这一点尤其成问题。修正它的用途或提供替代方案可能很难。特别是当我们的 API 被其他开发者用于我们不熟悉的项目时。如果是一个公共库，我们无法自行调整这些使用，而要用户自行更改，从用户的角度来看，这并不舒服。一个库中的小变化可能需要在代码库的不同部分进行许多变化。当用户害怕这种变化时，他们就会使用的旧的库版本。 这是一个很大的问题，因为更新对他们来说变得越来越困难，而新的更新可能包含他们需要的东西，比如 Bug 修复或漏洞修正。旧的库可能不再受支持，或者可能完全停止工作。当程序员害怕使用较新的稳定的库版本时，这是一种非常不健康的状况
2. 用户需要学习新的 API，他们通常不愿意花费这样的额外精力。更重要的是，他们需要更新理解那些被改变的地方，这对他们来说也是痛苦的，所以他们避免这样做。过时的知识可能会导致安全问题，而学习接口的变化会很困难，这都是不健康的。

另一方面，设计一个好的 API 是非常困难的，所以开发者想要做出改变来改进它，编程社区推荐的解决方案是指定 API 的稳定性。

最简单的方法，创建者应该在文档中明确说明 API 或其某部分是否不稳定。更正式的说，我们通过版本来指定整个库或模块的稳定性。有许多版本控制系统，但是现在有一个非常流行，几乎可以当做是业界标准来参考的，就是语言版本号（SemVer），在这个系统中，我们有三个部分组成： MAJOR.MINOR.PATCH 。每一个部分都是从 0 开始的正整数，当公共 API 中的更改具有具体重要性时，我们就增加它们的值。 规则是：

* 当你做了不兼容的 API 更改时，增加 MAJOR 版本号
* 当你以向后兼容的方式添加功能时， 增加 MINOR 版本号
* 当你向后兼容的修复了 Bug 时，增加 PATCH 版本号

当我们增加 MAJOR 时，我们需要设置 MINOR 和 PATCH 为0。 当我们增加 MINOR 时，我们将 PATCH 设置为0，预发布版和构建元数据的附加标签可以作为 MAJOR.MINOR.PATCH 格式的扩展。Major 版本为0（0.y.z）用于初始的开发，在这个版本中，任何东西都可能随时间发生变化，公共 API 不应该被认为是最稳定的，因此，当一个库或模块遵循 SemVer 且 Major 版本为0时，我们不应该期望它是稳定的。

不要担心长时间处于一个测试版本， Kotlin 花了5年的时间才升到 1.0 版，这是这种语言的一个非常重要的时期，因为它在这一时期发生了很多变化。

当我们将新元素引入一个稳定的 API 时，如果它们还不稳定，我们应该首先将它们保留在另一个分支中一段时间，当你希望允许一些用户使用它（通过将代码合入到主分支并发布它），你可以使用 `Experimental` 元注解来提示他们：这个 API 还不稳定。这个注解能让人们正常使用元素，但是会显示警告或错误（基于设置的级别）。

```kotlin
@Experimental(level = Experimental.Level.WARNING)
annotation class ExperimentalNewApi

@ExperimentalNewApi
suspend fun getUsers(): List<User> {
    //...
}
```

![](<../../.gitbook/assets/image (11) (1).png>)

我们应该预料到这些因素随时可能发生变化，同样，不要担心元素长时间处于实验状态，这样做虽然会减缓应用的速度，但也有助于我们设计出更好的 API。

当我们需要改变某个稳定 API 的一部分时，为了帮助用户处理这些变化，我们首先使用 `Deprecated` 注解来处理那些需要废弃的元素：

```kotlin
@Deprecated("Use suspending getUsers instead")
fun getUsers(callback: (List<User>)->Unit) {
    //...
}
```

此外，当有一个直接替代的方案时，使用 `ReplaceWith` 指定它，以允许 IDE 进行自动转化：

```kotlin
@Deprecated("Use suspending getUsers instead",
ReplaceWith("getUsers()"))
fun getUsers(callback: (List<User>)->Unit) {
    //...
}
```

下面是 stdlib 的一个例子：

```kotlin
@Deprecated("Use readBytes() overload without "+
"estimatedSize parameter",
ReplaceWith("readBytes()"))
public fun InputStream.readBytes(
    estimatedSize: Int = DEFAULT_BUFFER_SIZE
): ByteArray {
    //...
}
```

然后我们需要给用户时间来调整，这应该需要很长一段时间，因为用户除了适应他们所使用的库的新版本外，还有别的责任。广泛的使用 api 需要数年的时间，最后，在此之后的某个 major 版本中，我们可以删除已弃用的元素。

### 总结

用户需要了解 api 的稳定性，虽然稳定的 api 是首选的，但没有什么比在本该稳定的 api 中发生意外变化更糟糕了。这样的变化对用户来说是痛苦的。模块或库的创建者与其用户之间的纽带非常重要，我们通过使用版本名称、文档、注解注释来实现这些。而且，在一个稳定的 API 中，每次更改都需要经过一个漫长的弃用过程。