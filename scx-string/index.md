# SCX String

SCX String 是一个字符串工具库。

它提供 `ScxString`，用于补充 JDK `String` 中一些常用但写起来比较重复的操作，例如忽略大小写的前缀判断、后缀判断、查找、包含判断，以及按照 Unicode code point 处理字符串。

SCX String 本身不是模板引擎，也不是文本解析框架。它只是对常见字符串操作做了一层简单封装，让代码更直观、更容易复用。

当前版本为 `0.1.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-string</artifactId>
    <version>0.1.0</version>
</dependency>
```

## 基本概念

SCX String 当前只有一个核心类：

```text
ScxString    字符串工具类
```

它提供的能力可以分为两类：

```text
IgnoreCase    忽略大小写的字符串判断和查找
CodePoint     按 Unicode code point 处理字符串
```

IgnoreCase 相关方法包括：

```text
startsWithIgnoreCase
endsWithIgnoreCase
indexOfIgnoreCase
lastIndexOfIgnoreCase
containsIgnoreCase
```

CodePoint 相关方法包括：

```text
splitByCodePoint
lengthByCodePoint
charAtByCodePoint
substringByCodePoint
```

## 快速开始

判断字符串是否以某个前缀开始，忽略大小写：

```java
import dev.scx.string.ScxString;

var result = ScxString.startsWithIgnoreCase("Hello World", "hello");

System.out.println(result);
```

输出：

```text
true
```

查找字符串，忽略大小写：

```java
var index = ScxString.indexOfIgnoreCase("xxAbCxx", "abc");

System.out.println(index);
```

输出：

```text
2
```

判断是否包含某个字符串，忽略大小写：

```java
var result = ScxString.containsIgnoreCase("你好ABC", "abc");

System.out.println(result);
```

输出：

```text
true
```

按照 code point 计算长度：

```java
var length = ScxString.lengthByCodePoint("a😀b");

System.out.println(length);
```

输出：

```text
3
```

按照 code point 拆分字符串：

```java
var array = ScxString.splitByCodePoint("a😀b");

System.out.println(java.util.Arrays.toString(array));
```

输出：

```text
[a, 😀, b]
```

## 忽略大小写

SCX String 中的 IgnoreCase 方法用于补充 `String` 自带方法。

JDK 中已经有：

```java
string.startsWith(prefix)

string.endsWith(suffix)

string.indexOf(target)

string.lastIndexOf(target)

string.contains(target)
```

但是这些方法默认都是区分大小写的。

SCX String 提供了对应的忽略大小写版本：

```java
ScxString.startsWithIgnoreCase(string, prefix)

ScxString.endsWithIgnoreCase(string, suffix)

ScxString.indexOfIgnoreCase(string, target)

ScxString.lastIndexOfIgnoreCase(string, target)

ScxString.containsIgnoreCase(string, target)
```

例如：

```java
ScxString.containsIgnoreCase("Hello World", "hello")
```

结果为：

```text
true
```

## startsWithIgnoreCase

`startsWithIgnoreCase(...)` 用于判断字符串是否以指定前缀开始，忽略大小写。

```java
boolean startsWithIgnoreCase(String string, String prefix)
```

示例：

```java
var result = ScxString.startsWithIgnoreCase("aBc123", "ABC");

System.out.println(result);
```

输出：

```text
true
```

空前缀会返回 `true`：

```java
var result = ScxString.startsWithIgnoreCase("abc123", "");

System.out.println(result);
```

输出：

```text
true
```

如果前缀比原字符串更长，会返回 `false`：

```java
var result = ScxString.startsWithIgnoreCase("abc", "abcd");

System.out.println(result);
```

输出：

```text
false
```

### 指定偏移量

也可以从指定偏移量开始判断：

```java
boolean startsWithIgnoreCase(String string, String prefix, int toffset)
```

示例：

```java
var result = ScxString.startsWithIgnoreCase("xxxAbC123", "abc", 3);

System.out.println(result);
```

输出：

```text
true
```

这里表示从 `"xxxAbC123"` 的索引 `3` 开始判断是否匹配 `"abc"`。

如果偏移量不匹配，会返回 `false`：

```java
var result = ScxString.startsWithIgnoreCase("xxxAbC123", "abc", 2);

System.out.println(result);
```

输出：

```text
false
```

## endsWithIgnoreCase

`endsWithIgnoreCase(...)` 用于判断字符串是否以指定后缀结束，忽略大小写。

```java
boolean endsWithIgnoreCase(String string, String suffix)
```

示例：

```java
var result = ScxString.endsWithIgnoreCase("123aBc", "ABC");

System.out.println(result);
```

输出：

```text
true
```

空后缀会返回 `true`：

```java
var result = ScxString.endsWithIgnoreCase("abc123", "");

System.out.println(result);
```

输出：

```text
true
```

如果后缀比原字符串更长，会返回 `false`：

```java
var result = ScxString.endsWithIgnoreCase("abc", "zabc");

System.out.println(result);
```

输出：

```text
false
```

## indexOfIgnoreCase

`indexOfIgnoreCase(...)` 用于从前往后查找目标字符串，忽略大小写。

```java
int indexOfIgnoreCase(String string, String target)
```

示例：

```java
var index = ScxString.indexOfIgnoreCase("xxAbCxx", "abc");

System.out.println(index);
```

输出：

```text
2
```

如果找不到，会返回 `-1`：

```java
var index = ScxString.indexOfIgnoreCase("abc", "d");

System.out.println(index);
```

输出：

```text
-1
```

如果目标字符串为空字符串，会返回开始查找的位置。

```java
var index = ScxString.indexOfIgnoreCase("abc", "");

System.out.println(index);
```

输出：

```text
0
```

### 从指定位置开始查找

```java
int indexOfIgnoreCase(String string, String target, int fromIndex)
```

示例：

```java
var s = "abcABCabc";

System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 0));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 1));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 4));
```

输出：

```text
0
3
6
```

`fromIndex` 的处理方式和 `String#indexOf` 类似：

```text
fromIndex < 0              按 0 处理
fromIndex > string.length  按 string.length 处理
```

例如：

```java
var s = "abcABCabc";

System.out.println(ScxString.indexOfIgnoreCase(s, "abc", -99));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 99));
```

输出：

```text
0
-1
```

查找空字符串时，会返回处理后的 `fromIndex`：

```java
var s = "abcABCabc";

System.out.println(ScxString.indexOfIgnoreCase(s, "", -99));
System.out.println(ScxString.indexOfIgnoreCase(s, "", 2));
System.out.println(ScxString.indexOfIgnoreCase(s, "", 99));
```

输出：

```text
0
2
9
```

### 在指定范围内查找

```java
int indexOfIgnoreCase(String string, String target, int beginIndex, int endIndex)
```

这里的范围是：

```text
[beginIndex, endIndex)
```

也就是包含 `beginIndex`，不包含 `endIndex`。

示例：

```java
var s = "xxAbCxxABCxx";

System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 0, s.length()));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 3, s.length()));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 0, 5));
System.out.println(ScxString.indexOfIgnoreCase(s, "abc", 0, 4));
```

输出：

```text
2
7
2
-1
```

如果目标字符串为空，会返回 `beginIndex`：

```java
var s = "xxAbCxxABCxx";

System.out.println(ScxString.indexOfIgnoreCase(s, "", 4, 4));
System.out.println(ScxString.indexOfIgnoreCase(s, "", 4, 8));
```

输出：

```text
4
4
```

范围必须合法：

```text
beginIndex >= 0
beginIndex <= endIndex
endIndex <= string.length()
```

否则会抛出 `StringIndexOutOfBoundsException`。

## lastIndexOfIgnoreCase

`lastIndexOfIgnoreCase(...)` 用于从后往前查找目标字符串，忽略大小写。

```java
int lastIndexOfIgnoreCase(String string, String target)
```

示例：

```java
var index = ScxString.lastIndexOfIgnoreCase("abcABCabc", "abc");

System.out.println(index);
```

输出：

```text
6
```

如果找不到，会返回 `-1`：

```java
var index = ScxString.lastIndexOfIgnoreCase("abc", "d");

System.out.println(index);
```

输出：

```text
-1
```

如果目标字符串为空字符串，会返回字符串长度：

```java
var index = ScxString.lastIndexOfIgnoreCase("abc", "");

System.out.println(index);
```

输出：

```text
3
```

### 从指定位置向前查找

```java
int lastIndexOfIgnoreCase(String string, String target, int fromIndex)
```

示例：

```java
var s = "abcABCabc";

System.out.println(ScxString.lastIndexOfIgnoreCase(s, "abc", 99));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "abc", 6));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "abc", 5));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "abc", 2));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "abc", -1));
```

输出：

```text
6
6
3
0
-1
```

`fromIndex` 的处理方式如下：

```text
fromIndex < 0                         返回 -1
fromIndex > 最大可匹配位置             按最大可匹配位置处理
```

查找空字符串时，会返回处理后的 `fromIndex`：

```java
var s = "abcABCabc";

System.out.println(ScxString.lastIndexOfIgnoreCase(s, "", 99));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "", 1));
System.out.println(ScxString.lastIndexOfIgnoreCase(s, "", -1));
```

输出：

```text
9
1
-1
```

## containsIgnoreCase

`containsIgnoreCase(...)` 用于判断字符串中是否包含目标字符串，忽略大小写。

```java
boolean containsIgnoreCase(String string, String target)
```

它内部等价于：

```java
ScxString.indexOfIgnoreCase(string, target) >= 0
```

示例：

```java
var result = ScxString.containsIgnoreCase("xxAbCxx", "abc");

System.out.println(result);
```

输出：

```text
true
```

中文、数字、符号等不涉及大小写的字符会保持正常比较：

```java
var result = ScxString.containsIgnoreCase("你好ABC", "abc");

System.out.println(result);
```

输出：

```text
true
```

空字符串总是可以被包含：

```java
var result = ScxString.containsIgnoreCase("abc", "");

System.out.println(result);
```

输出：

```text
true
```

如果找不到目标字符串，会返回 `false`：

```java
var result = ScxString.containsIgnoreCase("xxAbCxx", "777");

System.out.println(result);
```

输出：

```text
false
```

## Code Point

Java 的 `String` 底层使用 UTF-16。

这意味着 `String#length()` 返回的是 UTF-16 code unit 的数量，而不一定是日常理解中的“字符数量”。

例如：

```java
System.out.println("a😀b".length());
```

输出：

```text
4
```

因为 `😀` 在 UTF-16 中会占用两个 `char`。

如果希望把 `😀` 这类字符按一个单位处理，可以使用 SCX String 中的 CodePoint 相关方法：

```text
splitByCodePoint
lengthByCodePoint
charAtByCodePoint
substringByCodePoint
```

需要注意：

```text
Code point 不完全等同于用户感知上的字符
```

例如带组合符号的字符可能由多个 code point 组成。SCX String 这里处理的是 Unicode code point，不是完整的 grapheme cluster。

## splitByCodePoint

`splitByCodePoint(...)` 用于把字符串按 code point 拆成 `String[]`。

```java
String[] splitByCodePoint(String string)
```

示例：

```java
import dev.scx.string.ScxString;

import java.util.Arrays;

var array = ScxString.splitByCodePoint("a😀b");

System.out.println(Arrays.toString(array));
```

输出：

```text
[a, 😀, b]
```

更复杂的例子：

```java
var array = ScxString.splitByCodePoint("🐷😂🤣😅😍😡123你好");

System.out.println(Arrays.toString(array));
```

输出：

```text
[🐷, 😂, 🤣, 😅, 😍, 😡, 1, 2, 3, 你, 好]
```

空字符串会返回空数组：

```java
var array = ScxString.splitByCodePoint("");

System.out.println(array.length);
```

输出：

```text
0
```

## lengthByCodePoint

`lengthByCodePoint(...)` 用于按 code point 计算字符串长度。

```java
int lengthByCodePoint(String string)
```

示例：

```java
System.out.println(ScxString.lengthByCodePoint(""));
System.out.println(ScxString.lengthByCodePoint("abc"));
System.out.println(ScxString.lengthByCodePoint("a😀b"));
System.out.println(ScxString.lengthByCodePoint("🐷😂🤣😅😍😡123你好"));
```

输出：

```text
0
3
3
11
```

对比 `String#length()`：

```java
System.out.println("a😀b".length());
System.out.println(ScxString.lengthByCodePoint("a😀b"));
```

输出：

```text
4
3
```

## charAtByCodePoint

`charAtByCodePoint(...)` 用于按照 code point 索引获取指定位置的字符。

```java
String charAtByCodePoint(String string, int index)
```

返回值是 `String`，不是 `char`。

这是因为一个 code point 不一定能用一个 Java `char` 表示，例如 emoji。

示例：

```java
var s = "a😀b你";

System.out.println(ScxString.charAtByCodePoint(s, 0));
System.out.println(ScxString.charAtByCodePoint(s, 1));
System.out.println(ScxString.charAtByCodePoint(s, 2));
System.out.println(ScxString.charAtByCodePoint(s, 3));
```

输出：

```text
a
😀
b
你
```

如果索引越界，会抛出 `IndexOutOfBoundsException`：

```java
ScxString.charAtByCodePoint("a😀b你", -1);

ScxString.charAtByCodePoint("a😀b你", 4);

ScxString.charAtByCodePoint("", 0);
```

## substringByCodePoint

`substringByCodePoint(...)` 用于按照 code point 索引截取字符串。

```java
String substringByCodePoint(String string, int beginIndex)
```

示例：

```java
var s = "a😀b你c";

System.out.println(ScxString.substringByCodePoint(s, 0));
System.out.println(ScxString.substringByCodePoint(s, 1));
System.out.println(ScxString.substringByCodePoint(s, 3));
System.out.println(ScxString.substringByCodePoint(s, 5));
```

输出：

```text
a😀b你c
😀b你c
你c

```

也可以指定结束索引：

```java
String substringByCodePoint(String string, int beginIndex, int endIndex)
```

这里的范围是：

```text
[beginIndex, endIndex)
```

示例：

```java
var s = "a😀b你c";

System.out.println(ScxString.substringByCodePoint(s, 0, 2));
System.out.println(ScxString.substringByCodePoint(s, 1, 3));
System.out.println(ScxString.substringByCodePoint(s, 3, 5));
System.out.println(ScxString.substringByCodePoint(s, 2, 2));
```

输出：

```text
a😀
😀b
你c

```

如果索引越界，或者范围不合法，会抛出 `IndexOutOfBoundsException`：

```java
ScxString.substringByCodePoint("a😀b你c", -1);

ScxString.substringByCodePoint("a😀b你c", 6);

ScxString.substringByCodePoint("a😀b你c", 3, 2);

ScxString.substringByCodePoint("a😀b你c", 0, 6);
```

## null 处理

SCX String 不做 `null` 安全处理。

也就是说，传入 `null` 时通常会直接抛出 `NullPointerException`。

例如：

```java
ScxString.startsWithIgnoreCase(null, "abc");

ScxString.startsWithIgnoreCase("abc", null);

ScxString.containsIgnoreCase(null, "abc");

ScxString.lengthByCodePoint(null);
```

这和 JDK 中很多字符串方法的行为保持一致。

如果业务中允许 `null`，应该在调用前自行处理：

```java
if (value != null && ScxString.containsIgnoreCase(value, "abc")) {
    // ...
}
```

或者：

```java
var result = value != null && ScxString.startsWithIgnoreCase(value, "abc");
```

## 空字符串行为

SCX String 对空字符串的处理遵循 JDK 字符串方法的常见语义。

常见规则如下：

```text
startsWithIgnoreCase("abc", "")             true
endsWithIgnoreCase("abc", "")               true
indexOfIgnoreCase("abc", "")                0
indexOfIgnoreCase("abc", "", 2)             2
lastIndexOfIgnoreCase("abc", "")            3
containsIgnoreCase("abc", "")               true
splitByCodePoint("")                        []
lengthByCodePoint("")                       0
```

## API 一览

### IgnoreCase

```java
public static boolean startsWithIgnoreCase(String string, String prefix)

public static boolean startsWithIgnoreCase(String string, String prefix, int toffset)

public static boolean endsWithIgnoreCase(String string, String suffix)

public static int indexOfIgnoreCase(String string, String target)

public static int indexOfIgnoreCase(String string, String target, int fromIndex)

public static int indexOfIgnoreCase(String string, String target, int beginIndex, int endIndex)

public static int lastIndexOfIgnoreCase(String string, String target)

public static int lastIndexOfIgnoreCase(String string, String target, int fromIndex)

public static boolean containsIgnoreCase(String string, String target)
```

### CodePoint

```java
public static String[] splitByCodePoint(String string)

public static int lengthByCodePoint(String string)

public static String charAtByCodePoint(String string, int index)

public static String substringByCodePoint(String string, int beginIndex)

public static String substringByCodePoint(String string, int beginIndex, int endIndex)
```

## 适用场景

SCX String 适合用于：

```text
不区分大小写地判断前缀
不区分大小写地判断后缀
不区分大小写地查找字符串
不区分大小写地判断包含关系
处理包含 emoji 的字符串长度
按 code point 拆分字符串
按 code point 截取字符串
按 code point 获取指定位置字符
```

例如，在处理 HTTP Header、配置项、命令参数、文件扩展名、用户输入搜索关键字时，经常需要忽略大小写：

```java
if (ScxString.endsWithIgnoreCase(fileName, ".jpg")) {
    // 处理 jpg 文件
}
```

在处理 emoji 或其它非 BMP 字符时，可以使用 code point 相关方法避免直接按 UTF-16 `char` 切分：

```java
var text = "a😀b";

var chars = ScxString.splitByCodePoint(text);
```

结果是：

```text
[a, 😀, b]
```

而不是把 `😀` 拆成两个 Java `char`。