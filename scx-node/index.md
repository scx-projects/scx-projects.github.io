# SCX Node

SCX Node 是一套中立的、平台无关的通用数据模型。

它提供一组简单的 `Node` 类型，用来表达对象、数组、字符串、数字、布尔值和 null。

SCX Node 本身不负责 JSON 解析，不负责 XML 解析，也不负责 Java 对象绑定。它只提供一个格式无关的数据树模型，供其它模块复用。

例如：

```text
JSON       -> Node
XML        -> Node
Java Bean  -> Node
Node       -> JSON
Node       -> XML
Node       -> Java Bean
```

当前版本为 `0.1.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-node</artifactId>
    <version>0.1.0</version>
</dependency>
```

## 基本概念

SCX Node 中最核心的概念包括：

```text
Node             所有节点的顶层接口
ValueNode        值节点
NumberNode       数字节点
ContainerNode    容器节点
ObjectNode       对象节点
ArrayNode        数组节点
StringNode       字符串节点
BooleanNode      布尔节点
NullNode         null 节点
```

类型层级可以简单理解为：

```text
Node
├── ValueNode
│   ├── NumberNode
│   │   ├── IntNode
│   │   ├── LongNode
│   │   ├── FloatNode
│   │   ├── DoubleNode
│   │   ├── BigIntegerNode
│   │   └── BigDecimalNode
│   ├── StringNode
│   └── BooleanNode
├── ContainerNode
│   ├── ObjectNode
│   └── ArrayNode
└── NullNode
```

也就是说：

```text
ObjectNode / ArrayNode 是可变容器
ValueNode / NullNode 是不可变值
```

## 快速开始

创建一个对象节点：

```java
import dev.scx.node.ArrayNode;
import dev.scx.node.ObjectNode;

import static dev.scx.node.NullNode.NULL;

var user = new ObjectNode();

user.put("id", 1);
user.put("name", "Tom");
user.put("age", 18);
user.put("active", true);
user.put("parent", NULL);

var tags = new ArrayNode();

tags.add("java");
tags.add("scx");

user.put("tags", tags);

System.out.println(user);
```

输出大致是：

```json
{
 "id": 1,
 "name": "Tom",
 "age": 18,
 "active": true,
 "parent": null,
 "tags": [
  "java",
  "scx"
 ]
}
```

需要注意，`toString()` 采用的是类 JSON 格式，主要用于调试和展示。

如果需要标准 JSON 序列化，应使用 `scx-format-json`。

## Node

`Node` 是所有节点的顶层接口。

它是一个 sealed interface，只允许下面几类节点实现：

```text
ValueNode
ContainerNode
NullNode
```

接口定义可以理解为：

```java
public sealed interface Node permits ValueNode, ContainerNode, NullNode {

    Node deepCopy();

}
```

`deepCopy()` 用于深拷贝。

不同节点的行为不同：

```text
ContainerNode    返回结构完全独立的副本
ValueNode        返回自身
NullNode         返回自身
```

原因是：

```text
ObjectNode / ArrayNode 是可变的
ValueNode / NullNode 是不可变的
```

## ContainerNode

`ContainerNode` 表示容器节点。

它有两个实现：

```text
ObjectNode
ArrayNode
```

接口可以理解为：

```java
public sealed interface ContainerNode extends Node permits ArrayNode, ObjectNode {

    int size();

    boolean isEmpty();

    void clear();

    @Override
    ContainerNode deepCopy();

}
```

容器节点是可变的。

因此 `deepCopy()` 会递归复制内部结构。

## ObjectNode

`ObjectNode` 表示对象节点。

它内部使用 `LinkedHashMap<String, Node>` 保存字段。

因此字段顺序会按插入顺序保留。

创建对象节点：

```java
var objectNode = new ObjectNode();
```

设置字段：

```java
objectNode.put("name", new StringNode("Tom"));
objectNode.put("age", new IntNode(18));
```

也可以使用便捷方法：

```java
objectNode.put("name", "Tom");
objectNode.put("age", 18);
objectNode.put("active", true);
```

获取字段：

```java
Node name = objectNode.get("name");
```

删除字段：

```java
Node old = objectNode.remove("name");
```

遍历字段：

```java
for (var field : objectNode) {
    System.out.println(field.getKey());
    System.out.println(field.getValue());
}
```

## ObjectNode 的 null 规则

`ObjectNode` 不允许字段名为 `null`。

```java
objectNode.put(null, "Tom");
```

会抛出：

```java
NullPointerException
```

`ObjectNode` 也不允许字段值为 Java `null`。

```java
objectNode.put("name", (Node) null);
```

会抛出：

```java
NullPointerException
```

如果要表示数据层面的 null，应使用：

```java
import static dev.scx.node.NullNode.NULL;

objectNode.put("name", NULL);
```

也就是说：

```text
Java null       表示没有对象
NullNode.NULL   表示数据中的 null
```

## ObjectNode 便捷 put 方法

`ObjectNode` 提供了一组便捷方法。

```java
objectNode.put("i", 1);
objectNode.put("l", 1L);
objectNode.put("f", 1.0F);
objectNode.put("d", 1.0D);
objectNode.put("bigInt", new BigInteger("123456789"));
objectNode.put("bigDecimal", new BigDecimal("123.45"));
objectNode.put("str", "hello");
objectNode.put("bool", true);
```

它们会自动包装成对应的节点：

```text
int         -> IntNode
long        -> LongNode
float       -> FloatNode
double      -> DoubleNode
BigInteger  -> BigIntegerNode
BigDecimal  -> BigDecimalNode
String      -> StringNode
boolean     -> BooleanNode
```

如果要放入 `null`，不要传 Java `null`，而是使用：

```java
objectNode.put("x", NullNode.NULL);
```

## ArrayNode

`ArrayNode` 表示数组节点。

它内部使用 `ArrayList<Node>` 保存元素。

创建数组节点：

```java
var arrayNode = new ArrayNode();
```

添加元素：

```java
arrayNode.add(new StringNode("Tom"));
arrayNode.add(new IntNode(18));
```

也可以使用便捷方法：

```java
arrayNode.add("Tom");
arrayNode.add(18);
arrayNode.add(true);
```

按索引插入：

```java
arrayNode.add(0, new StringNode("first"));
```

按索引替换：

```java
arrayNode.set(0, new StringNode("new value"));
```

按索引读取：

```java
Node first = arrayNode.get(0);
```

按索引删除：

```java
Node old = arrayNode.remove(0);
```

遍历数组：

```java
for (var item : arrayNode) {
    System.out.println(item);
}
```

## ArrayNode 的 null 规则

`ArrayNode` 不允许添加 Java `null`。

```java
arrayNode.add((Node) null);
```

会抛出：

```java
NullPointerException
```

如果要表示数据层面的 null，应使用：

```java
arrayNode.add(NullNode.NULL);
```

也就是说：

```text
Java null       不允许作为节点元素
NullNode.NULL   表示数组中的 null
```

## ArrayNode 便捷 add 方法

`ArrayNode` 提供了一组便捷方法。

```java
arrayNode.add(1);
arrayNode.add(1L);
arrayNode.add(1.0F);
arrayNode.add(1.0D);
arrayNode.add(new BigInteger("123456789"));
arrayNode.add(new BigDecimal("123.45"));
arrayNode.add("hello");
arrayNode.add(true);
```

它们会自动包装成对应节点：

```text
int         -> IntNode
long        -> LongNode
float       -> FloatNode
double      -> DoubleNode
BigInteger  -> BigIntegerNode
BigDecimal  -> BigDecimalNode
String      -> StringNode
boolean     -> BooleanNode
```

## ValueNode

`ValueNode` 表示值节点。

它有三类实现：

```text
NumberNode
StringNode
BooleanNode
```

`ValueNode` 提供一组类型转换方法：

```java
int asInt();

int asIntExact();

long asLong();

long asLongExact();

float asFloat();

float asFloatExact();

double asDouble();

double asDoubleExact();

BigInteger asBigInteger();

BigInteger asBigIntegerExact();

BigDecimal asBigDecimal();

String asString();

boolean asBoolean();
```

其中：

```text
asXxx()        允许按 Java / BigDecimal 规则转换，可能丢失精度
asXxxExact()   不允许丢失精度，精度丢失时抛 ArithmeticException
```

对于 `StringNode`，如果字符串不能解析为数字，对应数字转换会抛出 `NumberFormatException`。

## NumberNode

`NumberNode` 表示数字节点。

它有六种实现：

```text
IntNode
LongNode
FloatNode
DoubleNode
BigIntegerNode
BigDecimalNode
```

数字节点转换为其它数字类型时，不会出现 `NumberFormatException`。

但是精确转换仍然可能抛出：

```java
ArithmeticException
```

例如：

```java
var node = new BigDecimalNode(new BigDecimal("1.5"));

node.asIntExact();
```

会因为精度丢失而抛出异常。

## IntNode

`IntNode` 是 `int` 节点。

```java
var node = new IntNode(123);
```

常见转换：

```java
int i = node.asInt();

long l = node.asLong();

String s = node.asString();

boolean b = node.asBoolean();
```

其中：

```text
asBoolean() = value != 0
```

`IntNode` 是 record，因此值通过 `value()` 获取：

```java
int value = node.value();
```

## LongNode

`LongNode` 是 `long` 节点。

```java
var node = new LongNode(123L);
```

常见转换：

```java
long l = node.asLong();

int i = node.asInt();

int exact = node.asIntExact();
```

如果 `long` 值无法无损转换为 `int`，`asIntExact()` 会抛出 `ArithmeticException`。

## FloatNode

`FloatNode` 是 `float` 节点。

```java
var node = new FloatNode(1.25F);
```

它可以转换为其它数字类型。

但是精确转换到整数时，如果存在小数部分，会抛出 `ArithmeticException`。

## DoubleNode

`DoubleNode` 是 `double` 节点。

```java
var node = new DoubleNode(1.25D);
```

和 `FloatNode` 类似，普通转换允许精度变化，精确转换会检查精度丢失。

## BigIntegerNode

`BigIntegerNode` 是 `BigInteger` 节点。

```java
var node = new BigIntegerNode(new BigInteger("12345678901234567890"));
```

`BigIntegerNode` 不允许值为 `null`。

```java
new BigIntegerNode(null);
```

会抛出 `NullPointerException`。

## BigDecimalNode

`BigDecimalNode` 是 `BigDecimal` 节点。

```java
var node = new BigDecimalNode(new BigDecimal("123.45"));
```

`BigDecimalNode` 不允许值为 `null`。

```java
new BigDecimalNode(null);
```

会抛出 `NullPointerException`。

## StringNode

`StringNode` 表示字符串节点。

```java
var node = new StringNode("123");
```

`StringNode` 不允许值为 `null`。

```java
new StringNode(null);
```

会抛出 `NullPointerException`。

字符串可以转换为数字：

```java
int i = new StringNode("123").asInt();

long l = new StringNode("123").asLong();

BigDecimal d = new StringNode("123.45").asBigDecimal();
```

如果字符串不是合法数字，会抛出：

```java
NumberFormatException
```

字符串转布尔使用：

```java
Boolean.parseBoolean(value)
```

因此：

```java
new StringNode("true").asBoolean();
```

结果是：

```text
true
```

而：

```java
new StringNode("yes").asBoolean();
```

结果是：

```text
false
```

## BooleanNode

`BooleanNode` 表示布尔节点。

它不是 record，而是两个单例：

```java
BooleanNode.TRUE

BooleanNode.FALSE
```

通常使用：

```java
var node = BooleanNode.of(true);
```

布尔转数字规则：

```text
true   -> 1
false  -> 0
```

示例：

```java
var node = BooleanNode.TRUE;

int i = node.asInt();

String s = node.asString();

boolean b = node.asBoolean();
```

## NullNode

`NullNode` 表示数据层面的 null。

它是单例：

```java
NullNode.NULL
```

构造方法是私有的，因此不能手动 new。

```java
var node = NullNode.NULL;
```

`NullNode#toString()` 返回：

```text
null
```

`NullNode#deepCopy()` 返回自身。

## deepCopy

`deepCopy()` 用于复制节点。

对于不可变节点：

```text
StringNode
IntNode
LongNode
FloatNode
DoubleNode
BigIntegerNode
BigDecimalNode
BooleanNode
NullNode
```

`deepCopy()` 返回自身。

例如：

```java
var a = new StringNode("hello");
var b = a.deepCopy();

System.out.println(a == b);
```

结果是：

```text
true
```

对于容器节点：

```text
ObjectNode
ArrayNode
```

`deepCopy()` 会递归复制结构。

示例：

```java
var user = new ObjectNode();
user.put("name", "Tom");

var copy = user.deepCopy();

copy.put("name", "Jerry");

System.out.println(user.get("name"));
System.out.println(copy.get("name"));
```

原对象不会被修改。

需要注意，当前实现假设 `ObjectNode` / `ArrayNode` 不存在自引用。

不要构造递归引用结构。

## equals 和 hashCode

`ObjectNode` 的相等性由内部字段 map 决定。

`ArrayNode` 的相等性由内部元素 list 决定。

数字节点、字符串节点等 record 类型按 record 默认规则比较。

`BooleanNode` 按布尔值比较。

示例：

```java
var a = new ObjectNode();
a.put("name", "Tom");

var b = new ObjectNode();
b.put("name", "Tom");

System.out.println(a.equals(b));
```

结果是：

```text
true
```

## toString

SCX Node 的 `toString()` 采用类 JSON 格式。

例如：

```java
var node = new ObjectNode();

node.put("name", "Tom");
node.put("age", 18);

System.out.println(node);
```

输出大致是：

```json
{
 "name": "Tom",
 "age": 18
}
```

需要注意：

1. `toString()` 主要用于调试。
2. `StringNode#toString()` 只做简单引号包裹，不处理复杂 JSON 转义。
3. 如果需要严格 JSON 输出，应使用 `scx-format-json`。

## 与 JSON 的关系

SCX Node 的结构和 JSON 数据模型很接近。

可以这样理解：

```text
ObjectNode       JSON object
ArrayNode        JSON array
StringNode       JSON string
NumberNode       JSON number
BooleanNode      JSON boolean
NullNode.NULL    JSON null
```

但是 SCX Node 本身不是 JSON 库。

它不负责：

```text
解析 JSON 字符串
输出标准 JSON
处理 JSON 转义
处理 JSON 语法选项
```

这些能力应由 `scx-format-json` 提供。

## 与 XML 的关系

XML 和 Node 的结构不完全一致。

`scx-format-xml` 会把 XML 规范化映射到 Node。

例如 XML 属性、子元素、重复标签等都会按 XML 模块自己的规则转换成 `ObjectNode` / `ArrayNode` / `StringNode` / `NullNode`。

SCX Node 本身不关心这些规则。

## 与 SCX Object 的关系

`scx-object` 定义了：

```text
Object <-> Node
```

的抽象接口。

`scx-object-x` 提供默认实现。

也就是说，SCX Node 可以作为 Java 对象和外部格式之间的中间模型。

```text
Java Object
    ↓
Node
    ↓
JSON / XML
```

以及：

```text
JSON / XML
    ↓
Node
    ↓
Java Object
```

## 完整示例

```java
import dev.scx.node.ArrayNode;
import dev.scx.node.ObjectNode;

import static dev.scx.node.NullNode.NULL;

public class NodeDemo {

    public static void main(String[] args) {
        var user = new ObjectNode();

        user.put("id", 1);
        user.put("name", "Tom");
        user.put("age", 18);
        user.put("active", true);
        user.put("parent", NULL);

        var tags = new ArrayNode();

        tags.add("java");
        tags.add("scx");
        tags.add("node");

        user.put("tags", tags);

        var copy = user.deepCopy();

        ((ArrayNode) copy.get("tags")).add("copy");

        System.out.println(user);
        System.out.println(copy);
    }

}
```

## 设计说明

### 1. SCX Node 是中间数据模型

SCX Node 的目标不是替代 JSON/XML 库。

它是中间数据模型。

它让不同模块之间可以使用同一种数据树沟通。

### 2. 容器可变，值不可变

`ObjectNode` 和 `ArrayNode` 是可变的。

值节点和 `NullNode` 是不可变的。

这也是为什么：

```text
容器 deepCopy 返回新结构
值 deepCopy 返回自身
```

### 3. Java null 和 NullNode 分开

Java `null` 表示没有对象引用。

`NullNode.NULL` 表示数据中的 null。

SCX Node 不允许把 Java `null` 放进 `ObjectNode` 或 `ArrayNode`。

### 4. ObjectNode 保持字段顺序

`ObjectNode` 内部使用 `LinkedHashMap`。

因此字段顺序会按插入顺序保留。

这对调试输出、格式化输出和稳定测试比较有用。

### 5. toString 不是严格序列化器

`toString()` 只是调试友好输出。

它不是完整 JSON 序列化器。

严格格式输出应交给 `scx-format-json`。

## 常见问题

### SCX Node 是 JSON 库吗？

不是。

它只是通用数据模型。

### 可以把 Java null 放入 ObjectNode 吗？

不可以。

请使用 `NullNode.NULL`。

### 可以把 Java null 放入 ArrayNode 吗？

不可以。

请使用 `NullNode.NULL`。

### NullNode 可以 new 吗？

不可以。

使用：

```java
NullNode.NULL
```

### BooleanNode 可以 new 吗？

不可以。

使用：

```java
BooleanNode.of(true)
```

或者：

```java
BooleanNode.TRUE
BooleanNode.FALSE
```

### deepCopy 会复制所有节点吗？

容器会复制。

值节点和 null 节点会返回自身。

### ObjectNode 的字段有顺序吗？

有。

它按插入顺序保存字段。

### StringNode 的 toString 是严格 JSON 字符串吗？

不是。

它只是简单用引号包裹。

如果需要严格 JSON 转义，请使用 `scx-format-json`。

### 数字转换什么时候会抛异常？

精确转换时如果发生精度丢失，会抛 `ArithmeticException`。

字符串转数字时如果格式不正确，会抛 `NumberFormatException`。

### StringNode 的 asBoolean 怎么判断？

使用 Java 的 `Boolean.parseBoolean(...)`。

只有忽略大小写等于 `"true"` 时才是 `true`。

### NumberNode 的 asBoolean 怎么判断？

非零为 `true`。

零为 `false`。

### ObjectNode 和 ArrayNode 可以自引用吗？

不推荐。

当前 `deepCopy()` 和 `toString()` 都假设不存在自引用。