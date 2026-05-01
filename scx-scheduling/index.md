# SCX Scheduling

SCX Scheduling 是一个轻量的 Java 调度任务库。

它提供单次任务、固定频率任务、固定延迟任务和 Cron 任务四种调度方式，并通过统一的 `ScheduleHandle` 查看任务状态、运行次数、下一次运行时间或取消调度。SCX Scheduling 底层基于 `scx-timer`，Cron 表达式解析使用 `cron-utils`，当前版本为 `0.2.0`。([GitHub][1])

## 安装

### Maven

```xml id="9lo8qe"
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-scheduling</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX Scheduling 中最常用的几个概念是：

```text id="ohrtv5"
ScxScheduling      调度任务入口
ScheduleTask       调度任务构建接口
ScheduleHandle     调度句柄，用于查看状态、取消任务、查看运行次数
TaskContext        任务运行上下文
ExpirationPolicy   过期策略
ScheduleStatus     调度状态
```

`ScxScheduling` 是最主要的入口，它可以创建：

```java id="2cz8dd"
ScxScheduling.oneTime();     // 单次任务
ScxScheduling.fixedRate();   // 固定频率任务
ScxScheduling.fixedDelay();  // 固定延迟任务
ScxScheduling.cron();        // Cron 任务
```

这些方法默认使用全局默认 timer；也可以传入自定义 `ScxTimer` 创建任务。([GitHub][2])

## 快速开始

### 延迟执行一次

```java id="3wr6ok"
import dev.scx.scheduling.ScxScheduling;

import java.time.Duration;

public class Main {

    public static void main(String[] args) {
        ScxScheduling
            .oneTime()
            .startDelay(Duration.ofSeconds(3))
            .start(context -> {
                System.out.println("3 秒后执行一次");
            });
    }

}
```

### 每秒执行一次

```java id="t2u3ke"
import dev.scx.scheduling.ScxScheduling;

import java.time.Duration;

public class Main {

    public static void main(String[] args) {
        ScxScheduling
            .fixedRate()
            .interval(Duration.ofSeconds(1))
            .start(context -> {
                System.out.println("第 " + context.currentRunCount() + " 次执行");
            });
    }

}
```

### 执行 10 次后停止

```java id="w5mdy4"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .maxRunCount(10)
    .start(context -> {
        System.out.println("第 " + context.currentRunCount() + " 次执行");
    });
```

测试代码中也展示了 `oneTime()`、`fixedRate()`、`cron()`、`maxRunCount()` 和 `shutdownDefaultTimer()` 的用法。([GitHub][3])

## 单次任务

单次任务通过 `ScxScheduling.oneTime()` 创建。

```java id="hlijg2"
import dev.scx.scheduling.ScxScheduling;

import java.time.Instant;

ScxScheduling
    .oneTime()
    .startTime(Instant.now().plusSeconds(5))
    .start(context -> {
        System.out.println("5 秒后执行");
    });
```

也可以使用 `startDelay`：

```java id="7zflie"
ScxScheduling
    .oneTime()
    .startDelay(Duration.ofSeconds(5))
    .start(context -> {
        System.out.println("5 秒后执行");
    });
```

`OneTimeScheduleTask` 支持设置开始时间、延迟时间和过期策略。`startTime(Instant)` 和 `startDelay(Duration)` 是便捷方法，底层都会转换为开始时间供应器。([GitHub][4])

### 立即执行一次

如果没有设置 `startTime`，单次任务会以当前时间作为开始时间：

```java id="31p6pj"
ScxScheduling
    .oneTime()
    .start(context -> {
        System.out.println("立即执行一次");
    });
```

`OneTimeScheduleTaskImpl` 在 `startTimeSupplier` 为空时会使用当前时间作为开始时间。([GitHub][5])

## 固定频率任务

固定频率任务通过 `ScxScheduling.fixedRate()` 创建。

```java id="ou751d"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("每秒执行一次");
    });
```

固定频率任务按照计划时间点推进。也就是说，它更关注“从开始时间起，每隔一个 interval 到达一个计划执行点”。源码中 `FixedRatePeriodicScheduleTaskImpl` 通过 `startTime + interval * count` 计算计划执行时间。([GitHub][6])

### 指定开始时间

```java id="kk16n2"
ScxScheduling
    .fixedRate()
    .startTime(Instant.now().plusSeconds(10))
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("10 秒后开始，然后每秒执行一次");
    });
```

### 指定最大运行次数

```java id="01n85q"
ScheduleHandle handle = ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .maxRunCount(5)
    .start(context -> {
        System.out.println("第 " + context.currentRunCount() + " 次执行");
    });
```

`PeriodicScheduleTask` 支持 `startTime`、`startDelay`、`interval`、`maxRunCount` 和 `expirationPolicy`。([GitHub][7])

## 固定延迟任务

固定延迟任务通过 `ScxScheduling.fixedDelay()` 创建。

```java id="r388xc"
ScxScheduling
    .fixedDelay()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("上一次执行结束 1 秒后，再执行下一次");
    });
```

固定延迟任务会在一次任务执行结束后，再等待 `interval` 时间，然后执行下一次。源码中 `FixedDelayPeriodicScheduleTaskImpl` 会记录 `lastExecutionEndTime`，下一次运行时间基于 `lastExecutionEndTime + interval` 计算。([GitHub][8])

### fixedRate 和 fixedDelay 的区别

```text id="p2kkxp"
fixedRate   更关注固定计划时间点
fixedDelay  更关注两次任务执行之间的间隔
```

例如任务本身执行需要 800ms，间隔设置为 1s：

```text id="mtq9y6"
fixedRate   尽量按照 0s、1s、2s、3s 这样的计划点运行
fixedDelay  第一次结束后再等 1s，然后运行第二次
```

## Cron 任务

Cron 任务通过 `ScxScheduling.cron()` 创建。

```java id="bm5lcm"
ScxScheduling
    .cron()
    .cronExpression("*/5 * * * * ?")
    .start(context -> {
        System.out.println("每 5 秒执行一次");
    });
```

限制运行次数：

```java id="1mzx8k"
ScxScheduling
    .cron()
    .cronExpression("*/1 * * * * ?")
    .maxRunCount(3)
    .start(context -> {
        System.out.println("第 " + context.currentRunCount() + " 次执行");
    });
```

Cron 任务使用 `cron-utils` 解析表达式，源码中默认使用 `QUARTZ` 格式。因此示例中的表达式包含秒字段，例如 `*/5 * * * * ?`。([GitHub][9])

`CronScheduleTask` 支持：

```java id="q4of5o"
cronExpression(String cronExpression)
maxRunCount(long maxRunCount)
task(...)
onError(...)
start()
```

如果没有设置任务或没有设置 Cron 表达式，启动时会抛出 `IllegalStateException`。([GitHub][9])

## setTimeout 和 setInterval

SCX Scheduling 也提供了两个类似 JavaScript 的便捷方法。

### setTimeout

```java id="e5l91f"
ScheduleHandle handle = ScxScheduling.setTimeout(() -> {
    System.out.println("1 秒后执行一次");
}, 1000);
```

### setInterval

```java id="nr5xfz"
ScheduleHandle handle = ScxScheduling.setInterval(() -> {
    System.out.println("每 1 秒执行一次");
}, 1000);
```

`setTimeout` 内部使用 `oneTime().startDelay(Duration.ofMillis(delay))`，`setInterval` 内部使用 `fixedRate().interval(Duration.ofMillis(delay))`。([GitHub][2])

## ScheduleHandle

启动任务后会返回 `ScheduleHandle`。

```java id="wn7f05"
ScheduleHandle handle = ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("running");
    });
```

可以通过它查看调度状态、运行次数、下一次运行时间，或取消调度：

```java id="8efn9u"
System.out.println(handle.status());
System.out.println(handle.runCount());
System.out.println(handle.nextRunTime());

handle.cancel();
```

`ScheduleHandle` 提供的方法包括：

```java id="xrdut2"
void cancel();

ScheduleStatus status();

long runCount();

Instant nextRunTime();

Instant nextRunTime(int count);
```

`cancel()` 用于取消调度，但不包含已经开始执行的子任务；`runCount()` 返回子任务已经运行的次数；`nextRunTime()` 返回预计下一次运行时间，如果不会再运行则返回 `null`。([GitHub][10])

## ScheduleStatus

调度状态包括：

```java id="4aryxc"
RUNNING
DONE
CANCELLED
```

`ScheduleStatus` 表示宏观调度状态，而不是某一次子任务的运行状态。例如周期任务在两次执行之间仍然是 `RUNNING`。([GitHub][11])

示例：

```java id="j6hih4"
if (handle.status() == ScheduleStatus.RUNNING) {
    System.out.println("任务仍在调度中");
}
```

## TaskContext

任务执行时会收到一个 `TaskContext`。

```java id="s5muwk"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("当前是第 " + context.currentRunCount() + " 次运行");
    });
```

`TaskContext` 提供：

```java id="skpi4g"
long currentRunCount();

ScheduleHandle scheduleHandle();

void cancelSchedule();
```

`currentRunCount()` 是当前这次运行的次数快照；`scheduleHandle()` 可以拿到当前调度的句柄；`cancelSchedule()` 是 `scheduleHandle().cancel()` 的便捷方法。([GitHub][12])

### 在任务内部取消调度

```java id="3xu7ax"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("第 " + context.currentRunCount() + " 次执行");

        if (context.currentRunCount() >= 5) {
            context.cancelSchedule();
        }
    });
```

## 错误处理

可以通过 `onError(...)` 设置任务异常处理器。

```java id="haylt7"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .onError(error -> {
        System.err.println("任务执行失败");
        error.printStackTrace();
    })
    .start(context -> {
        throw new RuntimeException("boom");
    });
```

任务执行时抛出的异常会被捕获。如果设置了 `errorHandler`，会交给 `errorHandler` 处理；如果没有设置，则会写入日志。如果 `errorHandler` 自己也抛出异常，原异常会添加 suppressed 异常并记录日志。([GitHub][5])

## 过期策略

当任务的开始时间已经早于当前时间时，就会触发过期策略。

可用策略包括：

```java id="qgkvwz"
IMMEDIATE_IGNORE
BACKTRACKING_IGNORE
IMMEDIATE_COMPENSATION
BACKTRACKING_COMPENSATION
```

含义如下：

```text id="jmfdrb"
IMMEDIATE_IGNORE
立即忽略。单次任务不会执行；周期任务会跳过已经错过的时间点，runCount 不会补增。

BACKTRACKING_IGNORE
回溯忽略。单次任务不会执行，但会补一次 runCount；周期任务会跳过已经错过的时间点，同时补增 runCount。

IMMEDIATE_COMPENSATION
立即补偿。单次任务立即执行；周期任务立即执行一次，然后继续按正常调度时间点运行。

BACKTRACKING_COMPENSATION
回溯补偿。单次任务立即执行；周期任务会把错过的次数立即补偿执行，然后继续按正常调度时间点运行。
```

这些语义来自 `ExpirationPolicy` 枚举的源码注释。([GitHub][13])

### 单次任务过期处理

```java id="eav3iy"
import static dev.scx.scheduling.ExpirationPolicy.BACKTRACKING_COMPENSATION;

ScxScheduling
    .oneTime()
    .startTime(Instant.now().minusSeconds(10))
    .expirationPolicy(BACKTRACKING_COMPENSATION)
    .start(context -> {
        System.out.println("开始时间已过期，所以立即补偿执行");
    });
```

单次任务默认使用 `IMMEDIATE_COMPENSATION`，也就是开始时间已过期时立即执行。([GitHub][5])

### 周期任务过期处理

```java id="aquck5"
import static dev.scx.scheduling.ExpirationPolicy.IMMEDIATE_IGNORE;

ScxScheduling
    .fixedRate()
    .startTime(Instant.now().minusSeconds(10))
    .expirationPolicy(IMMEDIATE_IGNORE)
    .interval(Duration.ofSeconds(1))
    .maxRunCount(10)
    .start(context -> {
        System.out.println("第 " + context.currentRunCount() + " 次执行");
    });
```

周期任务默认也使用 `IMMEDIATE_COMPENSATION`。([GitHub][14])

## 默认 Timer

直接调用下面这些方法时，会使用 SCX Scheduling 的默认 timer：

```java id="7qa207"
ScxScheduling.oneTime();
ScxScheduling.fixedRate();
ScxScheduling.fixedDelay();
ScxScheduling.cron();
ScxScheduling.setTimeout(...);
ScxScheduling.setInterval(...);
```

默认 timer 会懒加载创建，底层使用 `ScheduledThreadPoolExecutor`，线程池大小为 `Runtime.getRuntime().availableProcessors() * 2`。([GitHub][2])

应用退出或不再需要调度时，可以关闭默认 timer：

```java id="dfhjiv"
ScxScheduling.shutdownDefaultTimer();
```

测试代码中也展示了在第 10 次运行时调用 `ScxScheduling.shutdownDefaultTimer()` 的用法。([GitHub][3])

## 使用自定义 Timer

如果你想自己管理 timer 生命周期，可以传入自定义 `ScxTimer`：

```java id="k3p5kl"
ScxTimer timer = ...;

ScheduleHandle handle = ScxScheduling
    .fixedRate(timer)
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("使用自定义 timer 执行");
    });
```

`ScxScheduling` 提供了 `oneTime(ScxTimer)`、`cron(ScxTimer)`、`fixedRate(ScxTimer)` 和 `fixedDelay(ScxTimer)` 四个重载方法。([GitHub][2])

## 完整示例

```java id="kmk38l"
import dev.scx.scheduling.ScheduleHandle;
import dev.scx.scheduling.ScxScheduling;

import java.time.Duration;
import java.time.Instant;

import static dev.scx.scheduling.ExpirationPolicy.IMMEDIATE_COMPENSATION;

public class SchedulingExample {

    public static void main(String[] args) {

        ScheduleHandle oneTimeHandle = ScxScheduling
            .oneTime()
            .startTime(Instant.now().plusSeconds(3))
            .expirationPolicy(IMMEDIATE_COMPENSATION)
            .onError(Throwable::printStackTrace)
            .start(context -> {
                System.out.println("单次任务执行，runCount = " + context.currentRunCount());
            });

        ScheduleHandle fixedRateHandle = ScxScheduling
            .fixedRate()
            .interval(Duration.ofSeconds(1))
            .maxRunCount(5)
            .onError(Throwable::printStackTrace)
            .start(context -> {
                System.out.println("固定频率任务，第 " + context.currentRunCount() + " 次执行");

                if (context.currentRunCount() == 5) {
                    System.out.println("固定频率任务执行完成");
                }
            });

        ScheduleHandle fixedDelayHandle = ScxScheduling
            .fixedDelay()
            .interval(Duration.ofSeconds(2))
            .maxRunCount(3)
            .start(context -> {
                System.out.println("固定延迟任务，第 " + context.currentRunCount() + " 次执行");
            });

        ScheduleHandle cronHandle = ScxScheduling
            .cron()
            .cronExpression("*/10 * * * * ?")
            .maxRunCount(3)
            .start(context -> {
                System.out.println("Cron 任务，第 " + context.currentRunCount() + " 次执行");
            });

        System.out.println("oneTime nextRunTime = " + oneTimeHandle.nextRunTime());
        System.out.println("fixedRate status = " + fixedRateHandle.status());
        System.out.println("fixedDelay status = " + fixedDelayHandle.status());
        System.out.println("cron nextRunTime = " + cronHandle.nextRunTime());
    }

}
```

## 设计说明

### 1. 调度和任务执行是分开的

`ScheduleHandle` 表示的是调度本身，而不是某一次子任务。周期任务在两次执行之间依然是 `RUNNING`。([GitHub][11])

### 2. 取消调度不等于中断正在运行的任务

`cancel()` 取消的是后续调度，不包含已经开始执行的子任务。([GitHub][10])

### 3. 周期任务使用取消标记

固定频率、固定延迟和 Cron 任务使用取消标记来控制后续执行。源码注释说明，取消后后续可能还有一次已安排的回调触发，但会在回调开头被拦截。([GitHub][9])

### 4. 默认 timer 需要按应用生命周期管理

默认 timer 会懒加载创建。如果应用需要干净退出，应该在合适时机调用：

```java id="p5o5yr"
ScxScheduling.shutdownDefaultTimer();
```

默认 timer 的创建和关闭逻辑都在 `ScxScheduling` 中。([GitHub][2])

### 5. SCX Scheduling 不负责持久化任务

SCX Scheduling 提供的是内存中的调度能力。任务定义、运行次数、下一次运行时间等状态都由当前进程中的调度对象维护；如果需要任务持久化、分布式调度或应用重启后恢复，需要在业务层或更上层框架中处理。这个结论来自它当前公开 API 只围绕 `ScxTimer`、`ScheduleTask` 和 `ScheduleHandle` 进行进程内调度，并没有持久化接口。([GitHub][2])

## 常见问题

### Cron 表达式是什么格式？

SCX Scheduling 的 Cron 任务默认使用 Quartz 格式。因此示例中使用的是带秒字段的表达式：

```java id="dwifsl"
"*/5 * * * * ?"
```

源码中 `CronScheduleTaskImpl` 使用 `CronParser(instanceDefinitionFor(QUARTZ))`。([GitHub][9])

### `fixedRate` 和 `fixedDelay` 应该怎么选？

需要按照固定计划时间点执行时，使用 `fixedRate()`；需要等上一次任务执行结束后再等待一段时间时，使用 `fixedDelay()`。两者的下一次运行时间计算方式不同：`fixedRate` 基于开始时间和运行次数计算，`fixedDelay` 基于上一次执行结束时间计算。([GitHub][6])

### 如何停止周期任务？

可以保存 `ScheduleHandle` 并调用 `cancel()`：

```java id="g4zae2"
var handle = ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .start(context -> {
        System.out.println("running");
    });

handle.cancel();
```

也可以在任务内部调用：

```java id="hsyi49"
context.cancelSchedule();
```

`TaskContext#cancelSchedule()` 内部就是调用 `scheduleHandle().cancel()`。([GitHub][12])

### 如何限制运行次数？

周期任务和 Cron 任务可以使用 `maxRunCount`：

```java id="9n30zf"
ScxScheduling
    .fixedRate()
    .interval(Duration.ofSeconds(1))
    .maxRunCount(10)
    .start(context -> {
        System.out.println(context.currentRunCount());
    });
```

`PeriodicScheduleTask` 和 `CronScheduleTask` 都提供 `maxRunCount(long maxRunCount)`。([GitHub][7])

### 任务抛异常后会不会停止调度？

任务异常会被捕获并交给 `onError` 或日志处理。源码中的固定频率、固定延迟、Cron 和单次任务实现都会捕获任务异常，不会把异常直接抛出到调用方。([GitHub][5])

[1]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/ScxScheduling.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/test/java/dev/scx/scheduling/test/ScxSchedulingTest.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/one_time/OneTimeScheduleTask.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/one_time/OneTimeScheduleTaskImpl.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/periodic/FixedRatePeriodicScheduleTaskImpl.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/periodic/PeriodicScheduleTask.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/periodic/FixedDelayPeriodicScheduleTaskImpl.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/cron/CronScheduleTaskImpl.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/ScheduleHandle.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/ScheduleStatus.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/TaskContext.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/ExpirationPolicy.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-scheduling/master/src/main/java/dev/scx/scheduling/periodic/AbstractPeriodicScheduleTask.java "raw.githubusercontent.com"
