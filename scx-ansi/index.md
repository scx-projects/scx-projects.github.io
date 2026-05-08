# SCX ANSI

SCX ANSI 是一个用于生成 ANSI 彩色控制台文本的轻量工具库。

它可以把普通文本、前景色、背景色和样式组合成 ANSI SGR 转义序列，然后输出到控制台，或者生成字符串交给调用者自己处理。

SCX ANSI 本身不是终端模拟器，也不是日志框架。它只负责构造 ANSI 文本，并在当前环境不支持 ANSI 时自动退化为普通文本。

当前版本为 `0.2.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-ansi</artifactId>
    <version>0.2.0</version>
</dependency>
```

SCX ANSI 依赖 `scx-ffi`。

这个依赖主要用于 Windows 10/11 上尝试调用 `Kernel32`，开启控制台的 ANSI 虚拟终端处理能力。

## 基本概念

SCX ANSI 中最核心的概念包括：

```text
Ansi                  ANSI 文本构建器
AnsiElement           ANSI 元素接口
AnsiColor             基本前景色
AnsiBackground        基本背景色
Ansi8BitColor         8 位前景色
Ansi8BitBackground    8 位背景色
AnsiStyle             文本样式
AnsiItem              单个文本片段和它对应的 ANSI 元素
AnsiHelper            ANSI 支持检测和 Windows ANSI 开启逻辑
```

它们之间的关系可以简单理解为：

```text
Ansi
    ↓
添加多个 AnsiItem
    ↓
每个 AnsiItem 保存 value + AnsiElement[]
    ↓
toString() 时拼接 ANSI 转义序列
    ↓
print() / println() 输出到 System.out
```

也就是说：

```text
Ansi 负责构建文本
AnsiElement 负责提供 ANSI code
AnsiItem 负责把文本片段包成 ANSI 片段
AnsiHelper 负责判断当前环境是否支持 ANSI
```

## 快速开始

最简单的用法是创建一个 `Ansi` 对象，然后添加带颜色的文本。

```java
import dev.scx.ansi.Ansi;

Ansi.ansi()
    .red("red text")
    .ln()
    .green("green text")
    .ln()
    .blue("blue text")
    .println();
```

也可以先构建字符串，再自己输出：

```java
import dev.scx.ansi.Ansi;

var text = Ansi.ansi()
    .red("error")
    .add(": ")
    .yellow("file not found")
    .toString();

System.out.println(text);
```

如果当前环境支持 ANSI，输出内容会带有颜色。

如果当前环境不支持 ANSI，输出内容会自动退化为普通文本。

## Ansi

`Ansi` 是整个库的入口类。

常见创建方式是：

```java
import dev.scx.ansi.Ansi;

var ansi = Ansi.ansi();
```

也可以直接使用构造方法：

```java
var ansi = new Ansi();
```

通常推荐使用静态方法：

```java
Ansi.ansi()
```

因为它更适合链式调用。

## 添加普通文本

`add(...)` 用于添加一个文本片段。

```java
Ansi.ansi()
    .add("hello")
    .add(" ")
    .add("world")
    .println();
```

如果不传入任何 ANSI 元素，那么它只是普通文本。

```java
Ansi.ansi()
    .add("plain text")
    .println();
```

`add(...)` 接收的是 `Object`，最终会通过 `StringBuilder` 拼接。

```java
Ansi.ansi()
    .add("count = ")
    .add(100)
    .add(", ok = ")
    .add(true)
    .println();
```

输出普通文本时大致是：

```text
count = 100, ok = true
```

如果传入 `null`，最终会按 `StringBuilder` 的规则输出为：

```text
null
```

## 添加带颜色的文本

`Ansi` 提供了一组便捷方法用于设置前景色。

```java
Ansi.ansi()
    .black("black")
    .red("red")
    .green("green")
    .yellow("yellow")
    .blue("blue")
    .magenta("magenta")
    .cyan("cyan")
    .white("white")
    .println();
```

也支持亮色版本：

```java
Ansi.ansi()
    .brightBlack("bright black")
    .brightRed("bright red")
    .brightGreen("bright green")
    .brightYellow("bright yellow")
    .brightBlue("bright blue")
    .brightMagenta("bright magenta")
    .brightCyan("bright cyan")
    .brightWhite("bright white")
    .println();
```

这些方法内部最终都会创建 `AnsiItem`，并把对应的 `AnsiColor` 加入该文本片段。

例如：

```java
Ansi.ansi().red("error");
```

等价于：

```java
import static dev.scx.ansi.AnsiColor.RED;

Ansi.ansi().add("error", RED);
```

## 基本前景色

`AnsiColor` 提供了标准前景色。

```java
import dev.scx.ansi.AnsiColor;
```

支持的值如下：

```text
DEFAULT          39

BLACK            30
RED              31
GREEN            32
YELLOW           33
BLUE             34
MAGENTA          35
CYAN             36
WHITE            37

BRIGHT_BLACK     90
BRIGHT_RED       91
BRIGHT_GREEN     92
BRIGHT_YELLOW    93
BRIGHT_BLUE      94
BRIGHT_MAGENTA   95
BRIGHT_CYAN      96
BRIGHT_WHITE     97
```

示例：

```java
import static dev.scx.ansi.AnsiColor.RED;
import static dev.scx.ansi.AnsiStyle.BOLD;

Ansi.ansi()
    .add("important", RED, BOLD)
    .println();
```

## 基本背景色

`AnsiBackground` 提供了标准背景色。

```java
import dev.scx.ansi.AnsiBackground;
```

支持的值如下：

```text
DEFAULT          49

BLACK            40
RED              41
GREEN            42
YELLOW           43
BLUE             44
MAGENTA          45
CYAN             46
WHITE            47

BRIGHT_BLACK     100
BRIGHT_RED       101
BRIGHT_GREEN     102
BRIGHT_YELLOW    103
BRIGHT_BLUE      104
BRIGHT_MAGENTA   105
BRIGHT_CYAN      106
BRIGHT_WHITE     107
```

示例：

```java
import static dev.scx.ansi.AnsiColor.WHITE;
import static dev.scx.ansi.AnsiBackground.RED;

Ansi.ansi()
    .add("error", WHITE, RED)
    .println();
```

生成的 ANSI 片段大致类似：

```text
ESC[37;41merrorESC[0m
```

其中：

```text
37    白色前景色
41    红色背景色
0     重置样式
```

## 8 位前景色

`Ansi8BitColor` 用于设置 8 位前景色。

8 位颜色取值范围是：

```text
0 ~ 255
```

示例：

```java
import dev.scx.ansi.Ansi8BitColor;

Ansi.ansi()
    .add("color 196", new Ansi8BitColor(196))
    .println();
```

它生成的 ANSI code 是：

```text
38;5;196
```

也就是说，8 位前景色的格式是：

```text
38;5;n
```

其中 `n` 是 `0 ~ 255` 之间的颜色编号。

如果颜色编号小于 `0` 或大于 `255`，会抛出 `IllegalArgumentException`。

```java
new Ansi8BitColor(-1);
```

或者：

```java
new Ansi8BitColor(256);
```

## 8 位背景色

`Ansi8BitBackground` 用于设置 8 位背景色。

8 位颜色取值范围同样是：

```text
0 ~ 255
```

示例：

```java
import dev.scx.ansi.Ansi8BitBackground;

Ansi.ansi()
    .add("background 21", new Ansi8BitBackground(21))
    .println();
```

它生成的 ANSI code 是：

```text
48;5;21
```

也就是说，8 位背景色的格式是：

```text
48;5;n
```

其中 `n` 是 `0 ~ 255` 之间的颜色编号。

如果颜色编号小于 `0` 或大于 `255`，会抛出 `IllegalArgumentException`。

## 文本样式

`AnsiStyle` 用于设置文本样式。

```java
import dev.scx.ansi.AnsiStyle;
```

支持的值如下：

```text
RESET              0
BOLD               1
FAINT              2
ITALIC             3
UNDERLINE          4
BLINK              5
REVERSE            7
HIDDEN             8
CROSSED_OUT        9
DOUBLE_UNDERLINE   21
OVERLINE           53
```

示例：

```java
import static dev.scx.ansi.AnsiColor.YELLOW;
import static dev.scx.ansi.AnsiStyle.BOLD;
import static dev.scx.ansi.AnsiStyle.UNDERLINE;

Ansi.ansi()
    .add("warning", YELLOW, BOLD, UNDERLINE)
    .println();
```

生成的 ANSI 片段大致类似：

```text
ESC[33;1;4mwarningESC[0m
```

## 组合颜色、背景色和样式

一个文本片段可以同时拥有：

```text
一个前景色
一个背景色
多个样式
```

示例：

```java
import static dev.scx.ansi.AnsiColor.BRIGHT_WHITE;
import static dev.scx.ansi.AnsiBackground.BLUE;
import static dev.scx.ansi.AnsiStyle.BOLD;
import static dev.scx.ansi.AnsiStyle.UNDERLINE;

Ansi.ansi()
    .add("title", BRIGHT_WHITE, BLUE, BOLD, UNDERLINE)
    .println();
```

也可以使用颜色便捷方法，再额外传入背景色和样式。

```java
import static dev.scx.ansi.AnsiBackground.BLUE;
import static dev.scx.ansi.AnsiStyle.BOLD;

Ansi.ansi()
    .brightWhite("title", BLUE, BOLD)
    .println();
```

## 元素过滤规则

SCX ANSI 会对每个文本片段传入的 `AnsiElement` 做过滤。

规则是：

```text
前景色只保留一个
背景色只保留一个
样式可以有多个，但会去重
```

例如：

```java
import static dev.scx.ansi.AnsiColor.RED;
import static dev.scx.ansi.AnsiColor.GREEN;
import static dev.scx.ansi.AnsiBackground.BLUE;
import static dev.scx.ansi.AnsiBackground.YELLOW;
import static dev.scx.ansi.AnsiStyle.BOLD;
import static dev.scx.ansi.AnsiStyle.UNDERLINE;

Ansi.ansi()
    .add("text", RED, GREEN, BLUE, YELLOW, BOLD, BOLD, UNDERLINE)
    .println();
```

最终会保留：

```text
GREEN
YELLOW
BOLD
UNDERLINE
```

原因是：

1. `RED` 和 `GREEN` 都是前景色，只保留最后一个前景色。
2. `BLUE` 和 `YELLOW` 都是背景色，只保留最后一个背景色。
3. `BOLD` 出现两次，但样式会去重。
4. `UNDERLINE` 会保留。

需要注意，样式去重使用的是 `EnumSet`，因此样式最终顺序由枚举顺序决定，而不是传入顺序。

## 颜色便捷方法的覆盖规则

颜色便捷方法会把对应颜色追加到参数列表后面，再执行过滤。

例如：

```java
import static dev.scx.ansi.AnsiColor.GREEN;

Ansi.ansi()
    .red("text", GREEN)
    .println();
```

虽然参数里传入了 `GREEN`，但是 `red(...)` 会额外追加 `RED`。

最终保留的是：

```text
RED
```

也就是说：

```java
red("text", GREEN)
```

大致等价于：

```java
add("text", GREEN, RED)
```

前景色只保留最后一个，所以结果是红色。

这可以让便捷方法拥有更高优先级。

## 换行

`ln()` 用于添加系统换行符。

```java
Ansi.ansi()
    .red("line 1")
    .ln()
    .green("line 2")
    .ln()
    .blue("line 3")
    .println();
```

`ln()` 内部添加的是：

```java
System.lineSeparator()
```

因此在不同平台上会使用对应系统换行符。

需要注意，`ln()` 会修改当前 `Ansi` 对象。

```java
var ansi = Ansi.ansi()
    .red("hello");

ansi.ln();
```

执行后，这个 `Ansi` 对象中已经包含了一个换行片段。

## print 和 println

`print()` 会把当前 `Ansi` 内容输出到 `System.out`。

```java
Ansi.ansi()
    .red("hello")
    .print();
```

`println()` 会先追加一个换行符，再输出。

```java
Ansi.ansi()
    .red("hello")
    .println();
```

它大致等价于：

```java
Ansi.ansi()
    .red("hello")
    .ln()
    .print();
```

需要注意，`println()` 会调用 `ln()`，所以它会修改当前 `Ansi` 对象。

例如：

```java
var ansi = Ansi.ansi()
    .red("hello");

ansi.println();
ansi.println();
```

第二次输出时，前一次 `println()` 添加的换行符仍然保留在对象中。

如果希望每次输出都是独立内容，推荐每次重新创建一个 `Ansi` 对象。

```java
Ansi.ansi().red("hello").println();

Ansi.ansi().green("world").println();
```

## 显式关闭 ANSI

`print(boolean useAnsi)` 和 `println(boolean useAnsi)` 可以显式控制是否使用 ANSI。

```java
Ansi.ansi()
    .red("error")
    .print(false);
```

当 `useAnsi` 为 `false` 时，会输出普通文本，不包含 ANSI 转义序列。

```java
Ansi.ansi()
    .red("error")
    .println(false);
```

这适合下面这些场景：

1. 输出到文件。
2. 输出到不支持 ANSI 的日志系统。
3. 单元测试中只想比较纯文本。
4. 根据用户配置关闭彩色输出。

## 生成字符串

`toString()` 默认会尝试生成带 ANSI 的字符串。

```java
var text = Ansi.ansi()
    .red("error")
    .toString();
```

它等价于：

```java
var text = Ansi.ansi()
    .red("error")
    .toString(true);
```

如果希望明确生成纯文本，可以使用：

```java
var text = Ansi.ansi()
    .red("error")
    .toString(false);
```

当 `useAnsi` 为 `false` 时，所有颜色、背景色和样式都会被忽略，只保留原始文本。

示例：

```java
var text = Ansi.ansi()
    .red("error")
    .add(": ")
    .yellow("file not found")
    .toString(false);
```

结果是：

```text
error: file not found
```

不会包含任何 ANSI 控制字符。

## ANSI 支持检测

SCX ANSI 会在 `Ansi` 类加载时检测当前环境是否支持 ANSI。

内部逻辑可以理解为：

```text
如果不是 Windows，认为支持 ANSI
如果是 Windows，但不是 Windows 10/11，认为不支持 ANSI
如果是 Windows 10/11，尝试开启虚拟终端处理
开启成功，则认为支持 ANSI
开启失败，则认为不支持 ANSI
```

因此最终是否输出 ANSI，需要同时满足两个条件：

```text
当前系统支持 ANSI
调用时 useAnsi 为 true
```

也就是说：

```java
ansi.toString(true);
```

并不一定会强行输出 ANSI。

如果系统检测结果是不支持 ANSI，那么即使传入 `true`，也会输出普通文本。

## Windows 10/11 支持

Windows 10/11 的控制台支持 ANSI，但通常需要开启虚拟终端处理。

SCX ANSI 会尝试调用 `Kernel32`：

```text
GetStdHandle
GetConsoleMode
SetConsoleMode
```

并给标准输出添加：

```text
ENABLE_VIRTUAL_TERMINAL_PROCESSING
```

对应值是：

```text
0x0004
```

这个逻辑通过 `scx-ffi` 调用 Windows API 完成。

如果调用失败，SCX ANSI 会认为当前环境不支持 ANSI，并自动退化为普通文本。

## AnsiElement

`AnsiElement` 是所有 ANSI 元素的统一接口。

接口非常简单：

```java
public interface AnsiElement {

    String code();

}
```

只要实现了 `AnsiElement`，就可以参与 ANSI 片段构造。

例如，`AnsiColor.RED` 的 `code()` 返回：

```text
31
```

`Ansi8BitColor(196)` 的 `code()` 返回：

```text
38;5;196
```

最终多个 code 会用分号拼接。

```text
31;1;4
```

然后包成完整的 SGR 序列：

```text
ESC[31;1;4m
```

## 自定义 ANSI 元素

因为 `AnsiElement` 只是一个接口，所以可以自定义 ANSI 元素。

示例：

```java
import dev.scx.ansi.AnsiElement;

public enum MyAnsiStyle implements AnsiElement {

    FRAMED("51"),
    ENCIRCLED("52");

    private final String code;

    MyAnsiStyle(String code) {
        this.code = code;
    }

    @Override
    public String code() {
        return this.code;
    }

}
```

使用：

```java
import static dev.scx.ansi.AnsiColor.CYAN;
import static dev.scx.ansi.AnsiStyle.BOLD;

Ansi.ansi()
    .add("custom", CYAN, BOLD, MyAnsiStyle.FRAMED)
    .println();
```

需要注意，内置过滤逻辑只会特殊识别下面这些类型：

```text
AnsiColor
Ansi8BitColor
AnsiBackground
Ansi8BitBackground
AnsiStyle
```

自定义 `AnsiElement` 不属于内置前景色、背景色或样式集合，因此不会参与颜色覆盖和样式去重。

如果自定义元素本身也是前景色或背景色，需要调用者自己避免冲突。

## ANSI 片段结构

每个带样式的文本片段都会生成下面这种结构：

```text
ESC[code1;code2;code3m文本ESC[0m
```

例如：

```java
import static dev.scx.ansi.AnsiColor.RED;
import static dev.scx.ansi.AnsiStyle.BOLD;

var text = Ansi.ansi()
    .add("error", RED, BOLD)
    .toString();
```

在支持 ANSI 的环境中，它大致会生成：

```text
ESC[31;1merrorESC[0m
```

其中：

```text
ESC[       ANSI 起始
31         红色前景色
1          加粗
m          ANSI 结束
error      文本内容
ESC[0m     重置样式
```

SCX ANSI 会在每个带样式的片段后面自动追加 `RESET`。

这样可以避免样式泄漏到后续文本。

## 样式不会泄漏

因为每个带 ANSI 的 `AnsiItem` 都会在文本后面添加 `RESET`，所以样式只影响当前片段。

示例：

```java
Ansi.ansi()
    .red("error")
    .add(" normal text")
    .println();
```

在支持 ANSI 的环境中，只有 `error` 是红色。

后面的 ` normal text` 不会继承红色。

内部生成大致是：

```text
ESC[31merrorESC[0m normal text
```

## 生成纯文本

当禁用 ANSI 或当前系统不支持 ANSI 时，`AnsiItem` 会直接输出原始值。

示例：

```java
var text = Ansi.ansi()
    .red("error")
    .add(": ")
    .yellow("file not found")
    .toString(false);
```

结果：

```text
error: file not found
```

不会生成：

```text
ESC[31m
ESC[33m
ESC[0m
```

## 彩色日志示例

可以把 SCX ANSI 用在简单日志输出中。

```java
import dev.scx.ansi.Ansi;

public final class Log {

    public static void info(String message) {
        Ansi.ansi()
            .brightBlue("[INFO] ")
            .add(message)
            .println();
    }

    public static void warn(String message) {
        Ansi.ansi()
            .brightYellow("[WARN] ")
            .add(message)
            .println();
    }

    public static void error(String message) {
        Ansi.ansi()
            .brightRed("[ERROR] ")
            .add(message)
            .println();
    }

}
```

使用：

```java
Log.info("server started");

Log.warn("config file not found, use default config");

Log.error("port already in use");
```

如果希望日志文件中不出现 ANSI 控制字符，可以自己控制：

```java
var message = Ansi.ansi()
    .brightRed("[ERROR] ")
    .add("port already in use")
    .toString(false);
```

## 表格输出示例

SCX ANSI 也可以用于构造简单的控制台表格。

```java
import dev.scx.ansi.Ansi;

import static dev.scx.ansi.AnsiBackground.BLUE;
import static dev.scx.ansi.AnsiColor.BRIGHT_WHITE;
import static dev.scx.ansi.AnsiStyle.BOLD;

Ansi.ansi()
    .add(" ID ", BRIGHT_WHITE, BLUE, BOLD)
    .add(" Name ", BRIGHT_WHITE, BLUE, BOLD)
    .add(" Status ", BRIGHT_WHITE, BLUE, BOLD)
    .ln()
    .add(" 1  ")
    .add(" Tom  ")
    .green(" OK ")
    .ln()
    .add(" 2  ")
    .add(" Jerry")
    .red(" FAIL ")
    .println();
```

## 8 位颜色表测试

测试代码中也展示了如何输出完整 8 位颜色组合。

```java
import dev.scx.ansi.Ansi;
import dev.scx.ansi.Ansi8BitBackground;
import dev.scx.ansi.Ansi8BitColor;

import static dev.scx.ansi.AnsiStyle.BOLD;
import static dev.scx.ansi.AnsiStyle.ITALIC;
import static dev.scx.ansi.AnsiStyle.UNDERLINE;

var ansi = Ansi.ansi();

for (int i = 0; i < 256; i = i + 1) {
    for (int j = 0; j < 256; j = j + 1) {
        ansi.add(
            "0",
            new Ansi8BitColor(i),
            new Ansi8BitBackground(j),
            BOLD,
            ITALIC,
            UNDERLINE
        );
    }
    ansi.ln();
}

ansi.println();
```

这个示例会生成大量输出，通常只适合本地测试终端颜色支持情况。

## 样式测试

可以输出所有内置样式：

```java
import dev.scx.ansi.Ansi;
import dev.scx.ansi.AnsiStyle;

var ansi = Ansi.ansi();

for (var style : AnsiStyle.values()) {
    for (int i = 0; i < 10; i = i + 1) {
        ansi.add(style.name() + " ", style);
    }
    ansi.ln();
}

ansi.println();
```

需要注意，不同终端对某些样式的支持并不一致。

例如：

```text
BLINK
HIDDEN
DOUBLE_UNDERLINE
OVERLINE
```

这些样式可能在某些终端中没有效果，或者显示效果和预期不同。

这属于终端支持差异，不是 SCX ANSI 的逻辑错误。

## 与 System.out 的关系

`print()` 和 `println()` 最终使用的是：

```java
System.out.print(...)
```

也就是说，SCX ANSI 不接管输出流。

如果你希望输出到其它地方，可以使用 `toString()`：

```java
var text = Ansi.ansi()
    .red("error")
    .toString();

writer.write(text);
```

或者生成纯文本：

```java
var plainText = Ansi.ansi()
    .red("error")
    .toString(false);

writer.write(plainText);
```

## 线程安全

`Ansi` 是一个可变构建器。

内部保存的是一个 `ArrayList`。

因此它不应该在多个线程之间共享修改。

推荐用法是：

```java
Ansi.ansi()
    .green("success")
    .println();
```

也就是每次构建一段文本时创建一个新的 `Ansi` 对象。

如果确实需要跨线程共享，应由调用方自己保证同步。

## 完整示例

下面是一个完整示例。

```java
import dev.scx.ansi.Ansi;
import dev.scx.ansi.Ansi8BitBackground;
import dev.scx.ansi.Ansi8BitColor;

import static dev.scx.ansi.AnsiBackground.BLUE;
import static dev.scx.ansi.AnsiColor.BRIGHT_WHITE;
import static dev.scx.ansi.AnsiColor.CYAN;
import static dev.scx.ansi.AnsiColor.GREEN;
import static dev.scx.ansi.AnsiColor.RED;
import static dev.scx.ansi.AnsiStyle.BOLD;
import static dev.scx.ansi.AnsiStyle.UNDERLINE;

public class AnsiDemo {

    public static void main(String[] args) {
        Ansi.ansi()
            .add(" SCX ANSI ", BRIGHT_WHITE, BLUE, BOLD)
            .ln()
            .green("success")
            .add(" ")
            .red("error")
            .add(" ")
            .add("custom 8 bit", new Ansi8BitColor(196), new Ansi8BitBackground(21), BOLD)
            .ln()
            .add("underlined cyan", CYAN, UNDERLINE)
            .println();
    }

}
```

如果当前终端支持 ANSI，输出会带有颜色和样式。

如果当前终端不支持 ANSI，输出会退化为普通文本。

## 设计说明

### 1. SCX ANSI 只是 ANSI 文本构建器

SCX ANSI 不负责模拟终端，也不负责解析 ANSI。

它只负责生成类似下面这种文本：

```text
ESC[31merrorESC[0m
```

然后交给控制台或其它输出目标处理。

### 2. 每个文本片段独立包裹

每个带样式的片段都会单独生成开始序列和重置序列。

例如：

```java
Ansi.ansi()
    .red("A")
    .green("B")
    .blue("C")
    .println();
```

大致会生成：

```text
ESC[31mAESC[0mESC[32mBESC[0mESC[34mCESC[0m
```

这样可以让每个片段互不影响。

### 3. 不支持 ANSI 时自动降级

SCX ANSI 在不支持 ANSI 的环境中不会抛异常，也不会强行输出控制字符。

它会直接输出普通文本。

这使得同一段代码可以同时用于：

```text
支持 ANSI 的终端
不支持 ANSI 的终端
日志文件
测试输出
CI 环境
```

### 4. Windows 支持是尽力而为

在 Windows 10/11 上，SCX ANSI 会尝试开启虚拟终端处理。

如果成功，就输出 ANSI。

如果失败，就退化为普通文本。

这意味着在某些受限环境中，即使系统版本是 Windows 10/11，也可能因为 API 调用失败而不输出 ANSI。

### 5. 前景色和背景色只保留最后一个

一个 ANSI 片段中同时设置多个前景色没有意义。

因此 SCX ANSI 会自动过滤，只保留最后一个前景色。

背景色同理，只保留最后一个背景色。

这样可以避免生成无效或冗余的 ANSI code。

### 6. 样式会去重

样式可以组合使用，例如：

```text
BOLD
UNDERLINE
ITALIC
```

但是重复样式没有意义。

因此 SCX ANSI 会对样式去重。

### 7. 不负责终端兼容性差异

不同终端对 ANSI 的支持不完全一致。

SCX ANSI 只生成标准 ANSI code。

具体显示效果取决于：

```text
操作系统
终端程序
字体
控制台配置
是否支持对应 ANSI 样式
```

例如某些终端可能不支持闪烁、双下划线或上划线。

## 常见问题

### SCX ANSI 是日志框架吗？

不是。

它只是 ANSI 文本构建器。你可以把它用于日志输出，但它本身不提供日志级别、日志输出目标、日志格式化或日志配置能力。

### SCX ANSI 会自动检测终端是否支持颜色吗？

会做基础检测。

非 Windows 系统默认认为支持 ANSI。

Windows 10/11 会尝试开启虚拟终端处理。

旧版 Windows 默认认为不支持 ANSI。

### `toString(true)` 一定会生成 ANSI 吗？

不一定。

`toString(true)` 只是表示调用者希望使用 ANSI。

最终还要看当前系统检测结果是否支持 ANSI。

如果系统不支持 ANSI，仍然会输出普通文本。

### 如何强制生成纯文本？

使用：

```java
toString(false)
```

或者：

```java
print(false)
```

或者：

```java
println(false)
```

### 如何输出红色文字？

```java
Ansi.ansi()
    .red("error")
    .println();
```

或者：

```java
import static dev.scx.ansi.AnsiColor.RED;

Ansi.ansi()
    .add("error", RED)
    .println();
```

### 如何同时设置前景色、背景色和加粗？

```java
import static dev.scx.ansi.AnsiColor.WHITE;
import static dev.scx.ansi.AnsiBackground.RED;
import static dev.scx.ansi.AnsiStyle.BOLD;

Ansi.ansi()
    .add("error", WHITE, RED, BOLD)
    .println();
```

### 如何使用 256 色？

使用 `Ansi8BitColor` 和 `Ansi8BitBackground`。

```java
Ansi.ansi()
    .add("text", new Ansi8BitColor(196), new Ansi8BitBackground(21))
    .println();
```

### 8 位颜色编号范围是多少？

范围是：

```text
0 ~ 255
```

小于 `0` 或大于 `255` 会抛出 `IllegalArgumentException`。

### 一个片段可以设置多个颜色吗？

可以传入多个，但最终只保留一个前景色和一个背景色。

前景色保留最后一个。

背景色保留最后一个。

### 样式可以叠加吗？

可以。

例如：

```java
Ansi.ansi()
    .add("text", BOLD, UNDERLINE, ITALIC)
    .println();
```

### 重复传入同一个样式会怎样？

会被去重。

```java
Ansi.ansi()
    .add("text", BOLD, BOLD, BOLD)
    .println();
```

最终只会保留一个 `BOLD`。

### 为什么某些样式没有效果？

因为终端可能不支持该样式。

例如：

```text
BLINK
HIDDEN
DOUBLE_UNDERLINE
OVERLINE
```

这些样式在不同终端中的表现可能不同。

### `println()` 会清空内容吗？

不会。

`Ansi` 是可变构建器，`println()` 只是追加换行并输出当前内容，不会清空内部 items。

如果要输出新的内容，推荐重新创建：

```java
Ansi.ansi().red("first").println();

Ansi.ansi().green("second").println();
```

### `ln()` 使用什么换行符？

使用：

```java
System.lineSeparator()
```

### 可以输出到文件吗？

可以先使用 `toString(false)` 生成纯文本。

```java
var text = Ansi.ansi()
    .red("error")
    .toString(false);
```

如果你希望文件中保留 ANSI 控制字符，也可以使用 `toString(true)`，但最终是否生成 ANSI 仍取决于系统支持检测。

### 可以自定义 ANSI code 吗？

可以实现 `AnsiElement`。

```java
public enum MyElement implements AnsiElement {

    FRAMED("51");

    private final String code;

    MyElement(String code) {
        this.code = code;
    }

    @Override
    public String code() {
        return this.code;
    }

}
```

然后：

```java
Ansi.ansi()
    .add("text", MyElement.FRAMED)
    .println();
```

### SCX ANSI 会解析已有 ANSI 字符串吗？

不会。

它只负责生成 ANSI 文本，不负责解析、清理或转换已有 ANSI 字符串。

### SCX ANSI 会影响全局控制台状态吗？

普通输出不会长期影响全局状态，因为每个带样式片段后都会自动追加 `RESET`。

但在 Windows 10/11 上，类加载时可能会尝试开启控制台的虚拟终端处理，这是对当前控制台模式的一次设置。

### 为什么 Windows 上没有颜色？

可能原因包括：

1. Windows 版本不是 10/11。
2. 当前输出不是标准控制台。
3. `Kernel32` 调用失败。
4. 当前终端不支持虚拟终端处理。
5. 输出被 IDE、CI 或日志系统截获。
6. 调用时使用了 `toString(false)`、`print(false)` 或 `println(false)`。

### 为什么日志里出现了奇怪的字符？

那通常是 ANSI 转义序列。

如果日志系统不支持 ANSI，应该使用：

```java
toString(false)
```

生成纯文本。