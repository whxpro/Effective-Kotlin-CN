# 第50条：限制操作步骤的数量

每一种集合处理函数都会有执行成本。对于标准集合处理，通常是对元素进行一次迭代，并在底层创建额外的集合。对于序列处理，它是包装整个集合的另个一个对象，该对象是用于进行操作的对象。两者都是昂贵的，特别是当处理的元素数量很多时。因此，我们主要通过组合那些处理操作，来限制集合处理步骤的数量。例如，我们使用 `filterNotNull`，而不是过滤非空元素然后再转化为非空类型。使用 `mapNotNull`，这样就不用先映射然后在过滤掉空值。

```kotlin
class Student(val name: String?)

// 使其工作的代码
fun List<Student>.getNames(): List<String> = this
    .map { it.name }
    .filter { it != null }
    .map { it!! }

// 更好的做法
fun List<Student>.getNames(): List<String> = this
    .map { it.name }
    .filterNotNull()

// 最好的做法
fun List<Student>.getNames(): List<String> = this
    .mapNotNull { it.name }
```

最大的问题，不是误解这些改变的重要性，而是我们缺乏应该使用哪些集合处理函数的知识。这是我们要去学习它的一个原因。此外，IDE 的警告常会建议我们引入更好的替代方案，这对我们也很有帮助。

![](<../../.gitbook/assets/image (6) (1) (1) (1).png>)

尽管如此，知道一些复合操作是更好的，下面是一些常用的函数调用和限制步骤数量的替代方法：

![](<../../.gitbook/assets/image (9) (1) (1).png>)

### 总结

Most collection processing steps require iteration over the whole collection and intermediate 大多数集合处理步骤需要对整个集合和中间集合创建进行迭代。可以通过使用更合适的集合处理功能来节约成本。
