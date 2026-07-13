# SCX Exception

SCX Exception 是一个用于异常包装和异常域隔离的轻量工具库。

它目前提供的核心类型是 `ScxWrappedException`。

`ScxWrappedException` 的主要作用不是创建一套复杂的异常体系，而是在方法接收外部逻辑时，用一个明确的包装异常区分：

```text
方法自身抛出的异常
外部逻辑抛出的异常
```

这在回调、高阶函数、模板方法、拦截器、事件处理器、流式处理等场景中非常有用。

[GitHub](https://github.com/scx-projects/scx-exception)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-exception</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Exception 中最核心的概念包括：

```text
ScxWrappedException     通用异常包装器
异常域                  异常来自哪一层逻辑
宿主逻辑                当前方法自身的逻辑
外部逻辑                当前方法接收并调用的外部回调、函数或处理器
解包                    从 ScxWrappedException 中取出真正的原始异常
```

它们之间的关系可以简单理解为：

```text
宿主方法
    ↓
调用外部逻辑
    ↓
外部逻辑抛出异常
    ↓
用 ScxWrappedException 包装
    ↓
调用方 catch ScxWrappedException
    ↓
通过 getUnwrappedCause() 获取真实异常
```

也就是说：

```text
宿主方法自己的异常     直接抛出
外部逻辑抛出的异常     包装成 ScxWrappedException 再抛出
```

这样调用方就能明确知道异常来自哪里。

## 快速开始

假设有一个方法 `read(...)`，它自己可能抛出 `IOException`，同时它接收的回调也可能抛出异常。

```java
public static void read(Func bytesConsumer, int length)
        throws IOException, ScxWrappedException {
    if (length > 2048) {
        throw new IOException("length too big");
    }

    try {
        bytesConsumer.apply(new byte[]{1, 2, 3});
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

调用时：

```java
try {
    read(bytes -> {
        throw new IOException("lambda error");
    }, 1024);
} catch (IOException e) {
    // 这里表示 read 方法自身抛出的 IOException
} catch (ScxWrappedException e) {
    // 这里表示外部逻辑抛出的异常
    Throwable realCause = e.getUnwrappedCause();
}
```

这样就可以区分：

```text
IOException             来自 read 方法自身
ScxWrappedException     来自 bytesConsumer 回调
```

## 为什么需要 ScxWrappedException

考虑下面这个方法：

```java
public void read(Func bytesConsumer, int length) throws IOException {
    // some code
    bytesConsumer.apply(new byte[]{1, 2, 3});
    // some code
}
```

调用方这样写：

```java
try {
    read(bytes -> {
        throw new IOException("lambda error");
    }, 1024);
} catch (IOException e) {
    e.printStackTrace();
}
```

这里有一个问题：

```text
这个 IOException 到底来自 read 方法自身？
还是来自 bytesConsumer 回调？
```

如果两边都可能抛出同一种异常，那么调用方只通过 `catch (IOException e)` 无法可靠判断异常来源。

这就是异常域模糊。

`ScxWrappedException` 的作用就是解决这个问题。

## 异常域隔离

异常域隔离的意思是：

```text
不要让宿主逻辑的异常和外部逻辑的异常混在同一个异常通道中
```

例如：

```java
public static void read(Func bytesConsumer, int length)
        throws IOException, ScxWrappedException {
    if (length > 2048) {
        throw new IOException("length too big");
    }

    try {
        bytesConsumer.apply(new byte[]{1, 2, 3});
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

在这个方法中：

```text
read 自己抛出的 IOException        直接抛出
bytesConsumer 抛出的任何异常        包装为 ScxWrappedException
```

调用方可以这样处理：

```java
try {
    read(bytes -> {
        throw new IOException("lambda error");
    }, 1024);
} catch (IOException e) {
    // 宿主方法 read 自己失败
} catch (ScxWrappedException e) {
    // 外部逻辑 bytesConsumer 失败
}
```

这样异常来源就是稳定的。

## 唯一推荐用法：包装外部逻辑抛出的所有异常

`ScxWrappedException` 的推荐用法是：

```text
只要异常来自外部逻辑，就统一包装
```

不要只包装某几种异常。

示例：

```java
public static void read(Func bytesConsumer, int length)
        throws IOException, ScxWrappedException {
    if (length > 2048) {
        throw new IOException("length too big");
    }

    try {
        bytesConsumer.apply(new byte[]{1, 2, 3});
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

这里不关心 `bytesConsumer` 抛出的到底是：

```text
IOException
RuntimeException
IllegalArgumentException
NullPointerException
Error
其它 Throwable
```

只要它来自 `bytesConsumer`，就统一包装为 `ScxWrappedException`。

这条规则非常重要。

因为它让异常来源判断变得简单、稳定、可组合。

```text
catch IOException             宿主逻辑异常
catch ScxWrappedException     外部逻辑异常
```

## 不推荐：选择性包装

不要只包装你认为“会混淆”的异常。

例如下面这种写法不推荐：

```java
public static void read(Func bytesConsumer, int length)
        throws IOException, ScxWrappedException {
    if (length > 2048) {
        throw new IOException("length too big");
    }

    try {
        bytesConsumer.apply(new byte[]{1, 2, 3});
    } catch (IOException e) {
        throw new ScxWrappedException(e);
    }
}
```

这看起来好像可以解决 `IOException` 的歧义。

但是它会引入新的问题：

```text
IOException             被包装
RuntimeException        直接透传
其它异常                可能直接透传
```

调用方又需要知道实现细节：

```text
哪些异常会被包装？
哪些异常不会被包装？
这个异常到底来自宿主逻辑还是外部逻辑？
```

这样异常域又变得不稳定。

## 选择性包装的问题

选择性包装在简单场景下看起来更“精确”，但在抽象层和组合场景下会迅速退化。

### 1. 规则不稳定

例如：

```java
try {
    handler.apply();
} catch (IOException e) {
    throw new ScxWrappedException(e);
}
```

这表示：

```text
外部逻辑抛 IOException      会被包装
外部逻辑抛 RuntimeException  不会被包装
```

那么调用方看到一个 `RuntimeException` 时，无法知道它来自哪里。

它可能来自：

```text
宿主方法自身
外部回调
回调内部调用的其它回调
更深层的组合逻辑
```

异常来源又变得模糊。

### 2. 类型系统无法表达规则

“只包装会混淆的异常”这种规则无法稳定写进方法签名。

方法签名只能表达：

```java
throws IOException, ScxWrappedException
```

它不能表达：

```text
如果异常类型和宿主方法异常冲突，就包装
否则不包装
```

调用方必须依赖文档或实现细节理解规则。

这种规则不适合长期维护。

### 3. 组合后规则会失效

假设一个方法接收回调。

这个回调内部又调用另一个接收回调的方法。

```java
outer(() -> {
    inner(() -> {
        throw new IOException("error");
    });
});
```

如果每一层都选择性包装，那么异常传播规则会变得非常复杂。

调用方很难判断最终拿到的是：

```text
IOException
ScxWrappedException
多层 ScxWrappedException
RuntimeException
```

而统一包装可以保持规则简单：

```text
只要来自外部逻辑，就一定是 ScxWrappedException
```

### 4. 统一错误处理会变脆弱

很多框架会根据异常类型做统一处理。

例如：

```text
IOException             映射为 500
NoPermissionException   映射为 403
ValidationException     映射为 400
```

如果外部逻辑异常有时被包装、有时直接透传，那么统一错误处理就无法可靠区分：

```text
这个异常是框架自己抛出的？
还是用户回调抛出的？
```

因此，框架级 API 更应该采用统一包装，而不是选择性包装。

## ScxWrappedException

`ScxWrappedException` 是一个 `final` 类。

它继承自 `RuntimeException`。

接口层面可以理解为：

```java
public final class ScxWrappedException extends RuntimeException {

    public ScxWrappedException(String message, Throwable cause) {
        ...
    }

    public ScxWrappedException(Throwable cause) {
        ...
    }

    public Throwable getUnwrappedCause() {
        ...
    }

}
```

它有三个关键点：

```text
1. 它是 RuntimeException
2. 它必须有 cause
3. 它可以递归解包
```

## 构造方法

`ScxWrappedException` 提供两个构造方法。

### ScxWrappedException(Throwable cause)

```java
throw new ScxWrappedException(e);
```

这是最常用的构造方式。

它表示：

```text
把外部逻辑抛出的异常 e 包装起来
```

示例：

```java
try {
    handler.apply();
} catch (Throwable e) {
    throw new ScxWrappedException(e);
}
```

### ScxWrappedException(String message, Throwable cause)

```java
throw new ScxWrappedException("handler failed", e);
```

这种方式可以附加额外说明。

示例：

```java
try {
    handler.apply();
} catch (Throwable e) {
    throw new ScxWrappedException("external handler failed", e);
}
```

## cause 不允许为 null

`ScxWrappedException` 是异常包装器。

包装器必须包装一个真实异常。

因此 `cause` 不允许为 `null`。

下面这种写法是不允许的：

```java
throw new ScxWrappedException(null);
```

如果传入 `null`，会抛出：

```java
NullPointerException
```

错误信息是：

```text
cause must not be null
```

这是为了避免出现没有真实原因的包装异常。

因为如果 `cause` 是 `null`，那么这个异常就失去了“包装器”的意义。

## getUnwrappedCause

`getUnwrappedCause()` 用于获取真实异常。

普通包装：

```java
var e = new ScxWrappedException(new IOException("lambda error"));

Throwable cause = e.getUnwrappedCause();
```

结果是：

```text
IOException("lambda error")
```

如果有多层包装，也会递归解包。

```java
var e = new ScxWrappedException(
    new ScxWrappedException(
        new IOException("lambda error")
    )
);

Throwable cause = e.getUnwrappedCause();
```

结果仍然是：

```text
IOException("lambda error")
```

也就是说，`getUnwrappedCause()` 会不断跳过中间的 `ScxWrappedException`，直到找到第一个不是 `ScxWrappedException` 的真实 cause。

## 单层包装示例

定义一个回调接口：

```java
@FunctionalInterface
public interface Func {

    void apply(byte[] bytes) throws Throwable;

}
```

定义一个宿主方法：

```java
public static void read(Func bytesConsumer, int length)
        throws ScxWrappedException, IOException {
    if (length > 2048) {
        throw new IOException("length too big");
    }

    try {
        bytesConsumer.apply(new byte[]{1, 2, 3});
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

如果宿主方法自身失败：

```java
try {
    read(bytes -> {
        throw new IOException("lambda error");
    }, 4096);
} catch (IOException e) {
    System.out.println(e.getMessage());
}
```

输出：

```text
length too big
```

这里的异常来自 `read(...)` 自身。

如果外部回调失败：

```java
try {
    read(bytes -> {
        throw new IOException("lambda error");
    }, 1024);
} catch (IOException e) {
    throw new AssertionError("不应该进入这里");
} catch (ScxWrappedException e) {
    System.out.println(e.getUnwrappedCause().getMessage());
}
```

输出：

```text
lambda error
```

这里的异常来自 `bytesConsumer` 回调。

## 多层包装示例

假设外层回调内部又调用了 `read(...)`。

```java
try {
    read(bytes -> {
        read(bytes2 -> {
            throw new IOException("lambda error");
        }, 1024);
    }, 1024);
} catch (IOException e) {
    throw new AssertionError("不应该进入这里");
} catch (ScxWrappedException e) {
    System.out.println(e.getUnwrappedCause().getMessage());
}
```

输出：

```text
lambda error
```

这里可能形成多层包装：

```text
ScxWrappedException
    ScxWrappedException
        IOException("lambda error")
```

但是调用：

```java
e.getUnwrappedCause()
```

可以直接拿到：

```text
IOException("lambda error")
```

这就是递归解包的意义。

## RuntimeException 的意义

`ScxWrappedException` 继承自 `RuntimeException`。

这意味着它可以在不声明 `throws` 的场景中继续传播。

例如很多函数式接口、回调接口或框架入口，不方便声明受检异常。

这时可以把受检异常包装起来：

```java
try {
    handler.apply();
} catch (Throwable e) {
    throw new ScxWrappedException(e);
}
```

调用方在边界处再解包：

```java
try {
    run();
} catch (ScxWrappedException e) {
    Throwable realCause = e.getUnwrappedCause();
}
```

需要注意，`RuntimeException` 并不表示这个异常可以忽略。

它只是让异常能够跨过不方便声明受检异常的 API 边界。

## 适用场景

`ScxWrappedException` 适合用于下面这些场景。

### 1. 高阶函数

例如一个方法接收外部函数：

```java
public void each(Handler handler) {
    try {
        handler.apply();
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

这里 `handler` 是外部传入的逻辑。

它抛出的异常应该和 `each(...)` 自己的异常区分开。

### 2. 模板方法

例如：

```java
public void withResource(Handler handler) throws IOException {
    var resource = openResource();

    try {
        handler.apply(resource);
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    } finally {
        resource.close();
    }
}
```

这里 `openResource()` 和 `close()` 可能抛出宿主方法自己的异常。

`handler.apply(...)` 抛出的异常应该被包装。

### 3. 拦截器

例如：

```java
public Object intercept(Invocation invocation) {
    try {
        return invocation.proceed();
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

这里 `invocation.proceed()` 属于外部逻辑。

包装后，上层可以知道异常来自被拦截的业务逻辑，而不是拦截器自身。

### 4. 事件处理器

例如：

```java
public void fire(Event event) {
    for (var listener : listeners) {
        try {
            listener.onEvent(event);
        } catch (Throwable e) {
            throw new ScxWrappedException(e);
        }
    }
}
```

这里 listener 是外部逻辑。

它抛出的异常应该和事件系统自身的异常区分。

### 5. 流式处理

例如：

```java
public void consume(Consumer consumer) {
    try {
        consumer.accept();
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

如果流式处理器本身也会抛异常，就更需要隔离外部逻辑的异常域。

## 不适用场景

`ScxWrappedException` 并不是所有异常都应该使用的包装器。

### 1. 没有外部逻辑

如果一个方法内部没有调用外部传入的逻辑，就通常不需要 `ScxWrappedException`。

```java
public void save() throws IOException {
    writeFile();
}
```

这里 `IOException` 来自方法自身，直接抛出即可。

### 2. 不存在异常域模糊

如果宿主逻辑和外部逻辑不可能抛出相同类型的异常，或者调用方根本不关心来源，也可以不用。

```java
public void run(Runnable runnable) {
    runnable.run();
}
```

如果这个方法本身不抛异常，外部异常来源已经很明确，就不一定需要包装。

### 3. 只是为了隐藏异常

不要为了“少写 throws”而随意包装异常。

`ScxWrappedException` 的目的不是吞掉异常，也不是掩盖异常类型。

它的目的是：

```text
隔离异常来源
```

如果只是想偷懒绕过受检异常，最好重新考虑 API 设计。

### 4. 业务异常体系

如果你的项目已经有明确的业务异常体系，例如：

```text
BizException
ValidationException
NoPermissionException
```

不要把所有业务异常都无脑包装成 `ScxWrappedException`。

只有当业务异常来自外部逻辑，并且需要区分异常域时，才应该包装。

## 与普通 RuntimeException 包装的区别

很多代码会这样写：

```java
try {
    handler.apply();
} catch (Exception e) {
    throw new RuntimeException(e);
}
```

这种写法只是把异常变成运行时异常。

但是它没有表达语义。

调用方无法知道：

```text
这是普通运行时异常？
这是宿主方法自己包装的异常？
这是外部回调抛出的异常？
```

而 `ScxWrappedException` 的语义更明确：

```text
这是被包装的外部逻辑异常
```

因此它更适合框架、工具库和高阶 API。

## 与 rethrow 的区别

有时候我们会直接重新抛出异常：

```java
try {
    handler.apply();
} catch (Throwable e) {
    throw e;
}
```

这不会改变异常类型。

看起来最“保真”。

但是它无法解决异常域模糊。

例如：

```java
public void read(Func handler) throws IOException {
    try {
        handler.apply();
    } catch (IOException e) {
        throw e;
    }
}
```

调用方看到 `IOException` 时仍然无法知道它来自：

```text
read 自己
handler
```

而包装后：

```text
IOException             read 自己
ScxWrappedException     handler
```

来源就清楚了。

## 与 addSuppressed 的区别

`addSuppressed(...)` 用于表达一个异常发生时，另一个异常也同时发生了。

典型场景是：

```java
try {
    doSomething();
} catch (Throwable e) {
    try {
        close();
    } catch (Throwable closeError) {
        e.addSuppressed(closeError);
    }
    throw e;
}
```

它表达的是：

```text
主异常
附带异常
```

而 `ScxWrappedException` 表达的是：

```text
异常来源属于外部逻辑
```

两者解决的问题不同。

## 与 checked exception 的关系

Java 的受检异常可以表达某个方法可能抛出某类异常。

例如：

```java
public void read() throws IOException
```

但是当一个方法既有宿主逻辑，又接收外部逻辑时，受检异常只能表达类型，无法表达来源。

例如：

```java
public void read(Func handler) throws IOException
```

如果 `handler` 也能抛出 `IOException`，那么 `throws IOException` 无法说明异常来自哪一边。

这时就可以使用：

```java
public void read(Func handler) throws IOException, ScxWrappedException
```

这样：

```text
IOException             read 自己
ScxWrappedException     handler
```

## 推荐写法

推荐写法是：

```java
public static void runExternal(ExternalHandler handler) throws ScxWrappedException {
    try {
        handler.apply();
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    }
}
```

如果宿主方法自己也有异常：

```java
public static void withFile(FileHandler handler)
        throws IOException, ScxWrappedException {
    var file = openFile();

    try {
        handler.apply(file);
    } catch (Throwable e) {
        throw new ScxWrappedException(e);
    } finally {
        file.close();
    }
}
```

调用方：

```java
try {
    withFile(file -> {
        throw new IOException("handler error");
    });
} catch (IOException e) {
    // withFile 自己的 IOException
} catch (ScxWrappedException e) {
    // handler 的异常
    Throwable cause = e.getUnwrappedCause();
}
```

## 解包后如何处理

`getUnwrappedCause()` 返回的是 `Throwable`。

调用方可以根据需要继续判断类型。

```java
try {
    runExternal(handler);
} catch (ScxWrappedException e) {
    Throwable cause = e.getUnwrappedCause();

    if (cause instanceof IOException ioException) {
        throw ioException;
    }

    if (cause instanceof RuntimeException runtimeException) {
        throw runtimeException;
    }

    throw e;
}
```

也可以转换为业务异常：

```java
try {
    runExternal(handler);
} catch (ScxWrappedException e) {
    throw new BizException("external handler failed", e.getUnwrappedCause());
}
```

或者只记录日志：

```java
try {
    runExternal(handler);
} catch (ScxWrappedException e) {
    log.error("external handler failed", e.getUnwrappedCause());
}
```

## 多层 API 中的使用方式

如果你正在写一个更高层 API，而它内部又调用了可能抛出 `ScxWrappedException` 的下层 API，推荐继续保持异常域语义。

例如：

```java
public void outer(Handler handler) throws ScxWrappedException {
    try {
        inner(handler);
    } catch (ScxWrappedException e) {
        throw new ScxWrappedException(e);
    }
}
```

这样可能产生多层包装。

```text
ScxWrappedException
    ScxWrappedException
        IOException
```

但是调用方可以通过：

```java
e.getUnwrappedCause()
```

直接拿到最里面的真实异常。

这比在每一层都尝试猜测“要不要解包再重抛”更加稳定。

## 什么时候不应该解包重抛

不要在中间层随意解包再重抛。

例如：

```java
try {
    inner(handler);
} catch (ScxWrappedException e) {
    throw e.getUnwrappedCause();
}
```

这种写法可能重新引入异常域模糊。

因为一旦真实异常被直接抛出，上层又无法判断它来自哪里。

推荐在边界层解包。

例如：

```text
框架边界
请求入口
任务入口
测试断言
最终日志记录位置
```

中间抽象层通常继续保持 `ScxWrappedException`。

## API 设计建议

如果一个方法接收外部逻辑，并且自身也可能抛出异常，推荐把方法签名设计成：

```java
public Result doSomething(Handler handler)
        throws HostException, ScxWrappedException
```

其中：

```text
HostException           宿主方法自己的异常
ScxWrappedException     handler 抛出的异常
```

示例：

```java
public byte[] read(Func bytesConsumer, int length)
        throws IOException, ScxWrappedException
```

这比下面这种签名更清晰：

```java
public byte[] read(Func bytesConsumer, int length)
        throws IOException
```

因为后一种写法中，`IOException` 的来源不清楚。

## 完整示例

下面是一个完整示例。

```java
import dev.scx.exception.ScxWrappedException;

import java.io.IOException;

public class WrappedExceptionDemo {

    public static void main(String[] args) {
        testHostException();

        testExternalException();

        testNestedExternalException();
    }

    public static void read(Func bytesConsumer, int length)
            throws ScxWrappedException, IOException {
        if (length > 2048) {
            throw new IOException("length too big");
        }

        try {
            bytesConsumer.apply(new byte[]{1, 2, 3});
        } catch (Throwable e) {
            throw new ScxWrappedException(e);
        }
    }

    public static void testHostException() {
        try {
            read(bytes -> {
                throw new IOException("lambda error");
            }, 4096);
        } catch (IOException e) {
            System.out.println(e.getMessage());
        } catch (ScxWrappedException e) {
            throw new AssertionError("不应该进入这里", e);
        }
    }

    public static void testExternalException() {
        try {
            read(bytes -> {
                throw new IOException("lambda error");
            }, 1024);
        } catch (IOException e) {
            throw new AssertionError("不应该进入这里", e);
        } catch (ScxWrappedException e) {
            System.out.println(e.getUnwrappedCause().getMessage());
        }
    }

    public static void testNestedExternalException() {
        try {
            read(bytes -> {
                read(bytes2 -> {
                    throw new IOException("lambda error");
                }, 1024);
            }, 1024);
        } catch (IOException e) {
            throw new AssertionError("不应该进入这里", e);
        } catch (ScxWrappedException e) {
            System.out.println(e.getUnwrappedCause().getMessage());
        }
    }

    @FunctionalInterface
    public interface Func {

        void apply(byte[] bytes) throws Throwable;

    }

}
```

输出：

```text
length too big
lambda error
lambda error
```

对应含义是：

```text
第一个 lambda error 没有出现，因为 read 自己先抛出了 length too big
第二个 lambda error 来自外部回调
第三个 lambda error 来自嵌套外部回调，并通过 getUnwrappedCause() 解包得到
```

## 设计说明

### 1. ScxWrappedException 是异常包装器

`ScxWrappedException` 的核心职责是包装另一个异常。

因此它必须有 `cause`。

没有 `cause` 的 `ScxWrappedException` 没有意义。

### 2. 它表达异常来源，而不是异常种类

`ScxWrappedException` 不表示某一种业务错误。

它表示：

```text
这个异常来自外部逻辑
```

真实的异常类型仍然保存在 `cause` 中。

例如：

```text
ScxWrappedException
    IOException
```

真实异常还是 `IOException`。

只是它被放进了“外部逻辑异常域”。

### 3. 它不吞异常

`ScxWrappedException` 不会吞掉原始异常。

原始异常会作为 `cause` 保存。

调用方可以通过：

```java
getCause()
```

或者：

```java
getUnwrappedCause()
```

获取。

### 4. 它支持多层包装

多层抽象中重复包装是可以接受的。

例如：

```text
ScxWrappedException
    ScxWrappedException
        ScxWrappedException
            IOException
```

调用：

```java
getUnwrappedCause()
```

仍然可以得到：

```text
IOException
```

### 5. 它鼓励统一包装，而不是选择性包装

如果某段异常来自外部逻辑，就统一包装。

不要根据异常类型决定是否包装。

统一包装可以形成稳定语义：

```text
ScxWrappedException = 外部逻辑异常
```

选择性包装会导致调用方必须理解实现细节。

### 6. 它适合框架和工具库边界

越是抽象层、框架层、模板层，越容易遇到异常域模糊。

例如：

```text
路由处理器
事件监听器
事务模板
资源模板
序列化回调
异步任务包装
拦截器链
```

这些场景都可能需要明确区分：

```text
框架自己失败
用户代码失败
```

`ScxWrappedException` 适合表达这种边界。

## 常见问题

### SCX Exception 是一个完整异常框架吗？

不是。

它目前主要提供 `ScxWrappedException`，用于异常包装和异常域隔离。

它不是全局异常处理框架，也不是业务异常体系。

### ScxWrappedException 是 checked exception 吗？

不是。

它继承自 `RuntimeException`。

### 为什么不用普通 RuntimeException？

普通 `RuntimeException` 没有明确语义。

`ScxWrappedException` 明确表示：

```text
这是被包装的外部逻辑异常
```

### 为什么 cause 不允许为 null？

因为它是包装异常。

包装异常必须包装一个真实原因。

如果没有 cause，就无法表达“包装了谁”。

### getCause 和 getUnwrappedCause 有什么区别？

`getCause()` 只返回当前这一层的 cause。

`getUnwrappedCause()` 会递归跳过所有 `ScxWrappedException`，返回最里面的真实异常。

例如：

```text
ScxWrappedException
    ScxWrappedException
        IOException
```

`getCause()` 返回的是内层 `ScxWrappedException`。

`getUnwrappedCause()` 返回的是 `IOException`。

### 什么时候应该使用 ScxWrappedException？

当一个方法接收外部逻辑，并且需要区分：

```text
方法自身异常
外部逻辑异常
```

时，应该使用。

### 什么时候不需要使用？

如果不存在异常域模糊，就不需要。

例如方法内部没有调用外部回调，或者调用方完全不关心异常来源。

### 应该包装哪些异常？

推荐包装外部逻辑抛出的所有异常。

不要只包装某几种异常。

### 可以只包装 checked exception 吗？

不推荐。

因为这样 `RuntimeException` 的来源仍然会变得模糊。

外部逻辑抛出的所有异常都应该统一包装。

### 可以包装 Error 吗？

当前推荐写法是捕获 `Throwable` 并包装。

这样可以保证异常域语义统一。

但是否应该在你的业务系统中捕获并处理 `Error`，需要根据具体场景决定。

如果只是为了保持异常域隔离，可以包装。

如果是 JVM 级严重错误，通常仍应在边界处记录并终止当前任务。

### 中间层应该解包吗？

通常不应该。

中间层随意解包再重抛，可能重新引入异常域模糊。

推荐在边界层解包，例如请求入口、任务入口、测试断言或最终日志处理处。

### 多层包装会不会拿不到真实异常？

不会。

可以使用：

```java
getUnwrappedCause()
```

递归获取真实异常。

### ScxWrappedException 会改变原始异常堆栈吗？

原始异常会作为 cause 保留。

查看完整堆栈时仍然可以看到原始异常。

### 可以用它做业务异常吗？

不推荐。

它表达的是异常来源，不是业务错误类型。

业务异常应该有自己的类型，例如：

```text
BizException
ValidationException
NoPermissionException
```

### 可以在 lambda 中使用吗？

可以。

这正是它适合的场景之一。

```java
try {
    handler.apply();
} catch (Throwable e) {
    throw new ScxWrappedException(e);
}
```

### 如果我想重新抛出真实异常怎么办？

可以在边界处解包后处理。

```java
try {
    runExternal(handler);
} catch (ScxWrappedException e) {
    Throwable cause = e.getUnwrappedCause();

    if (cause instanceof IOException ioException) {
        throw ioException;
    }

    throw e;
}
```

### 为什么推荐“统一包装所有外部异常”？

因为统一包装可以形成稳定规则：

```text
ScxWrappedException = 外部逻辑异常
```

选择性包装会让调用方无法可靠判断异常来源。

### 这个库适合放在哪里使用？

适合放在抽象边界处使用。

例如：

```text
调用用户回调前后
执行外部 handler 时
框架调用业务代码时
模板方法调用用户逻辑时
拦截器调用下一个节点时
```

不建议在普通业务代码里到处使用。
