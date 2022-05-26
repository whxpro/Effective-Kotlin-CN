# 第51条：性能关键处考虑使用原语的数组

我们不能在 Koltin 中声明原语，但作为一种优化，我们将在底层使用它们。这是一个重要的优化，已在第_45条：避免不必要的对象创建_中描述。原语是：

* 轻量的，因为每个对象都有额外的重量
* 更快的，因为通过访问器访问值有额外的成本

因此，在使用大量数据时，使用原语可能是一种重要的优化。一个问题是，在 Kotlin 中，典型的集合（如 `List` 或 `Set`）是泛型类型，原语不能用做泛型类型，因此我们最终改用包装类型。这是一种方便的解决方案并适合大部分场景，因为它比标准集合更容易进行处理。也就是说，在代码的性能关键部分，我们应该考虑使用带有原语的数组，如 `IntArray` 或 `LongArray`，因为它们的内存更轻量，处理效率更高。

| Kotlin type  | Java type       |
| ------------ | --------------- |
| `Int`        | `int`           |
| `List<Int>`  | `List<Integer>` |
| `Array<Int>` | `Integer[]`     |
| `IntArray`   | `int[]`         |

带原语的数组有多轻量？假设在 Kotlin/JVM 中，我们需要保存1,000,000个整数，我们可以将它保存在 `IntArray` 或 `List<Int>`。当你进行检查时，你会发现 `IntArray` 分配了4,00,016字节，而 `List<Int>` 则分配了20,000,040，是其5倍多！如果你对内存进行了优化，并且保留了具有基本类型的类型集合，比如 `Int`，则你阔以选择具有基本类型的数组。

```kotlin
import jdk.nashorn.internal.ir.debug.ObjectSizeCalculator.getObjectSize

fun main() {
    val ints = List(1_000_000) { it }
    val array: Array<Int> = ints.toTypedArray()
    val intArray: IntArray = ints.toIntArray()
    println(getObjectSize(ints)) // 20 000 040
    println(getObjectSize(array)) // 20 000 016
    println(getObjectSize(intArray)) // 4 000 016
}
```

它们在表现上也有差异，对于1,00,000,000个数字集合，在计算平均值时，对原语的处理速度大约快25%。

```kotlin
open class InlineFilterBenchmark {

    lateinit var list: List<Int>
    lateinit var array: IntArray
    
    @Setup
    fun init() {
        list = List(1_000_000) { it }
        array = IntArray(1_000_000) { it }
    }
    
    @Benchmark
    // 平均 1,260,593 ns
    fun averageOnIntList(): Double {
        return list.average()
    }
    
    @Benchmark
    // 平均 868,509 ns
    fun averageOnIntArray(): Double {
        return array.average()
    }
}
```

你看到了\~ 原语和带有原语的数组可以为代码中性能关键的部分进行优化。分配给它们的内存更少，处理速度更快，尽管在大多数情况下，这种优化还不足以使得带原语的数组能够代替列表（List）成为默认的集合。因为列表更直观，使用也更为频繁，所以在大多数情况下我们应该使用它。但是在需要优化某些性能的关键部分时，请记得使用数组。
