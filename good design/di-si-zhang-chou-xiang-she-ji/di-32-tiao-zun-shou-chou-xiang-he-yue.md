# 第32条：遵守抽象合约

合约和可见性都是开发者之间的一种协议。用户几乎总是可能违反此协议。从技术上来讲，一个项目中的任何地方都可能被骇掉。例如，可以使用反射来打开和使用我们想要的任何东西：

```kotlin
class Employee {
    private val id: Int = 2
    
    override fun toString() = "User(id=$id)"
    
    private fun privateFunction() {
        println("Private function called")
    }
}

fun callPrivateFunction(employee: Employee) {
    employee::class.declaredMemberFunctions
        .first { it.name == "privateFunction" }
        .apply { isAccessible = true }
        .call(employee)
}

fun changeEmployeeId(employee: Employee, newId: Int) {
    employee::class.java.getDeclaredField("id")
        .apply { isAccessible = true }
        .set(employee, newId)
}

fun main() {
    val employee = Employee()
    callPrivateFunction(employee) // Prints: Private function called
    changeEmployeeId(employee, 1)
    print(employee) // Prints: User(id=1)
}
```

仅仅因为你能做某件事情，并不意味着做这件事情就是好的。在这个例子中，我们非常依赖于实现细节，比如私有属性和私有函数的名称，它们根本不是合约的一部分，因此我们随时都可能发生变化，这种代码对我们的项目来说就像是一个定时炸弹。

记住，合约就像防火墙一样，只要你正确的使用你的电脑，防火墙就会保护你。 当你打开电脑并开始骇入它时，就同时失去了防火墙的保护。 同样的原则也适用于此：当你违反合约时，实现的更改导致代码停止了工作，就是你的问题。

### 合约是继承的

当我们从类继承或者从另一个库扩展接口时，遵守合约特别重要。记住，你的目标实现应该遵守它们的合约，例如，每个类都继承 `Any` 的 `equals` 和 `hashCode` 方法，我们都要遵守其完善的合约，否则对象就无法正常工作。比如，当 `hashCode` 和 `equals` 不一致时，对象在 `HashSet` 的行为就可能不正确，下面行为是不正确的，因为一个集合不应该允许重复：

```kotlin
class Id(val id: Int) {
    override fun equals(other: Any?) = other is Id && other.id == id
}

val mutableSet = mutableSetOf(Id(1))
mutableSet.add(Id(1))
mutableSet.add(Id(1))
print(mutableSet.size) // 3
```

在本例中，是 `hashCode` 的实现与 `equals` 不一致，我们将在_第6章：类的设计_中讨论一些重要的 Kotlin 合约，现在，请记住检查你的重写的函数的期望，并遵守这些期望。

### 总结

如果你希望你的程序是稳定的，遵守合约，当你不得不打破某些规则时，那么就请好好记录下这个事实，这些信息对维护你代码的人员非常有用。也许在几年后，你也会成为那样的人。
