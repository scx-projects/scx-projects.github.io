# SCX Random

SCX Random 是一个轻量随机工具库。

它提供 `ScxRandom`，用于生成随机 UUID、随机数字、随机布尔值、随机字节数组和随机字符串。

SCX Random 本身不是加密随机库，也不是复杂随机分布库。它主要是对 JDK 中常用随机能力的一层简单封装，让日常业务中生成随机值、随机字符串和随机 token 更方便。

[GitHub](https://github.com/scx-projects/scx-random)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-random</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Random 中最核心的概念包括：

```text
ScxRandom       随机工具类
CharPool        char 字符池
CodePointPool   Unicode code point 字符池
StringPool      字符串片段池
```

它们之间的关系可以简单理解为：

```text
ScxRandom
    ↓
randomInt / randomLong / randomFloat / randomDouble / randomBoolean
    ↓
ThreadLocalRandom.current()

ScxRandom
    ↓
randomUUID
    ↓
UUID.randomUUID()

ScxRandom
    ↓
randomString(count, pool)
    ↓
从 CharPool / CodePointPool / StringPool 中随机抽取
```

也就是说：

```text
ScxRandom 负责生成随机值
CharPool 负责提供 char 候选字符
CodePointPool 负责提供 Unicode code point 候选字符
StringPool 负责提供字符串片段候选项
```

## 快速开始

生成随机整数：

```java
import dev.scx.random.ScxRandom;

int value = ScxRandom.randomInt(100);

System.out.println(value);
```

结果范围是：

```text
0 <= value < 100
```

生成指定范围内的随机整数：

```java
int value = ScxRandom.randomInt(10, 20);
```

结果范围是：

```text
10 <= value < 20
```

生成随机字符串：

```java
import dev.scx.random.ScxRandom;

String token = ScxRandom.randomString(
    16,
    ScxRandom.NUMBER_AND_LETTER
);

System.out.println(token);
```

可能输出：

```text
A8d2Klm9Zq01xYpB
```

生成随机 UUID：

```java
String uuid = ScxRandom.randomUUID();

System.out.println(uuid);
```

可能输出：

```text
f47ac10b-58cc-4372-a567-0e02b2c3d479
```

生成随机字节数组：

```java
byte[] bytes = ScxRandom.randomBytes(32);
```

## ScxRandom

`ScxRandom` 是整个库的入口类。

它提供的能力可以分为几类：

```text
randomUUID       随机 UUID
randomInt        随机 int
randomLong       随机 long
randomFloat      随机 float
randomDouble     随机 double
randomBoolean    随机 boolean
randomBytes      随机 byte[]
randomString     随机字符串
```

`ScxRandom` 是纯静态工具类，通常不需要创建实例。

```java
ScxRandom.randomInt();

ScxRandom.randomString(16, ScxRandom.NUMBER_AND_LETTER);
```

## 预置字符池

`ScxRandom` 内置了一组常用 `CharPool`。

```java
public static final CharPool NUMBER

public static final CharPool UPPER_LETTER

public static final CharPool LOWER_LETTER

public static final CharPool LETTER

public static final CharPool NUMBER_AND_UPPER_LETTER

public static final CharPool NUMBER_AND_LOWER_LETTER

public static final CharPool NUMBER_AND_LETTER
```

对应内容如下：

```text
NUMBER                    0123456789

UPPER_LETTER              ABCDEFGHIJKLMNOPQRSTUVWXYZ

LOWER_LETTER              abcdefghijklmnopqrstuvwxyz

LETTER                    ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz

NUMBER_AND_UPPER_LETTER   0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ

NUMBER_AND_LOWER_LETTER   0123456789abcdefghijklmnopqrstuvwxyz

NUMBER_AND_LETTER         0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

示例：

```java
String code = ScxRandom.randomString(6, ScxRandom.NUMBER);
```

可能输出：

```text
592016
```

生成大写字母：

```java
String value = ScxRandom.randomString(8, ScxRandom.UPPER_LETTER);
```

可能输出：

```text
QAZKMPWT
```

生成大小写字母和数字：

```java
String token = ScxRandom.randomString(32, ScxRandom.NUMBER_AND_LETTER);
```

可能输出：

```text
x83LdP02QzNm91AbkT6YpRstUvW4cDeF
```

## randomUUID

`randomUUID()` 用于生成随机 UUID 字符串。

```java
String uuid = ScxRandom.randomUUID();
```

内部使用：

```java
UUID.randomUUID().toString()
```

返回格式通常是：

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

示例：

```java
String id = ScxRandom.randomUUID();

System.out.println(id);
```

可能输出：

```text
550e8400-e29b-41d4-a716-446655440000
```

需要注意：

```text
randomUUID() 返回的是 String
不是 UUID 对象
```

如果你需要 `UUID` 对象，可以直接使用 JDK：

```java
UUID uuid = UUID.randomUUID();
```

## randomInt

`randomInt()` 返回一个随机 `int`。

```java
int value = ScxRandom.randomInt();
```

它内部使用：

```java
ThreadLocalRandom.current().nextInt()
```

### randomInt(bound)

`randomInt(bound)` 返回 `[0, bound)` 范围内的随机整数。

```java
int value = ScxRandom.randomInt(100);
```

范围是：

```text
0 <= value < 100
```

适合生成数组索引、验证码数字、分页随机偏移等。

```java
String[] names = {"Tom", "Jerry", "Alice"};

String name = names[ScxRandom.randomInt(names.length)];
```

### randomInt(origin, bound)

`randomInt(origin, bound)` 返回 `[origin, bound)` 范围内的随机整数。

```java
int value = ScxRandom.randomInt(10, 20);
```

范围是：

```text
10 <= value < 20
```

需要注意，这里的区间是：

```text
左闭右开
```

也就是包含 `origin`，不包含 `bound`。

## randomLong

`randomLong()` 返回一个随机 `long`。

```java
long value = ScxRandom.randomLong();
```

内部使用：

```java
ThreadLocalRandom.current().nextLong()
```

### randomLong(bound)

`randomLong(bound)` 返回 `[0, bound)` 范围内的随机 long。

```java
long value = ScxRandom.randomLong(1_000_000L);
```

范围是：

```text
0 <= value < 1000000
```

### randomLong(origin, bound)

`randomLong(origin, bound)` 返回 `[origin, bound)` 范围内的随机 long。

```java
long value = ScxRandom.randomLong(1_000L, 10_000L);
```

范围是：

```text
1000 <= value < 10000
```

## randomFloat

`randomFloat()` 返回一个随机 `float`。

```java
float value = ScxRandom.randomFloat();
```

范围是：

```text
0.0 <= value < 1.0
```

内部使用：

```java
ThreadLocalRandom.current().nextFloat()
```

### randomFloat(bound)

`randomFloat(bound)` 返回 `[0.0, bound)` 范围内的随机 float。

```java
float value = ScxRandom.randomFloat(10.0F);
```

范围是：

```text
0.0 <= value < 10.0
```

### randomFloat(origin, bound)

`randomFloat(origin, bound)` 返回 `[origin, bound)` 范围内的随机 float。

```java
float value = ScxRandom.randomFloat(10.0F, 20.0F);
```

范围是：

```text
10.0 <= value < 20.0
```

## randomDouble

`randomDouble()` 返回一个随机 `double`。

```java
double value = ScxRandom.randomDouble();
```

范围是：

```text
0.0 <= value < 1.0
```

内部使用：

```java
ThreadLocalRandom.current().nextDouble()
```

### randomDouble(bound)

`randomDouble(bound)` 返回 `[0.0, bound)` 范围内的随机 double。

```java
double value = ScxRandom.randomDouble(100.0);
```

范围是：

```text
0.0 <= value < 100.0
```

### randomDouble(origin, bound)

`randomDouble(origin, bound)` 返回 `[origin, bound)` 范围内的随机 double。

```java
double value = ScxRandom.randomDouble(1.5, 9.5);
```

范围是：

```text
1.5 <= value < 9.5
```

## randomBoolean

`randomBoolean()` 返回一个随机布尔值。

```java
boolean value = ScxRandom.randomBoolean();
```

可能结果是：

```text
true
```

也可能是：

```text
false
```

示例：

```java
if (ScxRandom.randomBoolean()) {
    System.out.println("A");
} else {
    System.out.println("B");
}
```

## randomBytes

`randomBytes(byte[] bytes)` 用于填充已有 byte 数组。

```java
byte[] bytes = new byte[16];

ScxRandom.randomBytes(bytes);
```

调用后，`bytes` 中的内容会被随机字节覆盖。

内部使用：

```java
ThreadLocalRandom.current().nextBytes(bytes)
```

### randomBytes(size)

`randomBytes(size)` 创建一个指定长度的 byte 数组，并填充随机内容。

```java
byte[] bytes = ScxRandom.randomBytes(32);
```

等价于：

```java
byte[] bytes = new byte[32];

ScxRandom.randomBytes(bytes);
```

适合生成：

```text
随机 nonce
临时随机数据
测试数据
普通随机 token 的原始字节
```

需要注意：

```text
randomBytes(size) 不是加密安全随机
```

如果用于密码学场景，应使用 `SecureRandom`。

## randomString

`randomString(...)` 用于从指定 pool 中随机抽取内容，并拼成字符串。

当前有三种重载：

```java
String randomString(int count, CharPool pool)

String randomString(int count, CodePointPool pool)

String randomString(int count, StringPool pool)
```

这三种重载的区别是：

```text
CharPool        每次抽取一个 char
CodePointPool   每次抽取一个 Unicode code point
StringPool      每次抽取一个 String 片段
```

## randomString + CharPool

`CharPool` 适合普通 ASCII 字符池。

例如数字、大小写字母、简单符号。

```java
String code = ScxRandom.randomString(6, ScxRandom.NUMBER);
```

结果长度一定是：

```text
6
```

因为每次抽取的是一个 `char`。

示例：

```java
String token = ScxRandom.randomString(
    16,
    ScxRandom.NUMBER_AND_LETTER
);
```

可能输出：

```text
R8a2Zq91Lm0KpT7x
```

### 自定义 CharPool

可以通过字符串创建 `CharPool`。

```java
import dev.scx.random.CharPool;

var pool = CharPool.ofString("ABCDEF0123456789");

String value = ScxRandom.randomString(8, pool);
```

可能输出：

```text
A03F9B1C
```

也可以直接用构造方法：

```java
var pool = new CharPool('A', 'B', 'C');

String value = ScxRandom.randomString(10, pool);
```

可能输出：

```text
ABCCBAACBA
```

### CharPool 的长度语义

`CharPool#length()` 返回 `char` 数量。

```java
var pool = CharPool.ofString("ABC");

System.out.println(pool.length());
```

输出：

```text
3
```

`CharPool#charAt(index)` 返回指定位置的 char。

```java
char c = pool.charAt(0);
```

结果：

```text
A
```

### CharPool 不适合所有 Unicode 字符

`CharPool` 基于 Java `char`。

这意味着它适合 BMP 内的单个 `char` 字符。

例如：

```text
A
中
1
_
```

但对某些需要 surrogate pair 的字符，例如部分 emoji，`char` 不能完整表示一个 Unicode 字符。

这种场景应该使用：

```java
CodePointPool
```

而不是：

```java
CharPool
```

## randomString + CodePointPool

`CodePointPool` 适合按 Unicode code point 抽取字符。

例如中文、emoji 或其它可能不只占一个 `char` 的字符。

```java
import dev.scx.random.CodePointPool;
import dev.scx.random.ScxRandom;

var pool = CodePointPool.ofString("123你好");

String value = ScxRandom.randomString(10, pool);
```

可能输出：

```text
你1好23你12好3
```

### CodePointPool.ofString

`CodePointPool.ofString(s)` 会使用：

```java
s.codePoints().toArray()
```

也就是说，它按 Unicode code point 拆分字符串，而不是按 `char` 拆分。

示例：

```java
var pool = CodePointPool.ofString("😀😁😂");

String value = ScxRandom.randomString(5, pool);
```

这里每次抽取的是一个完整 emoji code point。

### count 不一定等于 String.length()

对 `CodePointPool` 来说，`count` 表示抽取次数。

```java
String value = ScxRandom.randomString(5, codePointPool);
```

表示抽取 5 个 code point。

但最终字符串的：

```java
value.length()
```

不一定等于 `5`。

原因是 Java `String#length()` 返回的是 UTF-16 code unit 数量，而不是 code point 数量。

例如 emoji 通常占两个 `char`。

```java
var pool = CodePointPool.ofString("😀");

String value = ScxRandom.randomString(5, pool);

System.out.println(value.codePointCount(0, value.length()));
System.out.println(value.length());
```

结果可能是：

```text
5
10
```

所以文档中需要区分：

```text
抽取次数 count
最终 String.length()
最终 codePointCount
```

## randomString + StringPool

`StringPool` 适合从多个字符串片段中随机抽取。

```java
import dev.scx.random.StringPool;
import dev.scx.random.ScxRandom;

var pool = new StringPool("red", "green", "blue");

String value = ScxRandom.randomString(3, pool);
```

可能输出：

```text
redbluegreen
```

这里 `count = 3` 表示抽取 3 次字符串片段。

每个片段可以是任意长度。

### count 不一定等于最终字符串长度

对 `StringPool` 来说，`count` 表示抽取次数。

```java
String value = ScxRandom.randomString(4, new StringPool("A", "BB", "CCC"));
```

最终字符串长度可能是：

```text
4
5
6
7
8
9
10
11
12
```

取决于每次随机抽到的片段长度。

### StringPool 可以包含空字符串

`StringPool` 构造方法不会拒绝空字符串。

例如：

```java
new StringPool("", "", "", "")
```

这种情况下：

```java
String value = ScxRandom.randomString(1000, pool);
```

最终结果会是空字符串。

这是因为每次抽取到的片段都是：

```text
""
```

## 字符池对象

### CharPool

`CharPool` 是 char 候选池。

```java
public final class CharPool {

    public CharPool(char... chars)

    public static CharPool ofString(String s)

    public int length()

    public char charAt(int index)

}
```

示例：

```java
var pool = new CharPool('0', '1', '2', '3');

char c = pool.charAt(0);
```

### CodePointPool

`CodePointPool` 是 Unicode code point 候选池。

```java
public final class CodePointPool {

    public CodePointPool(int... codePoints)

    public static CodePointPool ofString(String s)

    public int length()

    public int codePointAt(int index)

}
```

示例：

```java
var pool = CodePointPool.ofString("你好");

int codePoint = pool.codePointAt(0);
```

### StringPool

`StringPool` 是字符串片段候选池。

```java
public final class StringPool {

    public StringPool(String... strings)

    public int length()

    public String stringAt(int index)

}
```

示例：

```java
var pool = new StringPool("A", "BB", "CCC");

String s = pool.stringAt(1);
```

结果：

```text
BB
```

## 边界规则

### 随机范围是左闭右开

`randomInt(origin, bound)`、`randomLong(origin, bound)`、`randomFloat(origin, bound)`、`randomDouble(origin, bound)` 都遵循 JDK `ThreadLocalRandom` 的范围语义。

```text
origin <= value < bound
```

也就是：

```text
左闭右开
```

示例：

```java
int value = ScxRandom.randomInt(1, 7);
```

结果可能是：

```text
1
2
3
4
5
6
```

不会是：

```text
7
```

### bound 必须有效

这些方法最终调用的是 JDK `ThreadLocalRandom`。

因此需要遵守 JDK 对参数的要求。

例如：

```java
ScxRandom.randomInt(0);
```

会抛出异常。

```java
ScxRandom.randomInt(10, 10);
```

也会抛出异常。

常见规则是：

```text
bound > 0
origin < bound
```

对应的 `long`、`float`、`double` 版本也有类似要求。

### randomBytes size 不能为负数

```java
byte[] bytes = ScxRandom.randomBytes(-1);
```

会因为创建数组长度为负数而抛出：

```text
NegativeArraySizeException
```

### randomString count 不能为负数

```java
String value = ScxRandom.randomString(-1, ScxRandom.NUMBER);
```

会因为创建数组长度为负数而抛出异常。

具体异常取决于对应重载：

```text
CharPool        创建 char[]，负数会抛 NegativeArraySizeException
CodePointPool   创建 int[]，负数会抛 NegativeArraySizeException
StringPool      循环不会执行，当前会返回空字符串
```

从使用语义上看，`count` 应该传入：

```text
count >= 0
```

### pool 不能为空

`randomString(count, pool)` 会从 pool 中随机选择索引。

如果 pool 长度为 `0`，会调用：

```java
randomInt(0)
```

这会抛出异常。

因此下面这些用法是非法的：

```java
ScxRandom.randomString(10, new CharPool());

ScxRandom.randomString(10, new CodePointPool());

ScxRandom.randomString(10, new StringPool());
```

需要保证：

```text
pool.length() > 0
```

### pool 不能为 null

```java
ScxRandom.randomString(10, (CharPool) null);
```

会抛出：

```text
NullPointerException
```

因为内部会访问：

```java
pool.length()
```

## 生成验证码

数字验证码：

```java
String code = ScxRandom.randomString(6, ScxRandom.NUMBER);
```

可能输出：

```text
038194
```

大写字母验证码：

```java
String code = ScxRandom.randomString(6, ScxRandom.UPPER_LETTER);
```

可能输出：

```text
QKZMPA
```

数字 + 大写字母：

```java
String code = ScxRandom.randomString(
    8,
    ScxRandom.NUMBER_AND_UPPER_LETTER
);
```

可能输出：

```text
A83K2P9Q
```

需要注意，如果验证码用于安全敏感场景，例如登录、支付、找回密码，应考虑是否需要使用 `SecureRandom`。

SCX Random 当前使用的是 `ThreadLocalRandom`。

## 生成普通 token

```java
String token = ScxRandom.randomString(
    32,
    ScxRandom.NUMBER_AND_LETTER
);
```

可能输出：

```text
7kQpZ02aLm91xYBcdEFgHiJ34NoPqRst
```

适合：

```text
测试 token
临时标识
非安全场景下的随机字符串
普通演示数据
```

不适合：

```text
密码重置 token
登录态 token
API secret
密钥
加密 nonce
```

这类安全场景应使用加密安全随机数。

## 生成随机文件名

```java
String fileName = ScxRandom.randomString(
    24,
    ScxRandom.NUMBER_AND_LOWER_LETTER
) + ".tmp";
```

可能输出：

```text
9x2kmp31abcz8qwe7ytr6lop.tmp
```

也可以使用 UUID：

```java
String fileName = ScxRandom.randomUUID() + ".tmp";
```

可能输出：

```text
f47ac10b-58cc-4372-a567-0e02b2c3d479.tmp
```

## 生成随机测试数据

随机年龄：

```java
int age = ScxRandom.randomInt(1, 120);
```

随机价格：

```java
double price = ScxRandom.randomDouble(1.0, 1000.0);
```

随机状态：

```java
boolean enabled = ScxRandom.randomBoolean();
```

随机用户名：

```java
String username = "user_" + ScxRandom.randomString(
    8,
    ScxRandom.NUMBER_AND_LOWER_LETTER
);
```

随机二进制内容：

```java
byte[] data = ScxRandom.randomBytes(1024);
```

## Unicode 随机字符串

如果字符池中包含中文，可以使用 `CodePointPool`。

```java
import dev.scx.random.CodePointPool;
import dev.scx.random.ScxRandom;

var pool = CodePointPool.ofString("你好世界");

String value = ScxRandom.randomString(10, pool);
```

可能输出：

```text
你世好界你你世好界好
```

如果字符池中包含 emoji，也应该使用 `CodePointPool`。

```java
var pool = CodePointPool.ofString("😀😁😂🤣😎");

String value = ScxRandom.randomString(5, pool);
```

可能输出：

```text
😎😂😀🤣😁
```

不要用 `CharPool.ofString(...)` 处理这种字符池。

因为 `CharPool` 是按 `char` 拆分，可能把 surrogate pair 拆开。

## 字符串片段随机组合

`StringPool` 可以用来随机组合多个片段。

```java
import dev.scx.random.StringPool;
import dev.scx.random.ScxRandom;

var adjectives = new StringPool("red-", "blue-", "fast-", "tiny-");

var nouns = new StringPool("cat", "dog", "fox");

String name = ScxRandom.randomString(1, adjectives)
    + ScxRandom.randomString(1, nouns);
```

可能输出：

```text
fast-fox
```

也可以一次随机多个片段：

```java
var pool = new StringPool("ha", "yo", "la");

String value = ScxRandom.randomString(4, pool);
```

可能输出：

```text
hahalayo
```

## 方法总览

### UUID

```java
String randomUUID()
```

### int

```java
int randomInt()

int randomInt(int bound)

int randomInt(int origin, int bound)
```

### long

```java
long randomLong()

long randomLong(long bound)

long randomLong(long origin, long bound)
```

### float

```java
float randomFloat()

float randomFloat(float bound)

float randomFloat(float origin, float bound)
```

### double

```java
double randomDouble()

double randomDouble(double bound)

double randomDouble(double origin, double bound)
```

### boolean

```java
boolean randomBoolean()
```

### byte[]

```java
void randomBytes(byte[] bytes)

byte[] randomBytes(int size)
```

### String

```java
String randomString(int count, CharPool pool)

String randomString(int count, CodePointPool pool)

String randomString(int count, StringPool pool)
```

### CharPool

```java
public CharPool(char... chars)

public static CharPool ofString(String s)

public int length()

public char charAt(int index)
```

### CodePointPool

```java
public CodePointPool(int... codePoints)

public static CodePointPool ofString(String s)

public int length()

public int codePointAt(int index)
```

### StringPool

```java
public StringPool(String... strings)

public int length()

public String stringAt(int index)
```

## 完整示例：随机验证码

```java
import dev.scx.random.ScxRandom;

public class RandomCodeDemo {

    public static void main(String[] args) {
        String numericCode = ScxRandom.randomString(
            6,
            ScxRandom.NUMBER
        );

        String mixedCode = ScxRandom.randomString(
            8,
            ScxRandom.NUMBER_AND_UPPER_LETTER
        );

        System.out.println(numericCode);
        System.out.println(mixedCode);
    }

}
```

可能输出：

```text
492017
A8K2P9QZ
```

## 完整示例：随机 token

```java
import dev.scx.random.ScxRandom;

public class RandomTokenDemo {

    public static void main(String[] args) {
        String token = ScxRandom.randomString(
            32,
            ScxRandom.NUMBER_AND_LETTER
        );

        System.out.println(token);
    }

}
```

可能输出：

```text
x83LdP02QzNm91AbkT6YpRstUvW4cDeF
```

## 完整示例：随机 Unicode 字符串

```java
import dev.scx.random.CodePointPool;
import dev.scx.random.ScxRandom;

public class RandomUnicodeDemo {

    public static void main(String[] args) {
        var pool = CodePointPool.ofString("123你好😀😁");

        String value = ScxRandom.randomString(20, pool);

        System.out.println(value);

        System.out.println(value.codePointCount(0, value.length()));
    }

}
```

这里第二行输出的 code point 数量一定是：

```text
20
```

但 `value.length()` 不一定是 `20`。

## 完整示例：随机片段组合

```java
import dev.scx.random.ScxRandom;
import dev.scx.random.StringPool;

public class RandomStringPoolDemo {

    public static void main(String[] args) {
        var prefixPool = new StringPool(
            "red-",
            "blue-",
            "green-"
        );

        var namePool = new StringPool(
            "cat",
            "dog",
            "fox"
        );

        String value = ScxRandom.randomString(1, prefixPool)
            + ScxRandom.randomString(1, namePool);

        System.out.println(value);
    }

}
```

可能输出：

```text
blue-fox
```

## 完整示例：随机测试对象

```java
import dev.scx.random.ScxRandom;

public class RandomUserDemo {

    public static void main(String[] args) {
        var user = new User(
            ScxRandom.randomUUID(),
            "user_" + ScxRandom.randomString(8, ScxRandom.NUMBER_AND_LOWER_LETTER),
            ScxRandom.randomInt(1, 120),
            ScxRandom.randomBoolean()
        );

        System.out.println(user);
    }

    public record User(
        String id,
        String username,
        int age,
        boolean enabled
    ) {

    }

}
```

可能输出：

```text
User[id=..., username=user_a82k1pxz, age=36, enabled=true]
```

## 完整示例：随机字节

```java
import dev.scx.random.ScxRandom;

import java.util.HexFormat;

public class RandomBytesDemo {

    public static void main(String[] args) {
        byte[] bytes = ScxRandom.randomBytes(16);

        String hex = HexFormat.of().formatHex(bytes);

        System.out.println(hex);
    }

}
```

可能输出：

```text
4f9c2a13b8e7d00193ffac1023897abe
```

## 设计说明

### 1. SCX Random 是纯静态工具类

`ScxRandom` 中的方法都是静态方法。

常见用法是：

```java
ScxRandom.randomInt(100);

ScxRandom.randomString(16, ScxRandom.NUMBER_AND_LETTER);
```

它没有保存全局随机状态，也不需要创建实例。

### 2. 底层使用 ThreadLocalRandom

除 UUID 之外，当前随机数方法都使用：

```java
ThreadLocalRandom.current()
```

这适合普通业务随机场景，尤其是多线程环境下避免共享同一个 `Random` 实例带来的竞争。

但它不是加密安全随机。

如果需要加密安全随机，应使用：

```java
SecureRandom
```

### 3. randomUUID 使用 JDK UUID

`randomUUID()` 直接使用：

```java
UUID.randomUUID().toString()
```

它返回的是字符串形式，而不是 `UUID` 对象。

### 4. 范围方法遵循 JDK 语义

范围型方法都遵循左闭右开。

```text
origin <= value < bound
```

这和 JDK `ThreadLocalRandom` 的语义一致。

### 5. CharPool 面向 char

`CharPool` 是最简单的字符池。

它使用：

```java
char[]
```

适合：

```text
数字
英文字母
ASCII 符号
常见 BMP 字符
```

如果字符池中包含 emoji 或其它需要 surrogate pair 的字符，应使用 `CodePointPool`。

### 6. CodePointPool 面向 Unicode code point

`CodePointPool` 使用：

```java
int[]
```

保存 code point。

它适合 Unicode 字符更复杂的场景。

```java
CodePointPool.ofString("你好😀")
```

会按 code point 拆分，而不是按 `char` 拆分。

### 7. StringPool 面向字符串片段

`StringPool` 保存的是：

```java
String[]
```

它适合随机拼接片段，而不是随机字符。

例如：

```text
前缀
后缀
单词
emoji 字符串
模板片段
```

### 8. count 是抽取次数

`randomString(int count, pool)` 中的 `count` 表示从池中抽取多少次。

对不同 pool 来说，最终字符串长度语义不同。

```text
CharPool        count 等于最终 char 数量
CodePointPool   count 等于 code point 数量，但不一定等于 String.length()
StringPool      count 等于片段数量，不一定等于最终 String.length()
```

### 9. 不主动做参数校验

当前实现基本直接调用 JDK 或底层数组访问。

因此很多非法参数会由 JDK 自己抛出异常。

例如：

```text
bound <= 0
origin >= bound
count < 0
pool 为空
pool 为 null
```

调用方应传入合法参数。

### 10. 不适合安全敏感随机

SCX Random 的定位是普通随机工具。

它适合：

```text
测试数据
普通验证码
临时文件名
演示 token
随机选择
普通业务随机值
```

不适合直接用于：

```text
密码
密钥
登录 token
支付验证码
找回密码验证码
加密 nonce
API secret
```

这些场景应使用专门的加密安全随机方案。

## 常见问题

### SCX Random 是加密安全随机库吗？

不是。

它底层主要使用 `ThreadLocalRandom`。

如果需要加密安全随机，应使用 `SecureRandom`。

### randomInt(origin, bound) 包含 bound 吗？

不包含。

范围是：

```text
origin <= value < bound
```

### randomInt(bound) 的范围是什么？

范围是：

```text
0 <= value < bound
```

### randomFloat() 和 randomDouble() 的范围是什么？

范围是：

```text
0.0 <= value < 1.0
```

### randomBytes(size) 会返回多长的数组？

返回长度为 `size` 的 `byte[]`。

```java
byte[] bytes = ScxRandom.randomBytes(32);
```

结果数组长度是：

```text
32
```

### randomBytes(byte[]) 会创建新数组吗？

不会。

它会填充传入的数组。

```java
byte[] bytes = new byte[16];

ScxRandom.randomBytes(bytes);
```

调用后，`bytes` 本身被修改。

### randomUUID 返回 UUID 对象吗？

不是。

返回的是：

```java
String
```

如果需要 `UUID` 对象，请直接使用：

```java
UUID.randomUUID()
```

### randomString 的 count 是最终字符串长度吗？

不一定。

对 `CharPool` 来说，通常等于最终 `String.length()`。

对 `CodePointPool` 来说，`count` 是 code point 数量，不一定等于 `String.length()`。

对 `StringPool` 来说，`count` 是片段抽取次数，不一定等于最终字符串长度。

### 什么时候用 CharPool？

字符池只包含普通 `char` 时使用。

例如：

```text
数字
英文字母
ASCII 符号
```

### 什么时候用 CodePointPool？

字符池中可能包含 emoji 或其它非 BMP 字符时使用。

```java
CodePointPool.ofString("😀😁😂")
```

### 什么时候用 StringPool？

候选项不是单个字符，而是一段字符串时使用。

```java
new StringPool("red-", "blue-", "green-")
```

### CharPool.ofString 和 CodePointPool.ofString 有什么区别？

`CharPool.ofString(...)` 使用：

```java
s.toCharArray()
```

`CodePointPool.ofString(...)` 使用：

```java
s.codePoints().toArray()
```

所以前者按 `char` 拆分，后者按 Unicode code point 拆分。

### pool 可以为空吗？

不应该为空。

空 pool 会导致随机索引范围为 `0`，从而抛出异常。

### StringPool 可以包含空字符串吗？

可以。

如果所有候选项都是空字符串，那么随机结果也是空字符串。

### 生成验证码能直接用于安全场景吗？

不建议。

普通展示或测试可以使用。

登录、支付、找回密码等安全敏感验证码应考虑使用 `SecureRandom`。

### 生成 token 能直接用于登录态吗？

不建议。

登录 token、API secret、密码重置 token 等安全敏感内容应使用加密安全随机数。

### 为什么没有传入 RandomGenerator 的方法？

当前版本没有提供。

所有随机数都直接使用 `ThreadLocalRandom.current()`。

如果需要可复现随机序列，例如测试中指定 seed，当前版本不适合直接使用。

### 为什么 randomString 没有默认字符池重载？

当前版本要求显式传入 pool。

例如：

```java
ScxRandom.randomString(16, ScxRandom.NUMBER_AND_LETTER);
```

这样可以避免默认字符集语义不明确。

### 什么时候用 SCX Random？

适合下面这些场景：

1. 生成普通随机整数。
2. 生成普通随机 long。
3. 生成普通随机浮点数。
4. 生成随机 boolean。
5. 生成随机 byte[]。
6. 生成随机 UUID 字符串。
7. 生成数字验证码。
8. 生成英文字母和数字组成的随机字符串。
9. 从 Unicode 字符池中随机生成字符串。
10. 从字符串片段池中随机拼接结果。
11. 生成测试数据。
12. 生成非安全敏感的临时标识。
