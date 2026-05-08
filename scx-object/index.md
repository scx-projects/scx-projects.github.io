# SCX Object

SCX Object 是 `Object <-> Node` 的核心抽象库。

它定义了把 Java 对象转换成 `scx-node` 的 `Node`，以及把 `Node` 转回 Java 对象的统一接口。

SCX Object 本身不提供默认转换实现。

它只定义抽象接口、转换选项接口和异常类型。

真正的默认实现由 `scx-object-x` 提供。

当前版本为 `0.3.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-object</artifactId>
    <version>0.3.0</version>
</dependency>
```

通常情况下，如果你只是想直接使用默认对象转换能力，应引入：

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-object-x</artifactId>
    <version>0.4.0</version>
</dependency>
```

`scx-object-x` 已经依赖 `scx-object`。

## 基本概念

SCX Object 中最核心的概念包括：

```text
ObjectNodeConverter         Object 和 Node 之间的转换器接口
ObjectNodeConvertOptions    转换选项标记接口
ObjectToNodeException       Object -> Node 方向的异常
NodeToObjectException       Node -> Object 方向的异常
Node                        scx-node 提供的通用数据模型
TypeInfo                    scx-reflect 提供的类型信息
```

它们之间的关系可以简单理解为：

```text
Java Object
    ↓
ObjectNodeConverter#objectToNode(...)
    ↓
Node

Node
    ↓
ObjectNodeConverter#nodeToObject(...)
    ↓
Java Object
```

也就是说：

```text
scx-object      定义转换接口
scx-object-x    提供默认实现
scx-node        提供中间数据模型
scx-reflect     提供泛型类型信息
```

## 设计目标

SCX Object 的目标不是绑定某一种对象映射实现。

它只规定：

```text
如果一个模块要提供 Object <-> Node 转换能力，应该长什么样。
```

因此它适合作为上层格式库、序列化库、配置库、数据绑定库的共同抽象。

例如：

```text
JSON -> Node -> Object
XML  -> Node -> Object

Object -> Node -> JSON
Object -> Node -> XML
```

其中：

```text
格式解析由 scx-format-json / scx-format-xml 处理
对象绑定由 scx-object / scx-object-x 处理
数据模型由 scx-node 处理
```

## ObjectNodeConverter

`ObjectNodeConverter` 是核心接口。

它负责在 Java 对象和 `Node` 之间互相转换。

接口可以理解为：

```java
public interface ObjectNodeConverter<O extends ObjectNodeConvertOptions> {

    Node objectToNode(Object value, O options) throws ObjectToNodeException;

    <T> T nodeToObject(Node node, TypeInfo type, O options) throws NodeToObjectException;

    <T> T nodeToObject(Node node, Class<T> clazz, O options) throws NodeToObjectException;

}
```

它有两个主要方向：

```text
objectToNode     Object -> Node
nodeToObject     Node -> Object
```

## objectToNode

`objectToNode(...)` 表示：

```text
Java Object -> Node
```

示例：

```java
Node node = converter.objectToNode(
    user,
    options
);
```

具体如何转换由实现决定。

例如默认实现可能把：

```java
public class User {

    public String name;

    public Integer age;

}
```

转换为：

```json
{
  "name": "Tom",
  "age": 18
}
```

但是这些规则不属于 `scx-object` 核心层，而属于具体实现层。

## nodeToObject

`nodeToObject(...)` 表示：

```text
Node -> Java Object
```

可以传入 `Class<T>`：

```java
User user = converter.nodeToObject(
    node,
    User.class,
    options
);
```

也可以传入 `TypeInfo`：

```java
List<User> users = converter.nodeToObject(
    node,
    typeOf(new TypeReference<List<User>>() {}),
    options
);
```

`Class<T>` 适合普通类型。

`TypeInfo` 适合带泛型信息的类型。

例如：

```text
List<User>
Map<String, User>
User[]
```

## 为什么需要 TypeInfo

Java 的 `Class<T>` 无法表达完整泛型信息。

例如：

```java
List<User>.class
```

这种写法不存在。

如果只传入：

```java
List.class
```

转换器无法知道元素类型是 `User`。

因此 SCX Object 使用 `scx-reflect` 的 `TypeInfo` 表达完整类型信息。

示例：

```java
import static dev.scx.reflect.ScxReflect.typeOf;

var type = typeOf(new TypeReference<List<User>>() {});
```

然后：

```java
List<User> users = converter.nodeToObject(node, type, options);
```

## ObjectNodeConvertOptions

`ObjectNodeConvertOptions` 是转换选项接口。

它本身不定义任何方法。

```java
public interface ObjectNodeConvertOptions {

}
```

它的作用是给不同实现提供统一选项边界。

例如默认实现中有：

```text
DefaultObjectNodeConvertOptions
```

它可以包含：

```text
最大嵌套深度
mapper 专属选项
NodeTypeAdapter
primitive null 策略
日期格式策略
数字转换策略
bean 字段读写策略
```

这些都不属于核心抽象层。

核心层只规定：

```text
转换方法应该接收 options
```

具体 options 有哪些能力，由实现模块自己定义。

## ObjectToNodeException

`ObjectToNodeException` 表示：

```text
Object -> Node
```

这个方向发生的转换异常。

例如：

```java
try {
    Node node = converter.objectToNode(value, options);
} catch (ObjectToNodeException e) {
    // Java 对象无法转换为 Node
}
```

常见原因包括：

```text
找不到对应类型的 mapper
对象结构递归过深
对象中存在无法处理的字段
字段读取失败
类型不受支持
对象存在递归引用
```

`ObjectToNodeException` 继承自 `RuntimeException`。

它提供三个构造方法：

```java
public ObjectToNodeException(String message)

public ObjectToNodeException(Throwable cause)

public ObjectToNodeException(String message, Throwable cause)
```

## NodeToObjectException

`NodeToObjectException` 表示：

```text
Node -> Object
```

这个方向发生的转换异常。

例如：

```java
try {
    User user = converter.nodeToObject(node, User.class, options);
} catch (NodeToObjectException e) {
    // Node 无法转换为目标 Java 类型
}
```

常见原因包括：

```text
找不到对应类型的 mapper
Node 类型和目标 mapper 期望不匹配
数字精确转换失败
字符串无法解析为目标类型
对象创建失败
字段写入失败
primitive 类型遇到 NullNode
嵌套深度超过限制
```

`NodeToObjectException` 也继承自 `RuntimeException`。

它提供三个构造方法：

```java
public NodeToObjectException(String message)

public NodeToObjectException(Throwable cause)

public NodeToObjectException(String message, Throwable cause)
```

## 异常边界

SCX Object 把异常分成两个方向：

```text
ObjectToNodeException      Object -> Node 失败
NodeToObjectException      Node -> Object 失败
```

这样调用方可以很明确地知道异常发生在哪一步。

示例：

```java
try {
    Node node = converter.objectToNode(user, options);
    User user2 = converter.nodeToObject(node, User.class, options);
} catch (ObjectToNodeException e) {
    // Object -> Node 失败
} catch (NodeToObjectException e) {
    // Node -> Object 失败
}
```

## 与 SCX Node 的关系

SCX Object 依赖 `scx-node`。

它的转换边界是：

```text
Object <-> Node
```

也就是说，SCX Object 不直接处理 JSON、XML 或其它格式文本。

如果你需要完整链路：

```text
Object -> JSON
```

可以组合：

```text
Object -> Node -> JSON
```

如果你需要：

```text
JSON -> Object
```

可以组合：

```text
JSON -> Node -> Object
```

## 与 SCX Format 的关系

SCX Format 处理的是：

```text
外部格式 <-> Node
```

SCX Object 处理的是：

```text
Object <-> Node
```

它们组合起来就是完整格式对象绑定链路：

```text
JSON/XML
    ↓
SCX Format
    ↓
Node
    ↓
SCX Object
    ↓
Java Object
```

反方向：

```text
Java Object
    ↓
SCX Object
    ↓
Node
    ↓
SCX Format
    ↓
JSON/XML
```

## 与 SCX Object X 的关系

`scx-object` 是抽象层。

`scx-object-x` 是默认实现层。

```text
scx-object
    ObjectNodeConverter
    ObjectNodeConvertOptions
    ObjectToNodeException
    NodeToObjectException

scx-object-x
    DefaultObjectNodeConverter
    DefaultObjectNodeConvertOptions
    TypeNodeMapper
    TypeNodeMapperFactory
    默认 mapper 集合
```

一般使用者应直接使用：

```java
DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER
```

或者通过 builder 创建自己的 converter。

## 快速开始

如果你使用默认实现，需要引入 `scx-object-x`。

示例：

```java
import dev.scx.node.Node;
import dev.scx.object.x.DefaultObjectNodeConvertOptions;

import static dev.scx.object.x.DefaultObjectNodeConverter.DEFAULT_OBJECT_NODE_CONVERTER;

var options = new DefaultObjectNodeConvertOptions();

Node node = DEFAULT_OBJECT_NODE_CONVERTER.objectToNode(
    new User("Tom", 18),
    options
);

User user = DEFAULT_OBJECT_NODE_CONVERTER.nodeToObject(
    node,
    User.class,
    options
);
```

示例对象：

```java
public record User(String name, Integer age) {

}
```

## 自定义实现

如果你想提供自己的对象映射实现，可以实现 `ObjectNodeConverter`。

示例：

```java
public final class SimpleObjectNodeConverter
        implements ObjectNodeConverter<SimpleObjectNodeConvertOptions> {

    @Override
    public Node objectToNode(Object value, SimpleObjectNodeConvertOptions options)
            throws ObjectToNodeException {
        if (value == null) {
            return NullNode.NULL;
        }
        if (value instanceof String s) {
            return new StringNode(s);
        }
        throw new ObjectToNodeException("Unsupported type: " + value.getClass());
    }

    @Override
    public <T> T nodeToObject(Node node, TypeInfo type, SimpleObjectNodeConvertOptions options)
            throws NodeToObjectException {
        throw new NodeToObjectException("Not implemented");
    }

    @Override
    public <T> T nodeToObject(Node node, Class<T> clazz, SimpleObjectNodeConvertOptions options)
            throws NodeToObjectException {
        throw new NodeToObjectException("Not implemented");
    }

}
```

选项类型：

```java
public final class SimpleObjectNodeConvertOptions implements ObjectNodeConvertOptions {

}
```

这体现了 SCX Object 的扩展方式：

```text
新实现 = ObjectNodeConverter 实现 + Options 实现
```

## 设计说明

### 1. SCX Object 是抽象层

SCX Object 不规定：

```text
字段怎么读取
构造函数怎么选择
record 怎么处理
泛型怎么处理
日期怎么处理
枚举怎么处理
null 怎么处理
```

这些规则属于实现层。

### 2. Node 是统一中间层

SCX Object 不直接绑定 JSON/XML。

它只处理 `Object <-> Node`。

这样格式层和对象层可以解耦。

### 3. Options 由实现定义

核心层只定义 `ObjectNodeConvertOptions` 作为标记接口。

具体实现可以自由扩展自己的配置项。

### 4. TypeInfo 用于完整类型信息

`Class<T>` 只能表达普通 class。

`TypeInfo` 可以表达泛型类型。

因此接口同时提供：

```java
nodeToObject(Node, Class<T>, options)

nodeToObject(Node, TypeInfo, options)
```

### 5. 异常按方向区分

Object 转 Node 失败：

```text
ObjectToNodeException
```

Node 转 Object 失败：

```text
NodeToObjectException
```

这比统一抛一个异常更容易定位问题。

## 常见问题

### SCX Object 可以直接使用吗？

可以作为接口依赖使用。

但如果你需要默认实现，应使用 `scx-object-x`。

### SCX Object 会把对象转 JSON 吗？

不会。

它只负责：

```text
Object <-> Node
```

转 JSON 需要再使用 `scx-format-json`。

### SCX Object 会把 XML 转对象吗？

不会直接做。

完整链路应是：

```text
XML -> Node -> Object
```

其中 XML 转 Node 由 `scx-format-xml` 负责。

### 为什么接口需要 options？

因为不同实现需要不同转换策略。

核心层不知道具体有哪些策略，所以只保留统一参数位置。

### 为什么需要 TypeInfo？

因为 Java `Class` 无法表达泛型类型。

例如：

```text
List<User>
Map<String, User>
```

需要 `TypeInfo` 表达。

### ObjectToNodeException 是 checked exception 吗？

不是。

它继承自 `RuntimeException`。

### NodeToObjectException 是 checked exception 吗？

不是。

它继承自 `RuntimeException`。

### 默认实现在哪里？

在 `scx-object-x` 中。

### 自定义转换规则应该放在 scx-object 里吗？

不应该。

`scx-object` 是抽象层。

自定义规则应放在具体实现中，例如自定义 `TypeNodeMapper` 或自定义 `ObjectNodeConverter`。