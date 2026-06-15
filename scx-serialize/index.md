# SCX Serialize

SCX Serialize 是一个轻量的对象序列化 / 反序列化工具库。

它把 Java 对象、`Node`、JSON、XML 之间的转换统一到一组简单 API 中。核心路径是：

```text
Object <-> Node <-> JSON
Object <-> Node <-> XML
Object <-> Node <-> Object
```

[GitHub](https://github.com/scx-projects/scx-serialize)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-serialize</artifactId>
    <version>0.4.0</version>
</dependency>
```

## 基本概念

SCX Serialize 中最常用的几个概念是：

```text
ScxSerialize          序列化 / 反序列化入口类
Node                  中间数据结构
ObjectToNodeOptions   Object -> Node 选项
NodeToObjectOptions   Node -> Object 选项
ToJsonOptions         Object / Node -> JSON 选项
FromJsonOptions       JSON -> Node / Object 选项
ToXmlOptions          Object / Node -> XML 选项
FromXmlOptions        XML -> Node / Object 选项
ConvertObjectOptions  Object -> Node -> Object 转换选项
@Ignore               控制字段或 record component 是否参与序列化 / 反序列化
```

`ScxSerialize` 是静态工具类，提供对象、`Node`、JSON、XML 之间的常用转换方法，并内置默认的对象映射器、JSON 转换器和 XML 转换器。

## 快速开始

### 对象转 JSON

```java
import dev.scx.serialize.ScxSerialize;

var user = new User();
user.name = "Tom";
user.age = 18;

String json = ScxSerialize.toJson(user);

System.out.println(json);
```

### JSON 转对象

```java
import dev.scx.serialize.ScxSerialize;

String json = """
{
  "name": "Tom",
  "age": 18
}
""";

User user = ScxSerialize.fromJson(json, User.class);
```

### 对象转 XML

```java
String xml = ScxSerialize.toXml(user);
```

### XML 转对象

```java
User user = ScxSerialize.fromXml(xml, User.class);
```

常见用法包括字符串、集合、普通对象、record、数组、JSON 和 XML 的基础转换用法。

## 定义数据类型

普通 class 示例：

```java
public class User {

    public String name;

    public Integer age;

}
```

record 示例：

```java
public record Student(
    String name,
    Integer age
) {
}
```

可以同时使用普通 class 和 record 进行序列化 / 反序列化。

## JSON 序列化

### 基本用法

```java
User user = new User();
user.name = "Tom";
user.age = 18;

String json = ScxSerialize.toJson(user);
```

`toJson(Object)` 会先把对象转换为 `Node`，再把 `Node` 转换为 JSON 字符串。处理流程是 `objectToNode(object, options)` 后接 `toJson(node, options)`。

### 格式化输出

```java
import dev.scx.serialize.ToJsonOptions;

String json = ScxSerialize.toJson(
    user,
    new ToJsonOptions().prettyPrint(true)
);
```

`ToJsonOptions` 提供 `prettyPrint(boolean)`，默认值为 `false`。

### 忽略 null 值

```java
String json = ScxSerialize.toJson(
    user,
    new ToJsonOptions()
        .prettyPrint(true)
        .ignoreNullValue(true)
);
```

`ignoreNullValue` 来自 `ObjectToNodeOptions`，默认值是 `false`；开启后，普通对象、record 和 map 中的 null 值会被跳过。

## JSON 反序列化

### 转为普通类型

```java
Integer value = ScxSerialize.fromJson("123", Integer.class);
```

也可以转为基本类型：

```java
int value = ScxSerialize.fromJson("123", int.class);
```

也支持把 JSON 字符串转换为 `int.class` 这类基础类型。

### 转为对象

```java
String json = """
{
  "name": "Tom",
  "age": 18
}
""";

User user = ScxSerialize.fromJson(json, User.class);
```

### 转为数组

```java
int[] values = ScxSerialize.fromJson("[1, 2, 3]", int[].class);
```

常见用法包括 `List.of("123", 456)` 转 JSON 后，再反序列化为 `int[].class` 的用法。

### 转为泛型类型

对于带泛型的目标类型，可以使用 `TypeReference`：

```java
import dev.scx.reflect.TypeReference;

List<User> users = ScxSerialize.fromJson(
    json,
    new TypeReference<List<User>>() {}
);
```

`ScxSerialize#fromJson` 提供了 `Class`、`TypeInfo` 和 `TypeReference` 三类重载；`TypeReference` 会通过 `typeOf(type)` 转成 `TypeInfo`。

## XML 序列化

### 对象转 XML

```java
String xml = ScxSerialize.toXml(user);
```

### 使用 XML 选项

```java
import dev.scx.serialize.ToXmlOptions;

String xml = ScxSerialize.toXml(
    user,
    new ToXmlOptions()
        .ignoreNullValue(true)
);
```

`ToXmlOptions` 继承自 `ObjectToNodeOptions`，因此支持 `ignoreNullValue` 和 `useAnnotations`。

## XML 反序列化

### XML 转对象

```java
User user = ScxSerialize.fromXml(xml, User.class);
```

### XML 转泛型类型

```java
List<User> users = ScxSerialize.fromXml(
    xml,
    new TypeReference<List<User>>() {}
);
```

`ScxSerialize#fromXml` 和 `fromJson` 一样，支持从字符串或文件读取，也支持 `Class`、`TypeInfo` 和 `TypeReference` 作为目标类型。

## Node 转换

`Node` 是 SCX Serialize 的中间结构。你可以只做对象和 `Node` 之间的转换，也可以先解析 JSON / XML 得到 `Node`，再决定如何处理。

### Object 转 Node

```java
Node node = ScxSerialize.objectToNode(user);
```

### Node 转 Object

```java
User user = ScxSerialize.nodeToObject(node, User.class);
```

### JSON 转 Node

```java
Node node = ScxSerialize.fromJson(json);
```

### Node 转 JSON

```java
String json = ScxSerialize.toJson(node);
```

### XML 转 Node

```java
Node node = ScxSerialize.fromXml(xml);
```

### Node 转 XML

```java
String xml = ScxSerialize.toXml(node);
```

这些方法都在 `ScxSerialize` 中作为一等 API 暴露出来。

## 对象转换

`convertObject(...)` 用于把一个 Java 对象先转成 `Node`，再转成另一个 Java 类型。

```java
UserDTO dto = ScxSerialize.convertObject(user, UserDTO.class);
```

泛型目标类型：

```java
List<UserDTO> dtoList = ScxSerialize.convertObject(
    users,
    new TypeReference<List<UserDTO>>() {}
);
```

`convertObject` 的行为可以理解为：

```text
Object -> Node -> Object
```

它先调用 `objectToNode(value, options.objectToNodeOptions())`，再调用 `nodeToObject(node, type, options.nodeToObjectOptions())`。

## 文件读取

SCX Serialize 支持从文件读取 JSON 或 XML。

### 从 JSON 文件读取

```java
File file = new File("user.json");

User user = ScxSerialize.fromJson(file, User.class);
```

### 从 XML 文件读取

```java
File file = new File("user.xml");

User user = ScxSerialize.fromXml(file, User.class);
```

`ScxSerialize` 提供 `fromJson(File)`、`fromJson(File, type)`、`fromXml(File)` 和 `fromXml(File, type)` 等重载；文件读取方法可能抛出 `IOException`。

## 注解

SCX Serialize 当前提供 `@Ignore` 注解，用于控制字段或 record component 是否参与序列化和反序列化。

```java
import dev.scx.serialize.annotation.Ignore;

public class User {

    public String name;

    @Ignore
    public String password;

    public Integer age;

}
```

`@Ignore` 可以标注在字段、record component 或参数上，运行时保留；默认模式是 `BOTH`。

### 只在序列化时忽略

```java
import static dev.scx.serialize.annotation.Ignore.Mode.SERIALIZE;

public class User {

    public String name;

    @Ignore(SERIALIZE)
    public String password;

}
```

这种写法表示：对象转 JSON / XML 时忽略 `password`，但从 JSON / XML 反序列化时仍然允许读取。这适合密码、密钥等只允许读取、不希望输出的字段。

### 只在反序列化时忽略

```java
import static dev.scx.serialize.annotation.Ignore.Mode.DESERIALIZE;

public class User {

    public String name;

    @Ignore(DESERIALIZE)
    public String serverOnlyValue;

}
```

### 序列化和反序列化都忽略

```java
public record Student(
    String name,
    @Ignore String password,
    Integer age
) {
}
```

默认 `@Ignore` 等价于 `BOTH`。

## 关闭注解处理

默认会启用注解处理。可以通过 `useAnnotations(false)` 关闭。

### 序列化时关闭

```java
String json = ScxSerialize.toJson(
    user,
    new ToJsonOptions()
        .useAnnotations(false)
);
```

### 反序列化时关闭

```java
User user = ScxSerialize.fromJson(
    json,
    User.class,
    new FromJsonOptions()
        .useAnnotations(false)
);
```

`ObjectToNodeOptions` 和 `NodeToObjectOptions` 的 `useAnnotations` 默认值都是 `true`；`OptionsResolver` 会根据 `@Ignore` 和当前模式决定是否跳过字段或 record component。

## 反序列化选项

`FromJsonOptions` 和 `FromXmlOptions` 都继承自 `NodeToObjectOptions`，因此支持：

```java
useAnnotations(boolean)
numberConversionPolicy(NumberConversionPolicy)
primitiveNullPolicy(PrimitiveNullPolicy)
singleValueArrayCompatibility(boolean)
```

`FromJsonOptions` 和 `FromXmlOptions` 本身只是把这些方法改成返回自身类型，方便链式调用。

### 数字转换策略

```java
import dev.scx.object.x.mapper.NumberConversionPolicy;

User user = ScxSerialize.fromJson(
    json,
    User.class,
    new FromJsonOptions()
        .numberConversionPolicy(NumberConversionPolicy.EXACT)
);
```

`NodeToObjectOptions` 的默认数字转换策略是 `NumberConversionPolicy.DEFAULT`；`NumberConversionPolicy` 当前包含 `DEFAULT` 和 `EXACT` 两个值。

### 基本类型 null 策略

```java
import dev.scx.object.x.mapper.primitive.PrimitiveNullPolicy;

User user = ScxSerialize.fromJson(
    json,
    User.class,
    new FromJsonOptions()
        .primitiveNullPolicy(PrimitiveNullPolicy.DEFAULT_VALUE)
);
```

`NodeToObjectOptions` 的默认基本类型 null 策略是 `PrimitiveNullPolicy.ERROR`；`PrimitiveNullPolicy` 当前包含 `ERROR` 和 `DEFAULT_VALUE` 两个值。

### 单值和单元素数组兼容

```java
int value = ScxSerialize.fromJson(
    "[123]",
    int.class,
    new FromJsonOptions()
        .singleValueArrayCompatibility(true)
);
```

也可以把单值兼容成数组：

```java
int[] values = ScxSerialize.fromJson(
    "123",
    int[].class,
    new FromJsonOptions()
        .singleValueArrayCompatibility(true)
);
```

`singleValueArrayCompatibility(true)` 允许单值和单元素数组之间兼容转换，例如 `["a"] -> "a"` 或 `"a" -> ["a"]`；默认值是 `false`。这两类转换都可以按同一选项处理。

## 序列化选项

### ObjectToNodeOptions

```java
new ObjectToNodeOptions()
    .ignoreNullValue(true)
    .useAnnotations(true);
```

默认值：

```text
ignoreNullValue = false
useAnnotations = true
```

`ObjectToNodeOptions` 是对象转 `Node` 的基础选项，`ToJsonOptions` 和 `ToXmlOptions` 都继承自它。

### ToJsonOptions

```java
new ToJsonOptions()
    .prettyPrint(true)
    .ignoreNullValue(true)
    .useAnnotations(true);
```

默认值：

```text
prettyPrint = false
ignoreNullValue = false
useAnnotations = true
```

`prettyPrint` 只存在于 `ToJsonOptions` 中。

### ToXmlOptions

```java
new ToXmlOptions()
    .ignoreNullValue(true)
    .useAnnotations(true);
```

`ToXmlOptions` 支持对象转 `Node` 的通用选项，但没有 `prettyPrint`。

## 反序列化选项

### NodeToObjectOptions

```java
new NodeToObjectOptions()
    .useAnnotations(true)
    .numberConversionPolicy(NumberConversionPolicy.DEFAULT)
    .primitiveNullPolicy(PrimitiveNullPolicy.ERROR)
    .singleValueArrayCompatibility(false);
```

默认值：

```text
useAnnotations = true
numberConversionPolicy = DEFAULT
primitiveNullPolicy = ERROR
singleValueArrayCompatibility = false
```

这些默认值都在 `NodeToObjectOptions` 构造函数中设置。

### FromJsonOptions

```java
new FromJsonOptions()
    .useAnnotations(true)
    .singleValueArrayCompatibility(true);
```

### FromXmlOptions

```java
new FromXmlOptions()
    .useAnnotations(true)
    .primitiveNullPolicy(PrimitiveNullPolicy.DEFAULT_VALUE);
```

`FromJsonOptions` 和 `FromXmlOptions` 都是面向具体格式的反序列化选项。

## ConvertObjectOptions

`ConvertObjectOptions` 用于配置 `convertObject(...)` 的两段转换：

```text
Object -> Node
Node -> Object
```

```java
var options = new ConvertObjectOptions();

options.objectToNodeOptions()
    .ignoreNullValue(true);

options.nodeToObjectOptions()
    .singleValueArrayCompatibility(true);

UserDTO dto = ScxSerialize.convertObject(user, UserDTO.class, options);
```

`ConvertObjectOptions` 由一个 `ObjectToNodeOptions` 和一个 `NodeToObjectOptions` 组成。

## 访问底层 Converter

如果需要更底层的能力，可以直接获取公开的底层 converter：

```java
var objectNodeConverter = ScxSerialize.objectNodeConverter();
var jsonNodeConverter = ScxSerialize.jsonNodeConverter();
var xmlNodeConverter = ScxSerialize.xmlNodeConverter();
```

这三个方法分别返回默认的 `DefaultObjectNodeConverter`、`JsonNodeConverter` 和 `XmlNodeConverter` 实例。

## 异常

常见异常来自底层转换过程：

```text
ObjectToNodeException     Object -> Node 失败
NodeToObjectException     Node -> Object 失败
FormatToNodeException     JSON / XML -> Node 失败
NodeToFormatException     Node -> JSON / XML 失败
IOException               从文件读取 JSON / XML 失败
```

这些异常类型出现在 `ScxSerialize` 的方法签名中；不同入口会根据转换方向抛出对应异常。

## 完整示例

```java
import dev.scx.reflect.TypeReference;
import dev.scx.serialize.FromJsonOptions;
import dev.scx.serialize.ScxSerialize;
import dev.scx.serialize.ToJsonOptions;
import dev.scx.serialize.annotation.Ignore;

import java.util.List;

import static dev.scx.serialize.annotation.Ignore.Mode.SERIALIZE;

public class SerializeExample {

    public static void main(String[] args) {
        var user = new User();
        user.name = "Tom";
        user.password = "123456";
        user.age = null;

        String json = ScxSerialize.toJson(
            user,
            new ToJsonOptions()
                .prettyPrint(true)
                .ignoreNullValue(true)
        );

        System.out.println(json);

        User user2 = ScxSerialize.fromJson(json, User.class);

        List<User> users = ScxSerialize.fromJson(
            """
            [
              {"name": "Tom", "age": 18},
              {"name": "Jerry", "age": 20}
            ]
            """,
            new TypeReference<List<User>>() {}
        );

        String xml = ScxSerialize.toXml(user2);

        User user3 = ScxSerialize.fromXml(xml, User.class);

        int value = ScxSerialize.fromJson(
            "[123]",
            int.class,
            new FromJsonOptions()
                .singleValueArrayCompatibility(true)
        );
    }

    public static class User {

        public String name;

        @Ignore(SERIALIZE)
        public String password;

        public Integer age;

    }

}
```

## 设计说明

### 1. Node 是统一中间层

SCX Serialize 不是把 Object 直接写成 JSON / XML，也不是把 JSON / XML 直接写成 Object。它通过 `Node` 作为统一中间层，从而复用对象转换和格式转换能力。`toJson(Object)`、`toXml(Object)`、`fromJson(..., type)`、`fromXml(..., type)` 都沿用这个流程。

### 2. JSON 和 XML 共享对象映射规则

`ToJsonOptions` 和 `ToXmlOptions` 都继承 `ObjectToNodeOptions`；`FromJsonOptions` 和 `FromXmlOptions` 都继承 `NodeToObjectOptions`。因此忽略 null、注解处理、数字转换、基本类型 null 策略、单值数组兼容等规则在 JSON 和 XML 上保持一致。

### 3. 注解处理可以关闭

默认启用 `@Ignore`。如果需要原样序列化或反序列化所有字段，可以通过 `useAnnotations(false)` 关闭注解处理。常见用法包括开启和关闭注解后的不同反序列化行为。

### 4. 格式选项和对象选项是分层的

`prettyPrint` 属于 JSON 格式输出选项；`ignoreNullValue`、`useAnnotations` 属于对象转 `Node` 的选项；`numberConversionPolicy`、`primitiveNullPolicy`、`singleValueArrayCompatibility` 属于 `Node` 转对象的选项。这个分层由各个 Options 类的继承关系体现。

## 常见问题

### SCX Serialize 支持哪些格式？

当前公开 API 支持 JSON 和 XML，并提供 `fromJson`、`toJson`、`fromXml`、`toXml` 方法。

### 可以只用它做对象拷贝 / 类型转换吗？

可以。使用 `convertObject(...)` 即可，它会走 `Object -> Node -> Object` 流程。

### 默认会忽略 null 吗？

不会。`ObjectToNodeOptions` 的 `ignoreNullValue` 默认是 `false`。需要忽略 null 时，手动设置 `ignoreNullValue(true)`。

### 默认会启用注解吗？

会。`ObjectToNodeOptions` 和 `NodeToObjectOptions` 的 `useAnnotations` 默认都是 `true`。

### `@Ignore` 默认忽略哪个方向？

默认是 `Ignore.Mode.BOTH`，也就是序列化和反序列化都忽略。也可以指定 `SERIALIZE` 或 `DESERIALIZE`。

### JSON 可以格式化输出吗？

可以。使用 `new ToJsonOptions().prettyPrint(true)`。

### XML 有 prettyPrint 选项吗？

当前 `ToXmlOptions` 没有 `prettyPrint` 字段，只继承了 `ignoreNullValue` 和 `useAnnotations`。
