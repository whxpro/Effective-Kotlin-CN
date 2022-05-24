# 第9条： 使用 use 来关闭资源

有些资源是不能自动关闭的，一旦不再需要它们，我们就需要调用 `close` 方法，我们在 Kotlin/JVM 中使用 Java 标准库，涵盖了很多这样的资源，例如：

* `InputStream` 和 `OutputStream`
* java.sql.Connection
* java.io.Reader(`FileReader`、`BufferedReader`、`CSSParser`)
* java.new.Socket 和 java.util.Scanner

所有这些资源都实现了 `Closeable` 接口，其扩展了 `AutoCloseable` 接口

问题是在这些情况下，我们必须先确保我们不再需要这些资源，才能去执行其 `close` 方法，因为这些资源是相当昂贵的，它们不会轻易的关闭自身（如果我们不处理，垃圾收集器最终会收集它们，但这需要一些时间）。因此，为了确保关闭它们，我们通常将这些资源包装在 `try-finally` 块中，并在那里调用 close:

```
fun countCharacterInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    try {
        return reader.lineSequence().sumBy { it.length }
    } finally {
        reader.close()
    }
}
```

这样的代码结构既复杂又不正确，因为 `close` 函数可能会抛出错误，而这样的错误缺没有捕获。同样，如果 try 块 和 finally 块同时有错误，那么我们只能感知到某一个块的错误 。我们所期望的行为是将新出现错误的信息添加到前一个错误信息中去。这个函数的正确实现是漫长而又复杂的，但它也是常见的，因此它被抽象到 `use` 函数中。**它应该被用于资源和处理异常**。此函数可用于任何实现了 `Closeable` 的对象：

```
fun coyntCharactersInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    reader.use {
        return reader.lineSequence().sumBy { it.length }
    }
}
```

接收者(在本例中是 reader)作为参数传递到 Lambda 表达式中，所以该语法可以简化成:

```
fun coyntCharactersInFile(path: String): Int {
    BufferedReader(FileReader(path)).user { reader ->
          return reader.lineSequence().sumBy { it.length }
    }
}
```

由于文件经常需要这种支持，而且逐行读取也很常见，在 Kotlin 标准库中还有一个 `useLines` 函数，它为我们提供了一个所有行的序列(String)，并在处理完后关闭底层的 reader：

```
 fun countCharactersInFile(path: String): Int {
    File(path).useLines { lines ->
        return lines.sumBy { it.length }
    }
}
```

这是一个处理大文件的合适方式，因为我们需要读取文件多个 line，并且在内存中一次不保留超过一行，代价是这个序列只能使用一次，如果需要多次遍历文件中的内容，就需要多次打开文件。 `useLines` 也可以用作表达式：

```
 fun countCharactersInFile(path: String): Int = 
    File(path).useLines { lines -> lines.sumBy { it.length }}
```

上述所有实现都使用序列对文件进行操作，这是正确的做法。多亏了这一点，我们总是可以只准备一行，而不是加载整个文件内容。关于它的更多信息，请参见_第49条：在处理具有多个步骤的大型集合时，优先使用序列_。

### 总结

使用 `use` 代码块来处理实现了 `Closeable` 或 `AutoCloseable` 接口的资源。这是一个既安全又简单的选择，当需要对文件进行操作时，请考虑使用 `useLines` ，它会产生一个序列，用于读取每一行。
