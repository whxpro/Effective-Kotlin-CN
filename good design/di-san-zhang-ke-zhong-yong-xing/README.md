# 第三章：可重用性

Have you ever wondered how the System.out.print function works (print function in Kotlin/JVM)? It’s one of the most basic functions used again and again and yet, would you be able to implement it yourself if it would vanish one day? Truth is that this is not an easy task. Especially if the rest of java.io would vanish as well. You would need to implement the communication with the operating system in C using JNI, separately for each operating system you support²⁴. Believe me, implementing it once is terrifying. Implementing it again and again in every project would be a horror.

你是否想知道 `System.out.print()` 函数是如何工作的（Kotlin/JVM上的 `print` 函数）？ 这是最基础的函数之一，并被反反复复地使用，但是如果有一天它消失了，你能自己实现它吗？实际上，这并不是一项容易的任务。特别是假如剩下的 `java.io` 也消失了，你需要使用 JNI 来与 C 语言下的操作系统进行通信，然后分别在你支持的操作系统实现该函数。 相信我，实现它一次是非常可怕的，在每个项目中反反复复实现它将是一个更恐怖的事情。

The same applies to many other functions as well. Making Android views is so easy because Android has a complex API that supports it. A backend developer doesn’t need to know too much about HTTP and HTTPS protocols even though they work with them every day. You don’t need to know any sorting algorithms to call Iterable.sorted. Thankfully, we don’t need to have and use all this knowledge every day. Someone implemented it once, and now we can use it whenever we need. This demonstrates a key feature of Programming languages: reusability.

这同样适用于许多其它函数。创建 Android 视图非常容易，因为 Android 有一个复杂的 Api 支持它；后端开发人员不太需要了解 HTTP 和 HTTPS 协议的知识，即使他们每天都在使用这些协议。你调用 `Iterable.sorted` 时不需要知道任何排序算法的原理。值得庆幸的是，有人实现过一次，现在我们就能随时使用了。这说明了编程语言的一个关键特性：可重用性。

Sounds enthusiastic, but code reusability is as dangerous as it is powerful. A small change in the print function could break countless programs. If we extract a common part from A and B, we have an easier job in the future if we need to change them both, but it’s harder and more error-prone when we need to change only one.

这听起来很让人激动，但是代码复用很危险，因为它很强大。`print` 函数的一个小变化可能会破坏无数的程序，我们从 A 和 B 中提取一个公共的部分，假如未来需要同时更改它们，我们能很轻松的完成，但是假如我们只需要改其中一个，则会很困难，且容易出错。

This chapter is dedicated to reusability. It touches on many subjects that developers do intuitively. We do so because we learned them through practice. Mainly through observations of how something we did in the past has an impact on us now. We extracted something, and now it causes problems. We haven’t extracted something, and now we have a problem when we need to make some changes. Sometimes we deal with code written by different developers years ago, and we see how their decisions impact us now. Maybe we just looked at another language or project and we thought “Wow, this is short and readable because they did X”. This is the way we typically learn, and this is one of the best ways to learn.

本章专门讨论代码可重用性，它触及到程序员直觉上的许多主题。

讨论它是因为过去我们都是通过在实践中去学习代码重用性的，我们主要通过观察过去所做的事情对我们现在的影响。我们提取了一些东西，现在它引起了问题；我还没有提取一些东西，而现在需要进行一些更改时，就出现了问题；有时我们处理由不同开发人员多年以前写的代码，看到他们当时的决策如何影响到现在的我们。 也许我们只是看到另一种语言或项目，我们会说：“哇，这是简短而易读的，因为他们做了XXX”，这是我们通常的学习方式，也是最好的学习方式之一。

It has one problem though: it requires years of practice. To speed it up and to help you systematize this knowledge, this chapter will give you some generic rules to help you make your code better in the long term. It is a bit more theoretical than the rest of the book. If you’re looking for concrete rules (as presented in the previous chapters), feel free to skip it.

但这样学习有一个问题：它需要长年累月地去练习积累，为了加速这个过程，并帮助你系统化这些知识，本章提供了一些惯例，以帮助你的代码长远来看更加优秀。它比本书的其他部分更偏向理论，如果你想看具体的规则（如前几章所示），请随意的略过它。
