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
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Serialize 中最常用的几个概念是：

```text
ScxSerialize      使用默认 Serializer 的静态入口类
Serializer        可独立创建或继承扩展的序列化器
SerializeConfig   常用序列化 / 反序列化配置
SerializeOptions  底层转换配置接口
Node              中间数据结构
@Ignore           控制字段或 record component 是否参与序列化 / 反序列化
```

`ScxSerialize` 是静态工具类，所有方法都委托给默认的 `Serializer`。它提供对象、`Node`、JSON、XML 之间的常用转换方法，并内置默认的对象映射器、JSON 转换器和 XML 转换器。

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
import dev.scx.serialize.SerializeConfig;

String json = ScxSerialize.toJson(
    user,
    SerializeConfig.of().prettyPrint(true)
);
```

`SerializeConfig` 提供 `prettyPrint(boolean)`，默认值为 `false`。该配置只影响 JSON 输出。

### 忽略 null 值

```java
String json = ScxSerialize.toJson(
    user,
    SerializeConfig.of()
        .prettyPrint(true)
        .ignoreNullValue(true)
);
```

`ignoreNullValue` 默认值是 `false`；开启后，普通对象、record 和 map 中的 null 值会被跳过。

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

`ScxSerialize#fromJson` 提供了 `Class`、`TypeInfo` 和 `TypeReference` 三类目标类型重载。

## XML 序列化

### 对象转 XML

```java
String xml = ScxSerialize.toXml(user);
```

### 使用 XML 选项

```java
import dev.scx.serialize.SerializeConfig;

String xml = ScxSerialize.toXml(
    user,
    SerializeConfig.of()
        .ignoreNullValue(true)
);
```

JSON 和 XML 使用同一个 `SerializeConfig`。`ignoreNullValue`、`useAnnotations` 等对象映射配置会同时作用于两种格式；`prettyPrint` 只影响 JSON。

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

它先调用 `objectToNode(value, options)`，再调用 `nodeToObject(node, type, options)`；两段转换使用同一个 `SerializeOptions`。

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
    SerializeConfig.of()
        .useAnnotations(false)
);
```

### 反序列化时关闭

```java
User user = ScxSerialize.fromJson(
    json,
    User.class,
    SerializeConfig.of()
        .useAnnotations(false)
);
```

`SerializeConfig` 的 `useAnnotations` 默认值是 `true`。启用时，会根据 `@Ignore` 的模式决定是否跳过字段或 record component。

## 反序列化选项

`SerializeConfig` 同时用于 JSON 和 XML 反序列化，常用配置包括：

```java
useAnnotations(boolean)
numberConversionPolicy(NumberConversionPolicy)
primitiveNullPolicy(PrimitiveNullPolicy)
singleValueArrayCompatibility(boolean)
```

### 数字转换策略

```java
import dev.scx.object.x.mapper.NumberConversionPolicy;

User user = ScxSerialize.fromJson(
    json,
    User.class,
    SerializeConfig.of()
        .numberConversionPolicy(NumberConversionPolicy.EXACT)
);
```

默认数字转换策略是 `NumberConversionPolicy.DEFAULT`。

### 基本类型 null 策略

```java
import dev.scx.object.x.mapper.primitive.PrimitiveNullPolicy;

User user = ScxSerialize.fromJson(
    json,
    User.class,
    SerializeConfig.of()
        .primitiveNullPolicy(PrimitiveNullPolicy.DEFAULT_VALUE)
);
```

默认基本类型 null 策略是 `PrimitiveNullPolicy.ERROR`。

### 单值和单元素数组兼容

```java
int value = ScxSerialize.fromJson(
    "[123]",
    int.class,
    SerializeConfig.of()
        .singleValueArrayCompatibility(true)
);
```

也可以把单值兼容成数组：

```java
int[] values = ScxSerialize.fromJson(
    "123",
    int[].class,
    SerializeConfig.of()
        .singleValueArrayCompatibility(true)
);
```

`singleValueArrayCompatibility(true)` 允许单值和单元素数组之间兼容转换，例如 `["a"] -> "a"` 或 `"a" -> ["a"]`；默认值是 `false`。这两类转换都可以按同一配置处理。

## 序列化选项

### SerializeConfig

```java
SerializeConfig.of()
    .prettyPrint(true)
    .ignoreNullValue(true)
    .useAnnotations(true);
```

默认值：

```text
prettyPrint = false
ignoreNullValue = false
useAnnotations = true
numberConversionPolicy = DEFAULT
primitiveNullPolicy = ERROR
singleValueArrayCompatibility = false
mapperOptionsMap = null
```

`SerializeConfig` 是 `SerializeOptions` 的便捷实现，同一个配置对象可用于 `objectToNode`、`nodeToObject`、`convertObject`、JSON 和 XML 方法。也可以使用 `new SerializeConfig()` 创建配置，或使用 `SerializeConfig.copyOf(config)` 复制已有配置。

### 自定义 Mapper 选项

可以通过 `putMapperOptions(...)` 覆盖底层对象映射器的对应选项：

```java
import dev.scx.object.x.mapper.time.TemporalAccessorNodeMapperOptions;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

var config = SerializeConfig.of().putMapperOptions(
    new TemporalAccessorNodeMapperOptions().setFormatter(
        LocalDateTime.class,
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS")
    )
);

String json = ScxSerialize.toJson(car, config);
Car car2 = ScxSerialize.fromJson(json, Car.class, config);
```

用户通过 `putMapperOptions(...)` 指定的 mapper 配置会覆盖 `SerializeConfig` 根据快捷配置生成的同类型选项。

## SerializeOptions

`SerializeOptions` 是底层配置接口，包含三组转换选项：

```java
DefaultObjectNodeConvertOptions objectNodeConvertOptions();
JsonNodeConvertOptions jsonNodeConvertOptions();
XmlNodeConvertOptions xmlNodeConvertOptions();
```

日常使用直接传入 `SerializeConfig` 即可。如果需要完整控制 JSON、XML 或 Object / Node 的底层转换行为，可以自行实现 `SerializeOptions`。

## 对象转换选项

`convertObject(...)` 使用同一个 `SerializeOptions` 配置两段转换：

```text
Object -> Node
Node -> Object
```

```java
var config = SerializeConfig.of()
    .ignoreNullValue(true)
    .singleValueArrayCompatibility(true);

UserDTO dto = ScxSerialize.convertObject(user, UserDTO.class, config);
```

传入的配置会分别通过 `objectNodeConvertOptions()` 应用于对象转 `Node` 和 `Node` 转对象。

## 访问底层 Converter

如果需要更底层的能力，可以获取默认的 `Serializer` 或其 converter：

```java
var serializer = ScxSerialize.serializer();
var objectNodeConverter = ScxSerialize.objectNodeConverter();
var jsonNodeConverter = ScxSerialize.jsonNodeConverter();
var xmlNodeConverter = ScxSerialize.xmlNodeConverter();
```

三个 converter 方法分别返回默认的 `DefaultObjectNodeConverter`、`JsonNodeConverter` 和 `XmlNodeConverter` 实例。

也可以创建独立的 `Serializer`：

```java
var serializer = new Serializer();
String json = serializer.toJson(user);
```

`Serializer` 允许通过构造函数传入自定义 converter，也可以继承后扩展功能。

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
import dev.scx.serialize.ScxSerialize;
import dev.scx.serialize.SerializeConfig;
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
            SerializeConfig.of()
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
            SerializeConfig.of()
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

`SerializeConfig` 统一提供对象映射、JSON 和 XML 的常用配置。同一个配置对象可用于两个格式，因此忽略 null、注解处理、数字转换、基本类型 null 策略、单值数组兼容等对象映射规则在 JSON 和 XML 上保持一致。

### 3. 注解处理可以关闭

默认启用 `@Ignore`。如果需要原样序列化或反序列化所有字段，可以通过 `useAnnotations(false)` 关闭注解处理。常见用法包括开启和关闭注解后的不同反序列化行为。

### 4. 格式选项和对象选项是分层的

`SerializeConfig` 对外提供一组统一配置，但内部仍会分别生成 `DefaultObjectNodeConvertOptions`、`JsonNodeConvertOptions` 和 `XmlNodeConvertOptions`。其中 `prettyPrint` 只进入 JSON 配置，其余对象映射选项进入 Object / Node 配置。

## 常见问题

### SCX Serialize 支持哪些格式？

当前公开 API 支持 JSON 和 XML，并提供 `fromJson`、`toJson`、`fromXml`、`toXml` 方法。

### 可以只用它做对象拷贝 / 类型转换吗？

可以。使用 `convertObject(...)` 即可，它会走 `Object -> Node -> Object` 流程。

### 默认会忽略 null 吗？

不会。`SerializeConfig` 的 `ignoreNullValue` 默认是 `false`。需要忽略 null 时，手动设置 `ignoreNullValue(true)`。

### 默认会启用注解吗？

会。`SerializeConfig` 的 `useAnnotations` 默认是 `true`。

### `@Ignore` 默认忽略哪个方向？

默认是 `Ignore.Mode.BOTH`，也就是序列化和反序列化都忽略。也可以指定 `SERIALIZE` 或 `DESERIALIZE`。

### JSON 可以格式化输出吗？

可以。使用 `SerializeConfig.of().prettyPrint(true)`。

### XML 有 prettyPrint 选项吗？

`SerializeConfig` 虽然提供 `prettyPrint`，但该配置只会传给 JSON converter，不影响 XML 输出。
