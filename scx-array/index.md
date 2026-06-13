# SCX Array

SCX Array 是一个数组工具库。

它提供 `ScxArray`，用于补充 JDK 数组操作中比较常见但写起来重复的能力，例如基本类型数组和包装类型数组互转、交换元素、反转数组、拼接数组、截取子数组、切分数组、查找元素、查找子数组、打乱数组和随机取值。

SCX Array 本身不是集合框架，也不是 `List`、`Set`、`Map` 的替代品。它只处理 Java 数组，尤其是 Java 中比较麻烦的基本类型数组。

当前版本为 `0.0.1`。

[GitHub](https://github.com/scx-projects/scx-array)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-array</artifactId>
    <version>0.0.1</version>
</dependency>
```

## 基本概念

SCX Array 当前只有一个核心类：

```text
ScxArray    数组工具类
```

它提供的能力可以分为几类：

```text
toPrimitive    包装类型数组转基本类型数组
toWrapper      基本类型数组转包装类型数组
swap           交换数组中的两个元素
reverse        原地反转数组
concat         拼接多个数组
subArray       截取子数组
splitArray     按固定长度切分数组
indexOf        查找元素或子数组
shuffle        原地随机打乱数组
randomGet      从数组中随机获取一个元素
```

这些方法大多同时支持：

```text
byte
short
int
long
float
double
boolean
char
Object
```

其中，`toPrimitive(...)` 和 `toWrapper(...)` 用于基本类型和包装类型互转。

其它数组操作通常同时支持基本类型数组和对象数组。

## 快速开始

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var a = new int[]{1, 2, 3};
var b = new int[]{4, 5};

var c = ScxArray.concat(a, b);

System.out.println(Arrays.toString(c));
```

输出：

```text
[1, 2, 3, 4, 5]
```

反转数组：

```java
var array = new int[]{1, 2, 3, 4, 5};

ScxArray.reverse(array);

System.out.println(Arrays.toString(array));
```

输出：

```text
[5, 4, 3, 2, 1]
```

截取子数组：

```java
var array = new int[]{1, 2, 3, 4, 5};

var subArray = ScxArray.subArray(array, 1, 4);

System.out.println(Arrays.toString(subArray));
```

输出：

```text
[2, 3, 4]
```

查找子数组：

```java
var array = new byte[]{1, 2, 3, 4, 5, 6};

var index = ScxArray.indexOf(array, new byte[]{3, 4, 5});

System.out.println(index);
```

输出：

```text
2
```

## toPrimitive

`toPrimitive(...)` 用于把包装类型数组转换为基本类型数组。

支持的转换包括：

```text
Byte[]       -> byte[]
Short[]      -> short[]
Integer[]    -> int[]
Long[]       -> long[]
Float[]      -> float[]
Double[]     -> double[]
Boolean[]    -> boolean[]
Character[]  -> char[]
```

示例：

```java
import dev.scx.array.ScxArray;

Integer[] wrapperArray = {1, 2, 3};

int[] primitiveArray = ScxArray.toPrimitive(wrapperArray);
```

结果：

```text
[1, 2, 3]
```

也可以直接使用可变参数：

```java
int[] array = ScxArray.toPrimitive(1, 2, 3, 4, 5);
```

因为方法签名是：

```java
public static int[] toPrimitive(Integer... w)
```

所以数组和可变参数都可以传入。

### 空数组

如果传入空数组，会返回对应类型的空数组。

```java
Integer[] source = {};

int[] result = ScxArray.toPrimitive(source);
```

结果长度为：

```text
0
```

### null 元素

需要注意，包装类型数组中的元素不能为 `null`。

例如：

```java
Integer[] source = {1, null, 3};

int[] result = ScxArray.toPrimitive(source);
```

这里在拆箱时会抛出 `NullPointerException`。

这是 Java 自动拆箱的正常行为。

## toWrapper

`toWrapper(...)` 用于把基本类型数组转换为包装类型数组。

支持的转换包括：

```text
byte[]     -> Byte[]
short[]    -> Short[]
int[]      -> Integer[]
long[]     -> Long[]
float[]    -> Float[]
double[]   -> Double[]
boolean[]  -> Boolean[]
char[]     -> Character[]
```

示例：

```java
import dev.scx.array.ScxArray;

int[] primitiveArray = {1, 2, 3};

Integer[] wrapperArray = ScxArray.toWrapper(primitiveArray);
```

结果：

```text
[1, 2, 3]
```

也可以直接使用可变参数：

```java
Integer[] array = ScxArray.toWrapper(1, 2, 3, 4, 5);
```

因为方法签名是：

```java
public static Integer[] toWrapper(int... p)
```

所以数组和可变参数都可以传入。

## swap

`swap(...)` 用于交换数组中的两个元素。

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var array = new int[]{1, 2, 3, 4};

ScxArray.swap(array, 0, 3);

System.out.println(Arrays.toString(array));
```

输出：

```text
[4, 2, 3, 1]
```

对象数组也可以使用：

```java
var array = new String[]{"a", "b", "c"};

ScxArray.swap(array, 0, 2);

System.out.println(Arrays.toString(array));
```

输出：

```text
[c, b, a]
```

`swap(...)` 是原地操作。

它不会创建新数组，而是直接修改传入的数组。

支持的类型包括：

```text
byte[]
short[]
int[]
long[]
float[]
double[]
boolean[]
char[]
Object[]
```

### 索引检查

`swap(...)` 不做额外索引检查。

如果索引越界，会由 Java 数组访问本身抛出异常。

```java
var array = new int[]{1, 2, 3};

ScxArray.swap(array, 0, 10);
```

这会抛出：

```text
ArrayIndexOutOfBoundsException
```

## reverse

`reverse(...)` 用于原地反转数组。

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var array = new int[]{1, 2, 3, 4, 5};

ScxArray.reverse(array);

System.out.println(Arrays.toString(array));
```

输出：

```text
[5, 4, 3, 2, 1]
```

对象数组：

```java
var array = new String[]{"a", "b", "c"};

ScxArray.reverse(array);

System.out.println(Arrays.toString(array));
```

输出：

```text
[c, b, a]
```

`reverse(...)` 内部通过前后元素交换实现。

对于长度为奇数的数组，中间元素不会移动。

例如：

```text
[1, 2, 3, 4, 5]
```

反转后：

```text
[5, 4, 3, 2, 1]
```

其中 `3` 仍然位于中间。

### reverse 是原地操作

`reverse(...)` 不会返回新数组。

```java
var array = new int[]{1, 2, 3};

ScxArray.reverse(array);
```

调用后，`array` 本身已经被修改。

如果你希望保留原数组，可以先复制：

```java
var source = new int[]{1, 2, 3};

var copy = Arrays.copyOf(source, source.length);

ScxArray.reverse(copy);
```

### 可变参数调用

`reverse(...)` 的签名使用可变参数。

例如：

```java
public static void reverse(int... arr)
```

所以可以这样调用：

```java
ScxArray.reverse(1, 2, 3);
```

但这种写法没有意义，因为传入的是临时数组，方法内部反转后外部拿不到这个数组。

推荐写法是传入已有数组：

```java
var array = new int[]{1, 2, 3};

ScxArray.reverse(array);
```

## concat

`concat(...)` 用于拼接多个数组，并返回一个新数组。

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var a = new int[]{1, 2};
var b = new int[]{3, 4};
var c = new int[]{5};

var result = ScxArray.concat(a, b, c);

System.out.println(Arrays.toString(result));
```

输出：

```text
[1, 2, 3, 4, 5]
```

对象数组：

```java
var a = new String[]{"a", "b"};
var b = new String[]{"c"};

var result = ScxArray.concat(a, b);

System.out.println(Arrays.toString(result));
```

输出：

```text
[a, b, c]
```

支持的类型包括：

```text
byte[]
short[]
int[]
long[]
float[]
double[]
boolean[]
char[]
T[]
```

### concat 会创建新数组

`concat(...)` 不会修改原数组。

```java
var a = new int[]{1, 2};
var b = new int[]{3, 4};

var result = ScxArray.concat(a, b);
```

调用后：

```text
a      = [1, 2]
b      = [3, 4]
result = [1, 2, 3, 4]
```

### 拼接空数组

可以拼接空数组。

```java
var a = new int[]{};
var b = new int[]{1, 2};

var result = ScxArray.concat(a, b);
```

结果：

```text
[1, 2]
```

如果所有数组都是空数组，结果也是空数组。

```java
var result = ScxArray.concat(new int[]{}, new int[]{});
```

结果长度为：

```text
0
```

### 对象数组的组件类型

对象数组版本是：

```java
public static <T> T[] concat(T[]... arrays)
```

它会根据传入的二维数组参数推断结果数组的组件类型。

例如：

```java
Number[] a = new Integer[]{1, 2};
Number[] b = new Double[]{3.14};

Number[] result = ScxArray.concat(a, b);
```

结果数组的组件类型会按照调用点推断出的数组类型创建。

需要注意，Java 数组是协变的，所以对象数组拼接时仍然要遵守 Java 数组本身的类型规则。

## subArray

`subArray(...)` 用于截取子数组。

索引规则和 JDK 中常见的 `fromIndex` / `toIndex` 规则一致：

```text
fromIndex    起始索引，包含
toIndex      结束索引，不包含
```

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var array = new int[]{1, 2, 3, 4, 5};

var result = ScxArray.subArray(array, 1, 4);

System.out.println(Arrays.toString(result));
```

输出：

```text
[2, 3, 4]
```

因为截取范围是：

```text
[1, 4)
```

也就是索引 `1`、`2`、`3`。

### 截取整个数组

```java
var array = new long[]{1L, 2L, 3L};

var result = ScxArray.subArray(array, 0, array.length);
```

结果：

```text
[1, 2, 3]
```

需要注意，返回的是新数组，不是原数组本身。

### 截取空数组

当 `fromIndex == toIndex` 时，会返回空数组。

```java
var array = new int[]{1, 2, 3};

var result = ScxArray.subArray(array, 1, 1);
```

结果长度为：

```text
0
```

### 索引检查

`subArray(...)` 会检查索引范围。

下面这些情况都会抛出 `ArrayIndexOutOfBoundsException`：

```text
fromIndex < 0
toIndex > array.length
fromIndex > toIndex
```

示例：

```java
var array = new byte[]{1, 2, 3, 4, 5};

ScxArray.subArray(array, -1, 10);
```

这会抛出异常。

### 支持的类型

`subArray(...)` 支持：

```text
byte[]
short[]
int[]
long[]
float[]
double[]
boolean[]
char[]
T[]
```

对象数组示例：

```java
var array = new String[]{"a", "b", "c", "d"};

var result = ScxArray.subArray(array, 1, 3);
```

结果：

```text
[b, c]
```

对象数组版本会保留原数组的组件类型。

## splitArray

`splitArray(...)` 用于按照固定长度切分数组。

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var array = new int[]{1, 2, 3, 4, 5};

var result = ScxArray.splitArray(array, 2);

System.out.println(Arrays.deepToString(result));
```

输出：

```text
[[1, 2], [3, 4], [5]]
```

也就是说，`sliceSize` 表示每个子数组的最大长度。

如果最后剩余元素不足 `sliceSize`，最后一个子数组会更短。

### 切分数量

切分数量使用向上取整。

```text
numOfSlices = (array.length + sliceSize - 1) / sliceSize
```

例如：

```java
var array = new int[1001];

var result = ScxArray.splitArray(array, 9);
```

结果：

```text
result.length = 112
最后一个子数组长度 = 2
```

因为：

```text
1001 / 9 = 111 余 2
```

所以需要 `112` 个子数组。

### sliceSize 必须大于 0

如果 `sliceSize <= 0`，会抛出 `IllegalArgumentException`。

```java
var array = new int[]{1, 2, 3};

ScxArray.splitArray(array, 0);
```

异常信息类似：

```text
sliceSize must be > 0 : 0
```

### 空数组

如果传入空数组，结果是长度为 `0` 的二维数组。

```java
var array = new int[]{};

var result = ScxArray.splitArray(array, 3);
```

结果：

```text
[]
```

### 支持的类型

`splitArray(...)` 支持：

```text
byte[]
short[]
int[]
long[]
float[]
double[]
boolean[]
char[]
T[]
```

对象数组示例：

```java
var array = new String[]{"a", "b", "c", "d", "e"};

var result = ScxArray.splitArray(array, 2);
```

结果：

```text
[[a, b], [c, d], [e]]
```

## indexOf 元素

`indexOf(...)` 可以查找单个元素第一次出现的位置。

示例：

```java
import dev.scx.array.ScxArray;

var array = new int[]{10, 20, 30, 20};

var index = ScxArray.indexOf(array, 20);

System.out.println(index);
```

输出：

```text
1
```

如果没有找到，返回：

```text
-1
```

示例：

```java
var array = new int[]{10, 20, 30};

var index = ScxArray.indexOf(array, 99);
```

结果：

```text
-1
```

### 指定查找范围

也可以指定查找范围。

```java
var array = new int[]{10, 20, 30, 20};

var index = ScxArray.indexOf(array, 2, array.length, 20);
```

结果：

```text
3
```

范围规则是：

```text
startIndex    起始索引，包含
endIndex      结束索引，不包含
```

也就是说，查找范围是：

```text
[startIndex, endIndex)
```

### 对象数组查找

对象数组使用 `Objects.equals(...)` 判断是否相等。

```java
var array = new String[]{"a", null, "c"};

var index1 = ScxArray.indexOf(array, null);
var index2 = ScxArray.indexOf(array, "c");
```

结果：

```text
index1 = 1
index2 = 2
```

因为使用 `Objects.equals(...)`，所以可以正确处理 `null`。

### 基本类型数组查找

基本类型数组使用 `==` 判断是否相等。

这意味着：

```java
var array = new double[]{Double.NaN};

var index = ScxArray.indexOf(array, Double.NaN);
```

结果是：

```text
-1
```

原因是 Java 中：

```java
Double.NaN == Double.NaN
```

结果为：

```text
false
```

这不是 SCX Array 的特殊规则，而是 Java 浮点数比较规则。

## indexOf 子数组

`indexOf(...)` 也可以查找一个子数组第一次出现的位置。

示例：

```java
import dev.scx.array.ScxArray;

var array = new byte[]{1, 2, 3, 4, 5, 6};

var index = ScxArray.indexOf(array, new byte[]{3, 4, 5});

System.out.println(index);
```

输出：

```text
2
```

如果没有找到，返回：

```text
-1
```

```java
var array = new byte[]{1, 2, 3, 4, 5, 6};

var index = ScxArray.indexOf(array, new byte[]{5, 8});
```

结果：

```text
-1
```

### 指定范围查找子数组

可以指定查找范围。

```java
var array = new int[]{1, 2, 3, 4, 5, 3, 4};

var index = ScxArray.indexOf(array, 3, array.length, new int[]{3, 4});
```

结果：

```text
5
```

范围仍然是：

```text
[startIndex, endIndex)
```

### 子数组匹配规则

子数组匹配要求连续相等。

例如：

```java
var array = new int[]{1, 2, 3, 4, 5};

ScxArray.indexOf(array, new int[]{2, 3, 4});
```

结果为：

```text
1
```

但是：

```java
ScxArray.indexOf(array, new int[]{2, 4});
```

结果为：

```text
-1
```

因为 `2` 和 `4` 在原数组中不是连续出现。

### 对象子数组

对象数组查找子数组时，同样使用 `Objects.equals(...)`。

```java
var array = new String[]{"a", null, "c", "d"};

var index = ScxArray.indexOf(array, new String[]{null, "c"});
```

结果：

```text
1
```

### 空子数组

如果查找的子数组长度为 `0`，当前实现会返回 `startIndex`。

例如：

```java
var array = new int[]{1, 2, 3};

var index = ScxArray.indexOf(array, 0, array.length, new int[]{});
```

结果：

```text
0
```

这是由当前匹配循环自然得到的结果。

## shuffle

`shuffle(...)` 用于原地随机打乱数组。

示例：

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;

var array = new int[]{1, 2, 3, 4, 5};

ScxArray.shuffle(array);

System.out.println(Arrays.toString(array));
```

输出结果是不确定的，例如：

```text
[3, 1, 5, 2, 4]
```

`shuffle(...)` 是原地操作。

它不会返回新数组，而是直接修改传入数组。

### 指定 RandomGenerator

可以传入自己的 `RandomGenerator`。

```java
import java.util.Random;
import dev.scx.array.ScxArray;

var array = new int[]{1, 2, 3, 4, 5};

var random = new Random(100);

ScxArray.shuffle(array, random);
```

这适合下面这些场景：

1. 测试中希望结果可复现。
2. 希望使用指定随机数实现。
3. 希望统一管理随机源。
4. 不希望使用默认的 `ThreadLocalRandom.current()`。

### 默认随机源

不传入 `RandomGenerator` 时，会使用：

```java
ThreadLocalRandom.current()
```

示例：

```java
ScxArray.shuffle(array);
```

大致等价于：

```java
ScxArray.shuffle(array, ThreadLocalRandom.current());
```

### 支持的类型

`shuffle(...)` 支持：

```text
byte[]
short[]
int[]
long[]
float[]
double[]
boolean[]
char[]
Object[]
```

对象数组示例：

```java
var array = new String[]{"a", "b", "c", "d"};

ScxArray.shuffle(array);
```

## randomGet

`randomGet(...)` 用于从数组中随机获取一个元素。

示例：

```java
import dev.scx.array.ScxArray;

var array = new int[]{10, 20, 30};

var value = ScxArray.randomGet(array);

System.out.println(value);
```

输出可能是：

```text
10
```

也可能是：

```text
20
```

或者：

```text
30
```

### 指定 RandomGenerator

可以传入自己的 `RandomGenerator`。

```java
import dev.scx.array.ScxArray;

import java.util.Random;

var array = new String[]{"a", "b", "c"};

var random = new Random(100);

var value = ScxArray.randomGet(array, random);
```

这适合测试或需要可复现随机结果的场景。

### 默认随机源

不传入 `RandomGenerator` 时，会使用：

```java
ThreadLocalRandom.current()
```

示例：

```java
var value = ScxArray.randomGet(array);
```

大致等价于：

```java
var value = ScxArray.randomGet(array, ThreadLocalRandom.current());
```

### 空数组

如果数组长度为 `0`，会抛出异常。

```java
var array = new int[]{};

var value = ScxArray.randomGet(array);
```

原因是内部会调用：

```java
random.nextInt(arr.length)
```

当 `arr.length` 为 `0` 时，随机范围非法。

## 方法总览

### 包装类型转基本类型

```java
byte[] toPrimitive(Byte... w)

short[] toPrimitive(Short... w)

int[] toPrimitive(Integer... w)

long[] toPrimitive(Long... w)

float[] toPrimitive(Float... w)

double[] toPrimitive(Double... w)

boolean[] toPrimitive(Boolean... w)

char[] toPrimitive(Character... w)
```

### 基本类型转包装类型

```java
Byte[] toWrapper(byte... p)

Short[] toWrapper(short... p)

Integer[] toWrapper(int... p)

Long[] toWrapper(long... p)

Float[] toWrapper(float... p)

Double[] toWrapper(double... p)

Boolean[] toWrapper(boolean... p)

Character[] toWrapper(char... p)
```

### 交换元素

```java
void swap(byte[] arr, int i, int j)

void swap(short[] arr, int i, int j)

void swap(int[] arr, int i, int j)

void swap(long[] arr, int i, int j)

void swap(float[] arr, int i, int j)

void swap(double[] arr, int i, int j)

void swap(boolean[] arr, int i, int j)

void swap(char[] arr, int i, int j)

void swap(Object[] arr, int i, int j)
```

### 反转数组

```java
void reverse(byte... arr)

void reverse(short... arr)

void reverse(int... arr)

void reverse(long... arr)

void reverse(float... arr)

void reverse(double... arr)

void reverse(boolean... arr)

void reverse(char... arr)

void reverse(Object... arr)
```

### 拼接数组

```java
byte[] concat(byte[]... arrays)

short[] concat(short[]... arrays)

int[] concat(int[]... arrays)

long[] concat(long[]... arrays)

float[] concat(float[]... arrays)

double[] concat(double[]... arrays)

boolean[] concat(boolean[]... arrays)

char[] concat(char[]... arrays)

<T> T[] concat(T[]... arrays)
```

### 截取子数组

```java
byte[] subArray(byte[] array, int fromIndex, int toIndex)

short[] subArray(short[] array, int fromIndex, int toIndex)

int[] subArray(int[] array, int fromIndex, int toIndex)

long[] subArray(long[] array, int fromIndex, int toIndex)

float[] subArray(float[] array, int fromIndex, int toIndex)

double[] subArray(double[] array, int fromIndex, int toIndex)

boolean[] subArray(boolean[] array, int fromIndex, int toIndex)

char[] subArray(char[] array, int fromIndex, int toIndex)

<T> T[] subArray(T[] array, int fromIndex, int toIndex)
```

### 切分数组

```java
byte[][] splitArray(byte[] arr, int sliceSize)

short[][] splitArray(short[] arr, int sliceSize)

int[][] splitArray(int[] arr, int sliceSize)

long[][] splitArray(long[] arr, int sliceSize)

float[][] splitArray(float[] arr, int sliceSize)

double[][] splitArray(double[] arr, int sliceSize)

boolean[][] splitArray(boolean[] arr, int sliceSize)

char[][] splitArray(char[] arr, int sliceSize)

<T> T[][] splitArray(T[] arr, int sliceSize)
```

### 查找单个元素

```java
int indexOf(byte[] a, int startIndex, int endIndex, byte b)

int indexOf(short[] a, int startIndex, int endIndex, short b)

int indexOf(int[] a, int startIndex, int endIndex, int b)

int indexOf(long[] a, int startIndex, int endIndex, long b)

int indexOf(float[] a, int startIndex, int endIndex, float b)

int indexOf(double[] a, int startIndex, int endIndex, double b)

int indexOf(boolean[] a, int startIndex, int endIndex, boolean b)

int indexOf(char[] a, int startIndex, int endIndex, char b)

int indexOf(Object[] a, int startIndex, int endIndex, Object b)
```

简化版本：

```java
int indexOf(byte[] a, byte b)

int indexOf(short[] a, short b)

int indexOf(int[] a, int b)

int indexOf(long[] a, long b)

int indexOf(float[] a, float b)

int indexOf(double[] a, double b)

int indexOf(boolean[] a, boolean b)

int indexOf(char[] a, char b)

int indexOf(Object[] a, Object b)
```

### 查找子数组

```java
int indexOf(byte[] a, int startIndex, int endIndex, byte... b)

int indexOf(short[] a, int startIndex, int endIndex, short... b)

int indexOf(int[] a, int startIndex, int endIndex, int... b)

int indexOf(long[] a, int startIndex, int endIndex, long... b)

int indexOf(float[] a, int startIndex, int endIndex, float... b)

int indexOf(double[] a, int startIndex, int endIndex, double... b)

int indexOf(boolean[] a, int startIndex, int endIndex, boolean... b)

int indexOf(char[] a, int startIndex, int endIndex, char... b)

int indexOf(Object[] a, int startIndex, int endIndex, Object... b)
```

简化版本：

```java
int indexOf(byte[] a, byte... b)

int indexOf(short[] a, short... b)

int indexOf(int[] a, int... b)

int indexOf(long[] a, long... b)

int indexOf(float[] a, float... b)

int indexOf(double[] a, double... b)

int indexOf(boolean[] a, boolean... b)

int indexOf(char[] a, char... b)

int indexOf(Object[] a, Object... b)
```

### 打乱数组

```java
void shuffle(byte[] arr, RandomGenerator random)

void shuffle(short[] arr, RandomGenerator random)

void shuffle(int[] arr, RandomGenerator random)

void shuffle(long[] arr, RandomGenerator random)

void shuffle(float[] arr, RandomGenerator random)

void shuffle(double[] arr, RandomGenerator random)

void shuffle(boolean[] arr, RandomGenerator random)

void shuffle(char[] arr, RandomGenerator random)

void shuffle(Object[] arr, RandomGenerator random)
```

简化版本：

```java
void shuffle(byte... arr)

void shuffle(short... arr)

void shuffle(int... arr)

void shuffle(long... arr)

void shuffle(float... arr)

void shuffle(double... arr)

void shuffle(boolean... arr)

void shuffle(char... arr)

void shuffle(Object... arr)
```

### 随机获取元素

```java
byte randomGet(byte[] arr, RandomGenerator random)

short randomGet(short[] arr, RandomGenerator random)

int randomGet(int[] arr, RandomGenerator random)

long randomGet(long[] arr, RandomGenerator random)

float randomGet(float[] arr, RandomGenerator random)

double randomGet(double[] arr, RandomGenerator random)

boolean randomGet(boolean[] arr, RandomGenerator random)

char randomGet(char[] arr, RandomGenerator random)

<T> T randomGet(T[] arr, RandomGenerator random)
```

简化版本：

```java
byte randomGet(byte... arr)

short randomGet(short... arr)

int randomGet(int... arr)

long randomGet(long... arr)

float randomGet(float... arr)

double randomGet(double... arr)

boolean randomGet(boolean... arr)

char randomGet(char... arr)

<T> T randomGet(T... arr)
```

## 完整示例

下面是一个完整示例。

```java
import dev.scx.array.ScxArray;

import java.util.Arrays;
import java.util.Random;

public class ArrayDemo {

    public static void main(String[] args) {
        Integer[] wrapperArray = {1, 2, 3};

        int[] primitiveArray = ScxArray.toPrimitive(wrapperArray);

        int[] more = {4, 5, 6};

        int[] all = ScxArray.concat(primitiveArray, more);

        System.out.println(Arrays.toString(all));

        int[] subArray = ScxArray.subArray(all, 1, 5);

        System.out.println(Arrays.toString(subArray));

        int[][] slices = ScxArray.splitArray(all, 2);

        System.out.println(Arrays.deepToString(slices));

        int index = ScxArray.indexOf(all, new int[]{3, 4});

        System.out.println(index);

        ScxArray.reverse(all);

        System.out.println(Arrays.toString(all));

        ScxArray.shuffle(all, new Random(100));

        System.out.println(Arrays.toString(all));

        int value = ScxArray.randomGet(all);

        System.out.println(value);
    }

}
```

可能输出：

```text
[1, 2, 3, 4, 5, 6]
[2, 3, 4, 5]
[[1, 2], [3, 4], [5, 6]]
2
[6, 5, 4, 3, 2, 1]
[...]
...
```

其中 `shuffle(...)` 和 `randomGet(...)` 的结果取决于随机数。

## 设计说明

### 1. SCX Array 是纯静态工具类

`ScxArray` 是一个工具类。

所有能力都通过静态方法提供。

```java
ScxArray.concat(...)

ScxArray.reverse(...)

ScxArray.indexOf(...)
```

通常不需要创建 `ScxArray` 实例。

### 2. 优先支持基本类型数组

Java 的泛型不能直接处理基本类型数组。

例如：

```java
int[]
long[]
double[]
```

不能作为 `T[]` 的普通泛型数组处理。

所以 SCX Array 为每一种基本类型都提供了独立重载。

这也是为什么 API 中会有很多看起来相似的重载方法。

### 3. 修改类方法和返回类方法要区分

下面这些方法会修改原数组：

```text
swap
reverse
shuffle
```

下面这些方法会返回新数组：

```text
toPrimitive
toWrapper
concat
subArray
splitArray
```

下面这些方法不修改数组，只读取数组：

```text
indexOf
randomGet
```

使用时要注意这几类方法的区别。

### 4. fromIndex / toIndex 使用左闭右开

`subArray(...)` 和带范围的 `indexOf(...)` 都使用左闭右开区间。

```text
[fromIndex, toIndex)
```

这和 JDK 中很多 API 的习惯一致。

例如：

```java
ScxArray.subArray(array, 1, 4)
```

表示包含索引 `1`，不包含索引 `4`。

### 5. Object 数组使用 Objects.equals

对象数组的 `indexOf(...)` 使用 `Objects.equals(...)` 判断元素是否相等。

因此它可以处理 `null`。

```java
ScxArray.indexOf(new String[]{"a", null}, null)
```

结果是：

```text
1
```

### 6. 基本类型数组使用 ==

基本类型数组的查找使用 `==` 判断。

这对大多数基本类型来说是直观的。

但对 `float` 和 `double` 需要注意 `NaN`。

```java
Double.NaN == Double.NaN
```

结果是：

```text
false
```

因此 `indexOf(doubleArray, Double.NaN)` 不会匹配到 `NaN`。

### 7. 不做复杂算法优化

子数组查找使用的是直接匹配算法。

它会从起始位置开始，逐个尝试匹配子数组。

这种实现简单直接，适合一般数组工具场景。

如果需要在大数组中频繁查找长模式，可以根据业务场景选择更专门的算法。

### 8. shuffle 使用原地交换

`shuffle(...)` 使用原地交换实现。

它不会创建新数组。

如果需要保留原数组，应先复制：

```java
var copy = Arrays.copyOf(source, source.length);

ScxArray.shuffle(copy);
```

### 9. randomGet 不处理空数组

`randomGet(...)` 要求数组至少有一个元素。

空数组无法随机取值，因此会抛出异常。

调用方应该在业务层提前判断：

```java
if (array.length > 0) {
    var value = ScxArray.randomGet(array);
}
```

## 常见问题

### SCX Array 是集合库吗？

不是。

SCX Array 只处理 Java 数组。

如果你需要 `List`、`Set`、`Map` 等集合能力，应该使用 JDK 集合框架或其它集合工具库。

### 为什么有这么多重载方法？

因为 Java 的基本类型数组不能用普通泛型统一处理。

例如：

```java
int[]
```

不是：

```java
Integer[]
```

也不能作为普通的：

```java
T[]
```

来处理。

所以需要为 `byte[]`、`short[]`、`int[]` 等类型分别提供重载。

### reverse 会返回新数组吗？

不会。

`reverse(...)` 是原地操作。

```java
var array = new int[]{1, 2, 3};

ScxArray.reverse(array);
```

调用后，`array` 本身变成：

```text
[3, 2, 1]
```

### shuffle 会返回新数组吗？

不会。

`shuffle(...)` 也是原地操作。

如果需要新数组，应先复制原数组。

### concat 会修改原数组吗？

不会。

`concat(...)` 会创建并返回一个新数组。

### subArray 会修改原数组吗？

不会。

`subArray(...)` 会创建并返回一个新数组。

### splitArray 会修改原数组吗？

不会。

`splitArray(...)` 会创建新的二维数组，并复制每个切片。

### indexOf 找不到时返回什么？

返回：

```text
-1
```

### indexOf 的 endIndex 包含吗？

不包含。

查找范围是：

```text
[startIndex, endIndex)
```

### subArray 的 toIndex 包含吗？

不包含。

截取范围是：

```text
[fromIndex, toIndex)
```

### splitArray 的 sliceSize 可以是 0 吗？

不可以。

`sliceSize` 必须大于 `0`。

否则会抛出 `IllegalArgumentException`。

### randomGet 可以用于空数组吗？

不可以。

空数组没有可随机获取的元素，因此会抛出异常。

### 对象数组查找支持 null 吗？

支持。

对象数组使用 `Objects.equals(...)`，所以可以查找 `null`。

### 基本类型数组查找支持 NaN 吗？

`float[]` 和 `double[]` 使用 `==` 比较。

由于 Java 中 `NaN == NaN` 为 `false`，所以不能通过当前 `indexOf(...)` 查找到 `NaN`。

### toPrimitive 可以处理 null 元素吗？

不可以。

包装类型拆箱时，如果元素为 `null`，会抛出 `NullPointerException`。

### toWrapper 会产生 null 元素吗？

不会。

基本类型数组中的每个值都会被装箱为对应包装类型对象。

### concat 没有参数时能用吗？

对于基本类型数组，不推荐无参数调用，因为编译器可能无法推断你想要的数组类型。

推荐明确传入至少一个数组。

```java
var result = ScxArray.concat(new int[]{});
```

### 对象数组 concat 会保留类型吗？

会根据传入参数推断出来的数组组件类型创建结果数组。

例如：

```java
String[] a = {"a"};
String[] b = {"b"};

String[] result = ScxArray.concat(a, b);
```

结果仍然是 `String[]`。

### 什么时候用 SCX Array？

适合下面这些场景：

1. 需要频繁操作基本类型数组。
2. 想把 `Integer[]` 转成 `int[]`。
3. 想把 `int[]` 转成 `Integer[]`。
4. 想快速拼接多个数组。
5. 想截取数组的一部分。
6. 想按固定大小切分数组。
7. 想查找一个子数组。
8. 想原地反转或打乱数组。
9. 想从数组中随机取一个元素。
