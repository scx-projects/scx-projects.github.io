# SCX FFI

SCX FFI 是一个基于 JDK Foreign Function & Memory API 的轻量 FFI 封装库。

它允许你用 Java 接口声明 native 函数，然后通过 `ScxFFI.createFFI(...)` 创建代理对象，最终像调用普通 Java 方法一样调用本机函数。

SCX FFI 本身不负责提供 C 标准库、Windows API 或其它 native library。它只负责把 Java 接口方法映射到 native symbol，并在调用前后完成参数的内存转换和必要的回写。

[GitHub](https://github.com/scx-projects/scx-ffi)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-ffi</artifactId>
    <version>0.10.0</version>
</dependency>
```

SCX FFI 依赖 JDK 的 `java.lang.foreign` API，因此使用时需要运行在支持该 API 的 JDK 上。

## 基本概念

SCX FFI 中最核心的概念包括：

```text
ScxFFI        创建 FFI 代理对象的入口
FFMProxy      Java 动态代理 InvocationHandler
FFMMapper     Java 对象和 MemorySegment 之间的转换器
SymbolName    指定 Java 方法对应的 native symbol 名称
FFIStruct     表示 native 结构体
FFICallback   表示 native 回调函数
EncodedString 指定编码的字符串
ByteRef 等    用于模拟 C 指针出参
```

它们之间的关系可以简单理解为：

```text
Java 接口方法
    ↓
ScxFFI.createFFI(...)
    ↓
FFMProxy
    ↓
查找 native symbol
    ↓
创建 downcall MethodHandle
    ↓
调用时转换参数
    ↓
调用 native 函数
    ↓
必要时回写参数
```

也就是说：

```text
接口负责声明 native 函数
ScxFFI 负责创建代理
FFMProxy 负责拦截调用
FFMMapper 负责对象和 MemorySegment 的相互转换
```

## 快速开始

以 C 标准库中的 `strlen` 为例，可以先定义一个 Java 接口：

```java
import dev.scx.ffi.ScxFFI;

public interface C {

    C C = ScxFFI.createFFI(C.class);

    long strlen(String str);

}
```

然后直接调用：

```java
long length = C.C.strlen("abc123456");

System.out.println(length);
```

如果 native symbol 可以在默认 lookup 中找到，可以使用：

```java
ScxFFI.createFFI(C.class)
```

如果函数在指定动态库中，可以使用：

```java
ScxFFI.createFFI(User32.class, "user32")
```

或者使用动态库路径：

```java
ScxFFI.createFFI(MyLib.class, Path.of("/path/to/libxxx.so"))
```

## 定义 FFI 接口

SCX FFI 的主要使用方式是定义一个接口。

例如：

```java
import dev.scx.ffi.ScxFFI;

public interface C {

    C C = ScxFFI.createFFI(C.class);

    long strlen(String str);

    int abs(int x);

    double sqrt(double x);

}
```

然后：

```java
long a = C.C.strlen("hello");

int b = C.C.abs(-123);

double c = C.C.sqrt(88);
```

接口中的抽象方法会被映射为 native symbol。

默认情况下，Java 方法名就是 native symbol 名：

```text
strlen -> strlen
abs    -> abs
sqrt   -> sqrt
```

## 创建代理

`ScxFFI` 提供了四个入口方法。

### 使用默认 lookup

```java
public static <T> T createFFI(Class<T> clazz)
```

示例：

```java
var c = ScxFFI.createFFI(C.class);
```

这种方式使用：

```java
nativeLinker().defaultLookup()
```

适合调用默认可见的本机符号。

### 使用库名称

```java
public static <T> T createFFI(Class<T> clazz, String libraryName)
```

示例：

```java
var user32 = ScxFFI.createFFI(User32.class, "user32");
```

这种方式内部使用：

```java
SymbolLookup.libraryLookup(libraryName, Arena.global())
```

适合按库名称加载 native library。

例如 Windows 上：

```java
ScxFFI.createFFI(User32.class, "user32")
```

### 使用库路径

```java
public static <T> T createFFI(Class<T> clazz, Path libraryPath)
```

示例：

```java
var myLib = ScxFFI.createFFI(MyLib.class, Path.of("/opt/lib/libxxx.so"));
```

这种方式适合通过明确路径加载动态库。

### 使用 SymbolLookup

```java
public static <T> T createFFI(Class<T> clazz, SymbolLookup symbolLookup)
```

示例：

```java
var myLib = ScxFFI.createFFI(MyLib.class, customSymbolLookup);
```

这种方式适合需要自己控制 symbol 查找逻辑的场景。

## SymbolName

默认情况下，SCX FFI 使用 Java 方法名查找 native symbol。

如果 native symbol 名称不适合作为 Java 方法名，或者希望 Java 方法名和 native symbol 名分离，可以使用 `@SymbolName`。

示例：

```java
import dev.scx.ffi.annotation.SymbolName;

public interface C {

    @SymbolName("abs")
    int javaAbs(int x);

}
```

调用时：

```java
int value = C.C.javaAbs(-123);
```

实际查找的 native symbol 是：

```text
abs
```

不是：

```text
javaAbs
```

这适合下面这些场景：

1. native 函数名不是合法 Java 方法名。
2. native 函数名和 Java 侧命名风格不同。
3. 希望避免和 Java 默认方法、Object 方法或其它方法名冲突。
4. 希望给 native 方法提供更清晰的 Java 名称。

## 默认方法

FFI 接口中可以定义默认方法。

默认方法不会映射到 native symbol，而是作为普通 Java 默认方法调用。

示例：

```java
public interface C {

    C C = ScxFFI.createFFI(C.class);

    @SymbolName("abs")
    int javaAbs(int x);

    default int abs(int x) {
        return javaAbs(x);
    }

}
```

调用：

```java
int value = C.C.abs(-123);
```

这里：

1. `abs(...)` 是 Java 默认方法。
2. `javaAbs(...)` 才会映射到 native symbol `abs`。
3. 默认方法内部可以继续调用其它 FFI 方法。

## 继承接口

FFI 接口可以继承其它接口。

父接口中的抽象方法也会被扫描并映射为 native symbol。

示例：

```java
public interface CBase {

    double sin(double x);

}
```

```java
public interface C extends CBase {

    C C = ScxFFI.createFFI(C.class);

    long strlen(String str);

    double sqrt(double x);

}
```

调用：

```java
double a = C.C.sin(12);
double b = C.C.sqrt(88);
```

这里 `sin(...)` 来自父接口，但仍然可以被 FFI 代理处理。

## 支持的参数类型

SCX FFI 当前支持下面这些参数类型：

```text
byte
short
int
long
float
double
char

MemorySegment

ByteRef
ShortRef
IntRef
LongRef
FloatRef
DoubleRef
CharRef

String
EncodedString

byte[]
short[]
int[]
long[]
float[]
double[]
char[]

FFIStruct

FFICallback

FFMMapper

null
```

它们最终会被转换成下面两类 native 调用参数：

```text
基本类型
MemorySegment
```

其中：

1. 基本类型会直接传给 FFM。
2. `MemorySegment` 会直接传给 FFM。
3. Ref、字符串、数组、结构体、回调和自定义 mapper 会先转换成 `MemorySegment`。
4. `null` 会被转换成 `MemorySegment.NULL`。

## 支持的返回值类型

SCX FFI 当前支持下面这些返回值类型：

```text
void

byte
short
int
long
float
double
char

MemorySegment
```

例如：

```java
int abs(int x);

double sqrt(double x);

MemorySegment GetStdHandle(int nStdHandle);

void someVoidFunction();
```

需要注意：

1. 当前返回值不支持 `String`。
2. 当前返回值不支持 `FFIStruct`。
3. 当前返回值不支持 `FFICallback`。
4. 当前返回值不支持数组。
5. 当前返回值不支持 `boolean`。
6. 如果 native 函数返回指针，应使用 `MemorySegment` 接收。
7. 如果 native 函数通过指针写出数据，应使用 Ref、数组或结构体参数。

## 类型映射

常见类型映射可以理解为：

```text
Java byte        -> JAVA_BYTE
Java short       -> JAVA_SHORT
Java int         -> JAVA_INT
Java long        -> JAVA_LONG
Java float       -> JAVA_FLOAT
Java double      -> JAVA_DOUBLE
Java char        -> JAVA_CHAR
MemorySegment    -> ADDRESS
```

对于参数来说，下面这些类型会被映射为地址：

```text
ByteRef 等 Ref 类型    -> ADDRESS
String                 -> ADDRESS
EncodedString          -> ADDRESS
基本类型数组            -> ADDRESS
FFIStruct              -> ADDRESS
FFICallback            -> ADDRESS
FFMMapper              -> ADDRESS
null                   -> MemorySegment.NULL
```

对于返回值来说，`MemorySegment` 会映射为：

```text
ADDRESS
```

## 字符串

### String

`String` 参数会通过 `StringFFMMapper` 转换成 native 字符串内存。

示例：

```java
long strlen(String str);
```

调用：

```java
long length = C.C.strlen("abc123456");
```

`String` 是只读映射。

也就是说，native 函数如果修改了这块字符串内存，修改结果不会回写到原来的 Java `String` 中。

这是因为 Java `String` 本身是不可变对象。

### EncodedString

如果需要指定字符串编码，可以使用 `EncodedString`。

示例：

```java
import dev.scx.ffi.type.EncodedString;

import static java.nio.charset.StandardCharsets.UTF_16LE;

int MessageBoxW(MemorySegment hWnd,
                EncodedString lpText,
                EncodedString lpCaption,
                int uType);
```

调用：

```java
USER32.MessageBoxW(
    null,
    new EncodedString("MessageBoxW 测试中文内容", UTF_16LE),
    new EncodedString("测试标题", UTF_16LE),
    0
);
```

`EncodedString` 适合下面这些场景：

1. Windows `W` 系列 API。
2. native 函数要求 UTF-16LE。
3. native 函数要求特定编码。
4. 不希望依赖普通 `String` 的默认编码行为。

和 `String` 一样，`EncodedString` 也是只读映射，不会从 native 内存回写到 Java 对象。

## Ref 类型

SCX FFI 提供了一组 Ref 类型，用来模拟 C 中的指针出参。

```text
ByteRef
ShortRef
IntRef
LongRef
FloatRef
DoubleRef
CharRef
```

例如 C 风格函数：

```c
int get_value(int* out);
```

可以声明为：

```java
import dev.scx.ffi.type.IntRef;

int get_value(IntRef out);
```

调用：

```java
var out = new IntRef();

int result = lib.get_value(out);

System.out.println(out.value());
```

Ref 类型的基本语义是：

1. 调用前，根据当前 `value()` 分配 native 内存。
2. 调用 native 函数时，把这块内存地址传进去。
3. native 函数执行结束后，从内存中读取值。
4. 将读取到的值写回 Ref 对象。

以 `IntRef` 为例：

```java
var mode = new IntRef();

KERNEL32.GetConsoleMode(handle, mode);

int value = mode.value();
```

如果 native 函数修改了 `int*` 指向的值，调用结束后可以通过 `mode.value()` 读取。

### Ref 初始值

Ref 类型有无参构造和带初始值构造。

```java
var a = new IntRef();

var b = new IntRef(100);
```

无参构造默认值为对应基本类型的零值。

## 数组

SCX FFI 支持基本类型数组作为参数。

```text
byte[]
short[]
int[]
long[]
float[]
double[]
char[]
```

数组会被转换成一段连续 native 内存。

调用结束后，native 内存中的数据会回写到原 Java 数组对象中。

示例：

```java
void fill(int[] array, int length);
```

调用：

```java
var array = new int[10];

lib.fill(array, array.length);

System.out.println(Arrays.toString(array));
```

如果 native 函数修改了数组内容，Java 侧原数组会被更新。

测试中也使用了 `qsort` 对 `int[]` 进行排序：

```java
void qsort(int[] base, long nmemb, long size, Compar compar);
```

调用：

```java
var array = new int[]{2, 5, 7, 1, 4, 56, 12, 31, 99999, 90, 271, 2};

C.C.qsort(array, array.length, Integer.BYTES, (aAddr, bAddr) -> {
    int a = aAddr.get(JAVA_INT, 0);
    int b = bAddr.get(JAVA_INT, 0);
    return Integer.compare(a, b);
});
```

调用后，`array` 会被 native `qsort` 修改为排序后的结果。

需要注意：

1. 数组长度由 Java 数组本身决定。
2. native 函数不应写超过数组长度的内存。
3. SCX FFI 不会知道 native 函数实际写了多少。
4. 回写时会把 native 内存内容复制回原数组。
5. 数组对象本身不会被替换，外部持有的数组引用仍然有效。

## MemorySegment

如果参数或返回值本来就是 native 地址，可以直接使用 `MemorySegment`。

示例：

```java
MemorySegment GetStdHandle(int nStdHandle);
```

调用：

```java
MemorySegment handle = KERNEL32.GetStdHandle(-11);
```

也可以作为参数传入：

```java
int IsWindowVisible(MemorySegment hWnd);
```

```java
int visible = USER32.IsWindowVisible(hWnd);
```

当需要表示空指针时，可以传入 `null`：

```java
USER32.MessageBoxA(null, "内容", "标题", 0);
```

SCX FFI 会把 `null` 转换为：

```java
MemorySegment.NULL
```

## 结构体

如果 native 函数需要结构体指针，可以使用 `FFIStruct`。

结构体类需要实现 `FFIStruct`，并使用 public 非 static 字段表示结构体字段。

示例：

```java
import dev.scx.ffi.type.FFIStruct;

public static class POINT implements FFIStruct {

    public int x;

    public int y;

}
```

对应 C 结构体可以理解为：

```c
typedef struct POINT {
    int x;
    int y;
} POINT;
```

然后可以声明 native 函数：

```java
int GetCursorPos(POINT lpPoint);
```

调用：

```java
var point = new POINT();

USER32.GetCursorPos(point);

System.out.println(point.x);
System.out.println(point.y);
```

调用流程是：

1. 根据 `POINT` 的 public 非 static 字段创建结构体内存布局。
2. 调用前把 Java 字段写入 native 内存。
3. 调用 native 函数。
4. 调用结束后从 native 内存读取字段值。
5. 写回原 Java 对象。

### 支持的结构体字段类型

`FFIStruct` 当前支持下面这些字段类型：

```text
byte
short
int
long
float
double
char

MemorySegment

FFIStruct
```

也就是说，结构体可以嵌套结构体。

示例：

```java
public static class RECT implements FFIStruct {

    public int left;

    public int top;

    public int right;

    public int bottom;

}
```

### 字段顺序

默认情况下，结构体字段使用字段声明顺序。

如果需要自定义字段顺序，可以重写 `fieldOrder()`。

```java
public static class RECT implements FFIStruct {

    public int left;

    public int top;

    public int right;

    public int bottom;

    @Override
    public String[] fieldOrder() {
        return new String[]{
            "left",
            "top",
            "right",
            "bottom"
        };
    }

}
```

`fieldOrder()` 返回 `null` 表示使用默认顺序。

### 字段过滤规则

SCX FFI 只处理结构体中的：

```text
public
非 static
字段
```

也就是说，下面这些字段不会参与结构体布局：

```java
private int a;

protected int b;

public static int c;
```

### null 字段

结构体字段不允许为 `null`。

尤其是嵌套结构体字段和 `MemorySegment` 字段，需要在调用前初始化好。

例如：

```java
public static class Outer implements FFIStruct {

    public Inner inner = new Inner();

}
```

而不是：

```java
public static class Outer implements FFIStruct {

    public Inner inner;

}
```

### 递归嵌套

SCX FFI 不支持递归嵌套结构体。

例如下面这种结构是不支持的：

```java
public static class Node implements FFIStruct {

    public int value;

    public Node next;

}
```

因为它无法生成固定大小的结构体布局。

如果 native 结构体中包含指针字段，应使用 `MemorySegment` 表示指针：

```java
public static class Node implements FFIStruct {

    public int value;

    public MemorySegment next;

}
```

## 回调函数

如果 native 函数需要函数指针，可以使用 `FFICallback`。

回调对象需要实现 `FFICallback`，并提供一个回调方法。

默认回调方法名是：

```text
callback
```

示例：

```java
import dev.scx.ffi.type.FFICallback;
import java.lang.foreign.MemorySegment;

public interface Compar extends FFICallback {

    int callback(MemorySegment aAddr, MemorySegment bAddr);

}
```

然后声明 native 函数：

```java
void qsort(int[] base, long nmemb, long size, Compar compar);
```

调用：

```java
import static java.lang.foreign.ValueLayout.JAVA_INT;

var array = new int[]{2, 5, 7, 1};

C.C.qsort(array, array.length, Integer.BYTES, (aAddr, bAddr) -> {
    int a = aAddr.get(JAVA_INT, 0);
    int b = bAddr.get(JAVA_INT, 0);
    return Integer.compare(a, b);
});
```

这里 Java lambda 会被转换成 native function pointer，然后传给 `qsort`。

### 自定义回调方法名

如果回调方法不叫 `callback`，可以重写 `callbackMethodName()`。

```java
public interface MyCallback extends FFICallback {

    int apply(int value);

    @Override
    default String callbackMethodName() {
        return "apply";
    }

}
```

SCX FFI 会查找名为 `apply` 的方法作为回调入口。

### 回调参数和返回值类型

回调函数当前支持下面这些参数类型：

```text
byte
short
int
long
float
double
char

MemorySegment
```

回调函数当前支持下面这些返回值类型：

```text
void

byte
short
int
long
float
double
char

MemorySegment
```

### parameterTargetLayouts

当回调参数是 `MemorySegment`，并且希望它指向具体类型数据时，可以通过 `parameterTargetLayouts()` 提供目标内存布局。

示例：

```java
import dev.scx.ffi.type.FFICallback;

import java.lang.foreign.MemoryLayout;
import java.lang.foreign.MemorySegment;

import static java.lang.foreign.ValueLayout.JAVA_INT;

public interface Compar extends FFICallback {

    int callback(MemorySegment aAddr, MemorySegment bAddr);

    @Override
    default MemoryLayout[] parameterTargetLayouts() {
        return new MemoryLayout[]{
            JAVA_INT,
            JAVA_INT
        };
    }

}
```

这样 `aAddr` 和 `bAddr` 就可以按 `JAVA_INT` 指向的数据来读取：

```java
int a = aAddr.get(JAVA_INT, 0);
int b = bAddr.get(JAVA_INT, 0);
```

如果 `parameterTargetLayouts()` 返回 `null`，则 `MemorySegment` 参数会按普通地址处理。

## 自定义映射器

如果内置类型无法满足需求，可以实现 `FFMMapper`。

接口定义可以理解为：

```java
public interface FFMMapper {

    MemorySegment toMemorySegment(Arena arena) throws Exception;

    void fromMemorySegment(MemorySegment memorySegment) throws Exception;

}
```

其中：

1. `toMemorySegment(...)` 在调用 native 函数前执行。
2. 它负责把 Java 对象转换为 native 内存。
3. `fromMemorySegment(...)` 在 native 函数调用结束后执行。
4. 它负责把 native 内存中的数据写回 Java 对象。
5. 如果是只读映射，`fromMemorySegment(...)` 可以什么都不做。

示例：

```java
import dev.scx.ffi.mapper.FFMMapper;

import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;

import static java.lang.foreign.ValueLayout.JAVA_INT;

public final class IntBoxMapper implements FFMMapper {

    private final IntBox box;

    public IntBoxMapper(IntBox box) {
        this.box = box;
    }

    @Override
    public MemorySegment toMemorySegment(Arena arena) {
        return arena.allocateFrom(JAVA_INT, box.value);
    }

    @Override
    public void fromMemorySegment(MemorySegment memorySegment) {
        box.value = memorySegment.get(JAVA_INT, 0);
    }

}
```

使用时，接口方法参数可以直接写成该 mapper 类型：

```java
int get_value(IntBoxMapper out);
```

调用：

```java
var box = new IntBox();

lib.get_value(new IntBoxMapper(box));

System.out.println(box.value);
```

一般情况下，不需要直接使用内置 mapper。

推荐使用：

```text
IntRef
int[]
String
EncodedString
FFIStruct
FFICallback
```

只有在内置映射无法表达你的 native 类型时，再实现 `FFMMapper`。

## 调用生命周期

一次 FFI 方法调用大致经过下面几个步骤：

```text
1. 进入代理方法
2. 创建 Arena.ofConfined()
3. 把 Java 参数包装成基本类型、MemorySegment 或 FFMMapper
4. 把 FFMMapper 转换成 MemorySegment
5. 调用 native MethodHandle
6. 对 FFMMapper 参数执行 fromMemorySegment 回写
7. 关闭 Arena
8. 返回 native 调用结果
```

示意代码：

```java
try (var arena = Arena.ofConfined()) {
    var wrappedParameters = wrapParameters(args);
    var nativeParameters = prepareNativeParameters(wrappedParameters, arena);
    var result = methodHandle.invokeWithArguments(nativeParameters);
    writeBackParameters(wrappedParameters, nativeParameters);
    return result;
}
```

因此需要注意：

1. 调用参数中由 SCX FFI 分配的临时 native 内存，只在本次调用期间有效。
2. native 函数不应保存这些临时地址并在调用结束后继续使用。
3. 如果 native 函数需要长期保存指针，应由调用者自己管理对应内存。
4. 如果 native 函数返回 `MemorySegment`，这段内存的生命周期由 native 侧决定，SCX FFI 不负责释放。

## 完整示例：C 标准库

下面是一个简化的 C 标准库接口示例。

```java
import dev.scx.ffi.ScxFFI;
import dev.scx.ffi.annotation.SymbolName;
import dev.scx.ffi.type.FFICallback;

import java.lang.foreign.MemoryLayout;
import java.lang.foreign.MemorySegment;

import static java.lang.foreign.ValueLayout.JAVA_INT;

public interface C extends CBase {

    C C = ScxFFI.createFFI(C.class);

    long strlen(String str);

    @SymbolName("abs")
    int javaAbs(int x);

    double sqrt(double x);

    void qsort(int[] base, long nmemb, long size, Compar compar);

    default int abs(int x) {
        return javaAbs(x);
    }

    interface Compar extends FFICallback {

        int callback(MemorySegment aAddr, MemorySegment bAddr);

        @Override
        default MemoryLayout[] parameterTargetLayouts() {
            return new MemoryLayout[]{
                JAVA_INT,
                JAVA_INT
            };
        }

    }

}
```

父接口：

```java
public interface CBase {

    double sin(double x);

}
```

调用：

```java
import static java.lang.foreign.ValueLayout.JAVA_INT;

long length = C.C.strlen("abc123456");

int abs = C.C.abs(-123);

double sin = C.C.sin(12);

double sqrt = C.C.sqrt(88);

var array = new int[]{2, 5, 7, 1, 4, 56, 12, 31, 99999, 90, 271, 2};

C.C.qsort(array, array.length, Integer.BYTES, (aAddr, bAddr) -> {
    int a = aAddr.get(JAVA_INT, 0);
    int b = bAddr.get(JAVA_INT, 0);
    return Integer.compare(a, b);
});
```

调用结束后，`array` 会被排序。

## 完整示例：Windows Kernel32

下面是一个 Windows `Kernel32` 示例。

```java
import dev.scx.ffi.ScxFFI;
import dev.scx.ffi.type.IntRef;

import java.lang.foreign.MemorySegment;

public interface Kernel32 {

    Kernel32 KERNEL32 = ScxFFI.createFFI(Kernel32.class, "kernel32");

    MemorySegment GetStdHandle(int nStdHandle);

    int GetConsoleMode(MemorySegment hConsoleHandle, IntRef lpMode);

    int SetConsoleMode(MemorySegment hConsoleHandle, long dwMode);

}
```

调用：

```java
var handle = Kernel32.KERNEL32.GetStdHandle(-11);

var mode = new IntRef();

int ok = Kernel32.KERNEL32.GetConsoleMode(handle, mode);

if (ok != 0) {
    Kernel32.KERNEL32.SetConsoleMode(handle, mode.value());
}
```

这里 `IntRef` 用来接收 `GetConsoleMode` 写出的控制台模式。

## 完整示例：Windows User32

下面是一个 Windows `User32` 示例。

```java
import dev.scx.ffi.ScxFFI;
import dev.scx.ffi.type.EncodedString;

import java.lang.foreign.MemorySegment;

public interface User32 {

    User32 USER32 = ScxFFI.createFFI(User32.class, "user32");

    int MessageBoxA(MemorySegment hWnd, String lpText, String lpCaption, int uType);

    int MessageBoxW(MemorySegment hWnd, EncodedString lpText, EncodedString lpCaption, int uType);

    int GetCursorPos(POINT lpPoint);

    int SetCursorPos(int x, int y);

}
```

结构体：

```java
import dev.scx.ffi.type.FFIStruct;

public static class POINT implements FFIStruct {

    public int x;

    public int y;

}
```

调用 ANSI 版本：

```java
User32.USER32.MessageBoxA(
    null,
    "MessageBoxA 测试中文内容",
    "测试标题",
    0
);
```

调用 Unicode 版本：

```java
import dev.scx.ffi.type.EncodedString;

import static java.nio.charset.StandardCharsets.UTF_16LE;

User32.USER32.MessageBoxW(
    null,
    new EncodedString("MessageBoxW 测试中文内容", UTF_16LE),
    new EncodedString("测试标题", UTF_16LE),
    0
);
```

获取鼠标位置：

```java
var point = new POINT();

User32.USER32.GetCursorPos(point);

System.out.println(point.x);
System.out.println(point.y);
```

## 设计说明

### 1. Java 接口就是 native 函数声明

SCX FFI 不使用额外的 IDL 文件，也不需要手写 MethodHandle。

你只需要声明 Java 接口：

```java
public interface C {

    long strlen(String str);

}
```

然后：

```java
var c = ScxFFI.createFFI(C.class);
```

抽象方法会被映射为 native symbol。

### 2. 方法名默认就是 symbol 名

默认情况下，方法名就是 symbol 名。

```java
long strlen(String str);
```

会查找：

```text
strlen
```

如果需要修改 symbol 名，使用 `@SymbolName`。

### 3. Object 方法和默认方法不会走 FFI

`Object` 方法会直接调用代理自身的方法。

例如：

```java
toString()
hashCode()
equals(...)
```

接口默认方法也会直接作为 Java 默认方法执行。

只有抽象方法才会映射到 native symbol。

### 4. 参数先包装，再转换

调用 native 函数之前，SCX FFI 会先把参数转换为内部统一形式：

```text
基本类型
MemorySegment
FFMMapper
```

然后再把所有 `FFMMapper` 转换为 `MemorySegment`。

这样可以让不同高级类型共用同一套调用流程。

### 5. 调用结束后会回写 mapper

对于 Ref、数组、结构体等可写参数，调用结束后会从 native 内存回写。

例如：

```java
IntRef
int[]
FFIStruct
```

对于只读参数，回写方法会忽略。

例如：

```java
String
EncodedString
FFICallback
```

### 6. 每次调用使用独立 Arena

每次代理方法调用都会创建一个新的 `Arena.ofConfined()`。

这让一次调用中的临时内存能够自动释放。

但也意味着：

1. 临时字符串内存不能被 native 长期保存。
2. 临时数组内存不能被 native 长期保存。
3. 临时结构体内存不能被 native 长期保存。
4. 临时 callback stub 也只适合在本次调用期间使用。

如果 native 侧需要保存指针或回调，需要自己设计更长生命周期的内存管理方式。

### 7. SCX FFI 不隐藏 native 风险

FFI 调用本质上仍然是 native 调用。

SCX FFI 可以简化接口声明和参数转换，但不能消除下面这些风险：

1. native 函数签名写错。
2. 参数类型和真实 ABI 不匹配。
3. native 函数写越界。
4. 字符串编码错误。
5. 结构体字段顺序错误。
6. 结构体对齐和平台 ABI 不一致。
7. native 返回的指针生命周期不明确。
8. native library 不存在或 symbol 不存在。
9. 平台差异导致同一函数签名不一致。

因此使用时仍然需要对照 native API 文档确认函数签名。

## 常见问题

### SCX FFI 是 JNA 或 JNI 的替代吗？

它更接近一个基于 JDK FFM API 的轻量封装。

它不使用 JNI 代码，也不是 JNA 的完整替代品。它的目标是让简单 native 函数可以通过 Java 接口快速调用。

### FFI 接口必须是 interface 吗？

是的。

`ScxFFI.createFFI(...)` 内部使用 Java 动态代理创建对象，因此传入类型应该是接口。

### 方法名和 native 函数名不一致怎么办？

使用 `@SymbolName`。

```java
@SymbolName("abs")
int javaAbs(int x);
```

### 可以调用接口默认方法吗？

可以。

默认方法不会查找 native symbol，而是直接执行 Java 默认方法。

### 可以继承父接口吗？

可以。

父接口中的抽象方法也会被扫描并创建 native MethodHandle。

### 支持 boolean 吗？

当前不支持。

参数和返回值支持的基本类型是：

```text
byte
short
int
long
float
double
char
```

如果 native API 使用布尔值，通常可以根据平台 ABI 使用 `int` 或其它整数类型表达。

### 返回值可以是 String 吗？

当前不支持。

如果 native 函数返回字符串指针，应使用 `MemorySegment` 接收，然后由调用者按正确编码读取。

### 参数可以是 String 吗？

可以。

`String` 会被转换为 native 字符串内存。

如果需要指定编码，使用 `EncodedString`。

### String 参数会被回写吗？

不会。

Java `String` 是不可变对象，因此 `String` 参数是只读映射。

如果 native 函数需要写入字符串缓冲区，可以使用 `byte[]` 或 `char[]`。

### 如何传入 NULL 指针？

传入 `null` 即可。

SCX FFI 会把 `null` 转换为：

```java
MemorySegment.NULL
```

### 如何处理 C 的出参？

使用 Ref 类型。

例如：

```java
var out = new IntRef();

lib.get_value(out);

int value = out.value();
```

### 如何传入数组？

直接使用基本类型数组。

```java
var array = new int[]{1, 2, 3};

lib.process(array, array.length);
```

调用结束后，native 内存会回写到原数组。

### 数组会自动防止越界吗？

不会。

SCX FFI 会按 Java 数组长度分配内存，但 native 函数如果写越界，仍然是 native 侧的问题。

### 如何定义结构体？

实现 `FFIStruct`，并使用 public 非 static 字段。

```java
public static class POINT implements FFIStruct {

    public int x;

    public int y;

}
```

### 结构体字段可以是 private 吗？

不可以。

当前只处理 public 非 static 字段。

### 结构体字段顺序怎么控制？

重写 `fieldOrder()`。

```java
@Override
public String[] fieldOrder() {
    return new String[]{"x", "y"};
}
```

### 结构体支持嵌套吗？

支持非递归嵌套。

结构体字段可以继续是 `FFIStruct`。

### 结构体支持链表吗？

不直接支持递归结构体。

链表这种自引用结构应使用 `MemorySegment` 表示指针字段。

### 如何传入回调函数？

定义一个继承 `FFICallback` 的接口，并提供 `callback` 方法。

```java
public interface Compar extends FFICallback {

    int callback(MemorySegment aAddr, MemorySegment bAddr);

}
```

然后把 lambda 或实现对象传给 native 函数。

### 回调方法名必须叫 callback 吗？

默认是 `callback`。

如果不是，可以重写：

```java
default String callbackMethodName() {
    return "apply";
}
```

### 找不到 native 函数会发生什么？

创建 FFI MethodHandle 时会查找对应 symbol。

如果找不到，会抛出 `IllegalArgumentException`。

### 临时内存什么时候释放？

每次调用结束后，代理方法内部创建的 `Arena.ofConfined()` 会关闭。

因此本次调用中分配的临时字符串、数组、结构体、回调 stub 等内存都会随之失效。

### native 侧可以保存 Java 传进去的指针吗？

一般不应该保存由 SCX FFI 临时分配的指针。

这些指针只适合本次调用期间使用。

如果 native 侧需要长期保存指针，需要调用者自己管理生命周期更长的 native 内存。

### SCX FFI 会释放 native 返回的指针吗？

不会。

如果 native 函数返回 `MemorySegment`，这块内存的生命周期由 native 侧决定。SCX FFI 不会自动释放它。

### 为什么调用崩溃了？

常见原因包括：

1. 函数签名写错。
2. 参数类型和 native ABI 不一致。
3. 字符串编码不正确。
4. 结构体字段顺序不正确。
5. 结构体对齐和 native 侧不一致。
6. native 函数写越界。
7. 传入了已经失效的指针。
8. library 或 symbol 加载错误。

FFI 调用绕过了 Java 的大部分安全边界，因此签名必须和 native API 完全匹配。
