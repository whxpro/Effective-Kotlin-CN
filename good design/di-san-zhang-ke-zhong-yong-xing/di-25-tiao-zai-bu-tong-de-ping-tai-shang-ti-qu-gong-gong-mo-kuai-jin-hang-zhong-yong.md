---
description: Basics
---

# 第25条：在不同的平台上提取公共模块进行重用

公司很少只为一个平台编写应用程序，他们宁愿为两个或两个以上的平台开发产品，而如今，他们的产品往往会依赖于运行在不同平台上的几个应用程序。考虑到通过网络调用服务器进行通信的（不同平台上的）客户端应用程序，经常有可以重用的地方。针对不同平台的相同的产品的实现通常具有更多的相似性。特别是它们的业务逻辑几乎相同，这些项目可以从共享代码中获得巨大收益。

### 全栈开发

很多公司都是基于 web 开发。它们的产品是一个网站，但在大多数情况下，这些产品需要一个后端应用程序（也被称为服务器端），JavaScript 是 Web 开发的王者，它几乎垄断了这个平台，后端开发和 web 开发分离是很常见的。然而，事实可以改变，现在， Kotlin 正在成为 Java 后端开发的热门替代语言。例如，对于最流行的 Java 框架 Spring， Koltin 就是一个一等公民。 Kotlin 可以在每个框架中作为 Java 的替代语言使用，还有许多 Koltin 后端框架，例如 `Ktor`，这就是许多后端项目从 Java 迁移到 Kotlin 的原因。 Kotlin 的一个优点是它可还可以编译为 JavaScript， 现在已经有很多 Kotlin/JS 库，我们可以使用 Kotlin 来编写不同类型的 Web 应用程序。例如，我们可以使用 React 框架和 Kotlin/Js 编写一个 web 前端，这允许我们用 Kotlin 同时编写后端和网站，更棒的是，Kotlin 可以对某部分的代码同时编译为 JVM 字节码和 JavaScript，这些是多平台共享的部分，例如，我们可以在那里放置工具、API定义、通用抽象等我们可能想要复用的东西。

![](<../../.gitbook/assets/image (9).png>)

### 手机开发

这种能力在移动领域更为重要，我们很至少只针对 Android 开发应用，有时候我们可以没有服务器，但也要实现一个 iOS 应用。每个应用程序都使用不同的语言和工具为不同的平台编写。最后，同一个应用程序的 Android 和 iOS 版本非常相似，它们的设计通常不同，但它们的内部逻辑几乎是相同的。

使用 Kotlin 多平台的功能，我们可以只用实现这个逻辑一次，并在这两个平台之间重用它，我们可以在那里创建一个通用模块并实现业务逻辑， 无论如何，业务逻辑应该独立于框架和平台（Clean Architecture），这种通用逻辑可以用纯 Kotlin 编写，也可以使用其他通用模块，然后可以在不同的平台上使用。

在 Android 中，它可以直接使用，因为 Android 是用 Gradle 以同样的方式构建的，这种体验类似于我们的 Android 项目中的那些公共部分。

For iOS, we compile these common parts to an Objective-C framework using Kotlin/Native which 对于 iOS， 我们使用 Kotlin/Native 将这些公共的部分编译到一个 Object-C 的 framwork 中， Kotlin/Native 使用 LLVM 编译到 Native code 中，然后我们可以在 Xcode 或 AppCode 中用 Swift 使用这些代码，另外，我们也可以使用 Kotlin/Native 实现整个应用程序。

![](<../../.gitbook/assets/image (10).png>)

### 库

定义通用模块也是库的一个强大工具。 特殊的，那些不太依赖平台的模块可以很容易地转移到一个通用模块中，允许用户从运行在 JVM、 JavaScript 或原生的所有语言中使用它们（例如 Java、Scala、JavaScript、CoffeeScript、TypeScript、C、Objective-C、Switft、Python、C#等）

### 全部平台一起

我们可以同时使用所有这些平台。 使用 Kotlin，我们可以为几乎所有类型的主流设备和平台进行开发，并根据我们的需要在它们之间复用代码，这只是我们可以用 Kotlin 实现的几个例子：

* 后端的 Kotin/JVM， 例如 Spring 或 Ktor
* Web 的 Kotlin/JS， 例如 React
* 在 Android 的 Kotlin/JVM， 使用 Android SDK
* 可使用 Kotlin/Native 从 Objective-C 或 Switft 中使用 iOS 框架
* 桌面的 Kotlin/JVM， 例如 TornadoFX
* 树莓派， Linux 或 Mac OS 程序的 Kotlin/Native

下面是一个典型的多平台程序图片：

![](<../../.gitbook/assets/image (5).png>)

我们仍然在学习如何组织我们的代码，让代码重用从长远来看更加安全和高效。 了解这种方法可以很好的让我们知道一些可能性，它能使我们在不同模块间重用公共模块， 这是一个消除冗余、复用常规逻辑和常规算法的强大工具。
