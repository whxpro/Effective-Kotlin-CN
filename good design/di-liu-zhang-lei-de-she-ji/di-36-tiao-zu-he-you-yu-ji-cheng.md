---
description: '`Not Kotlin-specific`'
---

# 第36条：组合优于继承

继承是一个强大的特性，但它是被设计来创建具有 “is-a” 关系的对象层次结构。当这种关系不明确时，继承可能是有问题、有危险的。当我们所需要的只是简单的代码提取和重用时，应该谨慎使用继承，而考虑轻松的替代方案：类的组合。

### 简单的行为重用

让我们从一个简单的问题开始：假设我有两个具有部分功能相似的类 —— 进度条显示在逻辑之前，隐藏在逻辑之后。

```kotlin
class ProfileLoader {

    fun load() {
        // show progress
        // load profile
        // hide progress
    }
}

class ImageLoader {

    fun load() {
        // show progress
        // load image
        // hide progress
    }
}
```

从我的经验来看，大多数程序员都会提取这个行为到一个公共的超类上：

```kotlin
abstract class LoaderWithProgress {

    fun load() {
        // show progress
        innerLoad()
        // hide progress
    }
    
    abstract fun innerLoad()
}

class ProfileLoader: LoaderWithProgress() {
    override fun innerLoad() {
        // load profile
    }
}

class ImageLoader: LoaderWithProgress() {
    override fun innerLoad() {
        // load image
    }
}
```

这种方法的确可以用于这样一个简单的情况，但它有一个严重的缺点，我们应该注意到：

* 我们只能扩展一个类，使用继承提取功能通常会导致产生大量的 BaseXXX 类，这些类累积了许多功能，或者类型的层次结构上过于深入和复杂
* 当我们要扩展时，会从超类中继承一切，这导致子类可能具有它们不需要的功能和方法（违背了接口隔离原则）
* 使用超类功能不会那么显式。通常，在开发人员想要阅读一个函数，需要多次跳转到超类了解该函数如何工作时，这是一个不好的信号

这些都是促使我们思考另一种方案的有力理由，其中一个很好的方案就是组合。通过组合，我们将对象作为属性保存（我们组合它们）并重用它们的功能。下面是一个使用复合代替继承来解决问题的例子：

```kotlin
class Progress {
    fun showProgress() { /* show progress */ }
    fun hideProgress() { /* hide progress */ }
}

class ProfileLoader {
    val progress = Progress()
    
    fun load() {
        progress.showProgress()
        // load profile
        progress.hideProgress()
        }
}

class ImageLoader {
    val progress = Progress()
    
    fun load() {
        progress.showProgress()
        // load image
        progress.hideProgress()
    }
}
```

请注意，组合比较困难，因为我们需要包含组合对象并在每个类中使用它，这是许多人喜欢继承的关键原因。然而，这些额外的代码并不是无用的，它告诉读者 `progress` 如何被使用的。它还赋予开发人员更大的权力来决定 `progress` 如何运行。

另一件要注意的事情是，当我们想要提取多个功能块时，组合是更好的选择，例如，加载完成的信息：

```kotlin
class ImageLoader {
    private val progress = Progress()
    private val finishedAlert = FinishedAlert()
    
    fun load() {
        progress.showProgress()
        // load image
        progress.hideProgress()
        finishedAlert.show()
    }
}
```

我们不能扩展超过一个类，所以如果我们想使用继承，我们将被迫将这两个功能放在一个超类中。这通常会导致用于添加这些功能的类型的在层次结构上更加复杂。这种层次结构很难阅读，也很难修改。想想看，如果我们在前两个子类中需要 alert，而在第三个子类中不需要，会发生什么？处理这个问题的一种方法是引入参数化构造函数：

```kotlin
abstract class InternetLoader(val showAlert: Boolean) {
    fun load() {
        // show progress
        innerLoad()
        // hide progress
        if (showAlert) {
            // show alert
        }
    }
    
    abstract fun innerLoad()
}

class ProfileLoader : InternetLoader(showAlert = true) {
    override fun innerLoad() {
        // load profile
    }
}
class ImageLoader : InternetLoader(showAlert = false) {   
    override fun innerLoad() {
        // load image
    }
}
```

这是一个糟糕的解决方案，它接收一个子类不需要的功能并阻塞它。当子类不能阻挡这些它不需要的功能加入时，问题就更加复杂了。当使用继承时，我们从父类中得到一切，而不仅是我们只需要的，可能还有我们不想要的。

### 从包中继承一切

当我们使用继承时，我们会从超类中获取一切 —— 所有方法、合约和行为。因此继承是一个很好的表示对象的层次结构的工具，而不一定只是重用一些公共部分。对于这些情况，使用组合是更好的，因为能选择我们需要的行为。举个例子，假设在我们的系统中，要去设计一个可以 `bark`（叫） 和 `sniff`（闻） 的 `Dog` 类：

```kotlin
abstract class Dog {
    open fun bark() { /*...*/ }
    open fun sniff() { /*...*/ }
}
```

如果我们需要创造出一个机器狗， 它能 `bark`，但是不能 `sniff`，该怎么办呢？

```kotlin
class Labrador: Dog()

class RobotDog : Dog() {
    override fun sniff() {
        throw Error("Operation not supported")
        // Do you really want that?
    }
}
```

注意，这样的解决方案违反了_接口隔离原则_ —— `RobotDog` 有一个它不需要的方法，它还违反了_里氏替换原则_，破坏了超类的行为。另一方面，如果你的 `RobotDog` 也需要成为一个 `Robot` 类，因为 `Robot` 可以计算（有 `calculate`方法），该怎么办？ Kotlin 并不支持多继承呀！

```kotlin
abstract class Robot {
    open fun calculate() { /*...*/ }
}

class RobotDog : Dog(), Robot() // Error
```

当你没有使用组合时，这里就充斥了严重的设计问题和限制性。当我们使用组合时，我们选择要重用的东西。为了表示类层次结构，这里使用接口更加安全，我们可以实现多个接口。还没有说明的是，继承可能会导致意外的行为。

### 继承打破了封装

在某种程度上，当我们继承一个类时，我们不仅依赖于其外部工作方式，还依赖于其内部实现方式。这就是为什么我们认为继承破坏了封装。让我们来看一个受 Joshua Bloch 的《Effective Java》一书启发的例子。假设我们需要一个 `set`，它要知道在生命周期内添加了多少元素，这个集合可以通过继承 `HashSet` 来创建:

```kotlin
class CounterSet<T>: HashSet<T>() {
    var elementsAdded: Int = 0
        private set
    
    override fun add(element: T): Boolean {
        elementsAdded++
        return super.add(element)
    }
    
    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return super.addAll(elements)
    }
}
```

这种实现看起来可能不错，但它并不能正确地工作：

```kotlin
val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) // 6
```

这是为什么呢？ 原因是 `HashSet` 在 `addAll` 的底层调用了 `add` 方法，然后对于使用 `addAll` 添加的每个元素，计数器都会自增两次，这个问题可以简单地通过删除自定义 `addAll` 来解决：

```kotlin
class CounterSet<T>: HashSet<T>() {
    var elementsAdded: Int = 0
        private set
        
    override fun add(element: T): Boolean {
        elementsAdded++
        return super.add(element)
    }
}
```

这个解决方案很危险。如果 Java 的创建者决定优化 `HashSet`、`addAll` 和实现它的方式，不依赖于 `add` 函数，会出现什么问题？如果他们这样子做了，这个实现将会随着 Java 的更新而异常，与此同时，任何其它依赖于我们当前实现的库也会异常。Java 创建者知道这一点，所以他们对更改这些类型的实现会非常谨慎。同样的问题会影响到任何库的创建者，甚至是大型项目的开发人员。我们如何解决这个问题呢？ 答案是应该使用组合而不是继承：

```kotlin
class CounterSet<T> {
    private val innerSet = HashSet<T>()
    var elementsAdded: Int = 0
    private set
    
    fun add(element: T) {
        elementsAdded++
        innerSet.add(element)
    }

    fun addAll(elements: Collection<T>) {
        elementsAdded += elements.size
        innerSet.addAll(elements)
    }
}

val counterList = CounterSet<String>()
counterList.addAll(listOf("A", "B", "C"))
print(counterList.elementsAdded) // 3
```

一个问题是，在这种情况下，我们失去了多态的行为： `CounterSet` 不再是一个 `Set`。为了实现它，我们可以使用委托模式，委托模式是当我们的类实现了一个接口时，我们可以组合一个实现相同接口的对象，并将接口中定义的方法转发给这个组合的对象。这种方法称为_转发方法_，来看看下面这个例子：

```kotlin
class CounterSet<T> : MutableSet<T> {
    private val innerSet = HashSet<T>()
    
    var elementsAdded: Int = 0
        private set
        
    override fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)
    }
    
    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)
    }
    
    override val size: Int
        get() = innerSet.size

    override fun contains(element: T): Boolean =
        innerSet.contains(element)

    override fun containsAll(elements: Collection<T>):
Boolean = innerSet.containsAll(elements)

    override fun isEmpty(): Boolean = innerSet.isEmpty()
    
    override fun iterator() = innerSet.iterator()

    override fun clear() = innerSet.clear()
    
    override fun remove(element: T): Boolean = 
        innerSet.remove(element)
    
    override fun removeAll(elements: Collection<T>):
Boolean = innerSet.removeAll(elements)

    override fun retainAll(elements: Collection<T>):
Boolean = innerSet.retainAll(elements)
}
```

现在的问题是我们需要实现许多转发方法（本例为9个），值得庆幸的是，Kotlin 引入了接口委托的支持，旨在帮助解决这类场景。当我们将接口委托给对象时，Kotlin 将在编译期间生成所有需要的转发方法。以下是 Kotlin 接口委托一个实现：

```kotlin
class CounterSet<T>(
    private val innerSet: MutableSet<T> = mutableSetOf()
) : MutableSet<T> by innerSet {
    var elementsAdded: Int = 0
    private set
    
    override fun add(element: T): Boolean {
        elementsAdded++
        return innerSet.add(element)
    }
    
    override fun addAll(elements: Collection<T>): Boolean {
        elementsAdded += elements.size
        return innerSet.addAll(elements)
    }
}
```

在这种情况下，委托是一个很好的选择：我们需要多态的行为，而继承会很危险。在大多数情况下不需要多态行为，或者我们以不同的方式使用它，在这种情况下没有委托的组合更适合，它更容易理解。

实际上，继承破坏封装是一个安全问题。但在许多情况下，行为是在合约中指定的，否则我们不会依赖它们（当方法为继承而设计时通常是这样），选择组合还有其它原因，这种组合更容易重用，并为我们提供了更多灵活性。

### 限制重写

为了防止开发人员去继承那些不是为了被继承而设计的类，我们可以将它们设置为 `final`，尽管出于某种原因需要允许继承，但默认情况下所有的方法都是 `final` 的。为了让开发人员重写，它们必须被设置为 `open`：

```kotlin
open class Parent {
    fun a() {}
    open fun b() {}
}

class Child: Parent() {
    override fun a() {} // Error
    override fun b() {}
}
```

明智地使用这种机制，只 `open` 那些为了继承而设计的方法，还要记住，当你重写一个方法时，你可以将它设为 `final`，而禁止子类再重写它：

```kotlin
open class ProfileLoader: InternetLoader() {
    
    final override fun loadFromInterner() {
        // load profile
    }
}
```

通过这种方法，你可以限制子类中可以重写方法的数目。

### 总结

在组合和继承之间有一些重要的区别：

* **组合更加安全** —— 我们不依赖一个类是如何实现的，而只依赖于它的外部可观察行为
* **组合更加灵活** —— 类只能继承一个类，但是能组合多个类。当我们继承的时，我们不得不承担一切，而使用组合的话，则可以选择我们需要的。当我们改变超类的行为时，我们也改变了所有子类的行为。仅改变某些子类的行为是很难的，而如果当我们组合的类发生变化，只有当它改变了对外部世界的合约时，它才会改变我们的行为
* **组合更加清晰** —— 使用继承既有优点也有缺点。当我们使用超类的方法时，我们不需要引用任何接收者（无需使用 this 关键字），它没有那么明显，这意味着我们不那么显式地调用它。 但它可能会让人困惑且更危险，因为它的出处很容易被混淆（它是来自哪里？当前类？超类？顶级类还是一个扩展？）。当我们在组合对象上调用一个方法时，我们就知道这个方法来自哪里了
* **组合的要求更高** —— 我们需要显式地组合对象。当我们向超类添加一些功能时，我们通常不需要修改子类。当我们使用组合时，我们更多时候需要调整其使用。
* **继承给了我们强大的多态行为** —— 这也是一把双刃剑，把狗当做动物来对待是很舒服的，另一方法，它也非常具有约束性，它一定是一只动物，动物的每个子类应该与动物一致。超类设置合约，子类应该遵守它

一般的 OOP 规则更喜欢组合而不是继承，但 Kotlin 通过将所有类和方法默认为 `final`，并将接口委托作为一级公民，更鼓励我们去组合。使得这个规则在 Kotlin 项目中更加重要。

那么什么时候组合更合理呢？ 经验法则：当有明确 "is-a"（是什么） 的关系时，我们应该使用继承。 不仅是在语言上，而且意味着每个继承自超类的类需要 "be" （成为）它的超类，子类可以通过所有父类的单元测试（里斯替换原则）。用于显示视图的面向对象框架就是很好的例子： `JavaFX` 中的 `Application`，`Android` 中的 `Activity`，`iOS` 中的 `UIViewController` 和 `React` 中的 `React.Component`。 当我们定义自己的特殊类型的视图时也是如此，它总是具有相同的功能和特征集。只要记住设计这些需要被继承的类时，要明确应该如何继承。此外，那些不是为了被继承而设计的方法要被设置为 `final`。
