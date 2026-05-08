# SCX Timer

SCX Timer 是一个极简延迟任务抽象库。

它提供一套很薄的 Timer 接口，用来把“延迟执行一个任务”这件事从具体调度实现中抽象出来。

SCX Timer 本身不是任务调度平台，也不是 cron 框架。它不负责持久化任务，不负责分布式调度，不负责周期执行，也不负责应用关闭时自动管理线程池生命周期。

它只负责：

```text
提交一个延迟任务
返回一个 TaskHandle
通过 TaskHandle 查询任务状态
通过 TaskHandle 等待任务完成
通过 TaskHandle 获取任务结果或异常
通过 TaskHandle 取消尚未开始的任务
```

当前版本为 `0.1.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-timer</artifactId>
    <version>0.1.0</version>
</dependency>
```

SCX Timer 依赖：

```text
scx-function
scx-exception
```

其中：

```text
scx-function    用于表示可抛异常的任务函数
scx-exception   用于把任务执行异常包装为 ScxWrappedException
```

## 基本概念

SCX Timer 中最核心的概念包括：

```text
ScxTimer                  Timer 抽象接口
ScheduledExecutorTimer    基于 ScheduledExecutorService 的默认实现
TaskHandle                任务句柄
TaskStatus                任务状态
TaskStateException        任务状态异常
ScxWrappedException       包装任务执行过程中抛出的异常
```

它们之间的关系可以简单理解为：

```text
ScxTimer
    ↓
runAfter(...)
    ↓
提交延迟任务
    ↓
返回 TaskHandle
    ↓
通过 TaskHandle 查询状态、等待结果、取消任务
```

默认实现：

```text
ScheduledExecutorTimer
    ↓
ScheduledExecutorService#schedule(...)
    ↓
ScheduledFuture
    ↓
TaskHandleImpl
```

也就是说：

```text
ScxTimer 负责抽象 API
ScheduledExecutorTimer 负责桥接 ScheduledExecutorService
TaskHandle 负责暴露任务控制和结果读取能力
```

## 快速开始

创建一个基于 `ScheduledThreadPoolExecutor` 的 Timer。

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

var executor = new ScheduledThreadPoolExecutor(1);

ScxTimer timer = new ScheduledExecutorTimer(executor);
```

提交一个延迟任务：

```java
var handle = timer.runAfter(() -> {
    System.out.println("hello timer");
}, 1, TimeUnit.SECONDS);
```

等待任务完成：

```java
handle.await();
```

关闭线程池：

```java
executor.shutdown();
```

完整示例：

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerDemo {

    public static void main(String[] args) {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                System.out.println("hello timer");
            }, 1, TimeUnit.SECONDS);

            handle.await();
        } finally {
            executor.shutdown();
        }
    }

}
```

## ScxTimer

`ScxTimer` 是 Timer 抽象接口。

接口可以理解为：

```java
public interface ScxTimer {

    TaskHandle runAfter(Function0Void action, long delay, TimeUnit unit);

    TaskHandle runAfter(Function0 action, long delay, TimeUnit unit);

}
```

它提供两个重载：

```text
runAfter(Function0Void, delay, unit)    延迟执行无返回值任务
runAfter(Function0, delay, unit)        延迟执行有返回值任务
```

其中：

```text
action    要执行的任务
delay     延迟时间
unit      延迟时间单位
```

`unit` 使用 JDK 的 `TimeUnit`。

例如：

```java
TimeUnit.MILLISECONDS
TimeUnit.SECONDS
TimeUnit.MINUTES
```

## runAfter 无返回值任务

无返回值任务使用 `Function0Void`。

```java
var handle = timer.runAfter(() -> {
    System.out.println("task executed");
}, 1, TimeUnit.SECONDS);
```

等待完成：

```java
handle.await();
```

因为任务没有返回值，所以 `await()` 的返回值通常可以忽略。

```java
timer.runAfter(() -> {
    doSomething();
}, 500, TimeUnit.MILLISECONDS).await();
```

## runAfter 有返回值任务

有返回值任务使用 `Function0`。

```java
var handle = timer.runAfter(() -> {
    return 123;
}, 1, TimeUnit.SECONDS);
```

等待并获取结果：

```java
Integer result = handle.await();
```

也可以在确定任务已经成功后使用：

```java
Integer result = handle.result();
```

区别是：

```text
await()    如果任务未完成，会阻塞等待
result()   不等待，只能在任务成功后调用
```

## ScheduledExecutorTimer

`ScheduledExecutorTimer` 是 `ScxTimer` 的默认实现。

它基于 JDK 的 `ScheduledExecutorService`。

创建方式：

```java
var executor = new ScheduledThreadPoolExecutor(1);

var timer = new ScheduledExecutorTimer(executor);
```

它不会自己创建线程池。

它只接收外部传入的 `ScheduledExecutorService`。

这意味着：

```text
线程池大小由调用方决定
线程命名由调用方决定
线程生命周期由调用方决定
shutdown 由调用方负责
```

例如：

```java
var executor = new ScheduledThreadPoolExecutor(4);

var timer = new ScheduledExecutorTimer(executor);
```

使用完后：

```java
executor.shutdown();
```

## 为什么不自动关闭 executor

`ScheduledExecutorTimer` 不拥有 executor 的生命周期。

调用方传进来的 executor 可能被多个组件共享。

如果 Timer 自动关闭 executor，可能会影响其它使用者。

因此关闭责任属于调用方。

推荐写法：

```java
var executor = new ScheduledThreadPoolExecutor(1);

try {
    var timer = new ScheduledExecutorTimer(executor);

    // 使用 timer
} finally {
    executor.shutdown();
}
```

或者在应用关闭时统一关闭：

```java
Runtime.getRuntime().addShutdownHook(new Thread(executor::shutdown));
```

## TaskHandle

`TaskHandle` 表示一个已经提交的任务句柄。

它用于：

```text
取消任务
等待任务完成
查询任务状态
获取任务结果
获取任务异常
```

接口可以理解为：

```java
public interface TaskHandle {

    boolean cancel();

    <R> R await() throws ScxWrappedException, TaskStateException;

    TaskStatus status();

    <R> R result() throws TaskStateException;

    Throwable exception() throws TaskStateException;

}
```

需要注意：

```text
TaskHandle 不是 Future 的完整替代品
TaskHandle 是 SCX Timer 自己暴露的最小任务控制接口
```

## cancel

`cancel()` 用于取消任务。

```java
boolean ok = handle.cancel();
```

当前语义是：

```text
只取消尚未开始执行的任务
不会中断已经开始执行的任务
```

内部使用的是：

```java
future.cancel(false)
```

也就是说：

```text
mayInterruptIfRunning = false
```

因此：

```text
任务仍处于 PENDING 时，cancel() 可能成功
任务已经 RUNNING 时，cancel() 通常不会成功
任务已经 SUCCESS / FAILED 时，cancel() 不再有意义
```

示例：

```java
var handle = timer.runAfter(() -> {
    System.out.println("不会执行");
}, 10, TimeUnit.SECONDS);

boolean cancelled = handle.cancel();

System.out.println(cancelled);
System.out.println(handle.status());
```

如果取消成功，状态会变为：

```text
CANCELLED
```

## cancel 不会中断运行中任务

如果任务已经开始执行，`cancel()` 不会中断它。

示例：

```java
var handle = timer.runAfter(() -> {
    Thread.sleep(10_000);
    System.out.println("task done");
}, 0, TimeUnit.SECONDS);

boolean cancelled = handle.cancel();
```

因为任务很可能已经开始执行，所以：

```text
cancelled 可能是 false
任务仍会继续运行
```

这是当前实现的明确设计。

如果你需要可中断任务，应在任务内部自己检查中断状态或设计取消标记。

```java
var cancelled = new AtomicBoolean(false);

var handle = timer.runAfter(() -> {
    while (!cancelled.get()) {
        doOneStep();
    }
}, 0, TimeUnit.SECONDS);

cancelled.set(true);
```

## await

`await()` 用于同步等待任务完成。

```java
handle.await();
```

如果任务尚未完成，例如仍然是：

```text
PENDING
RUNNING
```

那么 `await()` 会阻塞，直到任务完成。

如果任务成功，会返回任务结果。

```java
Integer result = handle.await();
```

如果任务没有返回值，可以忽略结果。

```java
handle.await();
```

`await()` 是幂等的。

也就是说，任务完成后可以多次调用：

```java
handle.await();
handle.await();
handle.await();
```

对于成功任务，每次都会得到同一个结果。

## await 失败情况

`await()` 可能抛出两类异常。

### ScxWrappedException

如果任务执行过程中抛出异常，`await()` 会抛出 `ScxWrappedException`。

示例：

```java
var handle = timer.runAfter(() -> {
    throw new IOException("read failed");
}, 1, TimeUnit.SECONDS);

try {
    handle.await();
} catch (ScxWrappedException e) {
    Throwable realCause = e.getUnwrappedCause();
}
```

这里真正的异常是任务内部抛出的：

```text
IOException("read failed")
```

可以通过：

```java
e.getUnwrappedCause()
```

获取。

### TaskStateException

如果任务状态不允许获取结果，会抛出 `TaskStateException`。

例如：

```text
任务已取消
等待线程被中断
```

如果等待过程中当前线程被中断，`await()` 会：

```text
1. 恢复当前线程中断标记
2. 抛出 TaskStateException
```

也就是说，它不会静默吞掉中断。

## status

`status()` 用于获取当前任务状态。

```java
TaskStatus status = handle.status();
```

可能返回：

```text
PENDING
RUNNING
SUCCESS
FAILED
CANCELLED
```

状态含义如下：

```text
PENDING      任务已提交，尚未开始执行
RUNNING      任务正在执行
SUCCESS      任务执行成功
FAILED       任务执行失败
CANCELLED    任务在开始前被取消
```

示例：

```java
var handle = timer.runAfter(() -> {
    System.out.println("task");
}, 1, TimeUnit.SECONDS);

System.out.println(handle.status());

handle.await();

System.out.println(handle.status());
```

输出大致是：

```text
PENDING
SUCCESS
```

## TaskStatus

`TaskStatus` 是任务状态枚举。

```java
public enum TaskStatus {

    PENDING,

    RUNNING,

    SUCCESS,

    FAILED,

    CANCELLED

}
```

状态流转可以理解为：

```text
PENDING -> RUNNING -> SUCCESS
PENDING -> RUNNING -> FAILED
PENDING -> CANCELLED
```

当前实现中，`cancel()` 不会中断运行中任务，所以通常不会出现：

```text
RUNNING -> CANCELLED
```

## result

`result()` 用于获取任务结果。

```java
Integer value = handle.result();
```

它只在任务执行成功后可用。

如果任务尚未成功，例如：

```text
PENDING
RUNNING
FAILED
CANCELLED
```

调用 `result()` 会抛出 `TaskStateException`。

`result()` 不会阻塞等待任务完成。

如果你需要等待，应使用：

```java
handle.await()
```

示例：

```java
var handle = timer.runAfter(() -> {
    return 123;
}, 1, TimeUnit.SECONDS);

handle.await();

Integer value = handle.result();
```

## exception

`exception()` 用于获取任务执行失败时的真实异常。

```java
Throwable throwable = handle.exception();
```

它只在任务失败后可用。

如果任务不是失败状态，例如：

```text
PENDING
RUNNING
SUCCESS
CANCELLED
```

调用 `exception()` 会抛出 `TaskStateException`。

示例：

```java
var handle = timer.runAfter(() -> {
    throw new IOException("read failed");
}, 1, TimeUnit.SECONDS);

try {
    handle.await();
} catch (ScxWrappedException e) {
    Throwable realCause = handle.exception();
}
```

`exception()` 返回的是任务内部抛出的原始异常。

例如：

```text
IOException("read failed")
```

而不是外层的 `ScxWrappedException`。

## TaskStateException

`TaskStateException` 表示任务状态不允许执行当前操作。

例如：

```text
任务还没完成时调用 result()
任务没有失败时调用 exception()
任务已取消时 await()
等待过程中线程被中断
```

它继承自 `RuntimeException`。

接口非常简单：

```java
public final class TaskStateException extends RuntimeException {

    public TaskStateException(String message) {
        super(message);
    }

}
```

示例：

```java
try {
    handle.result();
} catch (TaskStateException e) {
    // 当前任务状态还不能获取 result
}
```

## 任务异常包装

任务执行时，如果 action 抛出异常，`ScheduledExecutorTimer` 会把它包装成 `ScxWrappedException`。

内部语义可以理解为：

```java
try {
    action.apply();
    status.set(SUCCESS);
} catch (Throwable e) {
    status.set(FAILED);
    throw new ScxWrappedException(e);
}
```

因此：

```text
任务内部异常            原始异常
ScheduledFuture 中异常   ScxWrappedException
await() 抛出的异常       ScxWrappedException
exception() 返回值       原始异常
```

示例：

```java
var handle = timer.runAfter(() -> {
    throw new IOException("read failed");
}, 1, TimeUnit.SECONDS);

try {
    handle.await();
} catch (ScxWrappedException e) {
    System.out.println(e.getUnwrappedCause());
}
```

## 为什么使用 ScxWrappedException

任务执行异常属于“外部任务逻辑”的异常。

Timer 自身也可能有状态异常。

如果任务内部异常直接混在 Timer 状态异常里，就会造成语义不清。

因此 SCX Timer 使用：

```text
ScxWrappedException    表示任务 action 自己抛出的异常
TaskStateException     表示 TaskHandle 当前状态不允许执行操作
```

这样调用方可以区分：

```java
try {
    handle.await();
} catch (ScxWrappedException e) {
    // 任务自身执行失败
} catch (TaskStateException e) {
    // 任务状态不允许获取结果
}
```

## 成功任务示例

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

var executor = new ScheduledThreadPoolExecutor(1);
ScxTimer timer = new ScheduledExecutorTimer(executor);

try {
    var handle = timer.runAfter(() -> {
        return "done";
    }, 1, TimeUnit.SECONDS);

    System.out.println(handle.status());

    String result = handle.await();

    System.out.println(result);
    System.out.println(handle.status());
    System.out.println(handle.result());
} finally {
    executor.shutdown();
}
```

输出大致是：

```text
PENDING
done
SUCCESS
done
```

## 失败任务示例

```java
import dev.scx.exception.ScxWrappedException;
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.io.IOException;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

var executor = new ScheduledThreadPoolExecutor(1);
ScxTimer timer = new ScheduledExecutorTimer(executor);

try {
    var handle = timer.runAfter(() -> {
        throw new IOException("read failed");
    }, 1, TimeUnit.SECONDS);

    try {
        handle.await();
    } catch (ScxWrappedException e) {
        Throwable realCause = e.getUnwrappedCause();

        System.out.println(realCause.getClass());
        System.out.println(realCause.getMessage());
    }

    Throwable exception = handle.exception();

    System.out.println(exception.getMessage());
} finally {
    executor.shutdown();
}
```

输出大致是：

```text
class java.io.IOException
read failed
read failed
```

## 取消任务示例

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

var executor = new ScheduledThreadPoolExecutor(1);
ScxTimer timer = new ScheduledExecutorTimer(executor);

try {
    var result = new AtomicReference<>("Not Executed");

    var handle = timer.runAfter(() -> {
        result.set("Task Executed");
    }, 5, TimeUnit.SECONDS);

    boolean cancelled = handle.cancel();

    System.out.println(cancelled);
    System.out.println(result.get());
    System.out.println(handle.status());
} finally {
    executor.shutdown();
}
```

如果任务还未开始，输出大致是：

```text
true
Not Executed
CANCELLED
```

## 查询状态示例

```java
var handle = timer.runAfter(() -> {
    TimeUnit.MILLISECONDS.sleep(500);
}, 1, TimeUnit.SECONDS);

System.out.println(handle.status());

handle.await();

System.out.println(handle.status());
```

常见状态变化是：

```text
PENDING
SUCCESS
```

如果任务执行期间查询，可能看到：

```text
RUNNING
```

例如：

```java
var handle = timer.runAfter(() -> {
    TimeUnit.SECONDS.sleep(3);
}, 0, TimeUnit.SECONDS);

while (handle.status() == TaskStatus.PENDING) {
    Thread.onSpinWait();
}

System.out.println(handle.status());
```

可能输出：

```text
RUNNING
```

## 非阻塞读取结果

如果你不想阻塞等待，可以先看状态。

```java
if (handle.status() == TaskStatus.SUCCESS) {
    String result = handle.result();
}
```

失败时：

```java
if (handle.status() == TaskStatus.FAILED) {
    Throwable e = handle.exception();
}
```

不要在任务未完成时直接调用：

```java
handle.result();
```

否则会抛出 `TaskStateException`。

## 有返回值和无返回值的区别

SCX Timer 提供两种任务。

### 无返回值任务

```java
var handle = timer.runAfter(() -> {
    doSomething();
}, 1, TimeUnit.SECONDS);
```

适合：

```text
延迟清理
延迟通知
延迟打印日志
延迟关闭资源
延迟执行副作用动作
```

### 有返回值任务

```java
var handle = timer.runAfter(() -> {
    return loadValue();
}, 1, TimeUnit.SECONDS);
```

适合：

```text
延迟计算
延迟加载
需要 await() 获取结果的任务
```

需要注意：

```text
Timer 不是异步任务框架
Timer 只是延迟提交一次任务
```

如果你需要大量复杂异步组合，应考虑使用更高层的并发抽象。

## 线程池大小的影响

`ScheduledExecutorTimer` 使用调用方传入的 `ScheduledExecutorService`。

因此任务是否能准时执行，和线程池配置有关。

例如单线程：

```java
var executor = new ScheduledThreadPoolExecutor(1);
```

如果前一个任务执行很久，后面的任务即使到了时间，也可能需要等待线程空闲。

多线程：

```java
var executor = new ScheduledThreadPoolExecutor(4);
```

可以让多个到期任务并发执行。

SCX Timer 不修改这个行为。

它完全遵循底层 `ScheduledExecutorService` 的调度语义。

## 延迟时间不是实时保证

`runAfter(...)` 表示：

```text
至少延迟指定时间后，交给 executor 调度执行
```

它不表示严格实时执行。

实际执行时间受很多因素影响：

```text
executor 线程池大小
前面任务执行时间
系统负载
JVM 调度
操作系统线程调度
GC 暂停
```

因此 SCX Timer 不适合用作硬实时计时器。

## TimeUnit

`runAfter(...)` 使用 `TimeUnit` 表示延迟单位。

示例：

```java
timer.runAfter(action, 500, TimeUnit.MILLISECONDS);

timer.runAfter(action, 1, TimeUnit.SECONDS);

timer.runAfter(action, 5, TimeUnit.MINUTES);
```

推荐写法是明确单位，不要用魔法数字表达时间。

```java
timer.runAfter(action, 30, TimeUnit.SECONDS);
```

比下面更清楚：

```java
timer.runAfter(action, 30000, TimeUnit.MILLISECONDS);
```

## 与 ScheduledExecutorService 的关系

SCX Timer 不是要替代 `ScheduledExecutorService`。

它是对常用延迟任务场景的轻量抽象。

底层仍然是：

```java
ScheduledExecutorService
```

默认实现调用的是：

```java
executor.schedule(...)
```

它额外提供的是：

```text
统一 ScxTimer 接口
统一 TaskHandle 句柄
统一 TaskStatus 状态
统一 ScxWrappedException 异常包装
```

如果你需要完整的 JDK 调度能力，例如：

```text
scheduleAtFixedRate
scheduleWithFixedDelay
invokeAll
submit
shutdown
awaitTermination
```

应该直接使用 `ScheduledExecutorService`。

## 与 Future 的关系

`TaskHandle` 可以看作是对 `ScheduledFuture` 的简化包装。

它没有暴露完整 `Future` API。

它只暴露当前库需要的能力：

```text
cancel()
await()
status()
result()
exception()
```

其中：

```text
await()      类似 Future#get()
result()     类似 Future#resultNow()
exception()  类似 Future#exceptionNow()
```

但异常语义被整理成了：

```text
ScxWrappedException
TaskStateException
```

## 与 SCX Function 的关系

`ScxTimer` 接收的是 `Function0Void` 和 `Function0`。

这让任务可以直接抛出异常。

```java
timer.runAfter(() -> {
    throw new IOException("read failed");
}, 1, TimeUnit.SECONDS);
```

不需要写成：

```java
throw new RuntimeException(e);
```

任务执行时，如果抛出任何异常，都会被包装成 `ScxWrappedException`。

## 与 SCX Exception 的关系

任务内部异常会被包装成：

```java
ScxWrappedException
```

调用方可以通过：

```java
e.getUnwrappedCause()
```

获取原始异常。

示例：

```java
try {
    handle.await();
} catch (ScxWrappedException e) {
    Throwable cause = e.getUnwrappedCause();
}
```

这和 `SCX Exception` 的设计保持一致：

```text
Timer 自身状态异常        TaskStateException
外部任务逻辑异常          ScxWrappedException
```

## 不提供周期任务

SCX Timer 当前只提供：

```text
runAfter(...)
```

也就是一次性延迟任务。

它不提供：

```text
runEvery(...)
scheduleAtFixedRate(...)
scheduleWithFixedDelay(...)
cron(...)
```

如果需要周期任务，可以直接使用底层 executor。

```java
executor.scheduleAtFixedRate(
    () -> {
        System.out.println("tick");
    },
    0,
    1,
    TimeUnit.SECONDS
);
```

或者在未来扩展 `ScxTimer` 接口。

## 不提供任务持久化

SCX Timer 不会把任务保存到数据库、文件或消息队列。

一旦 JVM 退出，尚未执行的任务就会丢失。

如果需要可靠任务调度，应使用：

```text
数据库任务表
消息队列
分布式调度系统
Quartz
操作系统 cron
云厂商任务调度服务
```

SCX Timer 的定位是进程内轻量延迟任务。

## 不提供分布式调度

SCX Timer 只在当前 JVM 内工作。

它不处理：

```text
多实例抢占
任务去重
leader election
分布式锁
失败重试
任务迁移
节点宕机恢复
```

这些都属于更高层的任务调度系统。

## 不提供重试

如果任务失败，状态变为：

```text
FAILED
```

SCX Timer 不会自动重试。

如果需要重试，可以在任务内部自己实现。

```java
var handle = timer.runAfter(() -> {
    int maxRetries = 3;

    for (int i = 0; i < maxRetries; i = i + 1) {
        try {
            doSomething();
            return;
        } catch (Exception e) {
            if (i == maxRetries - 1) {
                throw e;
            }
        }
    }
}, 1, TimeUnit.SECONDS);
```

也可以在更高层封装一个 retry timer。

## 不提供任务 ID

`TaskHandle` 本身就是任务句柄。

SCX Timer 不提供全局任务 ID。

如果业务需要任务 ID，可以自己包装。

```java
record NamedTask(String id, TaskHandle handle) {

}
```

示例：

```java
var task = new NamedTask(
    UUID.randomUUID().toString(),
    timer.runAfter(() -> {
        doSomething();
    }, 1, TimeUnit.SECONDS)
);
```

## 完整示例：延迟执行并获取结果

```java
import dev.scx.exception.ScxWrappedException;
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStateException;
import dev.scx.timer.TaskStatus;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerResultDemo {

    public static void main(String[] args) {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                return "hello";
            }, 1, TimeUnit.SECONDS);

            System.out.println(handle.status());

            String result = handle.await();

            System.out.println(result);
            System.out.println(handle.status());

            if (handle.status() == TaskStatus.SUCCESS) {
                System.out.println(handle.result());
            }
        } catch (ScxWrappedException e) {
            e.getUnwrappedCause().printStackTrace();
        } catch (TaskStateException e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
    }

}
```

## 完整示例：取消尚未执行的任务

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class TimerCancelDemo {

    public static void main(String[] args) {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var result = new AtomicReference<>("Not Executed");

            var handle = timer.runAfter(() -> {
                result.set("Task Executed");
            }, 10, TimeUnit.SECONDS);

            boolean cancelled = handle.cancel();

            System.out.println(cancelled);
            System.out.println(result.get());
            System.out.println(handle.status());
        } finally {
            executor.shutdown();
        }
    }

}
```

## 完整示例：处理任务异常

```java
import dev.scx.exception.ScxWrappedException;
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.io.IOException;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerExceptionDemo {

    public static void main(String[] args) {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                throw new IOException("task failed");
            }, 1, TimeUnit.SECONDS);

            try {
                handle.await();
            } catch (ScxWrappedException e) {
                Throwable cause = e.getUnwrappedCause();

                System.out.println(cause.getClass().getName());
                System.out.println(cause.getMessage());
            }

            Throwable exception = handle.exception();

            System.out.println(exception.getMessage());
        } finally {
            executor.shutdown();
        }
    }

}
```

## 完整示例：非阻塞轮询状态

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerStatusDemo {

    public static void main(String[] args) throws Exception {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                TimeUnit.SECONDS.sleep(2);
                return 123;
            }, 1, TimeUnit.SECONDS);

            while (true) {
                var status = handle.status();

                System.out.println(status);

                if (status == TaskStatus.SUCCESS) {
                    System.out.println(handle.result());
                    break;
                }

                if (status == TaskStatus.FAILED) {
                    handle.exception().printStackTrace();
                    break;
                }

                if (status == TaskStatus.CANCELLED) {
                    break;
                }

                TimeUnit.MILLISECONDS.sleep(200);
            }
        } finally {
            executor.shutdown();
        }
    }

}
```

## 完整示例：应用中的延迟清理

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class SessionStore {

    private final Map<String, Session> sessions = new ConcurrentHashMap<>();

    private final ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(1);

    private final ScxTimer timer = new ScheduledExecutorTimer(executor);

    public void put(String sessionId, Session session) {
        sessions.put(sessionId, session);

        timer.runAfter(() -> {
            sessions.remove(sessionId);
        }, 30, TimeUnit.MINUTES);
    }

    public Session get(String sessionId) {
        return sessions.get(sessionId);
    }

    public void close() {
        executor.shutdown();
    }

    public record Session(String userId) {

    }

}
```

这个例子适合简单进程内过期清理。

如果需要可靠会话过期、跨进程同步或服务重启恢复，不应只依赖 SCX Timer。

## 设计说明

### 1. SCX Timer 是进程内轻量延迟任务抽象

SCX Timer 的定位是：

```text
在当前 JVM 内，延迟执行一次任务
```

它不是完整任务调度系统。

### 2. 调度实现由外部 executor 提供

`ScheduledExecutorTimer` 不创建 executor。

它只使用传入的 `ScheduledExecutorService`。

这样调用方可以自己控制：

```text
线程数
线程工厂
异常策略
关闭时机
监控方式
```

### 3. TaskHandle 是任务控制边界

提交任务后，调用方只拿到 `TaskHandle`。

它提供最小控制能力：

```text
cancel
await
status
result
exception
```

不会暴露底层完整 `ScheduledFuture`。

### 4. cancel 不中断运行中任务

当前实现使用：

```java
future.cancel(false)
```

因此取消只对尚未执行的任务有意义。

这可以避免 Timer 层强行中断用户任务。

如果用户任务需要响应取消，应由任务自己设计取消机制。

### 5. 任务异常统一包装

任务 action 抛出的异常统一包装为 `ScxWrappedException`。

这可以区分：

```text
任务内部异常
TaskHandle 状态异常
```

### 6. result 和 exception 都是非阻塞读取

`result()` 和 `exception()` 不等待任务完成。

如果任务状态不满足条件，会抛 `TaskStateException`。

等待任务请使用：

```java
await()
```

### 7. status 是状态快照

`status()` 返回的是当前瞬间的状态。

在并发环境下，状态可能马上发生变化。

因此不要把一次 `status()` 结果当作永久事实。

例如：

```java
if (handle.status() == TaskStatus.PENDING) {
    // 下一行执行时任务可能已经 RUNNING
}
```

这是并发状态查询的正常语义。

### 8. 只提供 runAfter

当前接口只提供一次性延迟执行。

这让 API 保持很小。

周期任务、cron、重试、持久化、分布式调度都不属于当前模块职责。

## 常见问题

### SCX Timer 是定时任务框架吗？

不是完整定时任务框架。

它只是一个轻量延迟任务抽象。

### 支持周期任务吗？

不支持。

当前只支持：

```java
runAfter(...)
```

也就是一次性延迟任务。

### 支持 cron 表达式吗？

不支持。

如果需要 cron，应使用专门的调度框架。

### 支持任务持久化吗？

不支持。

任务只存在于当前 JVM 进程中。

### 支持分布式调度吗？

不支持。

它不处理多节点、抢占、去重、恢复等问题。

### runAfter 的 delay 是精确执行时间吗？

不是。

它表示延迟指定时间后交给 executor 调度。

实际执行时间取决于线程池和系统调度。

### ScheduledExecutorTimer 会自己创建线程池吗？

不会。

线程池由调用方传入。

### ScheduledExecutorTimer 会自动 shutdown executor 吗？

不会。

executor 生命周期由调用方管理。

### cancel 会中断任务吗？

不会。

当前实现使用：

```java
future.cancel(false)
```

因此只会取消尚未开始的任务。

### 任务已经运行后还能取消吗？

通常不能。

如果任务已经进入 `RUNNING`，`cancel()` 通常返回 `false`，任务继续执行。

### await 会阻塞吗？

会。

如果任务尚未完成，`await()` 会阻塞直到任务完成。

### await 可以调用多次吗？

可以。

`await()` 是幂等的。

### result 会阻塞吗？

不会。

`result()` 只在任务成功后可用。

任务未成功时会抛 `TaskStateException`。

### exception 会阻塞吗？

不会。

`exception()` 只在任务失败后可用。

任务未失败时会抛 `TaskStateException`。

### 任务抛异常后 await 抛什么？

抛 `ScxWrappedException`。

原始异常可以通过：

```java
e.getUnwrappedCause()
```

获取。

### exception 返回什么？

返回任务内部抛出的原始异常。

不是外层的 `ScxWrappedException`。

### 任务取消后 await 会怎样？

会抛 `TaskStateException`。

### 等待线程被中断后 await 会怎样？

会恢复当前线程中断标记，并抛 `TaskStateException`。

### status 有哪些状态？

```text
PENDING
RUNNING
SUCCESS
FAILED
CANCELLED
```

### 初始状态是什么？

任务提交后，执行前是：

```text
PENDING
```

### 成功完成后是什么状态？

```text
SUCCESS
```

### 执行异常后是什么状态？

```text
FAILED
```

### 取消成功后是什么状态？

```text
CANCELLED
```

### 为什么任务异常要包装成 ScxWrappedException？

因为任务异常来自外部 action。

`TaskStateException` 表示 Timer/TaskHandle 的状态异常。

两者语义不同。

### 可以直接使用 ScheduledExecutorService 吗？

可以。

如果你需要完整 JDK 调度能力，应该直接使用 `ScheduledExecutorService`。

SCX Timer 只是提供一个更小、更统一的抽象。

### 什么时候适合使用 SCX Timer？

适合：

```text
简单延迟任务
延迟清理
延迟通知
延迟执行一次动作
需要拿到 TaskHandle 查询状态的任务
希望任务异常统一包装的场景
```

不适合：

```text
复杂任务调度
周期任务
cron
分布式任务
任务持久化
可靠任务队列
严格实时计时
```