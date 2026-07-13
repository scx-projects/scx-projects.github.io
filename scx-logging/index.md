# SCX Logging

SCX Logging 是一个轻量日志库。

它提供一个很小的日志核心：`ScxLogging`、`ScxLogger`、`ScxLoggerConfig`、`ScxLogRecord` 和 `ScxLogRecorder`。同时它也通过 SPI 接入了 JDK `System.Logger`、SLF4J 和 Log4j2，让不同日志入口最终都可以输出到同一套 SCX Logging 记录器中。

SCX Logging 本身不是复杂日志框架。它不提供异步队列、复杂 layout 配置、滚动文件策略、MDC 完整语义或分布式日志能力。它的定位是一个简单、可嵌入、可配置、可桥接的日志核心。

[GitHub](https://github.com/scx-projects/scx-logging)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-logging</artifactId>
    <version>0.10.0</version>
</dependency>
```

`scx-logging` 对下面两个依赖是 optional：

```text
slf4j-api
log4j-api
```

也就是说：

1. 如果只使用 `ScxLogging` 自己的 API，不需要额外关心 SLF4J 或 Log4j2。
2. 如果希望通过 SLF4J 调用 SCX Logging，需要项目中存在 `slf4j-api`。
3. 如果希望通过 Log4j2 API 调用 SCX Logging，需要项目中存在 `log4j-api`。
4. 如果希望通过 JDK `System.Logger` 调用 SCX Logging，不需要额外依赖。

## 基本概念

SCX Logging 中最核心的概念包括：

```text
ScxLogging          全局日志入口和配置入口
ScxLogger           具体 logger
ScxLoggerConfig     logger 配置
ScxLogRecord        一条日志记录
ScxLogRecorder      日志记录器接口
ConsoleRecorder     控制台记录器
FileRecorder        文件记录器
```

它们之间的关系可以简单理解为：

```text
日志 API 调用
    ↓
ScxLogger
    ↓
判断 level 是否允许
    ↓
创建 ScxLogRecord
    ↓
交给 ScxLoggerConfig 中的 recorders
    ↓
ConsoleRecorder / FileRecorder / 自定义 Recorder
```

SCX Logging 同时支持三类外部入口：

```text
JDK System.Logger
SLF4J
Log4j2
```

这些入口最终都会转到同一个核心对象：

```text
ScxLogger
```

## 快速开始

最直接的方式是使用 `ScxLogging` 自己的 API。

```java
import dev.scx.logging.ScxLogging;

import static java.lang.System.Logger.Level.ERROR;
import static java.lang.System.Logger.Level.INFO;

var logger = ScxLogging.getLogger("demo");

logger.log(INFO, "这条默认不会输出", null);

logger.log(ERROR, "这条默认会输出", null);
```

默认配置是：

```text
level       ERROR
stackTrace  false
recorders   ConsoleRecorder
```

所以默认情况下，只有 `ERROR` 及以上级别会输出。

## 修改根配置

`ScxLoggerConfig` 是不可变配置对象。修改根配置时，需要创建新的配置并传给 `ScxLogging.rootConfig(...)`。

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;

import java.util.List;

import static java.lang.System.Logger.Level.DEBUG;

ScxLogging.rootConfig(new ScxLoggerConfig(
    DEBUG,
    true,
    List.of(new ConsoleRecorder())
));

var logger = ScxLogging.getLogger("demo");

logger.log(DEBUG, "debug message", null);
```

这里把根日志级别改成了 `DEBUG`，并开启了调用栈记录，所以 `DEBUG`、`INFO`、`WARNING`、`ERROR` 都可以输出。

## 使用 SLF4J

如果项目中使用 SLF4J，可以直接通过 SLF4J API 获取 logger。

```java
import org.slf4j.LoggerFactory;

var logger = LoggerFactory.getLogger("demo");

logger.debug("debug {}", 1);

logger.error("error {}", 2);

logger.error("error with throwable", new RuntimeException("test"));
```

只要 classpath 中存在 `scx-logging`，它会通过 `SLF4JServiceProvider` 接入 SLF4J 2.x。

SLF4J 的日志级别会映射为 JDK 日志级别：

```text
SLF4J ERROR  -> System.Logger.Level.ERROR
SLF4J WARN   -> System.Logger.Level.WARNING
SLF4J INFO   -> System.Logger.Level.INFO
SLF4J DEBUG  -> System.Logger.Level.DEBUG
SLF4J TRACE  -> System.Logger.Level.TRACE
```

SLF4J 的 `{}` 占位符会通过 SLF4J 的 `MessageFormatter` 格式化。

```java
logger.info("hello {}", "scx");
```

最终消息类似：

```text
hello scx
```

## 使用 JDK System.Logger

SCX Logging 也提供了 JDK `System.LoggerFinder` 实现。

可以直接使用 JDK API：

```java
import java.lang.System.Logger;

var logger = System.getLogger("demo");

logger.log(Logger.Level.ERROR, "error message");
```

如果使用格式化参数：

```java
logger.log(Logger.Level.ERROR, "hello {0}", "scx");
```

JDK 入口会使用 `MessageFormat` 风格进行格式化。

结果类似：

```text
hello scx
```

如果参数最后一个是 `Throwable`，会被当作异常对象处理。

```java
logger.log(
    Logger.Level.ERROR,
    "error {0}",
    "message",
    new RuntimeException("test")
);
```

## 使用 Log4j2 API

如果项目中使用 Log4j2 API，也可以直接调用：

```java
import org.apache.logging.log4j.LogManager;

var logger = LogManager.getLogger("demo");

logger.info("info message");

logger.error("error message", new RuntimeException("test"));
```

Log4j2 的日志级别会映射为 JDK 日志级别：

```text
Log4j2 FATAL  -> System.Logger.Level.ERROR
Log4j2 ERROR  -> System.Logger.Level.ERROR
Log4j2 WARN   -> System.Logger.Level.WARNING
Log4j2 INFO   -> System.Logger.Level.INFO
Log4j2 DEBUG  -> System.Logger.Level.DEBUG
Log4j2 TRACE  -> System.Logger.Level.TRACE
Log4j2 ALL    -> System.Logger.Level.ALL
Log4j2 OFF    -> System.Logger.Level.OFF
```

Log4j2 的 `Message` 会先转换为格式化后的字符串：

```java
message.getFormattedMessage()
```

然后交给 `ScxLogger` 输出。

## ScxLogging

`ScxLogging` 是全局入口类。

它负责：

```text
获取 logger
维护根配置
维护精确名称配置和正则配置
为 logger 解析当前配置
在配置变化后刷新 logger 的配置缓存
```

常用方法包括：

```java
public static ScxLogger getLogger(String name)

public static ScxLogger getLogger(Class<?> clazz)

public static ScxLoggerConfig rootConfig()

public static void rootConfig(ScxLoggerConfig rootConfig)

public static ScxLoggerConfig config(String name)

public static void config(String name, ScxLoggerConfig newConfig)

public static void removeConfig(String name)

public static ScxLoggerConfig config(Pattern regex)

public static void config(Pattern regex, ScxLoggerConfig newConfig)

public static void removeConfig(Pattern regex)
```

## getLogger

`getLogger(String name)` 根据名称获取 logger。

```java
var logger = ScxLogging.getLogger("dev.scx.demo.App");
```

同名 logger 会被缓存。

```java
var a = ScxLogging.getLogger("demo");

var b = ScxLogging.getLogger("demo");
```

`a` 和 `b` 指向同一个缓存 logger。

也可以通过 class 获取：

```java
var logger = ScxLogging.getLogger(App.class);
```

等价于：

```java
var logger = ScxLogging.getLogger(App.class.getName());
```

## 默认配置

SCX Logging 内部有一个默认根配置。

```text
level       ERROR
stackTrace  false
recorders   ConsoleRecorder
```

因此即使你不做任何配置，也可以直接输出 `ERROR` 级别日志。

```java
var logger = ScxLogging.getLogger("demo");

logger.log(System.Logger.Level.ERROR, "error", null);
```

默认会输出到控制台。

## rootConfig

`rootConfig()` 返回当前根配置。

```java
var config = ScxLogging.rootConfig();
```

设置根配置需要传入一个新的 `ScxLoggerConfig`：

```java
ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.DEBUG,
    true,
    List.of(new ConsoleRecorder())
));
```

根配置是所有 logger 配置解析的起点，并且三个字段都必须为非 `null`：

```text
level
stackTrace
recorders
```

普通名称配置和正则配置可以只设置部分字段，未设置的字段会保留之前解析出的值。

## ScxLoggerConfig

`ScxLoggerConfig` 表示 logger 配置。

它是一个 record：

```java
public record ScxLoggerConfig(
    Level level,
    Boolean stackTrace,
    List<ScxLogRecorder> recorders
) {

}
```

三个字段分别表示：

```text
level       最小日志级别
stackTrace  是否记录调用栈
recorders   日志记录器列表
```

根配置中的三个字段都必须有值。普通配置规则允许使用 `null` 表示“不覆盖这个字段”。

例如下面的配置只覆盖日志级别：

```java
var config = new ScxLoggerConfig(
    System.Logger.Level.INFO,
    null,
    null
);
```

`recorders` 不为 `null` 时，构造方法会使用 `List.copyOf(...)` 保存不可变副本。

## 设置日志级别

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.DEBUG,
    root.stackTrace(),
    root.recorders()
));
```

`ScxLogger#isLoggable(...)` 的判断逻辑是：

```text
当前日志 level 的 severity >= 配置 level 的 severity
```

也就是说，配置为 `INFO` 时：

```text
ERROR     输出
WARNING   输出
INFO      输出
DEBUG     不输出
TRACE     不输出
```

配置为 `DEBUG` 时：

```text
ERROR     输出
WARNING   输出
INFO      输出
DEBUG     输出
TRACE     不输出
```

配置为 `TRACE` 时，通常可以输出最详细日志。

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.TRACE,
    root.stackTrace(),
    root.recorders()
));
```

如果传入的日志 level 是 `null`，会直接认为不可输出。

```java
logger.isLoggable(null);
```

结果是：

```text
false
```

## 设置调用栈记录

`stackTrace` 用来控制是否记录当前日志调用上下文。

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    root.level(),
    true,
    root.recorders()
));
```

当 `stackTrace` 为 `true` 时，SCX Logging 会创建一个临时异常，并从中提取过滤后的调用栈。

输出时会追加这些调用栈信息。

如果当前日志已经带有 `Throwable`，则优先输出 `Throwable` 的堆栈，而不会再同时输出调用上下文栈。

也就是说：

```text
throwable != null       输出 throwable 堆栈
throwable == null
    且 stackTrace != null   输出调用上下文栈
```

## 调用栈过滤

SCX Logging 会过滤掉日志框架内部调用栈。

过滤规则大致是排除下面这些类名前缀：

```text
dev.scx.logging
org.slf4j.helpers
org.apache.logging.log4j
java.lang.System$Logger
```

这样输出的调用栈会更接近业务调用位置。

## 设置记录器

`recorders` 是真正处理日志记录的组件。

默认是：

```java
List.of(new ConsoleRecorder())
```

可以设置多个记录器：

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;
import java.util.List;

var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    root.level(),
    root.stackTrace(),
    List.of(
        new ConsoleRecorder(),
        new FileRecorder(Path.of("./logs"))
    )
));
```

设置为空列表表示日志通过级别检查后，不交给任何 recorder：

```java
ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of()
));
```

## 局部配置项

普通名称配置和正则配置允许使用 `null` 表示“不覆盖这个字段”。

例如只为某个 logger 设置 `DEBUG` 级别：

```java
ScxLogging.config(
    "demo.UserService",
    new ScxLoggerConfig(
        System.Logger.Level.DEBUG,
        null,
        null
    )
);
```

这里的 `stackTrace` 和 `recorders` 会继续使用前面已经解析出的值。

需要区分 `null` 和空 recorder 列表：

```text
recorders == null       不覆盖 recorder 配置
recorders == List.of()  明确覆盖为空列表
```

如果要取消整条名称配置，可以使用：

```java
ScxLogging.removeConfig("demo.UserService");
```

## 按 logger 名称配置

`ScxLogging.config(...)` 可以按精确 logger 名称或正则表达式设置配置。

字符串重载使用精确名称匹配：

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;

import static java.lang.System.Logger.Level.DEBUG;

ScxLogging.config(
    "dev.scx.demo.UserService",
    new ScxLoggerConfig(DEBUG, true, null)
);
```

正则匹配需要显式传入 `Pattern`：

```java
import java.util.regex.Pattern;

ScxLogging.config(
    Pattern.compile("dev\\.scx\\.demo\\..*"),
    new ScxLoggerConfig(DEBUG, true, null)
);
```

之后名称匹配这个正则的 logger 会应用该配置。

```java
var logger = ScxLogging.getLogger("dev.scx.demo.UserService");
```

### 匹配规则

正则配置内部使用：

```java
pattern.matcher(loggerName).matches()
```

所以它要求整个 loggerName 匹配正则。

例如：

```text
pattern: test.*
name:    test1
```

可以匹配。

```text
pattern: test
name:    test1
```

不能匹配。

如果只想匹配前缀，应写成：

```text
test.*
```

如果想匹配包名，`.` 需要转义：

```text
dev\.scx\.demo\..*
```

## 配置优先级

配置解析从根配置开始，然后按声明顺序应用所有匹配的精确名称配置和正则配置。

后声明的规则会覆盖前面规则中相同且非 `null` 的字段。

```java
var allDemo = Pattern.compile("dev\\.scx\\.demo\\..*");

ScxLogging.config(
    allDemo,
    new ScxLoggerConfig(System.Logger.Level.INFO, null, null)
);

ScxLogging.config(
    "dev.scx.demo.UserService",
    new ScxLoggerConfig(System.Logger.Level.DEBUG, true, null)
);
```

对于：

```text
dev.scx.demo.UserService
```

最终配置会得到：

```text
level       DEBUG
stackTrace  true
recorders   继续使用根配置中的值
```

精确名称规则和正则规则之间没有固定优先级，覆盖关系只取决于声明顺序。

对同一个名称或同一个 `Pattern` 再次调用 `config(...)`，新的声明会获得最新顺序。

## config 对已有 logger 的影响

`config(...)` 同时影响未来创建的 logger 和已经存在的 logger。

```java
var logger = ScxLogging.getLogger("test1");

ScxLogging.config(
    Pattern.compile("test.*"),
    new ScxLoggerConfig(
        System.Logger.Level.ERROR,
        null,
        null
    )
);
```

配置变化时，SCX Logging 会更新全局配置版本。已有 logger 在之后调用 `config()`、`isLoggable(...)` 或 `log(...)` 时会重新解析配置。

这个刷新是按需完成的，不会在 `config(...)` 调用中遍历并逐个修改所有 logger。

## removeConfig

`removeConfig(...)` 可以删除精确名称配置或正则配置。

```java
ScxLogging.removeConfig("demo.UserService");
```

```java
ScxLogging.removeConfig(Pattern.compile("demo\\..*"));
```

配置删除后，全局配置版本会发生变化。已经存在的 logger 在下一次解析配置时，会基于剩余规则和根配置重新计算结果。

如果对应配置不存在，`removeConfig(...)` 不执行其它操作。

## ScxLogger

`ScxLogger` 是实际日志对象。

常用方法包括：

```java
public boolean isLoggable(Level level)

public void log(Level level, String message, Throwable t)

public String name()

public ScxLoggerConfig config()
```

示例：

```java
var logger = ScxLogging.getLogger("demo");

if (logger.isLoggable(System.Logger.Level.DEBUG)) {
    logger.log(System.Logger.Level.DEBUG, "debug message", null);
}
```

`log(...)` 中的 `message` 和 `Throwable` 都允许为 `null`。

```java
logger.log(System.Logger.Level.ERROR, null, null);

logger.log(System.Logger.Level.ERROR, "error", null);

logger.log(System.Logger.Level.ERROR, "error", new RuntimeException("test"));
```

## ScxLogRecord

`ScxLogRecord` 表示一条日志记录。

它是一个 record：

```java
public record ScxLogRecord(
    LocalDateTime timeStamp,
    Level level,
    String loggerName,
    String message,
    String threadName,
    Throwable throwable,
    StackTraceElement[] stackTrace
) {

}
```

字段含义：

```text
timeStamp      日志时间
level          日志级别
loggerName     logger 名称
message        日志消息
threadName     当前线程名称
throwable      异常对象
stackTrace     调用上下文栈
```

`ScxLogger` 在真正输出前会创建 `ScxLogRecord`，然后交给所有 recorder。

## ScxLogRecorder

`ScxLogRecorder` 是日志记录器接口。

```java
public interface ScxLogRecorder {

    void record(ScxLogRecord logRecord);

}
```

所有输出目标都可以实现这个接口。

例如：

```text
控制台
文件
内存
数据库
HTTP
消息队列
测试断言收集器
```

只要实现 `record(...)`，就可以加入 `ScxLoggerConfig`。

## ConsoleRecorder

`ConsoleRecorder` 是控制台记录器。

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;

import java.util.List;

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(new ConsoleRecorder())
));
```

它的输出规则是：

```text
level >= ERROR    输出到 System.err
其它级别           输出到 System.out
```

也就是说，`ERROR` 级别日志走标准错误流。

```java
logger.log(System.Logger.Level.ERROR, "error", null);
```

会使用：

```java
System.err.print(...)
```

`INFO`、`DEBUG` 等级别会使用：

```java
System.out.print(...)
```

## FileRecorder

`FileRecorder` 是文件记录器。

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;
import java.util.List;

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(new FileRecorder(Path.of("./logs")))
));
```

它会按日期写入日志文件。

文件名格式是：

```text
yyyy-MM-dd.log
```

例如：

```text
2026-05-08.log
```

日志目录由构造方法传入：

```java
new FileRecorder(Path.of("./logs"))
```

最终写入路径类似：

```text
./logs/2026-05-08.log
```

### FileRecorder 的写入行为

`FileRecorder` 写文件时使用：

```text
APPEND
CREATE
SYNC
WRITE
```

也就是说：

1. 文件不存在时创建。
2. 文件存在时追加。
3. 写入时使用同步写入选项。
4. 如果父目录不存在，会尝试创建父目录。

如果写入失败，当前实现会吞掉异常，不向调用方抛出。

因此 `FileRecorder` 更偏向“尽力记录”，不会因为日志写入失败影响业务流程。

### storedDirectory 为 null

如果 `FileRecorder` 的目录是 `null`，会直接返回，不记录日志。

```java
new FileRecorder(null)
```

这种情况下不会抛异常，也不会写文件。

## 日志格式

默认格式由 `ScxLogRecordHelper.formatLogRecord(...)` 生成。

格式大致是：

```text
yyyy-MM-dd HH:mm:ss.SSS [threadName] LEVEL loggerName - message
```

示例：

```text
2026-05-08 12:30:15.123 [main] ERROR demo.App - something wrong
```

日志级别会格式化为固定宽度：

```text
ALL      -> ALL 
TRACE    -> TRACE
DEBUG    -> DEBUG
INFO     -> INFO 
WARNING  -> WARN 
ERROR    -> ERROR
OFF      -> OFF 
```

需要注意，`WARNING` 会显示为：

```text
WARN
```

## Throwable 输出

如果日志记录中包含 `Throwable`，默认格式化器会在日志行后追加异常堆栈。

示例：

```java
logger.log(
    System.Logger.Level.ERROR,
    "failed",
    new RuntimeException("test")
);
```

输出类似：

```text
2026-05-08 12:30:15.123 [main] ERROR demo.App - failed
java.lang.RuntimeException: test
    at ...
```

如果有 `Throwable`，就不会再输出调用上下文栈。

## 自定义 Recorder

可以实现自己的 `ScxLogRecorder`。

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.ScxLogRecorder;

import java.util.ArrayList;
import java.util.List;

public final class MemoryRecorder implements ScxLogRecorder {

    private final List<ScxLogRecord> records = new ArrayList<>();

    @Override
    public void record(ScxLogRecord logRecord) {
        records.add(logRecord);
    }

    public List<ScxLogRecord> records() {
        return records;
    }

}
```

使用：

```java
var memoryRecorder = new MemoryRecorder();

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(memoryRecorder)
));

var logger = ScxLogging.getLogger("demo");

logger.log(System.Logger.Level.ERROR, "error", null);

System.out.println(memoryRecorder.records().size());
```

这适合：

1. 单元测试。
2. 日志收集。
3. 自定义输出格式。
4. 写入数据库。
5. 写入外部系统。
6. 桥接到其它日志系统。

## 自定义 ConsoleRecorder 格式

`ConsoleRecorder` 提供了 `format(...)` 方法，可以通过继承覆盖。

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.recorder.ConsoleRecorder;

import java.util.List;

var recorder = new ConsoleRecorder() {

    @Override
    public String format(ScxLogRecord record) {
        return record.loggerName()
            + " : "
            + record.message()
            + System.lineSeparator();
    }

};

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(recorder)
));
```

这样可以快速改变控制台输出格式。

## 自定义 FileRecorder 格式

`FileRecorder` 也提供了 `format(...)` 方法。

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;
import java.util.List;

var recorder = new FileRecorder(Path.of("./logs")) {

    @Override
    public String format(ScxLogRecord record) {
        return record.timeStamp()
            + " "
            + record.level()
            + " "
            + record.message()
            + System.lineSeparator();
    }

};

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(recorder)
));
```

## 自定义文件名

`FileRecorder` 提供 `getLogFileName(...)` 方法。

默认是：

```text
yyyy-MM-dd.log
```

可以覆盖：

```java
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;
import java.time.format.DateTimeFormatter;
import java.time.temporal.TemporalAccessor;

var recorder = new FileRecorder(Path.of("./logs")) {

    private final DateTimeFormatter formatter =
        DateTimeFormatter.ofPattern("yyyy-MM-dd-HH");

    @Override
    public String getLogFileName(TemporalAccessor temporal) {
        return formatter.format(temporal) + ".log";
    }

};
```

这样可以按小时拆分：

```text
2026-05-08-12.log
```

需要注意，`FileRecorder` 本身不提供日志滚动、压缩、保留天数清理等能力。

## ServiceLoader 接入

SCX Logging 通过 `META-INF/services` 注册了三类 SPI。

```text
META-INF/services/java.lang.System$LoggerFinder
META-INF/services/org.slf4j.spi.SLF4JServiceProvider
META-INF/services/org.apache.logging.log4j.spi.Provider
```

对应实现是：

```text
dev.scx.logging.spi.jdk.ScxJDKLoggerFinder
dev.scx.logging.spi.slf4j.ScxSLF4JServiceProvider
dev.scx.logging.spi.log4j.ScxLog4jProvider
```

因此，在 classpath 中加入 `scx-logging` 后：

```text
JDK System.Logger 可以找到 ScxJDKLoggerFinder
SLF4J 2.x 可以找到 ScxSLF4JServiceProvider
Log4j2 可以找到 ScxLog4jProvider
```

## JDK Logger 适配

JDK 适配层包括：

```text
ScxJDKLogger
ScxJDKLoggerFinder
```

`ScxJDKLoggerFinder` 会把 JDK logger 名称转交给：

```java
ScxLogging.getLogger(name)
```

`ScxJDKLogger` 实现了：

```java
java.lang.System.Logger
```

核心行为包括：

1. `getName()` 返回底层 `ScxLogger` 的名称。
2. `isLoggable(level)` 使用 SCX Logging 的级别判断。
3. `log(level, bundle, msg, throwable)` 支持 ResourceBundle 文本查找。
4. `log(level, bundle, format, params)` 支持 `MessageFormat` 格式化。
5. 参数数组最后一个如果是 `Throwable`，会作为异常处理。

## SLF4J 适配

SLF4J 适配层包括：

```text
ScxSLF4JLogger
ScxSLF4JLoggerFactory
ScxSLF4JServiceProvider
```

`ScxSLF4JServiceProvider` 提供：

```text
ILoggerFactory     ScxSLF4JLoggerFactory
IMarkerFactory     BasicMarkerFactory
MDCAdapter         BasicMDCAdapter
```

`ScxSLF4JLoggerFactory` 会把 SLF4J logger 名称转交给：

```java
ScxLogging.getLogger(name)
```

`ScxSLF4JLogger` 继承：

```java
LegacyAbstractLogger
```

并在 `handleNormalizedLoggingCall(...)` 中把日志转发给 `ScxLogger`。

### SLF4J Marker

当前 SLF4J 适配中，`Marker` 参数不会写入 `ScxLogRecord`。

也就是说，Marker 不参与默认输出格式，也不参与过滤。

## Log4j2 适配

Log4j2 适配层包括：

```text
ScxLog4jLogger
ScxLog4jLoggerContext
ScxLog4jLoggerContextFactory
ScxLog4jProvider
```

`ScxLog4jProvider` 注册 `ScxLog4jLoggerContextFactory`。

`ScxLog4jLoggerContextFactory` 返回一个单例 `ScxLog4jLoggerContext`。

`ScxLog4jLoggerContext` 会把 logger 名称转交给：

```java
ScxLogging.getLogger(name)
```

并创建：

```java
new ScxLog4jLogger(...)
```

### Log4j2 Marker

当前 Log4j2 适配中，`Marker` 参数不会写入 `ScxLogRecord`。

也就是说，Marker 不参与默认输出格式，也不参与过滤。

### Log4j2 hasLogger

当前 `ScxLog4jLoggerContext#hasLogger(...)` 系列方法返回 `false`。

因此不要依赖它判断 logger 是否已经存在。

## 线程安全

SCX Logging 对全局 logger 缓存和配置表做了基础并发处理。

logger 缓存、精确名称配置和正则配置都使用：

```java
ConcurrentHashMap
```

配置版本和声明顺序使用：

```java
AtomicLong
```

根配置通过 `volatile` 引用发布。logger 内部缓存当前配置和对应的配置版本，版本变化后会在后续调用中重新解析。

配置解析不保证并发修改期间的严格快照一致性。一次解析可能看到修改前后的混合状态，但之后的调用会根据新版本再次解析并最终使用新配置。

需要注意：

```text
recorders 本身是否线程安全，取决于具体 recorder 实现
```

例如自定义 recorder 如果内部使用普通 `ArrayList` 收集日志，在多线程日志输出时需要自己保证同步。

## 完整示例：直接使用 ScxLogging

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;

import java.util.List;

import static java.lang.System.Logger.Level.DEBUG;
import static java.lang.System.Logger.Level.ERROR;

public class LoggingDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig(new ScxLoggerConfig(
            DEBUG,
            false,
            List.of(new ConsoleRecorder())
        ));

        var logger = ScxLogging.getLogger(LoggingDemo.class);

        logger.log(DEBUG, "debug message", null);

        logger.log(ERROR, "error message", new RuntimeException("test"));
    }

}
```

## 完整示例：SLF4J 入口

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;
import org.slf4j.LoggerFactory;

import java.util.List;

import static java.lang.System.Logger.Level.DEBUG;

public class Slf4jDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig(new ScxLoggerConfig(
            DEBUG,
            false,
            List.of(new ConsoleRecorder())
        ));

        var logger = LoggerFactory.getLogger(Slf4jDemo.class);

        logger.debug("debug {}", 1);

        logger.info("info {}", 2);

        logger.error("error {}", 3, new RuntimeException("test"));
    }

}
```

## 完整示例：按名称配置

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import org.slf4j.LoggerFactory;

import java.util.regex.Pattern;

import static java.lang.System.Logger.Level.DEBUG;
import static java.lang.System.Logger.Level.ERROR;

public class ConfigDemo {

    public static void main(String[] args) {
        ScxLogging.config(
            Pattern.compile("demo\\..*"),
            new ScxLoggerConfig(DEBUG, true, null)
        );

        ScxLogging.config(
            Pattern.compile("demo\\.quiet\\..*"),
            new ScxLoggerConfig(ERROR, null, null)
        );

        var logger1 = LoggerFactory.getLogger("demo.UserService");

        var logger2 = LoggerFactory.getLogger("demo.quiet.Job");

        logger1.debug("会输出");

        logger2.debug("不会输出");

        logger2.error("会输出");
    }

}
```

## 完整示例：控制台 + 文件

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;
import java.util.List;

import static java.lang.System.Logger.Level.INFO;

public class RecorderDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig(new ScxLoggerConfig(
            INFO,
            false,
            List.of(
                new ConsoleRecorder(),
                new FileRecorder(Path.of("./logs"))
            )
        ));

        var logger = ScxLogging.getLogger("demo");

        logger.log(INFO, "hello file and console", null);
    }

}
```

如果当前日期是：

```text
2026-05-08
```

文件输出路径类似：

```text
./logs/2026-05-08.log
```

## 完整示例：测试用内存 Recorder

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.ScxLogRecorder;
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;

import java.util.ArrayList;
import java.util.List;

import static java.lang.System.Logger.Level.DEBUG;

public class MemoryRecorderDemo {

    public static void main(String[] args) {
        var recorder = new MemoryRecorder();

        ScxLogging.rootConfig(new ScxLoggerConfig(
            DEBUG,
            false,
            List.of(recorder)
        ));

        var logger = ScxLogging.getLogger("demo");

        logger.log(DEBUG, "hello", null);

        System.out.println(recorder.records().size());
    }

    public static class MemoryRecorder implements ScxLogRecorder {

        private final List<ScxLogRecord> records = new ArrayList<>();

        @Override
        public void record(ScxLogRecord logRecord) {
            records.add(logRecord);
        }

        public List<ScxLogRecord> records() {
            return records;
        }

    }

}
```

## 方法总览

### ScxLogging

```java
public static ScxLogger getLogger(String name)

public static ScxLogger getLogger(Class<?> clazz)
```

```java
public static ScxLoggerConfig rootConfig()

public static void rootConfig(ScxLoggerConfig rootConfig)
```

```java
public static ScxLoggerConfig config(String name)

public static void config(String name, ScxLoggerConfig newConfig)

public static void removeConfig(String name)
```

```java
public static ScxLoggerConfig config(Pattern regex)

public static void config(Pattern regex, ScxLoggerConfig newConfig)

public static void removeConfig(Pattern regex)
```

### ScxLogger

```java
public boolean isLoggable(Level level)

public void log(Level level, String message, Throwable t)

public String name()

public ScxLoggerConfig config()
```

### ScxLoggerConfig

```java
public ScxLoggerConfig(
    Level level,
    Boolean stackTrace,
    List<ScxLogRecorder> recorders
)
```

```java
public Level level()

public Boolean stackTrace()

public List<ScxLogRecorder> recorders()
```

### ScxLogRecorder

```java
void record(ScxLogRecord logRecord)
```

### ConsoleRecorder

```java
public void record(ScxLogRecord logRecord)

public String format(ScxLogRecord logRecord)
```

### FileRecorder

```java
public FileRecorder(Path storedDirectory)

public static void writeToFile(Path path, String data)

public String getLogFileName(TemporalAccessor temporal)

public void record(ScxLogRecord logRecord)

public String format(ScxLogRecord logRecord)
```

## 设计说明

### 1. SCX Logging 的核心很小

核心对象只有几类：

```text
ScxLogging
ScxLogger
ScxLoggerConfig
ScxLogRecord
ScxLogRecorder
```

其它 JDK、SLF4J、Log4j2 相关类都是适配层。

这意味着你可以只用 SCX Logging 自己的 API，也可以通过已有日志门面接入它。

### 2. 级别体系统一使用 System.Logger.Level

SCX Logging 内部统一使用：

```java
java.lang.System.Logger.Level
```

SLF4J 和 Log4j2 入口都会先映射到 JDK level，再交给 `ScxLogger`。

这样可以减少内部级别体系数量。

### 3. logger 名称配置支持精确匹配和正则匹配

字符串重载用于精确 logger 名称：

```java
ScxLogging.config("dev.scx.demo.App", config);
```

`Pattern` 重载用于正则匹配：

```java
ScxLogging.config(Pattern.compile("dev\\.scx\\..*"), config);
```

正则使用 `matches()`，要求匹配完整 logger 名称。

### 4. 后声明的字段覆盖前面的字段

配置解析从根配置开始，然后按声明顺序合并全部匹配规则。

后声明的规则只覆盖自己非 `null` 的字段。

这让不同规则可以分别设置 level、stackTrace 和 recorders。

### 5. 记录器是同步调用

`ScxLogger` 创建 `ScxLogRecord` 后，会直接遍历 `recorders` 并调用：

```java
r.record(logRecord);
```

当前核心没有异步队列。

因此：

1. recorder 执行耗时会影响日志调用耗时。
2. recorder 抛出的 `Exception` 会被忽略，不会传播到用户代码。
3. 如果需要异步记录，应由自定义 recorder 自己实现异步队列。

需要注意，内置 `FileRecorder` 写入失败时也会吞掉异常。

### 6. FileRecorder 是简单文件输出

`FileRecorder` 只负责按日期写入文件。

它不负责：

```text
日志滚动
日志压缩
日志清理
文件大小切分
异步写入
并发写入优化
复杂 layout
```

如果需要这些能力，应扩展 recorder 或接入更完整的日志系统。

### 7. stackTrace 和 throwable 不重复输出

如果一条日志带有 `Throwable`，默认格式会输出异常堆栈。

如果没有 `Throwable`，但开启了 `stackTrace`，才会输出调用上下文栈。

这样可以避免同一条日志同时输出两套堆栈。

### 8. 配置变化后 logger 会按需重新解析

根配置、名称配置或正则配置发生变化后，全局配置版本会更新。

已经存在的 logger 会在之后的 `config()`、`isLoggable(...)` 或 `log(...)` 调用中重新解析配置。

### 9. 默认 recorder 只有 ConsoleRecorder

SCX Logging 默认只输出到控制台。

如果需要文件输出，需要显式设置 `FileRecorder`。

```java
ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(new FileRecorder(Path.of("./logs")))
));
```

### 10. optional 依赖意味着按需接入

`slf4j-api` 和 `log4j-api` 是 optional。

这表示 SCX Logging 可以作为很小的直接日志库使用。

只有当项目本身需要 SLF4J 或 Log4j2 API 时，才需要对应 API 出现在 classpath 中。

## 常见问题

### SCX Logging 是 SLF4J 实现吗？

它可以作为 SLF4J 2.x 的 provider 使用，但它本身不仅仅是 SLF4J 实现。

它还有自己的核心 API，并且同时接入 JDK `System.Logger` 和 Log4j2 API。

### 默认会输出哪些级别？

默认 level 是：

```text
ERROR
```

所以默认只输出 `ERROR` 及以上级别。

### 如何开启 DEBUG？

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.DEBUG,
    root.stackTrace(),
    root.recorders()
));
```

### 如何开启 TRACE？

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.TRACE,
    root.stackTrace(),
    root.recorders()
));
```

### 如何输出到文件？

```java
ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(new FileRecorder(Path.of("./logs")))
));
```

如果需要同时输出到控制台和文件，可以把 `ConsoleRecorder` 和 `FileRecorder` 一起放入列表。

### 文件日志叫什么名字？

默认按日期命名：

```text
yyyy-MM-dd.log
```

例如：

```text
2026-05-08.log
```

### FileRecorder 会自动创建目录吗？

如果写入时发现父目录不存在，会尝试创建目录。

### FileRecorder 写入失败会抛异常吗？

当前实现会吞掉写入异常。

也就是说，文件写入失败不会影响业务流程。

### 如何自定义日志格式？

可以继承 `ConsoleRecorder` 或 `FileRecorder`，覆盖 `format(...)`。

```java
new ConsoleRecorder() {

    @Override
    public String format(ScxLogRecord record) {
        return record.message() + System.lineSeparator();
    }

}
```

也可以直接实现 `ScxLogRecorder`。

### 如何自定义日志输出目标？

实现 `ScxLogRecorder`。

```java
public final class MyRecorder implements ScxLogRecorder {

    @Override
    public void record(ScxLogRecord logRecord) {
        // 自己处理
    }

}
```

然后放入配置：

```java
ScxLogging.rootConfig(new ScxLoggerConfig(
    System.Logger.Level.ERROR,
    false,
    List.of(new MyRecorder())
));
```

### config 的 String 参数是正则表达式吗？

不是。

```java
ScxLogging.config("demo.UserService", config);
```

这里使用精确名称匹配。正则匹配需要显式传入 `Pattern`：

```java
ScxLogging.config(Pattern.compile("demo\\..*"), config);
```

### 为什么 `dev.scx.*` 没匹配到？

如果传给 `config(String, ...)`，它会被当作完整 logger 名称，不会作为正则解析。

正则匹配应写成：

```java
Pattern.compile("dev\\.scx\\..*")
```

### config 会影响已经创建的 logger 吗？

会。

配置变化后，已有 logger 会在下一次解析配置时使用新规则。

### removeConfig 会恢复已经创建的 logger 吗？

会重新计算。

配置删除后，已有 logger 在下一次解析配置时会基于剩余规则和根配置得到新的结果。

### 可以直接改某个 logger 的配置吗？

`logger.config()` 返回当前解析出的不可变配置，不能直接修改。

可以使用精确名称配置：

```java
ScxLogging.config(
    "demo",
    new ScxLoggerConfig(System.Logger.Level.DEBUG, null, null)
);
```

### rootConfig 修改后所有 logger 都会变化吗？

根配置变化后，已有 logger 会在后续调用中重新解析。

如果某个字段被名称配置或正则配置覆盖，则该字段仍使用后声明规则中的值。

### stackTrace 是什么？

它表示是否在日志中附加当前调用上下文栈。

```java
var root = ScxLogging.rootConfig();

ScxLogging.rootConfig(new ScxLoggerConfig(
    root.level(),
    true,
    root.recorders()
));
```

### 有 Throwable 时还会输出调用上下文栈吗？

不会。

默认格式中，`Throwable` 优先。

如果 `Throwable` 不为 `null`，会输出 Throwable 堆栈；否则才可能输出调用上下文栈。

### SLF4J 的 Marker 会输出吗？

当前不会。

Marker 参数不会写入 `ScxLogRecord`。

### Log4j2 的 Marker 会输出吗？

当前不会。

Marker 参数不会写入 `ScxLogRecord`。

### SLF4J 的 MDC 会输出吗？

当前不会。

`ScxSLF4JServiceProvider` 提供的是 `BasicMDCAdapter`，但 `ScxLogRecord` 中没有 MDC 字段，默认格式也不会输出 MDC。

### Log4j2 配置文件会生效吗？

当前 SCX Logging 的 Log4j2 适配不会读取 Log4j2 配置文件。

它只是把 Log4j2 API 调用桥接到 `ScxLogger`。

### ConsoleRecorder 的 ERROR 为什么走 System.err？

`ConsoleRecorder` 中规定：

```text
level >= ERROR    System.err
其它级别           System.out
```

这样错误日志会走标准错误流。

### SCX Logging 是异步的吗？

不是。

recorders 是同步调用。

如果需要异步输出，可以自己实现异步 `ScxLogRecorder`。

### SCX Logging 线程安全吗？

logger 缓存和配置表使用并发容器，配置变化通过版本号通知已有 logger 重新解析。

配置解析在并发更新期间不保证严格快照一致性，但会最终使用新配置。

具体 recorder 是否线程安全，取决于 recorder 自身实现。

### 什么时候用 SCX Logging？

适合下面这些场景：

1. 想要一个很小的日志核心。
2. 想要同时接入 JDK `System.Logger`、SLF4J 和 Log4j2。
3. 想直接用 Java 代码配置日志。
4. 想按 logger 精确名称或正则配置级别和 recorder。
5. 想快速输出到控制台或简单文件。
6. 想自定义 recorder 收集日志。

