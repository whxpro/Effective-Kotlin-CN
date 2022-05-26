# 第23条：避免隐藏类型参数

可以定义具有同名和同类型的局部属性，它会屏蔽外部作用域的属性，而且没有警告，因为这种情况并不罕见，而且对于开发人员来说是显而易见的：

```kotlin
class Forest(val name: String) {
    
    fun addTree(name: String) {
        // ...
    }
}
```

另一方面，当我们用函数的类型参数来隐藏类的类型参数时，也会发生同样的情况。这种情况就不太明显了，还可能会导致严重的问题，这个错误通常是由于开发人员不知道泛型是如何工作所导致的：

```kotlin
interface Tree
class Birch: Tree
class Spruce: Tree

class Forest<T: Tree> {
    
    fun <T: Tree> addTree(tree: T) {
        // ...
    }
}
```

这里的问题是 `Forest` 和 `addTree` 的类型参数是相互独立的：

```kotlin
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce())
```

这是我们不想出现的情况，并且这种代码可能会让人疑惑。 一种解决方法是 `addTree` 应该使用类的类型参数 T：

```kotlin
class Forest<T: Tree> {
    
    fun addTree(tree: T) {
           // ...
    }
}

// Usage
val forest = Forest<Birch>()
forest.addTree(Birch())
forest.addTree(Spruce()) // ERROR, type mismatch
```

如果需要引入新的类型参数，最好将其命名为不同的名称，注意，它可以被约束为其他类型参数的子类型：

```kotlin
class Forest<T: Tree> {
    
    fun <ST: T> addTree(tree: ST) {
        // ...
    }
}
```

### 总结

避免隐藏类型参数，当你看到类型参数被隐藏时，需要小心，与其他类型的参数不同，它不是直观的，可能会非常混乱。
