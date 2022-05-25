# 第八章：高效的集合处理

集合是编程中最重要的概念之一。在 iOS 中，最重要的视图元素之一 ——`UICollectionView`，被设计用来表示一个集合。同样，在 `Android` 中，很难想象一个没有 `RecyclerView` 或 `ListView` 的应用程序。当需要编写带有新闻资讯的页面时，你将会拥有一个新闻列表，每个新闻信息可能都有一个作者列表和一个标签列表；当你创建一个在线商店时，会从产品列表开始，这些产品可能有分类列表和一个不同变体的列表；当用户购买时，它们可能有购物车，里面是产品及其数量的集合；然后他们需要从一系列的送货方式和付款方式中做出选择。在某些语言中，`String` 只是许多字符的集合，集合在编程中无处不在！只要着眼于你当前的应用程序，很快就会看到大量的集合。

这一事实也可以反映在编程语言中，大多数现代语言都有一些集合的声明用法：

```kotlin
// Python
primes = [2, 3, 5, 7, 13]
// Swift
let primes = [2, 3, 5, 7, 13]
```

良好的集合处理是函数式编程语言的标志性功能。 `Lisp` 语言的名字就代表了“列表处理（list processing）”。大多数现代语言都对集合处理提供了良好的支持，这些语言也包括了 Kotlin， 它具有一组最强大的集合处理工具。来看看下面的集合处理：

```kotlin
val visibleNews = mutableListOf<News>()
for (n in news) {
    if(n.visible) {
        visibleNews.add(n)
    }
}

Collections.sort(visibleNews,
    { n1, n2 -> n2.publishedAt - n1.publishedAt })
val newsItemAdapters = mutableListOf<NewsItemAdapter>()
for (n in visibleNews) {
    newsItemAdapters.add(NewsItemAdapter(n))
}
```

在 Kotlin 中，上面的代码可以用下面的代码来替代：

```kotlin
val newsItemAdapters = news
    .filter { it.visible }
    .sortedByDescending { it.publishedAt }
    .map(::NewsItemAdapter)
```

这样的语法不仅更短，而且可读性更强。每一步都对元素列表进行具体的转换。以下是上述处理的可视化过程：

![](<../../.gitbook/assets/image (5).png>)

上述示例的性能看似非常相似。但事实并非总是那么简单。 Kotlin 有很多集合处理方法，我们有各种方式不同但效果相同的函数来处理列表。例如，下面的处理实现具有相同的结果，但性能差异迥异：

```kotlin
fun productsListProcessing(): String {
    return clientsList
        .filter { it.adult }
        .flatMap { it.products }
        .filter { it.bought }
        .map { it.price }
        .filterNotNull()
        .map { "$$it" }
        .joinToString(separator = " + ")
}

fun productsSequenceProcessing(): String {
    return clientsList.asSequence()
        .filter { it.adult }
        .flatMap { it.products.asSequence() }
        .filter { it.bought }
        .mapNotNull { it.price }
        .joinToString(separator = " + ") { "$$it" }
}
```

集合处理的优化不仅仅是一个智力难题。

集合处理极其重要，在大型系统中，性能通常是至关重要的。这就是为什么提高程序性能往往是很关键的，特别是当我们进行后端应用程序开发或数据分析时。但是，当我们实现前端客户端时，我们也会面临集合的处理，这也会限制应用程序的性能。作为一名顾问，在看过很多项目后，我的经验是，我在很多不同的地方看到了一次又一次的集合处理，这不是一件容易被忽略的事情。

好消息是，集合处理优化不难掌握，有一些规则和少数事情需要去记住，但事实上任何人都可以高效的完成。这就是我们这一章要学习的内容。
