# SCX Timer

SCX Timer 是一个极简延迟任务抽象库。

它提供一套很薄的 Timer 接口，用来把“延迟执行一个任务”这件事从具体调度实现中抽象出来。

SCX Timer 本身不是任务调度平台，也不是 cron 框架。它不负责持久化任务，不负责分布式调度，不负责周期执行，也不负责应用关闭时自动管理线程池生命周期。

它只负责：

```text
提交一个延迟任务
返回一个 TaskHandle
通过 TaskHandle 查询任务状态
通过 TaskHandle 获取任务异常
通过 TaskHandle 尝试取消任务
```

[GitHub](https://github.com/scx-projects/scx-timer)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-timer</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Timer 中最核心的概念包括：

```text
ScxTimer                  Timer 抽象接口
ScheduledExecutorTimer    基于 ScheduledExecutorService 的默认实现
TaskHandle                任务句柄
TaskStatus                任务状态
TaskStateException        任务状态异常
ScxWrappedException       内部用于包装任务执行过程中抛出的异常
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
通过 TaskHandle 查询状态、取消任务、读取异常
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
TaskHandle 负责暴露任务控制、状态查询和异常读取能力
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

查询任务状态：

```java
System.out.println(handle.status());
```

关闭线程池：

```java
executor.shutdown();
```

完整示例：

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerDemo {

    public static void main(String[] args) throws InterruptedException {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);
            var completed = new CountDownLatch(1);

            var handle = timer.runAfter(() -> {
                try {
                    System.out.println("hello timer");
                } finally {
                    completed.countDown();
                }
            }, 1, TimeUnit.SECONDS);

            System.out.println(handle.status());

            completed.await();

            System.out.println(handle.status());
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

    TaskHandle runAfter(Function0Void<?> action, long delay, TimeUnit unit);

}
```

参数含义：

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

## runAfter

`runAfter(...)` 用于提交一个一次性延迟任务。

```java
var handle = timer.runAfter(() -> {
    System.out.println("task executed");
}, 1, TimeUnit.SECONDS);
```

任务类型是 `Function0Void<?>`，因此任务可以直接抛出异常。

```java
var handle = timer.runAfter(() -> {
    throw new IOException("read failed");
}, 500, TimeUnit.MILLISECONDS);
```

提交成功后会立即返回 `TaskHandle`。

```java
TaskHandle handle = timer.runAfter(() -> {
    doSomething();
}, 1, TimeUnit.SECONDS);
```

之后可以通过 `TaskHandle`：

```text
查询状态
尝试取消任务
在任务失败后读取原始异常
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
尝试取消任务
查询任务状态
获取任务异常
```

接口可以理解为：

```java
public interface TaskHandle {

    boolean cancel();

    TaskStatus status();

    Throwable exception() throws TaskStateException;

}
```

需要注意：

```text
TaskHandle 不是 Future 的完整替代品
TaskHandle 是 SCX Timer 自己暴露的最小任务控制接口
```

## cancel

`cancel()` 用于尝试取消任务。

```java
boolean ok = handle.cancel();
```

内部使用的是：

```java
future.cancel(false)
```

也就是说：

```text
mayInterruptIfRunning = false
```

因此它不会中断已经开始执行的任务。

如果任务尚未开始，取消成功后状态会变为：

```text
CANCELLED
```

示例：

```java
var handle = timer.runAfter(() -> {
    System.out.println("可能不会执行");
}, 10, TimeUnit.SECONDS);

boolean cancelled = handle.cancel();

System.out.println(cancelled);
System.out.println(handle.status());
```

如果取消发生在任务开始前，输出大致是：

```text
true
CANCELLED
```

`cancel()` 的返回值表示底层 `ScheduledFuture` 是否接受了本次取消请求。

它不表示 action 一定已经停止。

## cancel 不会中断运行中任务

如果任务已经开始执行，`cancel()` 仍然不会中断它。

```java
var started = new CountDownLatch(1);
var completed = new CountDownLatch(1);

var handle = timer.runAfter(() -> {
    started.countDown();

    try {
        TimeUnit.SECONDS.sleep(1);
    } finally {
        completed.countDown();
    }
}, 0, TimeUnit.SECONDS);

started.await();

boolean cancelled = handle.cancel();

completed.await();

System.out.println(cancelled);
System.out.println(handle.status());
```

对于 JDK 的 `ScheduledFuture` 实现，任务运行期间调用 `cancel(false)` 也可能返回 `true`。

但因为不会中断运行线程，所以 action 仍会继续执行。

当前 `TaskHandle` 的状态由 action 的实际执行过程决定：

```text
运行期间仍可能看到 RUNNING
正常执行结束后为 SUCCESS
抛出异常后为 FAILED
```

因此不要把 `cancel()` 返回 `true` 理解为“运行中的 action 已经停止”。

如果任务需要在运行中响应取消，应在任务内部使用取消标记等协作机制。

```java
var stopped = new AtomicBoolean(false);

var handle = timer.runAfter(() -> {
    while (!stopped.get()) {
        doOneStep();
    }
}, 0, TimeUnit.SECONDS);

stopped.set(true);
```

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
var completed = new CountDownLatch(1);

var handle = timer.runAfter(() -> {
    try {
        System.out.println("task");
    } finally {
        completed.countDown();
    }
}, 1, TimeUnit.SECONDS);

System.out.println(handle.status());

completed.await();

System.out.println(handle.status());
```

输出大致是：

```text
PENDING
SUCCESS
```

`status()` 返回当前瞬间的状态。

在并发环境中，状态可能在方法返回后立即发生变化。

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

## exception

`exception()` 用于获取任务执行失败时的真实异常。

```java
Throwable throwable = handle.exception();
```

它只在任务失败并且底层 `ScheduledFuture` 已经完成后可用。

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

while (handle.status() == TaskStatus.PENDING ||
       handle.status() == TaskStatus.RUNNING) {
    TimeUnit.MILLISECONDS.sleep(10);
}

if (handle.status() == TaskStatus.FAILED) {
    Throwable exception = handle.exception();

    System.out.println(exception.getClass());
    System.out.println(exception.getMessage());
}
```

`exception()` 返回的是任务内部抛出的原始异常。

例如：

```text
IOException("read failed")
```

而不是内部使用的 `ScxWrappedException`。

## TaskStateException

`TaskStateException` 表示任务状态不允许执行当前操作。

当前公开 API 中，它会在任务尚未失败完成时调用 `exception()` 的情况下抛出。

例如：

```text
任务尚未执行
任务正在执行
任务执行成功
任务已取消
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
    handle.exception();
} catch (TaskStateException e) {
    // 当前任务状态还不能获取 exception
}
```

## 任务异常包装

任务执行时，如果 action 抛出异常，`ScheduledExecutorTimer` 会把它包装成 `ScxWrappedException`，交给底层 `ScheduledFuture` 保存。

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
任务内部异常                原始异常
ScheduledFuture 中保存的异常 ScxWrappedException
exception() 返回值           原始异常
```

调用方通常不需要直接处理 `ScxWrappedException`。

任务失败后，通过 `exception()` 即可获取原始异常。

```java
if (handle.status() == TaskStatus.FAILED) {
    Throwable cause = handle.exception();
}
```

## 为什么使用 ScxWrappedException

任务执行异常属于“外部任务逻辑”的异常。

默认实现需要把任意 `Throwable` 交给 `ScheduledFuture` 保存，因此内部统一使用 `ScxWrappedException` 包装。

`TaskHandle#exception()` 会取出它的 cause，并返回任务内部抛出的原始异常。

```text
ScxWrappedException    默认实现内部的异常包装
TaskStateException     TaskHandle 当前状态不允许读取异常
```

这样调用方只需要区分：

```java
if (handle.status() == TaskStatus.FAILED) {
    Throwable taskException = handle.exception();
}
```

## 成功任务示例

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

var executor = new ScheduledThreadPoolExecutor(1);
ScxTimer timer = new ScheduledExecutorTimer(executor);

try {
    var completed = new CountDownLatch(1);

    var handle = timer.runAfter(() -> {
        try {
            System.out.println("done");
        } finally {
            completed.countDown();
        }
    }, 1, TimeUnit.SECONDS);

    System.out.println(handle.status());

    completed.await();

    System.out.println(handle.status());
} finally {
    executor.shutdown();
}
```

输出大致是：

```text
PENDING
done
SUCCESS
```

## 失败任务示例

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.io.IOException;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

var executor = new ScheduledThreadPoolExecutor(1);
ScxTimer timer = new ScheduledExecutorTimer(executor);

try {
    var handle = timer.runAfter(() -> {
        throw new IOException("read failed");
    }, 1, TimeUnit.SECONDS);

    while (handle.status() == TaskStatus.PENDING ||
           handle.status() == TaskStatus.RUNNING) {
        TimeUnit.MILLISECONDS.sleep(10);
    }

    if (handle.status() == TaskStatus.FAILED) {
        Throwable exception = handle.exception();

        System.out.println(exception.getClass());
        System.out.println(exception.getMessage());
    }
} finally {
    executor.shutdown();
}
```

输出大致是：

```text
class java.io.IOException
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
var completed = new CountDownLatch(1);

var handle = timer.runAfter(() -> {
    try {
        TimeUnit.MILLISECONDS.sleep(500);
    } finally {
        completed.countDown();
    }
}, 1, TimeUnit.SECONDS);

System.out.println(handle.status());

completed.await();

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

## 非阻塞读取异常

如果不想阻塞等待，可以先查询状态。

```java
if (handle.status() == TaskStatus.FAILED) {
    Throwable exception = handle.exception();
}
```

不要在任务尚未失败完成时直接调用：

```java
handle.exception();
```

否则会抛出 `TaskStateException`。

需要注意，`status()` 是状态快照。

任务处于 `PENDING` 或 `RUNNING` 时，下一次查询可能已经进入终态。

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
统一任务异常读取方式
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
status()
exception()
```

其中：

```text
cancel()      基于 Future#cancel(false)
status()      结合 Future 状态和内部任务状态判断
exception()   基于 Future#exceptionNow() 读取异常
```

`TaskHandle` 不暴露底层 `ScheduledFuture`，也不提供阻塞等待和任务结果读取能力。

## 与 SCX Function 的关系

`ScxTimer` 接收的是 `Function0Void<?>`。

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

任务执行时，如果抛出任何异常，默认实现都会把它包装成 `ScxWrappedException` 保存。

调用方可以在任务失败后通过 `TaskHandle#exception()` 获取原始异常。

## 与 SCX Exception 的关系

默认实现会在内部把任务异常包装成：

```java
ScxWrappedException
```

`TaskHandle#exception()` 会返回其中保存的原始异常。

示例：

```java
if (handle.status() == TaskStatus.FAILED) {
    Throwable cause = handle.exception();
}
```

这和 `SCX Exception` 的设计保持一致：

```text
Timer 状态异常             TaskStateException
默认实现内部任务异常包装    ScxWrappedException
调用方读取到的任务异常      原始 Throwable
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

## 完整示例：延迟执行并查询状态

```java
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerStatusDemo {

    public static void main(String[] args) throws InterruptedException {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);
            var completed = new CountDownLatch(1);

            var handle = timer.runAfter(() -> {
                try {
                    System.out.println("hello");
                } finally {
                    completed.countDown();
                }
            }, 1, TimeUnit.SECONDS);

            System.out.println(handle.status());

            completed.await();

            if (handle.status() == TaskStatus.SUCCESS) {
                System.out.println("task success");
            }
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
import dev.scx.timer.ScheduledExecutorTimer;
import dev.scx.timer.ScxTimer;
import dev.scx.timer.TaskStatus;

import java.io.IOException;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TimerExceptionDemo {

    public static void main(String[] args) throws InterruptedException {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                throw new IOException("task failed");
            }, 1, TimeUnit.SECONDS);

            while (handle.status() == TaskStatus.PENDING ||
                   handle.status() == TaskStatus.RUNNING) {
                TimeUnit.MILLISECONDS.sleep(10);
            }

            if (handle.status() == TaskStatus.FAILED) {
                Throwable exception = handle.exception();

                System.out.println(exception.getClass().getName());
                System.out.println(exception.getMessage());
            }
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

public class TimerPollingDemo {

    public static void main(String[] args) throws Exception {
        var executor = new ScheduledThreadPoolExecutor(1);

        try {
            ScxTimer timer = new ScheduledExecutorTimer(executor);

            var handle = timer.runAfter(() -> {
                TimeUnit.SECONDS.sleep(2);
            }, 1, TimeUnit.SECONDS);

            while (true) {
                var status = handle.status();

                System.out.println(status);

                if (status == TaskStatus.SUCCESS) {
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
status
exception
```

不会暴露底层完整 `ScheduledFuture`。

### 4. cancel 不中断运行中任务

当前实现使用：

```java
future.cancel(false)
```

它不会中断已经开始执行的任务。

这可以避免 Timer 层强行中断用户任务。

如果用户任务需要响应取消，应由任务自己设计取消机制。

### 5. 任务异常统一包装

任务 action 抛出的异常会在默认实现内部统一包装为 `ScxWrappedException`。

`TaskHandle#exception()` 会返回任务内部抛出的原始异常。

### 6. exception 是非阻塞读取

`exception()` 不等待任务完成。

如果任务状态不满足条件，会抛 `TaskStateException`。

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

因此不会中断已经开始执行的任务。

### 任务已经运行后还能取消吗？

可以调用 `TaskHandle#cancel()`，但它不会中断运行线程。

底层 `ScheduledFuture` 可能接受取消请求并返回 `true`，action 仍会继续执行，最终状态由 action 的执行结果决定。

### exception 会阻塞吗？

不会。

`exception()` 只在任务失败并完成后可用。

任务状态不符合要求时会抛 `TaskStateException`。

### exception 返回什么？

返回任务内部抛出的原始异常。

不是内部使用的 `ScxWrappedException`。

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

默认实现使用 `ScxWrappedException` 把任意任务异常交给 `ScheduledFuture` 保存，再由 `exception()` 返回原始异常。

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
需要在任务失败后读取原始异常的场景
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
