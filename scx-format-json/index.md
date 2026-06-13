# SCX Format JSON

SCX Format JSON 是 `scx-format` 的 JSON 实现模块。

它负责在 JSON 文本和 `scx-node` 的 `Node` 数据模型之间互相转换。

SCX Format JSON 基于 Jackson Core 实现 JSON 读写，但对外暴露的是 `Node` 转换接口，而不是 Jackson 的对象模型。

当前版本为 `0.3.0`。

[GitHub](https://github.com/scx-projects/scx-format-json)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-format-json</artifactId>
    <version>0.3.0</version>
</dependency>
```

`scx-format-json` 依赖：

```text
scx-format
jackson-core
```

因此普通使用场景中，只需要引入 `scx-format-json`。

## 基本概念

SCX Format JSON 中最核心的概念包括：

```text
JsonNodeConverter         JSON 和 Node 之间的转换器
JsonNodeConvertOptions    JSON 转换选项
JsonSerializer            Node -> JSON 的底层序列化器
JsonDeserializer          JSON -> Node 的底层反序列化器
DuplicateFieldPolicy      JSON 对象重复字段处理策略
LightJsonFactory          轻量 JsonFactory
```

它们之间的关系可以简单理解为：

```text
JSON 文本
    ↓
JsonNodeConverter#formatToNode(...)
    ↓
JsonDeserializer
    ↓
Node

Node
    ↓
JsonNodeConverter#nodeToFormat(...)
    ↓
JsonSerializer
    ↓
JSON 文本
```

也就是说：

```text
JsonNodeConverter 负责入口
JsonDeserializer 负责读取 JSON token 并生成 Node
JsonSerializer 负责把 Node 写成 JSON token
JsonNodeConvertOptions 负责配置读取和写出行为
```

## 快速开始

最简单的用法是把 JSON 字符串解析成 `Node`。

```java
import dev.scx.format.json.JsonNodeConvertOptions;
import dev.scx.format.json.JsonNodeConverter;
import dev.scx.node.Node;

var converter = new JsonNodeConverter();

Node node = converter.formatToNode(
    """
    {
      "name": "Tom",
      "age": 18,
      "active": true
    }
    """,
    new JsonNodeConvertOptions()
);
```

再把 `Node` 输出回 JSON：

```java
String json = converter.nodeToFormatString(
    node,
    new JsonNodeConvertOptions()
);
```

如果想要格式化输出：

```java
String prettyJson = converter.nodeToFormatString(
    node,
    new JsonNodeConvertOptions().prettyPrint(true)
);
```

## JsonNodeConverter

`JsonNodeConverter` 是本模块的核心入口。

它实现了 `FormatNodeConverter<JsonNodeConvertOptions>`。

常见方法包括：

```java
Node formatToNode(String text, JsonNodeConvertOptions options);

Node formatToNode(byte[] bytes, Charset charset, JsonNodeConvertOptions options);

Node formatToNode(Reader reader, JsonNodeConvertOptions options)
        throws IOException;

Node formatToNode(InputStream inputStream, Charset charset, JsonNodeConvertOptions options)
        throws IOException;

Node formatToNode(File file, Charset charset, JsonNodeConvertOptions options)
        throws IOException;

String nodeToFormatString(Node node, JsonNodeConvertOptions options);

byte[] nodeToFormatBytes(Node node, Charset charset, JsonNodeConvertOptions options);

void nodeToFormat(Node node, Writer writer, JsonNodeConvertOptions options)
        throws IOException;

void nodeToFormat(Node node, OutputStream outputStream, Charset charset, JsonNodeConvertOptions options)
        throws IOException;

File nodeToFormatFile(Node node, File file, Charset charset, JsonNodeConvertOptions options)
        throws IOException;
```

## JSON 转 Node

示例：

```java
var json = """
    {
      "user": {
        "id": 12345,
        "name": "小明",
        "active": true,
        "score": 99.99,
        "tags": ["程序员", "摄影师", "旅行者"],
        "updated_at": null
      }
    }
    """;

Node node = new JsonNodeConverter().formatToNode(
    json,
    new JsonNodeConvertOptions()
);
```

转换结果大致是：

```text
ObjectNode
    user -> ObjectNode
        id         -> IntNode(12345)
        name       -> StringNode("小明")
        active     -> BooleanNode.TRUE
        score      -> DoubleNode(99.99)
        tags       -> ArrayNode
        updated_at -> NullNode.NULL
```

## Node 转 JSON

示例：

```java
import dev.scx.node.ArrayNode;
import dev.scx.node.ObjectNode;

import static dev.scx.node.NullNode.NULL;

var user = new ObjectNode();

user.put("id", 12345);
user.put("name", "小明");
user.put("active", true);
user.put("score", 99.99);

var tags = new ArrayNode();
tags.add("程序员");
tags.add("摄影师");
tags.add("旅行者");

user.put("tags", tags);
user.put("updated_at", NULL);

var root = new ObjectNode();
root.put("user", user);

String json = new JsonNodeConverter().nodeToFormatString(
    root,
    new JsonNodeConvertOptions().prettyPrint(true)
);
```

输出类似：

```json
{
  "user" : {
    "id" : 12345,
    "name" : "小明",
    "active" : true,
    "score" : 99.99,
    "tags" : [ "程序员", "摄影师", "旅行者" ],
    "updated_at" : null
  }
}
```

具体空格和换行格式由 Jackson 的 pretty printer 决定。

## 支持的 Node 类型

JSON 序列化支持以下 `Node` 类型：

```text
ObjectNode
ArrayNode
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

对应 JSON 类型可以理解为：

```text
ObjectNode       -> JSON object
ArrayNode        -> JSON array
StringNode       -> JSON string
IntNode          -> JSON number
LongNode         -> JSON number
FloatNode        -> JSON number
DoubleNode       -> JSON number
BigIntegerNode   -> JSON number
BigDecimalNode   -> JSON number
BooleanNode      -> JSON boolean
NullNode         -> JSON null
```

## JSON 数字映射

JSON 读取时，整数和浮点数会根据 Jackson 返回的 number type 映射为不同 `Node`。

整数：

```text
INT          -> IntNode
LONG         -> LongNode
BIG_INTEGER  -> BigIntegerNode
```

浮点数：

```text
FLOAT        -> FloatNode
DOUBLE       -> DoubleNode
BIG_DECIMAL  -> BigDecimalNode
```

示例：

```java
Node node = converter.formatToNode(
    """
    {
      "i": 1,
      "l": 1234567890123,
      "d": 1.25
    }
    """,
    new JsonNodeConvertOptions()
);
```

读取后会得到对应数字节点。

## 标量 JSON

SCX Format JSON 不要求 JSON 顶层必须是对象或数组。

下面这些都是合法输入：

```json
"hello"
```

```json
123
```

```json
true
```

```json
null
```

例如：

```java
Node node = converter.formatToNode(
    "123",
    new JsonNodeConvertOptions()
);
```

会得到一个数字节点。

## 空输入和多余内容

如果输入中没有任何有效内容，会抛出 `FormatToNodeException`。

```java
converter.formatToNode("", new JsonNodeConvertOptions());
```

如果一个合法 JSON 值后面还有多余内容，也会抛出异常。

```java
converter.formatToNode(
    "{} {}",
    new JsonNodeConvertOptions()
);
```

这可以避免只读取前半段 JSON，而静默忽略后面的脏数据。

## DuplicateFieldPolicy

JSON 对象中理论上可能出现重复字段。

例如：

```json
{
  "name": "Tom",
  "name": "Jerry"
}
```

`JsonNodeConvertOptions` 提供 `duplicateFieldPolicy(...)` 用来控制重复字段处理策略。

可选值包括：

```text
USE_NEW    使用新值
USE_OLD    使用旧值
THROW      抛出异常
MERGE      合并为数组
```

默认值是：

```text
USE_NEW
```

### USE_NEW

使用新值覆盖旧值。

```java
var options = new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.USE_NEW);
```

输入：

```json
{
  "name": "Tom",
  "name": "Jerry"
}
```

结果相当于：

```json
{
  "name": "Jerry"
}
```

### USE_OLD

保留旧值，忽略新值。

```java
var options = new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.USE_OLD);
```

输入：

```json
{
  "name": "Tom",
  "name": "Jerry"
}
```

结果相当于：

```json
{
  "name": "Tom"
}
```

### THROW

发现重复字段时直接抛出异常。

```java
var options = new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.THROW);
```

输入：

```json
{
  "name": "Tom",
  "name": "Jerry"
}
```

会抛出 `FormatToNodeException`。

### MERGE

把重复字段合并成数组。

```java
var options = new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.MERGE);
```

输入：

```json
{
  "name": "Tom",
  "name": "Jerry"
}
```

结果相当于：

```json
{
  "name": ["Tom", "Jerry"]
}
```

如果旧值已经是数组，则会继续往这个数组中追加新值。

## 宽松 JSON 读取

`JsonNodeConvertOptions` 支持一些 Jackson JSON read features。

例如允许 Java 风格注释：

```java
var options = new JsonNodeConvertOptions()
    .allowJavaComments(true);
```

允许 YAML 风格注释：

```java
var options = new JsonNodeConvertOptions()
    .allowYamlComments(true);
```

允许单引号：

```java
var options = new JsonNodeConvertOptions()
    .allowSingleQuotes(true);
```

允许未加引号的属性名：

```java
var options = new JsonNodeConvertOptions()
    .allowUnquotedPropertyNames(true);
```

允许尾随逗号：

```java
var options = new JsonNodeConvertOptions()
    .allowTrailingComma(true);
```

允许缺失值：

```java
var options = new JsonNodeConvertOptions()
    .allowMissingValues(true);
```

这些选项适合处理非标准 JSON 或兼容历史数据。

如果你需要严格 JSON，应保持默认配置。

## 数字读取选项

可配置的数字读取相关选项包括：

```java
new JsonNodeConvertOptions()
    .allowLeadingDecimalPointForNumbers(true)
    .allowLeadingPlusSignForNumbers(true)
    .allowLeadingZerosForNumbers(true)
    .allowNonNumericNumbers(true)
    .allowTrailingDecimalPointForNumbers(true);
```

它们分别用于控制：

```text
.5
+1
001
NaN / Infinity
1.
```

这类非标准 JSON 数字形式。

## JSON 写出选项

可配置的写出选项包括：

```java
new JsonNodeConvertOptions()
    .quotePropertyNames(true)
    .writeNanAsStrings(true)
    .escapeNonAscii(false)
    .writeNumbersAsStrings(false);
```

含义可以理解为：

```text
quotePropertyNames       是否给属性名加引号
writeNanAsStrings        是否把 NaN 写为字符串
escapeNonAscii           是否转义非 ASCII 字符
writeNumbersAsStrings    是否把数字写为字符串
```

示例：转义非 ASCII 字符。

```java
var json = converter.nodeToFormatString(
    node,
    new JsonNodeConvertOptions().escapeNonAscii(true)
);
```

## Pretty Print

`prettyPrint(true)` 用于格式化输出。

```java
var options = new JsonNodeConvertOptions()
    .prettyPrint(true);

String json = converter.nodeToFormatString(node, options);
```

未开启 pretty print 时，输出更紧凑。

```java
var options = new JsonNodeConvertOptions()
    .prettyPrint(false);
```

默认值是：

```text
false
```

## 读写限制

`JsonNodeConvertOptions` 支持设置读写约束。

```java
var options = new JsonNodeConvertOptions()
    .maxNestingDepth(200)
    .maxDocumentLength(10_000_000)
    .maxTokenCount(1_000_000)
    .maxNumberLength(1_000)
    .maxStringLength(1_000_000)
    .maxNameLength(10_000);
```

这些选项用于限制输入规模，避免异常大输入造成资源消耗过高。

常见用途：

```text
限制最大嵌套深度
限制最大文档长度
限制最大 token 数量
限制数字长度
限制字符串长度
限制字段名长度
```

## 最大嵌套深度

`maxNestingDepth(...)` 会影响 JSON 解析时允许的最大嵌套层级。

示例：

```java
var json = "[".repeat(100) + "]".repeat(100);

converter.formatToNode(
    json,
    new JsonNodeConvertOptions().maxNestingDepth(80)
);
```

因为输入嵌套深度超过 `80`，所以会抛出 `FormatToNodeException`。

这个限制也间接保护递归序列化时的栈深度。

## CharacterEscapes

如果需要自定义字符转义，可以设置 `characterEscapes(...)`。

```java
var options = new JsonNodeConvertOptions()
    .characterEscapes(myCharacterEscapes);
```

这个选项直接传给底层 Jackson generator。

## Root Value Separator

可以设置根值分隔符。

```java
var options = new JsonNodeConvertOptions()
    .rootValueSeparator(mySeparator);
```

通常只有在连续写多个 root value 的特殊场景下才需要关注。

## highestNonEscapedChar

可以设置最高非转义字符。

```java
var options = new JsonNodeConvertOptions()
    .highestNonEscapedChar(127);
```

这类配置适合对输出字符集或转义策略有特殊要求的场景。

## quoteChar

默认 JSON 字符串和字段名使用双引号。

```text
"
```

可以通过 `quoteChar(...)` 修改。

```java
var options = new JsonNodeConvertOptions()
    .quoteChar('\'');
```

需要注意，修改 quote char 可能会生成非标准 JSON。

## Reader / InputStream / File

可以从字符流读取：

```java
try (var reader = new FileReader(file, StandardCharsets.UTF_8)) {
    Node node = converter.formatToNode(
        reader,
        new JsonNodeConvertOptions()
    );
}
```

可以从字节流读取：

```java
try (var inputStream = new FileInputStream(file)) {
    Node node = converter.formatToNode(
        inputStream,
        StandardCharsets.UTF_8,
        new JsonNodeConvertOptions()
    );
}
```

可以直接从文件读取：

```java
Node node = converter.formatToNode(
    file,
    StandardCharsets.UTF_8,
    new JsonNodeConvertOptions()
);
```

## 输出到 Writer / OutputStream / File

输出到字符流：

```java
try (var writer = new FileWriter(file, StandardCharsets.UTF_8)) {
    converter.nodeToFormat(
        node,
        writer,
        new JsonNodeConvertOptions().prettyPrint(true)
    );
}
```

输出到字节流：

```java
try (var outputStream = new FileOutputStream(file)) {
    converter.nodeToFormat(
        node,
        outputStream,
        StandardCharsets.UTF_8,
        new JsonNodeConvertOptions().prettyPrint(true)
    );
}
```

输出到文件：

```java
converter.nodeToFormatFile(
    node,
    file,
    StandardCharsets.UTF_8,
    new JsonNodeConvertOptions().prettyPrint(true)
);
```

## 异常处理

JSON 解析失败时，会抛出 `FormatToNodeException`。

```java
try {
    Node node = converter.formatToNode(text, options);
} catch (FormatToNodeException e) {
    // JSON -> Node 失败
}
```

Node 输出 JSON 失败时，会抛出 `NodeToFormatException`。

```java
try {
    String json = converter.nodeToFormatString(node, options);
} catch (NodeToFormatException e) {
    // Node -> JSON 失败
}
```

如果使用 Reader、InputStream、Writer、OutputStream 或 File，还可能出现 `IOException`。

```java
try {
    Node node = converter.formatToNode(file, StandardCharsets.UTF_8, options);
} catch (FormatToNodeException e) {
    // JSON 内容错误
} catch (IOException e) {
    // 文件读取错误
}
```

## 完整示例

```java
import dev.scx.format.json.JsonNodeConvertOptions;
import dev.scx.format.json.JsonNodeConverter;
import dev.scx.node.ArrayNode;
import dev.scx.node.ObjectNode;

import static dev.scx.node.NullNode.NULL;

public class JsonFormatDemo {

    public static void main(String[] args) {
        var converter = new JsonNodeConverter();

        var root = new ObjectNode();

        var user = new ObjectNode();
        user.put("id", 12345);
        user.put("name", "小明");
        user.put("active", true);
        user.put("score", 99.99);
        user.put("updated_at", NULL);

        var tags = new ArrayNode();
        tags.add("程序员");
        tags.add("摄影师");
        tags.add("旅行者");

        user.put("tags", tags);
        root.put("user", user);

        var options = new JsonNodeConvertOptions()
            .prettyPrint(true);

        String json = converter.nodeToFormatString(root, options);

        System.out.println(json);

        var node2 = converter.formatToNode(json, new JsonNodeConvertOptions());

        System.out.println(root.equals(node2));
    }

}
```

## 设计说明

### 1. JsonNodeConverter 是入口

使用者通常只需要直接使用：

```java
new JsonNodeConverter()
```

不需要直接操作 `JsonSerializer` 或 `JsonDeserializer`。

### 2. 基于 Jackson Core，而不是 Jackson Databind

本模块使用 Jackson Core 的 streaming API。

它不依赖 Jackson Databind，也不使用 Jackson 的 POJO 绑定模型。

转换边界是：

```text
JSON <-> Node
```

不是：

```text
JSON <-> Java Bean
```

### 3. 反序列化容器使用非递归方式

JSON 读取时，对象和数组容器使用栈结构处理。

这样可以避免深层 JSON 读取时直接依赖 Java 方法递归。

但序列化阶段仍然是递归下降方式。

因此仍然需要通过 `maxNestingDepth(...)` 限制极端深度。

### 4. 序列化阶段按 Node 类型写出

`JsonSerializer` 会根据具体 `Node` 类型调用对应的 JSON 写出方法。

例如：

```text
ObjectNode      writeStartObject / writeName / writeEndObject
ArrayNode       writeStartArray / writeEndArray
StringNode      writeString
IntNode         writeNumber
BooleanNode     writeBoolean
NullNode        writeNull
```

### 5. 重复字段策略是 JSON 模块自己的语义

`DuplicateFieldPolicy` 不是 `scx-format` 的核心语义。

它只属于 JSON 读取过程。

因为 JSON 对象中重复字段如何处理，并不是所有格式都共有的问题。

### 6. JsonNodeConvertOptions 是可变配置对象

`JsonNodeConvertOptions` 的 setter 方法会返回 `this`。

因此可以链式配置：

```java
var options = new JsonNodeConvertOptions()
    .prettyPrint(true)
    .allowTrailingComma(true)
    .duplicateFieldPolicy(DuplicateFieldPolicy.MERGE);
```

### 7. JsonNodeConverter 会复用 Jackson 读写对象

`JsonNodeConverter` 构造时会创建并复用部分较重对象。

因此推荐复用同一个 `JsonNodeConverter` 实例，而不是每次转换都创建新实例。

```java
private static final JsonNodeConverter JSON = new JsonNodeConverter();
```

## 常见问题

### SCX Format JSON 是 JSON 对象绑定库吗？

不是。

它只负责：

```text
JSON <-> Node
```

如果需要：

```text
Node <-> Java Object
```

应使用其它对象绑定模块。

### 它使用 Jackson 吗？

使用。

底层基于 Jackson Core。

### 它使用 Jackson Databind 吗？

不使用。

它不通过 Jackson Databind 绑定 Java Bean。

### 默认会格式化输出吗？

不会。

默认 `prettyPrint` 是 `false`。

需要格式化时：

```java
new JsonNodeConvertOptions().prettyPrint(true)
```

### 支持 JSON 顶层字符串、数字、布尔和 null 吗？

支持。

顶层不要求必须是对象或数组。

### 遇到重复字段默认怎么处理？

默认使用新值。

```text
DuplicateFieldPolicy.USE_NEW
```

### 如何让重复字段抛异常？

```java
new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.THROW)
```

### 如何把重复字段合并成数组？

```java
new JsonNodeConvertOptions()
    .duplicateFieldPolicy(DuplicateFieldPolicy.MERGE)
```

### 支持注释吗？

默认按 Jackson 默认值。

如果需要允许 Java 注释：

```java
new JsonNodeConvertOptions().allowJavaComments(true)
```

如果需要允许 YAML 注释：

```java
new JsonNodeConvertOptions().allowYamlComments(true)
```

### 支持单引号吗？

可以开启：

```java
new JsonNodeConvertOptions().allowSingleQuotes(true)
```

### 支持尾随逗号吗？

可以开启：

```java
new JsonNodeConvertOptions().allowTrailingComma(true)
```

### 如何限制 JSON 最大嵌套深度？

```java
new JsonNodeConvertOptions().maxNestingDepth(200)
```

### `formatToNode` 会自动关闭我传入的 Reader 吗？

不会。

外部传入资源由调用方关闭。

### `nodeToFormat` 会自动关闭我传入的 Writer 吗？

不会。

外部传入资源由调用方关闭。

### JSON 解析失败抛什么异常？

抛 `FormatToNodeException`。

底层 Jackson 异常会作为 cause 保存。

### JSON 输出失败抛什么异常？

抛 `NodeToFormatException`。

### 可以输出 UTF-8 字节数组吗？

可以。

```java
byte[] bytes = converter.nodeToFormatBytes(
    node,
    StandardCharsets.UTF_8,
    options
);
```

### 可以直接写入文件吗？

可以。

```java
converter.nodeToFormatFile(
    node,
    file,
    StandardCharsets.UTF_8,
    options
);
```

### JSON 和 Node 往返是否稳定？

对于 JSON 能表达的标准数据结构，通常是稳定的。

例如：

```text
JSON -> Node -> JSON -> Node
```

前后两个 `Node` 相等。

但如果你开启某些非标准选项，或者输入中包含重复字段，最终结果会受到选项影响。
