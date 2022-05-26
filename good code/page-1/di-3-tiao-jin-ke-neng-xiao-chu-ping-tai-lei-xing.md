# 第3条：尽可能消除平台类型

Kotlin 关于空安全的介绍非常棒。在编程社区中，Java 是因空指针异常（NPE）而闻名的，而 Kotlin 的空安全机制使这种异常很少出现，甚至完全消除了它们。尽管有一点不能完全保证，那就是不具有可靠安全性的语言（如 Java 和 C）与 Kotlin 之间的连接。假设我们使用一个 Java 方法声明 `String` 作为返回类型。在 Kotlin 中应该是什么类型？

如果它使用 `@Nullable` 注解，那么我们假设它是空的，并将指定为 `String?`。如果它带有 `@NotNull` 注解，那么我们可以信任这个非空注解，并指定其为 `String`，但是，如果这个返回类型没有使用这两个注释中的任何一个进行注释呢？

```kotlin
public class JavaTest {
    public String giveName() {
        // ...    
    }
}
```

我们可能会说：应该将这种类型视为可空。这确实是一个安全的方法，因为在 Java 中一切是可空的。然而，我们经常知道有些东西不是空的，所以我们最终会在代码很多地方使用非空断言 `!!`。

真正的问题出现在从 Java 中去获取泛型类型。假设 Java API 返回一个完全没有注解的类`List<User>`，如果 Kotlin 默认为可空类型，并且我们知道这个列表和那些用户都不是空的，我们不仅需要断言整个列表， 还要去过滤空值：

```
// Java 代码
public class UserRepo {
    public List<User> getUsers() {
            // ...    
    }
}

// Kotlin
val users: List<User> = UserRepo().users!!.filterNotNull()
```

如果函数返回的是一个 `List<List<User>>` 呢？ 那将会变得更加复杂了

```
val users: List<List<User>> = UserRepo().groupedUsers!!.map { it!!.filterNotNull }
```

列表至少有像 `map` 、 `filterNotNull` 这样的函数，如果在其它泛型类型中，可空性将会是一个更大的问题。这就是为什么在 Kotlin 中，来自 Java 并具有未知可空性的类型是一个特殊类型，而不是默认情况下可以直接视为可空类型的原因。我们称这种类型为**平台类型（platform type）**。

> 平台类型——来自另一种语言的类型，具有未知的可空性

平台类型在类型名称之后用一个感叹号 `!` 来表示，例如 `String!`，尽管这种表示方法并不能在代码中使用，**平台类型是不可表示的，这意味着不能在语言中明确地写出它们**。当平台类型被分配给 Kotlin 变量或属性时，它可以被推断出来，但是不能被明确地设置。相反的是，我们在选择赋予平台类型时，可以选择两种期望的类型，要么是可空类型，要么是非空类型。如下面代码所示：

```
// Java 代码
public class UserRepo {
    public User getUser() {
         // ...    
    }
}

// Kotlin 代码
val repo = UserRepo()
val user1 = repo.user        // user1 是 User!
val user2: User = repo.User  // user2 是 User
val user3: User? = repo.User // user3 是 User? 
```

得益于这个事实，我们就可以直接从 Kotlin 中获取 Java 的泛型类型：

```
val users: List<User> = UserRepo().users
val users: List<List<User>> = UserRepo().groupedUsers
```

问题是这样的做法仍然很危险，因为我们假设非空的东西，其有可能是空的。为了安全起见，我总是建议从 Java 获取平台类型时，需要非常谨慎地对待。记住，**即使一个函数现在不返回 null，也不会意味它将来不会改变**。如果设计者没有通过注解或者在注释中描述它，他们可以在不改变任何合约的情况下这样做。

如果你对需要与 Kotlin 互操作的 Java 代码有一定的控制，那么尽可能引入 `@Nullable` 和 \`@NotNull\` 注解。

```
import org.jetbrains.annotations.NotNull;

public class UserRepo {
    public @NotNull User getUser() {
        // ...    
    }
}
```

当我们想要更好的支持 Kotlin 开发者时，这是最重要的步骤之一（对于 Java 开发者来说，这也是重要的信息）。在 Kotlin 成为一等公民后，对许多暴露出来的类型加上注解，是 Android API 引入的最重要变化之一。这使得 Android API 对 Kotlin 更加友好。

注意，关于可空和非空的注解，有许多不同的类型，包括：

* JetBrains(来自 `org.jetbrains.jannotations` 的 `@Nullable` 和 `@NotNull`)
* Android (来自 `androidx.annotation` 的 `@Nullable` 和 `@NotNull`，同样的注解在 `android.support.annotations` 也有
* JSR-305 (来自 `javax.annotation` 的 `@Nullable`、`@CheckForNull` 和 `@Nonnull`
* JavaX (来自 `javax.annotation` 的 `@Nullable`、 `@CheckForNull` 和 `@Nonnull`
* FindBugs (来自 `edu.umd.cs.findbugs.annotations` 的 `@Nullable`、 `@CheckForNull` 、 `@PossiblyNull` 和 `@NonNull`
* ReactiveX (来自 `io.reactivex.annotations` 的 `@Nullable` 和 `@NonNull`
* Eclipse (来自 `org.eclipse.jdt.annotation` 的 `@Nullable` 和 `@NonNull`
* Lombok (来自 `lombok`) 的 `@NonNull`

或者，你可以考虑在 Java 中使用 JSR305 的 `@ParametersAreNonnullByDefault` 注解来指定所有类型默认为非空。

我们也可以在 Kotlin 代码中做一些事情。出于安全考虑，我的建议是尽快取消这些平台类型，至于为什么，请思考 `startedType` 和 `platformType` 在本例中的行为差异。

```
// Java
public class JavaClass {
    public String getValue() {
        return null;    
    }
}

// Kotlin
fun statedType() {
    val value: String = JavaClass().value    
    // ...    
    println(value.length)
}

fun platformType() {
    val value = JavaClass().value
    // ...    
    println(value.length)
}
```

在这两种情况下，开发人员都认为 `getValue` 不会返回 null，而这样写都是错误的。在这两种情况下都会导致 NPE，但错误发生的位置有所不同。

在 `statedType` 中， NPE 会在 Java 获取值的同一行中被抛出，很明显，我们错误地假设了非空类型，结果得到了 null。 我们只需要修改这个类型为空，然后以此调整其余部分的代码。

在 `platformType` 中，当我们把这个值作为非空类型使用时，将会抛出 NPE。 可能来自某个更复杂的表达式里，作为平台类型的变量可以同时被视为可空或者非空。这样的变量可能会被安全地使用几次，然后不安全地使用并抛出 NPE。当我们使用这些属性时，类型系统并不能保护我们。这与 Java 中的情况类似，但在 Kotlin 中，我们并不期望仅仅使用了一个来自 Java 的对象就出现 NPE。 很有可能迟早会有人不安全地使用它，然后我们的程序将会遇到运行时异常而崩溃，而其原因可能不那么容易找到。

```
// Java
public class JavaClass {
    public String getValue() {
        return null;
    }
}

// Kotlin
fun statedType() {
    val value: String = JavaClass().value  // NPE
    // ...
    println(value.length)
}
fun platformType() {
    val value = JavaClass().value
    // ...    
    println(value.length)  // NPE
}
```

更危险的是，平台类型可能会进一步传递，例如，我们可能将一个平台类型作为接口的一部分公开：

```
interface UserRepo {
    fun getUserName() = JavaClass().value
}
```

在本例中，函数推断类型是平台类型，这意味着任何人都可以决定它是否为空。你可以在选择定义时视其为可空，而在使用时视其不可空：

```
class RepoImpl : UserRepo {
    override fun getUserName(): String? {
        return null    
    }
}

fun main() {
    val repo: UserRepo = RepoImpl()    
    val text: String = repo.getUserName() // 运行时NPE    
    print("User name length is ${text.length}")
}
```

如上所示，**传递平台类型是导致重大灾难的原因**，它们是有问题的，出于安全原因，我们应该尽快消除它们，在本例中,IDEA IntelliJ 会抛出一个警告来帮助我们：

![](<../../.gitbook/assets/image (1) (1).png>)

### 总结

**来自另一种语言并具有未知的可空性类型被称为平台类型，**它们是危险的，我们应该尽快消灭它们，不要让它们散播出去。如果我们使用的 Java 构造函数、方法、字段上有指定的可空或非空的注解，那也是非常有用的，对于使用这些元素的 Java 和 Kotlin 开发人员来说，这都是宝贵的信息。
