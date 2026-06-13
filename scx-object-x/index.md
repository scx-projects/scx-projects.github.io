# SCX Object X

SCX Object X 是 `scx-object` 的默认实现模块。

它提供 `DefaultObjectNodeConverter`，用于在 Java 对象和 `scx-node` 的 `Node` 数据模型之间互相转换。

SCX Object X 的核心设计是：

```text
DefaultObjectNodeConverter 持有类型分派规则
DefaultObjectNodeConvertOptions 表示本次转换策略
TypeNodeMapper 负责具体类型和 Node 的双向映射
Context 表示一次转换过程中的执行现场
```

当前版本为 `0.5.0`。

[GitHub](https://github.com/scx-projects/scx-object-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-object-x</artifactId>
    <version>0.5.0</version>
</dependency>
```

`scx-object-x` 依赖：

```text
scx-object
scx-node
scx-reflect
```

因此普通使用场景中，只需要引入 `scx-object-x`。

## 基本概念

SCX Object X 中最核心的概念包括：

```text
DefaultObjectNodeConverter           默认 Object <-> Node 转换器
DefaultObjectNodeConvertOptions      默认转换选项
DefaultObjectNodeConverterBuilder    转换器构建器
TypeNodeMapper                       单个 Java 类型和 Node 类型之间的映射器
TypeNodeMapperFactory                根据 TypeInfo 创建 mapper 的工厂
TypeNodeMapperSelector               根据目标类型选择 mapper
TypeNodeMapperOptions                mapper 专属运行选项
ObjectToNodeContext                  Object -> Node 转换上下文
NodeToObjectContext                  Node -> Object 转换上下文
NodeTypeAdapter                      Node 类型不匹配时的适配器
```

它们之间的关系可以简单理解为：

```text
Object
    ↓
DefaultObjectNodeConverter
    ↓
ObjectToNodeContext
    ↓
TypeNodeMapperSelector
    ↓
TypeNodeMapper
    ↓
Node
```

反方向：

```text
Node
    ↓
DefaultObjectNodeConverter
    ↓
NodeToObjectContext
    ↓
TypeNodeMapperSelector
    ↓
TypeNodeMapper
    ↓
Object
```

## 快速开始

最简单的方式是使用默认转换器：

```java
import dev.scx.node.Node;
import dev.scx.object.x.DefaultObjectNodeConvertOptions;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

var options = new DefaultObjectNodeConvertOptions();

var user = new User("Tom", 18);

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(user, options);

User user2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    User.class,
    options
);
```

示例 record：

```java
public record User(String name, Integer age) {

}
```

如果目标类型带泛型，需要使用 `TypeInfo`：

```java
import dev.scx.reflect.TypeReference;

import static dev.scx.reflect.ScxReflect.typeOf;

var type = typeOf(new TypeReference<List<User>>() {});

List<User> users = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    type,
    new DefaultObjectNodeConvertOptions()
);
```

## 默认转换器

默认转换器是：

```java
DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER
```

它由下面逻辑创建：

```java
DefaultObjectNodeConverter
    .builder()
    .registerDefaultMappers()
    .build();
```

也就是说，默认转换器已经注册了内置 mapper 和 mapper factory。

一般情况下，可以直接使用它。

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    value,
    new DefaultObjectNodeConvertOptions()
);
```

## 自定义转换器

如果需要注册自定义 mapper，可以使用 builder。

```java
var converter = DefaultObjectNodeConverter.builder()
    .registerDefaultMappers()
    .registerMapper(new MoneyNodeMapper())
    .build();
```

如果需要注册 mapper factory：

```java
var converter = DefaultObjectNodeConverter.builder()
    .registerDefaultMappers()
    .registerMapperFactory(new MoneyNodeMapperFactory())
    .build();
```

也可以指定 factory 顺序：

```java
var converter = DefaultObjectNodeConverter.builder()
    .registerDefaultMappers()
    .registerMapperFactory(new MyMapperFactory(), 100)
    .build();
```

推荐规则：

```text
稳定的类型处理能力       放在 converter / builder 中
本次调用临时策略         放在 DefaultObjectNodeConvertOptions 中
```

## 默认支持的类型

默认 mapper 覆盖了常见 Java 类型。

### 基本类型和包装类型

```text
byte / Byte
short / Short
int / Integer
long / Long
float / Float
double / Double
boolean / Boolean
char / Character
```

示例：

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    123,
    new DefaultObjectNodeConvertOptions()
);

Integer value = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    Integer.class,
    new DefaultObjectNodeConvertOptions()
);
```

### 字符串

```text
String
```

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    "hello",
    new DefaultObjectNodeConvertOptions()
);
```

### 大数字

```text
BigInteger
BigDecimal
```

### 时间类型

默认支持：

```text
LocalDateTime
LocalDate
LocalTime
OffsetDateTime
OffsetTime
ZonedDateTime
Year
Month
MonthDay
YearMonth
DayOfWeek
Instant
Duration
Period
Date
```

### Node 类型

默认支持直接处理 `scx-node` 中的节点类型：

```text
Node
ValueNode
ContainerNode
NullNode
NumberNode
StringNode
BooleanNode
IntNode
LongNode
FloatNode
DoubleNode
BigIntegerNode
BigDecimalNode
ArrayNode
ObjectNode
```

这意味着如果对象本身已经是 `Node`，可以直接参与转换。

### 其它常见类型

```text
File
Path
Charset
UUID
URI
Enum
```

### 容器类型

默认支持：

```text
数组
Collection
Map
Bean
Record
Object
```

其中数组、集合、Map、Bean、Record、Enum 由 mapper factory 根据目标 `TypeInfo` 动态创建 mapper。

## Object -> Node

`objectToNode(...)` 用于把 Java 对象转换成 Node。

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    value,
    new DefaultObjectNodeConvertOptions()
);
```

如果 `value` 是 Java `null`，会转换为：

```java
NullNode.NULL
```

如果找不到对应 mapper，会抛出：

```java
ObjectToNodeException
```

## Node -> Object

`nodeToObject(...)` 用于把 Node 转换成 Java 对象。

```java
User user = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    User.class,
    new DefaultObjectNodeConvertOptions()
);
```

带泛型类型：

```java
List<User> users = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    typeOf(new TypeReference<List<User>>() {}),
    new DefaultObjectNodeConvertOptions()
);
```

如果传入的 `node` 是 Java `null`，会抛出：

```java
NodeToObjectException
```

如果要表达数据层面的 null，应使用：

```java
NullNode.NULL
```

## DefaultObjectNodeConvertOptions

`DefaultObjectNodeConvertOptions` 是默认转换选项。

默认值包括：

```text
maxNestingDepth    200
mapperOptionsMap   null
nodeTypeAdapter    null
```

创建：

```java
var options = new DefaultObjectNodeConvertOptions();
```

设置最大嵌套深度：

```java
var options = new DefaultObjectNodeConvertOptions()
    .maxNestingDepth(100);
```

添加 mapper 专属选项：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(PrimitiveNullPolicy.DEFAULT_VALUE);
```

设置 Node 类型适配器：

```java
var options = new DefaultObjectNodeConvertOptions()
    .nodeTypeAdapter(SingleValueWrapArrayAdapter.SINGLE_VALUE_WRAP_ARRAY_ADAPTER);
```

## 最大嵌套深度

`maxNestingDepth(...)` 用于限制转换递归深度。

默认值是：

```text
200
```

示例：

```java
var options = new DefaultObjectNodeConvertOptions()
    .maxNestingDepth(50);
```

如果嵌套深度超过限制，会抛出：

```text
ObjectToNodeException
NodeToObjectException
```

需要注意，当前实现主要使用最大嵌套深度限制递归。

它没有启用完整的递归引用检测。

因此如果对象存在循环引用，最终会因为深度限制或其它递归问题失败。

## mapper 专属选项

`addMapperOptions(...)` 用于添加 mapper 专属运行选项。

示例：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        PrimitiveNullPolicy.DEFAULT_VALUE,
        NumberConversionPolicy.EXACT
    );
```

内部会按选项对象的 class 保存。

```text
PrimitiveNullPolicy.class        -> PrimitiveNullPolicy.DEFAULT_VALUE
NumberConversionPolicy.class     -> NumberConversionPolicy.EXACT
```

如果多次添加同一类型选项，后添加的会覆盖之前的。

## PrimitiveNullPolicy

`PrimitiveNullPolicy` 用于控制 `NullNode` 转 primitive 时的行为。

可选值：

```text
ERROR
DEFAULT_VALUE
```

默认行为是：

```text
ERROR
```

例如：

```java
DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    NullNode.NULL,
    int.class,
    new DefaultObjectNodeConvertOptions()
);
```

会抛出 `NodeToObjectException`。

如果使用默认值策略：

```java
var value = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    NullNode.NULL,
    int.class,
    new DefaultObjectNodeConvertOptions()
        .addMapperOptions(PrimitiveNullPolicy.DEFAULT_VALUE)
);
```

结果是：

```text
0
```

不同 primitive 的默认值可以理解为：

```text
byte       0
short      0
int        0
long       0
float      0.0
double     0.0
boolean    false
char       '\0'
```

包装类型遇到 `NullNode.NULL` 时返回 Java `null`。

## NumberConversionPolicy

`NumberConversionPolicy` 用于控制数字转换策略。

可选值：

```text
DEFAULT
EXACT
```

默认行为是：

```text
DEFAULT
```

例如把字符串 `"123"` 转成 `int`：

```java
var value = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    new StringNode("123"),
    int.class,
    new DefaultObjectNodeConvertOptions()
);
```

如果设置为精确转换：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(NumberConversionPolicy.EXACT);
```

则会调用对应 Node 的 `asXxxExact()` 方法。

这意味着一旦发生精度丢失，会抛出异常。

## 时间转换选项

时间类型使用 `TemporalAccessorNodeMapperOptions`。

默认情况下，时间类型会转换为字符串。

```java
var now = OffsetDateTime.now();

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    now,
    new DefaultObjectNodeConvertOptions()
);
```

如果希望使用时间戳：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new TemporalAccessorNodeMapperOptions()
            .useTimestamp(true)
    );
```

再转换：

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(now, options);
```

会生成 `LongNode`。

需要注意，时间戳模式使用 epoch milli。

### 自定义时间格式

可以按类型设置 formatter：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new TemporalAccessorNodeMapperOptions()
            .setFormatter(
                LocalDate.class,
                DateTimeFormatter.ofPattern("yyyy/MM/dd")
            )
    );
```

然后：

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    LocalDate.now(),
    options
);
```

## Date 转换选项

`java.util.Date` 使用 `DateNodeMapperOptions`。

默认情况下，`Date` 会按 `DateFormat` 输出为字符串。

如果希望使用时间戳：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new DateNodeMapperOptions()
            .useTimestamp(true)
    );
```

转换：

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    new Date(),
    options
);
```

会生成 `LongNode`。

也可以设置自定义 `DateFormat`：

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new DateNodeMapperOptions()
            .dateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"))
    );
```

## Bean 转换

普通 Java Bean 会转换为 `ObjectNode`。

示例：

```java
public class User {

    public String name;

    public Integer age;

}
```

转换：

```java
var user = new User();

user.name = "Tom";
user.age = 18;

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    user,
    new DefaultObjectNodeConvertOptions()
);
```

结果大致是：

```json
{
  "name": "Tom",
  "age": 18
}
```

再转回：

```java
User user2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    User.class,
    new DefaultObjectNodeConvertOptions()
);
```

## Bean 字段策略

`BeanNodeMapperOptions` 可以配置字段读写策略。

对象转 Node 时，可以控制字段是否写出。

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new BeanNodeMapperOptions()
            .beanFieldWritePolicy((fieldInfo, value) -> {
                if (value == null) {
                    return BeanFieldWriteResult.ofSkip();
                }
                return null;
            })
    );
```

上面表示：

```text
Object -> Node 时跳过 null 字段
```

Node 转对象时，可以控制字段是否读取。

```java
var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new BeanNodeMapperOptions()
            .beanFieldReadPolicy(fieldInfo -> {
                if (fieldInfo.name().equals("ignoreMe")) {
                    return BeanFieldReadResult.ofSkip();
                }
                return null;
            })
    );
```

这适合处理：

```text
忽略某些字段
跳过 null 字段
模拟 JsonIgnore
自定义字段读写策略
```

## Record 转换

Record 会按 record component 转换。

示例：

```java
public record User(String name, Integer age) {

}
```

转换：

```java
var user = new User("Tom", 18);

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    user,
    new DefaultObjectNodeConvertOptions()
);
```

结果大致是：

```json
{
  "name": "Tom",
  "age": 18
}
```

再转回：

```java
User user2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    User.class,
    new DefaultObjectNodeConvertOptions()
);
```

## 数组和集合

数组和集合会转换为 `ArrayNode`。

示例：

```java
var list = List.of(1, 2, 3);

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    list,
    new DefaultObjectNodeConvertOptions()
);
```

结果：

```json
[1, 2, 3]
```

转回数组：

```java
Integer[] array = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    Integer[].class,
    new DefaultObjectNodeConvertOptions()
);
```

转回带泛型的 List：

```java
List<Integer> list2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    typeOf(new TypeReference<List<Integer>>() {}),
    new DefaultObjectNodeConvertOptions()
);
```

## Map 转换

Map 通常会转换为 `ObjectNode`。

示例：

```java
var map = new LinkedHashMap<String, Object>();

map.put("name", "Tom");
map.put("age", 18);

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    map,
    new DefaultObjectNodeConvertOptions()
);
```

结果大致是：

```json
{
  "name": "Tom",
  "age": 18
}
```

如果 Map 中存在递归引用，当前实现不会完整检测引用图，通常会因为嵌套深度限制而失败。

## Enum 转换

枚举会转换为枚举常量名。

示例：

```java
public enum Status {
    OK,
    FAIL
}
```

转换：

```java
Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    Status.OK,
    new DefaultObjectNodeConvertOptions()
);
```

结果是：

```json
"OK"
```

转回：

```java
Status status = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    new StringNode("OK"),
    Status.class,
    new DefaultObjectNodeConvertOptions()
);
```

需要注意，枚举使用的是 `name()`，不是 `toString()`。

## Path、File、Charset、UUID、URI

这些类型通常会转换为字符串。

示例：

```java
Path path = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    new StringNode("x/a/b/c.txt"),
    Path.class,
    new DefaultObjectNodeConvertOptions()
);
```

再转为 File：

```java
File file = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    new StringNode("x/a/b/c.txt"),
    File.class,
    new DefaultObjectNodeConvertOptions()
);
```

Charset 示例：

```java
Charset charset = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    new StringNode("UTF-8"),
    Charset.class,
    new DefaultObjectNodeConvertOptions()
);
```

## Node 类型直接转换

如果对象本身就是 `Node`，默认转换器可以直接处理。

例如：

```java
var node = new ObjectNode();

node.put("name", "Tom");

Node result = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    node,
    new DefaultObjectNodeConvertOptions()
);
```

Node 转 Node 类型也可以：

```java
ObjectNode objectNode = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    ObjectNode.class,
    new DefaultObjectNodeConvertOptions()
);
```

这在组合格式转换时很有用。

## Object / Untyped 转换

默认转换器中包含 Untyped mapper。

这类 mapper 用于处理目标类型为 `Object` 的场景。

例如：

```java
Object value = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    Object.class,
    new DefaultObjectNodeConvertOptions()
);
```

通常可以把 Node 转换为比较自然的 Java 值：

```text
ObjectNode   -> Map 或类似对象结构
ArrayNode    -> List 或类似数组结构
ValueNode    -> 对应标量值
NullNode     -> null
```

具体行为以当前 Untyped mapper 实现为准。

## NodeTypeAdapter

`NodeTypeAdapter` 用于 `Node -> Object` 阶段。

当实际 Node 类型和目标 mapper 期望的 Node 类型不匹配时，可以尝试做一次 Node 形状适配。

接口可以理解为：

```java
public interface NodeTypeAdapter {

    Node adapt(Node node, Class<?> expectedNodeType);

}
```

如果无法处理，返回 `null`。

### SingleValueWrapArrayAdapter

`SingleValueWrapArrayAdapter` 用于把单个值包装成数组。

例如目标类型期望 `ArrayNode`，但输入是普通值：

```json
1
```

可以适配成：

```json
[1]
```

使用：

```java
var options = new DefaultObjectNodeConvertOptions()
    .nodeTypeAdapter(
        SingleValueWrapArrayAdapter.SINGLE_VALUE_WRAP_ARRAY_ADAPTER
    );
```

### SingleElementArrayUnwrapAdapter

`SingleElementArrayUnwrapAdapter` 用于把单元素数组拆成单个值。

例如目标类型期望普通值，但输入是：

```json
[1]
```

可以适配成：

```json
1
```

使用：

```java
var options = new DefaultObjectNodeConvertOptions()
    .nodeTypeAdapter(
        SingleElementArrayUnwrapAdapter.SINGLE_ELEMENT_ARRAY_UNWRAP_ADAPTER
    );
```

### CompositeNodeTypeAdapter

如果需要组合多个适配器，可以使用：

```java
var adapter = new CompositeNodeTypeAdapter(
    SingleElementArrayUnwrapAdapter.SINGLE_ELEMENT_ARRAY_UNWRAP_ADAPTER,
    SingleValueWrapArrayAdapter.SINGLE_VALUE_WRAP_ARRAY_ADAPTER
);

var options = new DefaultObjectNodeConvertOptions()
    .nodeTypeAdapter(adapter);
```

组合适配器会按顺序尝试。

第一个返回非 null 的适配结果会被使用。

## TypeNodeMapper

`TypeNodeMapper` 是具体类型映射器。

接口可以理解为：

```java
public interface TypeNodeMapper<V, N extends Node> {

    TypeInfo valueType();

    Class<N> nodeType();

    N valueToNode(V value, ObjectToNodeContext context)
            throws ObjectToNodeException;

    V nodeToValue(N node, NodeToObjectContext context)
            throws NodeToObjectException;

    default V nullNodeToValue(NodeToObjectContext context)
            throws NodeToObjectException {
        return null;
    }

}
```

它负责一个 Java 类型和一个 Node 类型之间的双向转换。

例如：

```text
Integer <-> IntNode / ValueNode
String  <-> StringNode
Date    <-> StringNode 或 LongNode
Enum    <-> StringNode
Bean    <-> ObjectNode
List    <-> ArrayNode
```

## TypeNodeMapperFactory

有些 mapper 不能提前枚举出来。

例如：

```text
String[]
List<User>
Map<String, Integer>
某个 Bean 类型
某个 Record 类型
某个 Enum 类型
```

这些 mapper 需要根据目标 `TypeInfo` 动态创建。

这时使用 `TypeNodeMapperFactory`。

接口可以理解为：

```java
public interface TypeNodeMapperFactory {

    TypeNodeMapper<?, ?> createMapper(TypeInfo typeInfo);

}
```

如果不能处理，返回 `null`。

默认注册的 factory 包括：

```text
ArrayNodeMapperFactory
CollectionNodeMapperFactory
MapNodeMapperFactory
PathNodeMapperFactory
CharsetNodeMapperFactory
BeanNodeMapperFactory
RecordNodeMapperFactory
EnumNodeMapperFactory
```

## TypeNodeMapperSelector

`TypeNodeMapperSelector` 负责根据目标类型查找 mapper。

接口可以理解为：

```java
public interface TypeNodeMapperSelector {

    void registerMapper(TypeNodeMapper<?, ?> mapper);

    void registerMapperFactory(TypeNodeMapperFactory mapperFactory);

    void registerMapperFactory(TypeNodeMapperFactory mapperFactory, int order);

    TypeNodeMapper<?, ?> findMapper(TypeInfo type);

    TypeNodeMapper<?, ?> findMapper(Class<?> type);

}
```

`DefaultObjectNodeConverter` 持有一个 selector。

这代表这套 converter 的类型处理能力。

## Context

转换时会创建上下文对象。

```text
ObjectToNodeContextImpl
NodeToObjectContextImpl
```

它们负责：

```text
保存当前嵌套深度
读取本次 options
查找 mapper
递归转换子值
调用已选 mapper
处理 NodeTypeAdapter
```

Context 表示一次转换过程。

它不拥有 mapper 注册表，也不修改 selector。

## 完整示例：Bean 转 Node 再转回

```java
import dev.scx.node.Node;
import dev.scx.object.x.DefaultObjectNodeConvertOptions;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

public class ObjectXDemo {

    public static void main(String[] args) {
        var user = new User();

        user.name = "Tom";
        user.age = 18;

        var options = new DefaultObjectNodeConvertOptions();

        Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(user, options);

        User user2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
            node,
            User.class,
            options
        );

        System.out.println(node);
        System.out.println(user2.name);
        System.out.println(user2.age);
    }

    public static class User {

        public String name;

        public Integer age;

    }

}
```

## 完整示例：泛型 List

```java
import dev.scx.object.x.DefaultObjectNodeConvertOptions;
import dev.scx.reflect.TypeReference;

import java.util.List;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;
import static dev.scx.reflect.ScxReflect.typeOf;

var users = List.of(
    new User("Tom", 18),
    new User("Jerry", 20)
);

var options = new DefaultObjectNodeConvertOptions();

var node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(users, options);

var type = typeOf(new TypeReference<List<User>>() {});

List<User> users2 = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    type,
    options
);

public record User(String name, Integer age) {

}
```

## 完整示例：忽略 null 字段

```java
import dev.scx.object.x.DefaultObjectNodeConvertOptions;
import dev.scx.object.x.mapper.bean.BeanFieldWriteResult;
import dev.scx.object.x.mapper.bean.BeanNodeMapperOptions;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new BeanNodeMapperOptions()
            .beanFieldWritePolicy((fieldInfo, value) -> {
                if (value == null) {
                    return BeanFieldWriteResult.ofSkip();
                }
                return null;
            })
    );

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(user, options);
```

## 完整示例：primitive null 默认值

```java
import dev.scx.node.NullNode;
import dev.scx.object.x.DefaultObjectNodeConvertOptions;
import dev.scx.object.x.mapper.primitive.PrimitiveNullPolicy;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(PrimitiveNullPolicy.DEFAULT_VALUE);

int value = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    NullNode.NULL,
    int.class,
    options
);
```

结果：

```text
0
```

## 完整示例：时间戳模式

```java
import dev.scx.object.x.DefaultObjectNodeConvertOptions;
import dev.scx.object.x.mapper.time.DateNodeMapperOptions;
import dev.scx.object.x.mapper.time.TemporalAccessorNodeMapperOptions;

import java.time.OffsetDateTime;
import java.util.Date;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

var options = new DefaultObjectNodeConvertOptions()
    .addMapperOptions(
        new TemporalAccessorNodeMapperOptions().useTimestamp(true),
        new DateNodeMapperOptions().useTimestamp(true)
    );

var node1 = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    OffsetDateTime.now(),
    options
);

var node2 = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    new Date(),
    options
);
```

## 设计说明

### 1. DefaultObjectNodeConverter 管规则归属

`DefaultObjectNodeConverter` 持有 `TypeNodeMapperSelector`。

它决定：

```text
哪些类型能处理
每种类型由哪个 mapper 处理
mapper factory 如何创建 mapper
mapper 查找缓存如何工作
```

如果这些规则变化，就应该创建新的 converter。

### 2. Options 管本次运行策略

`DefaultObjectNodeConvertOptions` 表示一次转换的运行策略。

它可以控制：

```text
最大嵌套深度
mapper 专属 options
NodeTypeAdapter
primitive null 策略
数字转换策略
日期格式
timestamp 模式
bean 字段策略
```

但它不应该注册 mapper，也不应该替换 selector。

### 3. Mapper 构造函数绑定身份和结构

`TypeNodeMapper` 的构造函数适合绑定：

```text
目标类型
component type
element type
key / value type
record components
bean fields
构造参数结构
```

这些信息定义 mapper 是谁。

普通运行策略应放在 options 中。

### 4. TypeNodeMapperOptions 绑定 mapper 本次运行参数

例如：

```text
PrimitiveNullPolicy
NumberConversionPolicy
DateNodeMapperOptions
TemporalAccessorNodeMapperOptions
BeanNodeMapperOptions
```

它们不会改变哪个 mapper 被选中。

只改变已选 mapper 本次如何运行。

### 5. Context 是一次转换现场

Context 同时知道 selector 和 options。

但它不拥有这些规则。

它只是负责执行：

```text
找 mapper
递归转换
检查深度
读取 options
调用 mapper
适配 Node 类型
```

### 6. NodeTypeAdapter 只适配 Node 形状

`NodeTypeAdapter` 不决定 Java 类型由哪个 mapper 处理。

它只在 mapper 已经选中后，尝试把输入 Node 调整成 mapper 期望的 Node 类型。

例如：

```text
单值 -> 单元素数组
单元素数组 -> 单值
```

## 常见问题

### SCX Object X 和 SCX Object 有什么区别？

`scx-object` 是接口抽象层。

`scx-object-x` 是默认实现层。

### 默认转换器是什么？

```java
DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER
```

### 默认转换器已经注册 mapper 了吗？

是的。

它通过：

```java
builder().registerDefaultMappers().build()
```

创建。

### 如何注册自定义 mapper？

```java
var converter = DefaultObjectNodeConverter.builder()
    .registerDefaultMappers()
    .registerMapper(new MyMapper())
    .build();
```

### 如何注册自定义 mapper factory？

```java
var converter = DefaultObjectNodeConverter.builder()
    .registerDefaultMappers()
    .registerMapperFactory(new MyMapperFactory())
    .build();
```

### options 可以注册 mapper 吗？

不应该。

options 只表示本次转换运行策略。

mapper 注册属于 converter 构建阶段。

### Java null 会转成什么？

Object -> Node 时，Java `null` 会转成：

```java
NullNode.NULL
```

### Node -> Object 时可以传 Java null 吗？

不可以。

传入 Java `null` 会抛 `NodeToObjectException`。

如果要表达 null，请传入：

```java
NullNode.NULL
```

### primitive 遇到 NullNode 会怎样？

默认抛 `NodeToObjectException`。

如果设置：

```java
PrimitiveNullPolicy.DEFAULT_VALUE
```

则返回 primitive 默认值。

### 包装类型遇到 NullNode 会怎样？

返回 Java `null`。

### 数字转换默认是精确的吗？

默认不是。

默认使用 `asXxx()`。

如果需要精确转换，设置：

```java
NumberConversionPolicy.EXACT
```

### 枚举使用 name 还是 toString？

使用 `name()`。

因此即使枚举重写了 `toString()`，转换结果仍然是枚举常量名。

### 日期默认转字符串还是时间戳？

默认转字符串。

如果需要时间戳，使用对应 options 的 `useTimestamp(true)`。

### 时间戳单位是什么？

epoch milli。

### 是否支持循环引用？

当前不做完整递归引用检测。

主要通过最大嵌套深度限制递归。

循环引用对象通常会导致转换失败。

### 如何处理单值和数组兼容？

使用 `NodeTypeAdapter`。

单值转数组：

```java
SingleValueWrapArrayAdapter.SINGLE_VALUE_WRAP_ARRAY_ADAPTER
```

单元素数组转单值：

```java
SingleElementArrayUnwrapAdapter.SINGLE_ELEMENT_ARRAY_UNWRAP_ADAPTER
```

组合使用：

```java
new CompositeNodeTypeAdapter(...)
```

### Bean 字段怎么忽略？

使用 `BeanNodeMapperOptions` 的字段读写策略。

Object -> Node 时用：

```java
beanFieldWritePolicy(...)
```

Node -> Object 时用：

```java
beanFieldReadPolicy(...)
```

### 泛型集合怎么转换？

使用 `TypeInfo`。

```java
var type = typeOf(new TypeReference<List<User>>() {});
```

然后传给：

```java
nodeToObject(node, type, options)
```

### 什么时候需要自定义 converter？

当你要改变“哪些类型由哪个 mapper 处理”时。

例如：

```text
新增 Money 类型 mapper
替换 LocalDateTime 的处理方式
改变 mapperFactory 顺序
添加某个业务类型专用转换规则
```

### 什么时候只需要 options？

当你只是改变“本次怎么处理”时。

例如：

```text
这次忽略 null 字段
这次日期用时间戳
这次 primitive null 用默认值
这次数字必须精确转换
这次启用单值数组兼容
```
