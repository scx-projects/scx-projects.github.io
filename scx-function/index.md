# SCX Function

SCX Function 是一组结构统一的函数式接口。

它的目标是提供比 JDK 自带 `Supplier`、`Consumer`、`Function`、`BiFunction` 更中立、更统一，并且支持受检异常传播的函数接口。

SCX Function 本身不提供执行框架，也不提供异步能力。它只提供函数式接口定义。

这些接口适合用于：

```text
回调
模板方法
事务作用域
资源作用域
路由处理
拦截器
事件监听
高阶函数
需要抛出 checked exception 的 lambda
```

[GitHub](https://github.com/scx-projects/scx-function)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-function</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Function 中最核心的概念包括：

```text
Function0        0 个参数，有返回值
Function1        1 个参数，有返回值
Function2        2 个参数，有返回值
...
Function9        9 个参数，有返回值

Function0Void    0 个参数，无返回值
Function1Void    1 个参数，无返回值
Function2Void    2 个参数，无返回值
...
Function9Void    9 个参数，无返回值
```

命名规则非常直接：

```text
Function + 参数个数
Function + 参数个数 + Void
```

例如：

```text
Function0        () -> R
Function1        A -> R
Function2        (A, B) -> R
Function3Void    (A, B, C) -> void
```

所有接口的方法名都固定为：

```java
apply
```

所有接口都支持声明异常类型：

```java
X extends Throwable
```

也就是说，SCX Function 的核心特点是：

```text
结构统一
命名中立
支持 checked exception
只表达函数形状，不表达业务语义
```

## 为什么不直接用 JDK 函数式接口

JDK 已经有很多函数式接口，例如：

```text
Supplier
Consumer
Function
BiFunction
Predicate
Runnable
Callable
```

但是这些接口存在几个问题。

### 1. 命名不够中立

例如：

```text
Supplier     表示供应者
Consumer     表示消费者
Function     表示函数
BiFunction   表示二元函数
Predicate    表示断言
```

这些名称有一定语义色彩。

但很多时候我们只想表达：

```text
这个回调接收几个参数
有没有返回值
会不会抛异常
```

而不是表达“供应者”“消费者”“断言”这种语义。

SCX Function 选择了更中立的命名：

```text
Function0
Function1
Function2
Function1Void
Function2Void
```

它只表达函数形状。

### 2. JDK 接口不支持 checked exception

例如 JDK 的 `Function<T, R>` 是：

```java
R apply(T t);
```

它不能声明：

```java
throws IOException
```

所以如果 lambda 内部抛出受检异常，通常需要手动包装。

```java
Function<Path, String> readText = path -> {
    try {
        return Files.readString(path);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
};
```

SCX Function 可以直接写：

```java
Function1<Path, String, IOException> readText = path -> {
    return Files.readString(path);
};
```

异常可以自然传播。

### 3. JDK 接口命名体系不统一

JDK 中：

```text
0 参数有返回值       Supplier
0 参数无返回值       Runnable
1 参数有返回值       Function
1 参数无返回值       Consumer
2 参数有返回值       BiFunction
2 参数无返回值       BiConsumer
返回 boolean        Predicate
```

它们不是一套统一命名体系。

SCX Function 使用统一命名：

```text
Function0
Function0Void
Function1
Function1Void
Function2
Function2Void
...
Function9
Function9Void
```

看名字就知道函数形状。

## 快速开始

### 有返回值函数

```java
import dev.scx.function.Function1;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function1<Path, String, IOException> readText = path -> {
    return Files.readString(path);
};

String text = readText.apply(Path.of("README.md"));
```

这里的类型含义是：

```text
Path         参数类型
String       返回值类型
IOException 可能抛出的异常类型
```

### 无返回值函数

```java
import dev.scx.function.Function1Void;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function1Void<Path, IOException> deleteFile = path -> {
    Files.delete(path);
};

deleteFile.apply(Path.of("temp.txt"));
```

这里：

```text
Path         参数类型
IOException 可能抛出的异常类型
```

因为是 `Void` 版本，所以没有返回值类型。

## 接口列表

SCX Function 当前提供 0 到 9 个参数的函数接口。

### 有返回值接口

```text
Function0<R, X>
Function1<A, R, X>
Function2<A, B, R, X>
Function3<A, B, C, R, X>
Function4<A, B, C, D, R, X>
Function5<A, B, C, D, E, R, X>
Function6<A, B, C, D, E, F, R, X>
Function7<A, B, C, D, E, F, G, R, X>
Function8<A, B, C, D, E, F, G, H, R, X>
Function9<A, B, C, D, E, F, G, H, I, R, X>
```

其中：

```text
A, B, C...   参数类型
R            返回值类型
X            异常类型，必须 extends Throwable
```

### 无返回值接口

```text
Function0Void<X>
Function1Void<A, X>
Function2Void<A, B, X>
Function3Void<A, B, C, X>
Function4Void<A, B, C, D, X>
Function5Void<A, B, C, D, E, X>
Function6Void<A, B, C, D, E, F, X>
Function7Void<A, B, C, D, E, F, G, X>
Function8Void<A, B, C, D, E, F, G, H, X>
Function9Void<A, B, C, D, E, F, G, H, I, X>
```

其中：

```text
A, B, C...   参数类型
X            异常类型，必须 extends Throwable
```

因为是 `Void` 版本，所以没有 `R`。

## Function0

`Function0` 表示 0 个参数、有返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function0<R, X extends Throwable> {

    R apply() throws X;

}
```

示例：

```java
import dev.scx.function.Function0;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function0<String, IOException> readConfig = () -> {
    return Files.readString(Path.of("config.txt"));
};

String config = readConfig.apply();
```

适合表示：

```text
延迟求值
工厂方法
资源创建
无参数计算
可抛异常的 Supplier
```

## Function0Void

`Function0Void` 表示 0 个参数、无返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function0Void<X extends Throwable> {

    void apply() throws X;

}
```

示例：

```java
import dev.scx.function.Function0Void;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function0Void<IOException> clean = () -> {
    Files.deleteIfExists(Path.of("temp.txt"));
};

clean.apply();
```

适合表示：

```text
清理动作
无参数任务
资源关闭
可抛异常的 Runnable
```

## Function1

`Function1` 表示 1 个参数、有返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function1<A, R, X extends Throwable> {

    R apply(A a) throws X;

}
```

示例：

```java
import dev.scx.function.Function1;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function1<Path, String, IOException> readText = path -> {
    return Files.readString(path);
};

String text = readText.apply(Path.of("README.md"));
```

适合表示：

```text
单参数转换
资源处理
解析函数
可抛异常的 Function
```

## Function1Void

`Function1Void` 表示 1 个参数、无返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function1Void<A, X extends Throwable> {

    void apply(A a) throws X;

}
```

示例：

```java
import dev.scx.function.Function1Void;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function1Void<Path, IOException> delete = path -> {
    Files.delete(path);
};

delete.apply(Path.of("temp.txt"));
```

适合表示：

```text
单参数消费
事件监听
资源处理
可抛异常的 Consumer
```

## Function2

`Function2` 表示 2 个参数、有返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function2<A, B, R, X extends Throwable> {

    R apply(A a, B b) throws X;

}
```

示例：

```java
import dev.scx.function.Function2;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function2<Path, String, Path, IOException> writeText = (path, text) -> {
    return Files.writeString(path, text);
};

Path path = writeText.apply(Path.of("out.txt"), "hello");
```

适合表示：

```text
双参数转换
二元计算
写入函数
可抛异常的 BiFunction
```

## Function2Void

`Function2Void` 表示 2 个参数、无返回值的函数。

可以理解为：

```java
@FunctionalInterface
public interface Function2Void<A, B, X extends Throwable> {

    void apply(A a, B b) throws X;

}
```

示例：

```java
import dev.scx.function.Function2Void;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function2Void<Path, String, IOException> writeText = (path, text) -> {
    Files.writeString(path, text);
};

writeText.apply(Path.of("out.txt"), "hello");
```

适合表示：

```text
双参数消费
写入动作
回调处理
可抛异常的 BiConsumer
```

## Function3 到 Function9

`Function3` 到 `Function9` 用于表达更多参数的函数。

例如 `Function3`：

```java
@FunctionalInterface
public interface Function3<A, B, C, R, X extends Throwable> {

    R apply(A a, B b, C c) throws X;

}
```

示例：

```java
import dev.scx.function.Function3;

Function3<Integer, Integer, Integer, Integer, Exception> sum3 = (a, b, c) -> {
    return a + b + c;
};

int result = sum3.apply(1, 2, 3);
```

`Function9` 可以理解为：

```java
@FunctionalInterface
public interface Function9<A, B, C, D, E, F, G, H, I, R, X extends Throwable> {

    R apply(A a, B b, C c, D d, E e, F f, G g, H h, I i) throws X;

}
```

示例：

```java
import dev.scx.function.Function9;

Function9<
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Integer,
    Exception
> sum9 = (a, b, c, d, e, f, g, h, i) -> {
    return a + b + c + d + e + f + g + h + i;
};

int result = sum9.apply(1, 2, 3, 4, 5, 6, 7, 8, 9);
```

需要注意，参数越多，可读性越差。

如果参数数量超过 3 个，通常应该考虑是否需要把参数封装成对象。

## Function3Void 到 Function9Void

`Function3Void` 到 `Function9Void` 用于表达更多参数、无返回值的函数。

例如 `Function3Void`：

```java
@FunctionalInterface
public interface Function3Void<A, B, C, X extends Throwable> {

    void apply(A a, B b, C c) throws X;

}
```

示例：

```java
import dev.scx.function.Function3Void;

Function3Void<String, Integer, Boolean, Exception> print = (name, age, active) -> {
    System.out.println(name);
    System.out.println(age);
    System.out.println(active);
};

print.apply("Tom", 18, true);
```

`Function9Void` 可以理解为：

```java
@FunctionalInterface
public interface Function9Void<A, B, C, D, E, F, G, H, I, X extends Throwable> {

    void apply(A a, B b, C c, D d, E e, F f, G g, H h, I i) throws X;

}
```

## checked exception 支持

SCX Function 最大的实际价值之一，是支持 checked exception。

例如 JDK 的 `Function` 不能直接这样写：

```java
Function<Path, String> readText = path -> {
    return Files.readString(path);
};
```

因为 `Files.readString(...)` 会抛出 `IOException`。

使用 SCX Function 可以自然表达：

```java
import dev.scx.function.Function1;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

Function1<Path, String, IOException> readText = path -> {
    return Files.readString(path);
};
```

调用时也可以继续向外抛出：

```java
public String load(Path path) throws IOException {
    Function1<Path, String, IOException> readText = p -> {
        return Files.readString(p);
    };

    return readText.apply(path);
}
```

这比把异常强行包装成 `RuntimeException` 更自然。

## 多异常类型

泛型参数 `X` 只能是一个异常类型。

如果一个函数可能抛出多种 checked exception，可以使用它们的共同父类型。

例如：

```java
Function1<Path, String, Exception> load = path -> {
    if (!Files.exists(path)) {
        throw new FileNotFoundException(path.toString());
    }
    return Files.readString(path);
};
```

也可以使用更宽的：

```java
Function1<Path, String, Throwable> load = path -> {
    return Files.readString(path);
};
```

推荐根据 API 边界选择尽可能准确的异常类型。

```text
能写 IOException 就不要写 Exception
能写 Exception 就不要写 Throwable
```

## 无异常函数

如果函数不会抛出 checked exception，可以把异常类型写成 `RuntimeException`。

```java
Function1<Integer, Integer, RuntimeException> square = x -> {
    return x * x;
};
```

或者写成不会实际抛出的异常类型。

```java
Function2<Integer, Integer, Integer, RuntimeException> add = (a, b) -> {
    return a + b;
};
```

因为 lambda 内部不抛 checked exception，所以这里不会有额外负担。

## 与 JDK 接口对照

可以把 SCX Function 和 JDK 接口大致对应起来。

```text
Function0<R, X>              Supplier<R>
Function0Void<X>             Runnable
Function1<A, R, X>           Function<A, R>
Function1Void<A, X>          Consumer<A>
Function2<A, B, R, X>        BiFunction<A, B, R>
Function2Void<A, B, X>       BiConsumer<A, B>
```

但是 SCX Function 和 JDK 接口有两个关键差异：

```text
1. SCX Function 支持 throws X
2. SCX Function 命名完全按参数个数和返回值有无统一
```

## 在模板方法中使用

SCX Function 很适合用在模板方法中。

例如一个资源作用域：

```java
import dev.scx.function.Function1;

import java.io.IOException;

public static <R, X extends Throwable> R withResource(
        Function1<Resource, R, X> handler
) throws IOException, X {
    var resource = openResource();

    try {
        return handler.apply(resource);
    } finally {
        resource.close();
    }
}
```

调用：

```java
String result = withResource(resource -> {
    return resource.readText();
});
```

这里 `handler` 可以抛出自己的 checked exception。

模板方法不需要把它包装成运行时异常。

## 在无返回值模板中使用

如果 handler 不需要返回值，可以使用 `Function1Void`。

```java
import dev.scx.function.Function1Void;

import java.io.IOException;

public static <X extends Throwable> void withResource(
        Function1Void<Resource, X> handler
) throws IOException, X {
    var resource = openResource();

    try {
        handler.apply(resource);
    } finally {
        resource.close();
    }
}
```

调用：

```java
withResource(resource -> {
    resource.write("hello");
});
```

## 在事务中使用

SCX Function 也适合事务作用域。

例如：

```java
import dev.scx.function.Function0;

public <R, X extends Throwable> R autoTransaction(
        Function0<R, X> handler
) throws X {
    var tx = begin();

    try {
        R result = handler.apply();
        tx.commit();
        return result;
    } catch (Throwable e) {
        tx.rollback();
        throw e;
    } finally {
        tx.close();
    }
}
```

调用：

```java
var result = autoTransaction(() -> {
    repo1.update(...);
    repo2.delete(...);
    return repo3.find(...);
});
```

无返回值版本：

```java
import dev.scx.function.Function0Void;

public <X extends Throwable> void autoTransaction(
        Function0Void<X> handler
) throws X {
    autoTransaction(() -> {
        handler.apply();
        return null;
    });
}
```

## 在回调中使用

普通回调接口通常会写成业务语义名称。

例如：

```java
@FunctionalInterface
public interface UserHandler {

    void handle(User user) throws Exception;

}
```

如果你不想专门定义一个接口，也可以直接使用：

```java
Function1Void<User, Exception> handler
```

示例：

```java
public void eachUser(Function1Void<User, Exception> handler) throws Exception {
    for (var user : users) {
        handler.apply(user);
    }
}
```

调用：

```java
eachUser(user -> {
    System.out.println(user.name());
});
```

如果这个回调在业务上非常重要，仍然推荐定义具名接口。

如果只是工具方法或内部高阶函数，使用 `Function1Void` 会更轻量。

## 在拦截器中使用

可以用 `Function0` 表示“继续执行后续逻辑”。

```java
import dev.scx.function.Function0;

public <R, X extends Throwable> R intercept(
        Function0<R, X> next
) throws X {
    System.out.println("before");

    try {
        return next.apply();
    } finally {
        System.out.println("after");
    }
}
```

调用：

```java
String result = intercept(() -> {
    return service.doSomething();
});
```

如果后续逻辑无返回值：

```java
import dev.scx.function.Function0Void;

public <X extends Throwable> void intercept(
        Function0Void<X> next
) throws X {
    System.out.println("before");

    try {
        next.apply();
    } finally {
        System.out.println("after");
    }
}
```

## 在资源关闭中使用

可以用 `Function0Void` 表示可抛异常的关闭动作。

```java
import dev.scx.function.Function0Void;

public static <X extends Throwable> void closeQuietly(
        Function0Void<X> closeAction
) {
    try {
        closeAction.apply();
    } catch (Throwable ignored) {
        // ignore
    }
}
```

调用：

```java
closeQuietly(() -> {
    resource.close();
});
```

## 在解析器中使用

例如有一个解析函数：

```java
import dev.scx.function.Function1;

public static <T, X extends Throwable> T parse(
        String text,
        Function1<String, T, X> parser
) throws X {
    return parser.apply(text);
}
```

调用：

```java
Integer value = parse("123", text -> {
    return Integer.parseInt(text);
});
```

如果解析器可能抛出 checked exception：

```java
Config config = parse(text, source -> {
    return loadConfig(source);
});
```

## 与业务接口的取舍

SCX Function 的命名是结构化的。

例如：

```text
Function2<User, Role, Boolean, Exception>
```

它表达的是：

```text
两个参数
返回 Boolean
可能抛 Exception
```

但它不表达业务含义。

如果某个函数在业务上有明确语义，建议定义具名接口。

例如：

```java
@FunctionalInterface
public interface PermissionChecker {

    boolean check(User user, Role role) throws PermissionException;

}
```

这比下面这种更有业务可读性：

```java
Function2<User, Role, Boolean, PermissionException>
```

推荐原则是：

```text
工具层、抽象层、内部高阶函数      使用 SCX Function
稳定业务概念、公共业务 API        定义具名接口
```

## 参数数量建议

虽然 SCX Function 提供到 9 个参数，但并不表示推荐写 9 参数函数。

一般建议：

```text
0 ~ 2 个参数    非常适合
3 个参数        可以接受
4 个以上参数    应考虑封装参数对象
```

例如，不推荐：

```java
Function6<String, Integer, Boolean, Long, String, Double, Result, Exception> func;
```

更推荐：

```java
Function1<Request, Result, Exception> func;
```

其中：

```java
public record Request(
    String name,
    Integer age,
    Boolean active,
    Long id,
    String type,
    Double score
) {

}
```

参数对象更容易维护，也更容易扩展。

## 为什么没有基本类型特化

JDK 中有一些基本类型特化接口：

```text
IntFunction
ToIntFunction
IntConsumer
LongSupplier
DoublePredicate
```

它们可以避免装箱拆箱。

但如果要对所有参数数量、所有基本类型组合、所有返回类型全面特化，接口数量会爆炸。

例如：

```text
int 参数
long 参数
double 参数
boolean 参数
多参数组合
返回值组合
void / non-void
checked exception
```

组合数量会迅速变得不可维护。

SCX Function 选择保持轻量，只提供泛型版本。

如果某个性能热点确实需要基本类型特化，可以在具体业务代码中定义专用接口。

## 异常传播模式

SCX Function 推荐让异常自然向外传播。

例如：

```java
public <R, X extends Throwable> R run(Function0<R, X> handler) throws X {
    return handler.apply();
}
```

调用：

```java
String text = run(() -> {
    return Files.readString(Path.of("README.md"));
});
```

这里 `IOException` 不会丢失。

它会通过泛型 `X` 向外传播。

## 捕获异常

如果你希望在工具方法内部捕获异常，也可以捕获 `Throwable`。

```java
public static <R, X extends Throwable> R runAndLog(
        Function0<R, X> handler
) throws X {
    try {
        return handler.apply();
    } catch (Throwable e) {
        System.err.println(e.getMessage());
        throw e;
    }
}
```

需要注意，如果捕获 `Throwable` 后再抛出，应确保不要吞掉原始异常。

## 异常包装

如果你的 API 不想暴露 checked exception，可以在边界处包装。

```java
public static <R> R unchecked(Function0<R, ?> handler) {
    try {
        return handler.apply();
    } catch (Throwable e) {
        throw new RuntimeException(e);
    }
}
```

调用：

```java
String text = unchecked(() -> {
    return Files.readString(Path.of("README.md"));
});
```

但这不是 SCX Function 的默认建议。

SCX Function 的优势正是让 checked exception 可以自然传播。

只有在 API 边界确实不允许 checked exception 时，才应该包装。

## 与 lambda 的关系

所有 SCX Function 接口都带有：

```java
@FunctionalInterface
```

因此可以直接使用 lambda。

```java
Function2<Integer, Integer, Integer, RuntimeException> add = (a, b) -> {
    return a + b;
};
```

也可以使用方法引用。

```java
Function1<String, Integer, NumberFormatException> parseInt = Integer::parseInt;
```

无返回值：

```java
Function1Void<String, RuntimeException> println = System.out::println;
```

## 完整示例：资源作用域

```java
import dev.scx.function.Function1;
import dev.scx.function.Function1Void;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class FunctionDemo {

    public static void main(String[] args) throws Exception {
        String text = withTempFile(path -> {
            Files.writeString(path, "hello");
            return Files.readString(path);
        });

        System.out.println(text);

        withTempFileVoid(path -> {
            Files.writeString(path, "world");
            System.out.println(Files.readString(path));
        });
    }

    public static <R, X extends Throwable> R withTempFile(
            Function1<Path, R, X> handler
    ) throws IOException, X {
        var path = Files.createTempFile("scx-function-", ".txt");

        try {
            return handler.apply(path);
        } finally {
            Files.deleteIfExists(path);
        }
    }

    public static <X extends Throwable> void withTempFileVoid(
            Function1Void<Path, X> handler
    ) throws IOException, X {
        withTempFile(path -> {
            handler.apply(path);
            return null;
        });
    }

}
```

这个示例中：

```text
withTempFile 自己可能抛 IOException
handler 也可以抛自己的 X
两种异常都可以自然传播
```

## 完整示例：组合函数

可以用 `Function1` 实现简单组合。

```java
import dev.scx.function.Function1;

public static <A, B, C, X extends Throwable> Function1<A, C, X> compose(
        Function1<A, B, X> first,
        Function1<B, C, X> second
) {
    return a -> {
        B b = first.apply(a);
        return second.apply(b);
    };
}
```

调用：

```java
Function1<String, Integer, Exception> parse = text -> {
    return Integer.parseInt(text);
};

Function1<Integer, String, Exception> format = value -> {
    return "value = " + value;
};

var f = compose(parse, format);

String result = f.apply("123");
```

结果：

```text
value = 123
```

## 完整示例：事务模板

```java
import dev.scx.function.Function0;
import dev.scx.function.Function0Void;

public class TxTemplate {

    public <R, X extends Throwable> R autoTransaction(
            Function0<R, X> handler
    ) throws X {
        var tx = begin();

        try {
            R result = handler.apply();
            tx.commit();
            return result;
        } catch (Throwable e) {
            tx.rollback();
            throw e;
        } finally {
            tx.close();
        }
    }

    public <X extends Throwable> void autoTransaction(
            Function0Void<X> handler
    ) throws X {
        autoTransaction(() -> {
            handler.apply();
            return null;
        });
    }

    private Tx begin() {
        return new Tx();
    }

    private static class Tx {

        void commit() {

        }

        void rollback() {

        }

        void close() {

        }

    }

}
```

调用：

```java
var template = new TxTemplate();

String result = template.autoTransaction(() -> {
    return "ok";
});

template.autoTransaction(() -> {
    System.out.println("done");
});
```

## 设计说明

### 1. 命名保持绝对中立

SCX Function 不使用：

```text
Supplier
Consumer
Predicate
Handler
Callback
Processor
```

这些带有语义色彩的名称。

它只使用：

```text
Function0
Function1
Function2
Function0Void
Function1Void
Function2Void
```

这种结构化命名。

这样接口本身只表达函数形状，不表达业务意图。

### 2. 参数个数写在类型名里

`Function3` 表示 3 个参数。

`Function3Void` 表示 3 个参数且无返回值。

这让类型名称和函数形状保持一致。

### 3. Void 表示无返回值

`Void` 后缀表示方法返回 `void`。

例如：

```text
Function1       A -> R
Function1Void   A -> void
```

它不是返回 `java.lang.Void`。

### 4. 方法名统一为 apply

所有接口的方法名都是：

```java
apply
```

这样调用方式保持一致：

```java
function.apply(...)
```

不会出现：

```text
get
accept
test
run
call
handle
```

等多种方法名混杂的情况。

### 5. 支持 checked exception

每个接口都有异常泛型：

```java
X extends Throwable
```

这让 lambda 可以直接抛出 checked exception。

这也是它和 JDK 函数式接口最大的差异之一。

### 6. 不做基本类型全面特化

SCX Function 只提供泛型版本。

这保持了接口数量可控。

如果某个性能热点需要避免装箱拆箱，应在那个热点上定义专用接口，而不是让基础库提供指数级数量的组合接口。

### 7. 不提供函数组合工具

SCX Function 当前只定义接口。

它不提供：

```text
compose
andThen
memoize
retry
unchecked
```

这类函数工具。

这些能力如果需要，可以在上层工具库中提供。

## 常见问题

### SCX Function 是干什么的？

它是一组支持 checked exception 的函数式接口。

主要用于 lambda、回调、高阶函数和模板方法。

### 它和 JDK Function 有什么区别？

主要区别是：

```text
SCX Function 支持 throws X
命名更统一
接口覆盖 0 到 9 个参数
同时提供 void 版本
```

### 为什么叫 Function1，而不是 Function？

因为 `Function1` 明确表示 1 个参数。

`Function2` 表示 2 个参数。

这种命名比 `Function` / `BiFunction` 更统一。

### Function1Void 和 Consumer 有什么区别？

`Function1Void<A, X>` 类似于 `Consumer<A>`。

但它可以抛出 `X extends Throwable`。

```java
Function1Void<Path, IOException> delete = path -> {
    Files.delete(path);
};
```

### Function0 和 Supplier 有什么区别？

`Function0<R, X>` 类似于 `Supplier<R>`。

但它可以抛出 `X extends Throwable`。

### Function0Void 和 Runnable 有什么区别？

`Function0Void<X>` 类似于 `Runnable`。

但它可以抛出 `X extends Throwable`。

### 为什么异常类型是泛型？

因为不同函数可能抛出不同异常。

例如：

```java
Function1<Path, String, IOException>
Function1<String, Integer, NumberFormatException>
Function0<String, SQLException>
```

这样 API 可以保留准确的异常类型。

### 如果不抛异常，X 写什么？

通常可以写：

```java
RuntimeException
```

例如：

```java
Function1<Integer, Integer, RuntimeException> square = x -> x * x;
```

### 可以抛多个 checked exception 吗？

一个 `X` 只能表示一个类型。

如果可能抛多种异常，可以使用共同父类型。

```java
Function1<Path, String, Exception>
```

或者：

```java
Function1<Path, String, Throwable>
```

### 为什么没有 Predicate？

因为 Predicate 本质上就是返回 `boolean` 的函数。

可以使用：

```java
Function1<User, Boolean, Exception>
```

如果业务语义很重要，也可以自己定义：

```java
@FunctionalInterface
public interface UserPredicate {

    boolean test(User user) throws Exception;

}
```

### 为什么没有 Supplier / Consumer？

因为它们可以由结构化命名表达：

```text
Supplier<T>     -> Function0<T, X>
Consumer<T>     -> Function1Void<T, X>
```

### 为什么只到 9 个参数？

9 个参数已经覆盖绝大多数工具层场景。

如果还需要更多参数，通常应该封装参数对象。

### 推荐使用很多参数的 Function9 吗？

不推荐。

`Function9` 只是提供能力。

实际设计中，如果参数超过 3 到 4 个，通常应该考虑参数对象。

### Void 版本是不是返回 java.lang.Void？

不是。

Void 版本返回真正的 `void`。

例如：

```java
void apply(A a) throws X;
```

### 可以用方法引用吗？

可以。

```java
Function1<String, Integer, NumberFormatException> parse = Integer::parseInt;
```

### 可以用 lambda 吗？

可以。

所有接口都有 `@FunctionalInterface`。

### SCX Function 会捕获异常吗？

不会。

接口只声明异常。

是否捕获、包装、记录、重试，都由调用方决定。

### SCX Function 会把 checked exception 转 RuntimeException 吗？

不会。

它的设计目标正好相反：让 checked exception 可以自然传播。

### 它适合写公共业务 API 吗？

看情况。

如果接口有明确业务语义，推荐定义具名业务接口。

如果只是工具层、高阶函数或内部回调，使用 SCX Function 更轻量。

### 它适合和 SCX Transaction 一起用吗？

适合。

事务模板、作用域绑定、自动事务等场景都需要接收可抛异常的回调。

SCX Function 正好适合这种场景。

### 它有运行时依赖吗？

没有复杂运行时逻辑。

它主要是一组接口定义。
