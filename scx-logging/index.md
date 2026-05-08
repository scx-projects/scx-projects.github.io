# SCX Logging

SCX Logging 是一个轻量日志库。

它提供一个很小的日志核心：`ScxLogging`、`ScxLogger`、`ScxLoggerConfig`、`ScxLogRecord` 和 `ScxLogRecorder`。同时它也通过 SPI 接入了 JDK `System.Logger`、SLF4J 和 Log4j2，让不同日志入口最终都可以输出到同一套 SCX Logging 记录器中。

SCX Logging 本身不是复杂日志框架。它不提供异步队列、复杂 layout 配置、滚动文件策略、MDC 完整语义或分布式日志能力。它的定位是一个简单、可嵌入、可配置、可桥接的日志核心。

当前版本为 `0.0.1`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-logging</artifactId>
    <version>0.0.1</version>
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

可以通过 `ScxLogging.rootConfig()` 修改根配置。

```java
import dev.scx.logging.ScxLogging;

import static java.lang.System.Logger.Level.DEBUG;

ScxLogging.rootConfig()
    .setLevel(DEBUG)
    .setStackTrace(true);

var logger = ScxLogging.getLogger("demo");

logger.log(DEBUG, "debug message", null);
```

这里把根日志级别改成了 `DEBUG`，所以 `DEBUG`、`INFO`、`WARNING`、`ERROR` 都可以输出。

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
维护按名称匹配的配置
把配置应用到已有 logger
给新 logger 选择匹配配置
```

常用方法包括：

```java
public static ScxLoggerConfig rootConfig()

public static ScxLogger getLogger(String name)

public static ScxLogger getLogger(Class clazz)

public static void setConfig(String name, ScxLoggerConfig newConfig)

public static void removeConfig(String name)
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

SCX Logging 内部有一个默认配置。

```text
level       ERROR
stackTrace  false
recorders   ConsoleRecorder
```

根配置 `ROOT_CONFIG` 会继承这个默认配置。

因此即使你不做任何配置，也可以直接输出 `ERROR` 级别日志。

```java
var logger = ScxLogging.getLogger("demo");

logger.log(System.Logger.Level.ERROR, "error", null);
```

默认会输出到控制台。

## rootConfig

`rootConfig()` 返回根配置。

```java
var config = ScxLogging.rootConfig();
```

可以直接修改根配置：

```java
ScxLogging.rootConfig()
    .setLevel(System.Logger.Level.DEBUG)
    .setStackTrace(true);
```

根配置会作为普通 logger 配置的父配置。

也就是说，如果某个 logger 自己没有设置 level、stackTrace 或 recorders，就会向父配置查找，最终回退到默认配置。

## ScxLoggerConfig

`ScxLoggerConfig` 表示 logger 配置。

它包含三类配置项：

```text
level       最小日志级别
stackTrace  是否记录调用栈
recorders   日志记录器列表
```

配置支持父子继承。

```java
var config = new ScxLoggerConfig(parentConfig);
```

如果当前配置没有设置某个字段，就会从 parent 中查找。

例如：

```java
var parent = new ScxLoggerConfig()
    .setLevel(System.Logger.Level.INFO)
    .setStackTrace(false);

var child = new ScxLoggerConfig(parent);

System.out.println(child.level());
System.out.println(child.stackTrace());
```

结果来自父配置：

```text
INFO
false
```

如果当前配置和所有父配置都没有对应字段，会抛出 `IllegalStateException`。

## 设置日志级别

```java
import static java.lang.System.Logger.Level.DEBUG;

ScxLogging.rootConfig()
    .setLevel(DEBUG);
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
ScxLogging.rootConfig()
    .setLevel(System.Logger.Level.TRACE);
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
ScxLogging.rootConfig()
    .setStackTrace(true);
```

当 `stackTrace` 为 `true`，并且当前日志没有传入 `Throwable` 时，SCX Logging 会创建一个临时异常，并从中提取过滤后的调用栈。

输出时会追加这些调用栈信息。

如果当前日志已经带有 `Throwable`，则优先输出 `Throwable` 的堆栈，而不会再同时输出 context stack。

也就是说：

```text
throwable != null       输出 throwable 堆栈
throwable == null
    且 contextStack != null   输出 context stack
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
new ConsoleRecorder()
```

可以追加记录器：

```java
import dev.scx.logging.recorder.ConsoleRecorder;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;

ScxLogging.rootConfig()
    .addRecorder(
        new ConsoleRecorder(),
        new FileRecorder(Path.of("./logs"))
    );
```

也可以覆盖记录器列表：

```java
ScxLogging.rootConfig()
    .setRecorder(List.of(
        new ConsoleRecorder()
    ));
```

需要注意：

```text
addRecorder(...) 是追加
setRecorder(...) 是覆盖
```

## 清除配置项

`ScxLoggerConfig` 支持清除某个本地配置项。

```java
config.clearLevel();

config.clearStackTrace();

config.clearRecorders();
```

清除后，如果有父配置，就会重新从父配置读取。

例如：

```java
var parent = new ScxLoggerConfig()
    .setLevel(System.Logger.Level.INFO);

var child = new ScxLoggerConfig(parent)
    .setLevel(System.Logger.Level.DEBUG);

System.out.println(child.level());
```

结果：

```text
DEBUG
```

清除后：

```java
child.clearLevel();

System.out.println(child.level());
```

结果：

```text
INFO
```

## 按 logger 名称配置

`ScxLogging.setConfig(...)` 可以按 logger 名称设置配置。

第一个参数不是普通字符串匹配，而是正则表达式。

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;

import static java.lang.System.Logger.Level.DEBUG;

ScxLogging.setConfig(
    "dev\\.scx\\.demo\\..*",
    new ScxLoggerConfig()
        .setLevel(DEBUG)
        .setStackTrace(true)
);
```

之后名称匹配这个正则的 logger 会应用该配置。

```java
var logger = ScxLogging.getLogger("dev.scx.demo.UserService");
```

这个 logger 会匹配：

```text
dev\.scx\.demo\..*
```

### 匹配规则

内部使用的是：

```java
Pattern.matches(pattern, loggerName)
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

`setConfig(...)` 会把新的配置插入到配置列表最前面。

查找配置时按插入顺序从前到后遍历。

因此：

```text
新的配置优先级更高
旧的配置优先级更低
```

示例：

```java
ScxLogging.setConfig(
    "dev\\.scx\\..*",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.INFO)
);

ScxLogging.setConfig(
    "dev\\.scx\\.demo\\..*",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.DEBUG)
);
```

对于：

```text
dev.scx.demo.UserService
```

后设置的：

```text
dev\.scx\.demo\..*
```

会优先命中。

## setConfig 对已有 logger 的影响

`setConfig(...)` 不只影响未来创建的 logger，也会更新已经存在且名称匹配的 logger。

```java
var logger = ScxLogging.getLogger("test1");

ScxLogging.setConfig(
    "test.*",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.ERROR)
);
```

这里已经存在的 `test1` 也会被更新。

更新方式是：

```text
把 newConfig 中显式设置过的字段复制到 logger 当前 config 中
```

也就是说：

```text
level
stackTrace
recorders
```

会按 `newConfig` 的本地字段更新。

## removeConfig

`removeConfig(name)` 会删除某个正则配置。

```java
ScxLogging.removeConfig("test.*");
```

需要注意，当前实现只是从全局配置表中移除这条配置。

它不会自动把已经存在的 logger 重新计算并恢复到旧配置。

也就是说：

1. `removeConfig(...)` 会影响之后创建 logger 时的配置匹配。
2. 已经被 `setConfig(...)` 更新过的 logger，不会因为 `removeConfig(...)` 自动回滚。
3. 如果需要修改已有 logger，需要再次 `setConfig(...)` 或直接修改该 logger 的 `config()`。

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
    StackTraceElement[] contextStack
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
contextStack   调用上下文栈
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
import dev.scx.logging.recorder.ConsoleRecorder;

ScxLogging.rootConfig()
    .addRecorder(new ConsoleRecorder());
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
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;

ScxLogging.rootConfig()
    .addRecorder(new FileRecorder(Path.of("./logs")));
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

默认格式由 `ScxLogRecorderHelper.formatLogRecord(...)` 生成。

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

如果有 `Throwable`，就不会再输出 context stack。

## 自定义 Recorder

可以实现自己的 `ScxLogRecorder`。

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.recorder.ScxLogRecorder;

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

ScxLogging.rootConfig()
    .addRecorder(memoryRecorder);

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
import dev.scx.logging.recorder.ConsoleRecorder;

var recorder = new ConsoleRecorder() {

    @Override
    public String format(ScxLogRecord record) {
        return record.loggerName()
            + " : "
            + record.message()
            + System.lineSeparator();
    }

};

ScxLogging.rootConfig()
    .addRecorder(recorder);
```

这样可以快速改变控制台输出格式。

## 自定义 FileRecorder 格式

`FileRecorder` 也提供了 `format(...)` 方法。

```java
import dev.scx.logging.ScxLogRecord;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;

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

ScxLogging.rootConfig()
    .addRecorder(recorder);
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

logger 缓存使用：

```java
ConcurrentHashMap
```

配置表使用：

```java
LinkedHashMap
```

并通过：

```java
ReentrantReadWriteLock
```

保护配置表的读写。

`setConfig(...)` 会在写锁内：

1. 插入新的配置规则。
2. 遍历已有 logger。
3. 更新匹配的 logger 配置。

`getLogger(...)` 创建 logger 时，会查找当前配置表中第一个匹配的配置。

测试中也覆盖了“并发 setConfig 和 getLogger”的场景。

需要注意：

```text
recorders 本身是否线程安全，取决于具体 recorder 实现
```

例如自定义 recorder 如果内部使用普通 `ArrayList` 收集日志，在多线程日志输出时需要自己保证同步。

## 完整示例：直接使用 ScxLogging

```java
import dev.scx.logging.ScxLogging;

import static java.lang.System.Logger.Level.DEBUG;
import static java.lang.System.Logger.Level.ERROR;

public class LoggingDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig()
            .setLevel(DEBUG)
            .setStackTrace(false);

        var logger = ScxLogging.getLogger(LoggingDemo.class);

        logger.log(DEBUG, "debug message", null);

        logger.log(ERROR, "error message", new RuntimeException("test"));
    }

}
```

## 完整示例：SLF4J 入口

```java
import dev.scx.logging.ScxLogging;
import org.slf4j.LoggerFactory;

import static java.lang.System.Logger.Level.DEBUG;

public class Slf4jDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig()
            .setLevel(DEBUG);

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

import static java.lang.System.Logger.Level.DEBUG;
import static java.lang.System.Logger.Level.ERROR;

public class ConfigDemo {

    public static void main(String[] args) {
        ScxLogging.setConfig(
            "demo\\..*",
            new ScxLoggerConfig()
                .setLevel(DEBUG)
                .setStackTrace(true)
        );

        ScxLogging.setConfig(
            "demo\\.quiet\\..*",
            new ScxLoggerConfig()
                .setLevel(ERROR)
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
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ConsoleRecorder;
import dev.scx.logging.recorder.FileRecorder;

import java.nio.file.Path;

import static java.lang.System.Logger.Level.INFO;

public class RecorderDemo {

    public static void main(String[] args) {
        ScxLogging.rootConfig()
            .setLevel(INFO)
            .setRecorder(List.of(
                new ConsoleRecorder(),
                new FileRecorder(Path.of("./logs"))
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
import dev.scx.logging.ScxLogging;
import dev.scx.logging.recorder.ScxLogRecorder;

import java.util.ArrayList;
import java.util.List;

import static java.lang.System.Logger.Level.DEBUG;

public class MemoryRecorderDemo {

    public static void main(String[] args) {
        var recorder = new MemoryRecorder();

        ScxLogging.rootConfig()
            .setLevel(DEBUG)
            .addRecorder(recorder);

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
public static ScxLoggerConfig rootConfig()

public static ScxLogger getLogger(String name)

public static ScxLogger getLogger(Class clazz)

public static void setConfig(String name, ScxLoggerConfig newConfig)

public static void removeConfig(String name)
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
public ScxLoggerConfig()

public ScxLoggerConfig(ScxLoggerConfig parent)
```

```java
public Level level()

public Boolean stackTrace()

public List recorders()
```

```java
public ScxLoggerConfig setLevel(Level newLevel)

public ScxLoggerConfig setStackTrace(Boolean newStackTrace)

public ScxLoggerConfig setRecorder(List recorders)
```

```java
public ScxLoggerConfig clearLevel()

public ScxLoggerConfig clearStackTrace()

public ScxLoggerConfig clearRecorders()
```

```java
public ScxLoggerConfig addRecorder(ScxLogRecorder... recorders)

public ScxLoggerConfig removeRecorder(ScxLogRecorder recorder)

public ScxLoggerConfig updateConfig(ScxLoggerConfig newConfig)
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

### 3. logger 名称配置使用正则

`setConfig(...)` 的第一个参数是正则表达式。

这让配置可以很灵活，例如：

```text
dev\.scx\..*
.*Controller
test.*
```

但也意味着普通包名中的 `.` 需要转义。

### 4. 新配置优先

配置表使用倒序插入。

最新设置的配置会优先被匹配。

这让更具体的配置可以后设置，从而覆盖更宽泛的配置。

### 5. 记录器是同步调用

`ScxLogger` 创建 `ScxLogRecord` 后，会直接遍历 `recorders` 并调用：

```java
r.record(logRecord);
```

当前核心没有异步队列。

因此：

1. recorder 执行耗时会影响日志调用耗时。
2. recorder 抛异常会影响日志调用。
3. 如果需要异步记录，应由自定义 recorder 自己实现异步队列。

需要注意，内置 `FileRecorder` 写入失败时会吞掉异常，因此它不会因为文件写入失败影响调用方。

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

如果没有 `Throwable`，但开启了 `stackTrace`，才会输出 context stack。

这样可以避免同一条日志同时输出两套堆栈。

### 8. removeConfig 不会自动回滚已有 logger

`removeConfig(...)` 只删除全局配置规则。

已经存在的 logger 不会自动重新计算配置。

这是当前实现的一个重要边界。

如果需要让已有 logger 改回某个状态，应再次调用 `setConfig(...)` 或直接修改对应 logger 的 `config()`。

### 9. 默认 recorder 只有 ConsoleRecorder

SCX Logging 默认只输出到控制台。

如果需要文件输出，需要显式添加 `FileRecorder`。

```java
ScxLogging.rootConfig()
    .addRecorder(new FileRecorder(Path.of("./logs")));
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
ScxLogging.rootConfig()
    .setLevel(System.Logger.Level.DEBUG);
```

### 如何开启 TRACE？

```java
ScxLogging.rootConfig()
    .setLevel(System.Logger.Level.TRACE);
```

### 如何输出到文件？

```java
ScxLogging.rootConfig()
    .addRecorder(new FileRecorder(Path.of("./logs")));
```

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

然后加入配置：

```java
ScxLogging.rootConfig()
    .addRecorder(new MyRecorder());
```

### setConfig 的 name 是普通字符串吗？

不是。

它是正则表达式。

内部使用：

```java
Pattern.matches(name, loggerName)
```

### 为什么 `dev.scx.*` 没匹配到？

因为 `.` 在正则中表示任意字符。

如果想匹配包名点号，应写成：

```text
dev\.scx\..*
```

Java 字符串里则要写：

```java
"dev\\.scx\\..*"
```

### setConfig 会影响已经创建的 logger 吗？

会。

`setConfig(...)` 会遍历已经存在的 logger，并更新名称匹配的 logger 配置。

### removeConfig 会恢复已经创建的 logger 吗？

不会。

`removeConfig(...)` 只是删除配置规则，不会自动回滚已有 logger。

### 可以直接改某个 logger 的配置吗？

可以。

```java
ScxLogging.getLogger("demo")
    .config()
    .setLevel(System.Logger.Level.DEBUG);
```

### rootConfig 修改后所有 logger 都会变化吗？

对于没有本地覆盖配置的 logger，会通过父配置读取到新的 root 配置。

如果某个 logger 已经被 `setConfig(...)` 或直接 `config()` 设置了本地字段，则对应字段会优先使用自己的值。

### stackTrace 是什么？

它表示是否在没有 `Throwable` 的日志中附加当前调用上下文栈。

```java
ScxLogging.rootConfig()
    .setStackTrace(true);
```

### 有 Throwable 时还会输出 context stack 吗？

不会。

默认格式中，`Throwable` 优先。

如果 `Throwable` 不为 `null`，会输出 Throwable 堆栈；否则才可能输出 context stack。

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

logger 缓存和配置表有基础并发保护。

但具体 recorder 是否线程安全，取决于 recorder 自身实现。

自定义 recorder 如果有内部可变状态，需要自己保证线程安全。

### 什么时候用 SCX Logging？

适合下面这些场景：

1. 想要一个很小的日志核心。
2. 想要同时接入 JDK `System.Logger`、SLF4J 和 Log4j2。
3. 想直接用 Java 代码配置日志。
4. 想按 logger 名称正则配置级别和 recorder。
5. 想快速输出到控制台或简单文件。
6. 想自定义 recorder 收集日志。
7. 不需要复杂日志框架的完整能力。