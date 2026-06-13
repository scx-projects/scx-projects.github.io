# SCX Format

SCX Format 是一个通用格式转换抽象库。

它定义了“外部文本格式”和 `scx-node` 的 `Node` 数据模型之间的转换接口。

SCX Format 本身不解析 JSON，也不解析 XML。它只提供统一抽象，让不同格式模块可以按照同一套接口完成：

```text
格式文本 -> Node
Node -> 格式文本
```

例如：

```text
JSON -> Node
Node -> JSON

XML -> Node
Node -> XML
```

真正的 JSON 实现由 `scx-format-json` 提供。

真正的 XML 实现由 `scx-format-xml` 提供。

当前版本为 `0.1.0`。

[GitHub](https://github.com/scx-projects/scx-format)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-format</artifactId>
    <version>0.1.0</version>
</dependency>
```

通常情况下，你不需要单独引入 `scx-format`。

如果你要使用 JSON，应引入：

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-format-json</artifactId>
    <version>0.3.0</version>
</dependency>
```

如果你要使用 XML，应引入：

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-format-xml</artifactId>
    <version>0.3.0</version>
</dependency>
```

这两个模块都会依赖 `scx-format`。

## 基本概念

SCX Format 中最核心的概念包括：

```text
FormatNodeConverter         格式和 Node 之间的转换器接口
FormatNodeConvertOptions    转换选项标记接口
FormatToNodeException       格式解析为 Node 时的异常
NodeToFormatException       Node 输出为格式文本时的异常
Node                        scx-node 提供的通用数据模型
```

它们之间的关系可以简单理解为：

```text
外部格式文本
    ↓
FormatNodeConverter#formatToNode(...)
    ↓
Node

Node
    ↓
FormatNodeConverter#nodeToFormat(...)
    ↓
外部格式文本
```

其中：

```text
scx-format         定义抽象接口
scx-format-json    实现 JSON <-> Node
scx-format-xml     实现 XML <-> Node
```

## 设计目标

SCX Format 的目标不是把所有格式都强行变成同一种语义。

它的目标是提供一套统一转换边界。

也就是说，不同格式模块可以共享下面这些调用方式：

```java
formatToNode(...)
nodeToFormat(...)
nodeToFormatString(...)
nodeToFormatBytes(...)
nodeToFormatFile(...)
```

但每一种格式如何映射到 `Node`，由具体格式模块决定。

例如：

```text
JSON 和 Node 语义天然接近
XML 和 Node 语义并不完全一致
```

所以 JSON 模块可以做比较直接的结构映射。

XML 模块则需要定义一套 data-centric XML 到 Node 的规范化映射规则。

## FormatNodeConverter

`FormatNodeConverter` 是核心接口。

它负责在某种格式和 `Node` 之间互相转换。

接口可以理解为：

```java
public interface FormatNodeConverter<O extends FormatNodeConvertOptions> {

    Node formatToNode(Reader reader, O options)
            throws FormatToNodeException, IOException;

    Node formatToNode(InputStream inputStream, Charset charset, O options)
            throws FormatToNodeException, IOException;

    Node formatToNode(String text, O options)
            throws FormatToNodeException;

    Node formatToNode(byte[] bytes, Charset charset, O options)
            throws FormatToNodeException;

    Node formatToNode(File file, Charset charset, O options)
            throws FormatToNodeException, IOException;

    void nodeToFormat(Node node, Writer writer, O options)
            throws NodeToFormatException, IOException;

    void nodeToFormat(Node node, OutputStream outputStream, Charset charset, O options)
            throws NodeToFormatException, IOException;

    String nodeToFormatString(Node node, O options)
            throws NodeToFormatException;

    byte[] nodeToFormatBytes(Node node, Charset charset, O options)
            throws NodeToFormatException;

    File nodeToFormatFile(Node node, File file, Charset charset, O options)
            throws NodeToFormatException, IOException;

}
```

这个接口只描述转换能力，不限制底层实现。

实现可以基于：

```text
Jackson
Woodstox
JDK XML
自定义 parser
二进制 parser
其它格式库
```

## 转换方向

`FormatNodeConverter` 有两个主要方向。

### formatToNode

`formatToNode(...)` 表示：

```text
外部格式 -> Node
```

例如：

```java
Node node = converter.formatToNode(text, options);
```

常见输入包括：

```text
Reader
InputStream
String
byte[]
File
```

### nodeToFormat

`nodeToFormat(...)` 表示：

```text
Node -> 外部格式
```

例如：

```java
String text = converter.nodeToFormatString(node, options);
```

常见输出包括：

```text
Writer
OutputStream
String
byte[]
File
```

## Node 是中间数据模型

SCX Format 使用 `scx-node` 的 `Node` 作为中间数据模型。

常见节点类型包括：

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

可以把 `Node` 理解为一个格式无关的数据树。

例如 JSON：

```json
{
  "name": "Tom",
  "age": 18,
  "tags": ["java", "scx"]
}
```

可以转换成类似这样的 `Node`：

```text
ObjectNode
    name -> StringNode("Tom")
    age  -> IntNode(18)
    tags -> ArrayNode
        StringNode("java")
        StringNode("scx")
```

然后这个 `Node` 还可以被输出为其它格式。

## 快速开始

SCX Format 本身只是抽象层，通常配合具体实现使用。

以 JSON 为例：

```java
import dev.scx.format.json.JsonNodeConvertOptions;
import dev.scx.format.json.JsonNodeConverter;
import dev.scx.node.Node;

var converter = new JsonNodeConverter();

Node node = converter.formatToNode("""
    {
      "name": "Tom",
      "age": 18
    }
    """, new JsonNodeConvertOptions());

String json = converter.nodeToFormatString(
    node,
    new JsonNodeConvertOptions().prettyPrint(true)
);
```

以 XML 为例：

```java
import dev.scx.format.xml.XmlNodeConverter;
import dev.scx.format.xml.XmlNodeConverterOptions;
import dev.scx.node.Node;

var converter = new XmlNodeConverter();

Node node = converter.formatToNode("""
    <user>
        <name>Tom</name>
        <age>18</age>
    </user>
    """, new XmlNodeConverterOptions());

String xml = converter.nodeToFormatString(
    node,
    new XmlNodeConverterOptions().rootName("user")
);
```

## Reader 和 Writer

如果你已经有字符流，可以使用 `Reader` 和 `Writer`。

```java
try (var reader = new StringReader(text)) {
    Node node = converter.formatToNode(reader, options);
}
```

写出：

```java
try (var writer = new StringWriter()) {
    converter.nodeToFormat(node, writer, options);
    String text = writer.toString();
}
```

需要注意：

```text
FormatNodeConverter 不会自动关闭传入的 Reader / Writer
```

关闭资源由调用方负责。

## InputStream 和 OutputStream

如果你处理的是字节流，可以使用 `InputStream` 和 `OutputStream`。

```java
try (var inputStream = new FileInputStream("data.json")) {
    Node node = converter.formatToNode(
        inputStream,
        StandardCharsets.UTF_8,
        options
    );
}
```

写出：

```java
try (var outputStream = new FileOutputStream("data.json")) {
    converter.nodeToFormat(
        node,
        outputStream,
        StandardCharsets.UTF_8,
        options
    );
}
```

这种方式需要显式传入 `Charset`。

```java
StandardCharsets.UTF_8
```

## String 和 byte[]

如果数据已经在内存中，可以直接使用 `String` 或 `byte[]`。

字符串读取：

```java
Node node = converter.formatToNode(text, options);
```

字节数组读取：

```java
Node node = converter.formatToNode(
    bytes,
    StandardCharsets.UTF_8,
    options
);
```

输出为字符串：

```java
String text = converter.nodeToFormatString(node, options);
```

输出为字节数组：

```java
byte[] bytes = converter.nodeToFormatBytes(
    node,
    StandardCharsets.UTF_8,
    options
);
```

## File

如果输入输出目标是文件，可以使用 `File` 方法。

```java
Node node = converter.formatToNode(
    new File("data.json"),
    StandardCharsets.UTF_8,
    options
);
```

写出到文件：

```java
File file = converter.nodeToFormatFile(
    node,
    new File("data.json"),
    StandardCharsets.UTF_8,
    options
);
```

返回值就是写入的 `File` 对象。

## FormatNodeConvertOptions

`FormatNodeConvertOptions` 是转换选项接口。

它本身没有定义任何方法。

```java
public interface FormatNodeConvertOptions {

}
```

它的作用是给不同格式模块提供统一的选项类型边界。

例如：

```text
JsonNodeConvertOptions implements FormatNodeConvertOptions
XmlNodeConverterOptions  implements FormatNodeConvertOptions
```

JSON 和 XML 的配置项完全不同，因此不能放在核心接口中。

JSON 可以有：

```text
prettyPrint
allowSingleQuotes
allowTrailingComma
duplicateFieldPolicy
maxNestingDepth
```

XML 可以有：

```text
rootName
itemName
maxNestingDepth
maxChildCount
maxStringLength
```

这些都属于具体格式模块自己的选项。

## FormatToNodeException

`FormatToNodeException` 表示：

```text
外部格式 -> Node
```

这个方向发生的转换异常。

例如：

```java
try {
    Node node = converter.formatToNode(text, options);
} catch (FormatToNodeException e) {
    // 格式解析失败，或者无法映射为 Node
}
```

常见原因包括：

```text
格式文本不合法
格式语义无法映射为 Node
超过最大嵌套深度
超过长度限制
重复字段策略要求抛异常
XML 出现 mixed content
底层 parser 抛出异常
```

`FormatToNodeException` 继承自 `RuntimeException`。

它提供三个构造方法：

```java
public FormatToNodeException(String message)

public FormatToNodeException(Throwable cause)

public FormatToNodeException(String message, Throwable cause)
```

## NodeToFormatException

`NodeToFormatException` 表示：

```text
Node -> 外部格式
```

这个方向发生的转换异常。

例如：

```java
try {
    String text = converter.nodeToFormatString(node, options);
} catch (NodeToFormatException e) {
    // Node 无法输出为目标格式
}
```

常见原因包括：

```text
Node 结构不能表示为目标格式
超过最大嵌套深度
字段名不合法
底层 writer 抛出异常
```

`NodeToFormatException` 继承自 `RuntimeException`。

它提供三个构造方法：

```java
public NodeToFormatException(String message)

public NodeToFormatException(Throwable cause)

public NodeToFormatException(String message, Throwable cause)
```

## 异常边界

SCX Format 把异常分成两类：

```text
FormatToNodeException      读取和解析方向
NodeToFormatException      输出和序列化方向
```

这样调用方可以很清楚地区分异常发生在哪一步。

示例：

```java
try {
    Node node = converter.formatToNode(text, options);
    String output = converter.nodeToFormatString(node, options);
} catch (FormatToNodeException e) {
    // 输入格式解析失败
} catch (NodeToFormatException e) {
    // Node 输出失败
}
```

如果使用流或文件，还可能出现 `IOException`。

```java
try {
    Node node = converter.formatToNode(file, StandardCharsets.UTF_8, options);
} catch (FormatToNodeException e) {
    // 文件内容无法转换为 Node
} catch (IOException e) {
    // 文件读取失败
}
```

## 资源关闭规则

`FormatNodeConverter` 的注释明确说明：

```text
不会自动关闭传入的资源
```

也就是说，如果你传入的是调用方创建的资源：

```java
Reader
InputStream
Writer
OutputStream
```

那么关闭责任属于调用方。

推荐写法：

```java
try (var reader = new FileReader(file, StandardCharsets.UTF_8)) {
    Node node = converter.formatToNode(reader, options);
}
```

或者：

```java
try (var writer = new FileWriter(file, StandardCharsets.UTF_8)) {
    converter.nodeToFormat(node, writer, options);
}
```

对于 `String`、`byte[]`、`File` 这类便捷方法，实现会自行创建必要的临时流或 reader。

## 与 SCX Node 的关系

SCX Format 依赖 `scx-node`。

SCX Format 不负责 Java Bean 绑定，也不直接把 JSON/XML 转成某个 Java 类。

它的转换边界是：

```text
格式文本 <-> Node
```

如果你需要：

```text
Node <-> Java Object
```

应使用 `scx-node` 或其它上层模块完成。

这让格式层保持简单：

```text
格式模块只关心格式语法
Node 模块只关心数据模型
对象绑定模块只关心 Java 对象映射
```

## 与 JSON/XML 实现模块的关系

SCX Format 是核心抽象。

```text
scx-format
    FormatNodeConverter
    FormatNodeConvertOptions
    FormatToNodeException
    NodeToFormatException
```

JSON 实现模块：

```text
scx-format-json
    JsonNodeConverter
    JsonNodeConvertOptions
    DuplicateFieldPolicy
```

XML 实现模块：

```text
scx-format-xml
    XmlNodeConverter
    XmlNodeConverterOptions
    XmlElementConverter
    ElementNodeConverter
    Element
    TagElement
    TextElement
```

一般使用者直接使用具体模块。

例如：

```java
new JsonNodeConverter()
```

或者：

```java
new XmlNodeConverter()
```

只有在你要写一个新的格式模块时，才需要直接关注 `FormatNodeConverter`。

## 实现一个自定义格式转换器

如果你想给新的格式实现转换能力，可以实现 `FormatNodeConverter`。

例如一个简单的行文本格式：

```java
public final class LinesNodeConverter
        implements FormatNodeConverter<LinesNodeConvertOptions> {

    @Override
    public Node formatToNode(String text, LinesNodeConvertOptions options)
            throws FormatToNodeException {
        var array = new ArrayNode();
        for (var line : text.split("\\R")) {
            array.add(line);
        }
        return array;
    }

    @Override
    public String nodeToFormatString(Node node, LinesNodeConvertOptions options)
            throws NodeToFormatException {
        if (!(node instanceof ArrayNode arrayNode)) {
            throw new NodeToFormatException("Expected ArrayNode");
        }

        var sb = new StringBuilder();

        for (var item : arrayNode) {
            if (!sb.isEmpty()) {
                sb.append(System.lineSeparator());
            }
            sb.append(item.asString());
        }

        return sb.toString();
    }

    // 其它 Reader/InputStream/File 方法可按需要委托到 String/byte[] 方法
}
```

自定义选项：

```java
public final class LinesNodeConvertOptions implements FormatNodeConvertOptions {

}
```

这体现了 SCX Format 的扩展方式：

```text
新格式 = 新的 FormatNodeConverter 实现 + 新的 Options 类型
```

## 常见用法

### JSON 转 Node

```java
var converter = new JsonNodeConverter();

Node node = converter.formatToNode(
    """
    {
      "name": "Tom",
      "age": 18
    }
    """,
    new JsonNodeConvertOptions()
);
```

### Node 转 JSON

```java
String json = converter.nodeToFormatString(
    node,
    new JsonNodeConvertOptions().prettyPrint(true)
);
```

### XML 转 Node

```java
var converter = new XmlNodeConverter();

Node node = converter.formatToNode(
    """
    <user>
        <name>Tom</name>
        <age>18</age>
    </user>
    """,
    new XmlNodeConverterOptions()
);
```

### Node 转 XML

```java
String xml = converter.nodeToFormatString(
    node,
    new XmlNodeConverterOptions().rootName("user")
);
```

## 设计说明

### 1. SCX Format 是抽象层

SCX Format 本身不绑定任何具体格式。

它只定义接口。

具体格式由其它模块实现。

### 2. Node 是统一中间层

所有格式都先转换为 `Node`。

这样可以避免每个格式模块都直接绑定 Java Bean。

```text
JSON -> Node -> Java Object
XML  -> Node -> Java Object
```

中间的 `Node` 可以被复用。

### 3. Options 由具体格式定义

不同格式需要不同配置。

所以核心层只定义空的 `FormatNodeConvertOptions`。

具体模块自己扩展。

### 4. 输入输出形式统一

无论是 JSON、XML 还是其它格式，都可以使用类似方法：

```java
formatToNode(String, options)
nodeToFormatString(Node, options)
```

这让调用方可以在抽象层面处理格式转换。

### 5. 不自动关闭外部资源

传入的 `Reader`、`InputStream`、`Writer` 和 `OutputStream` 不会由转换器自动关闭。

这是为了避免转换器意外关闭调用方仍然要使用的资源。

### 6. 异常方向明确

读取方向使用：

```text
FormatToNodeException
```

输出方向使用：

```text
NodeToFormatException
```

这样异常语义比较清楚。

## 常见问题

### SCX Format 可以直接解析 JSON 吗？

不可以。

`scx-format` 只是抽象层。

解析 JSON 需要使用 `scx-format-json`。

### SCX Format 可以直接解析 XML 吗？

不可以。

解析 XML 需要使用 `scx-format-xml`。

### 为什么不直接把 JSON/XML 转 Java Bean？

因为 SCX Format 的职责是格式转换。

它只负责：

```text
格式文本 <-> Node
```

对象绑定属于另一层职责。

### Reader 会被自动关闭吗？

不会。

传入的外部资源由调用方关闭。

### InputStream 会被自动关闭吗？

不会。

传入的外部资源由调用方关闭。

### String 和 byte[] 方法会涉及资源关闭吗？

这类方法通常由实现创建临时 reader 或 stream。

调用方不需要关心临时资源。

### FormatToNodeException 和 NodeToFormatException 都是运行时异常吗？

是的。

它们都继承自 `RuntimeException`。

### 为什么还会抛 IOException？

如果方法接收 `Reader`、`InputStream`、`Writer`、`OutputStream` 或 `File`，底层 I/O 仍然可能失败。

因此这些方法会保留 `IOException`。

### 自定义格式应该怎么接入？

实现 `FormatNodeConverter`，并定义自己的 `FormatNodeConvertOptions`。

### JSON 和 XML 的转换结果一定完全可逆吗？

不一定。

这取决于具体格式模块。

JSON 和 Node 的结构比较接近，通常更容易保持结构一致。

XML 和 Node 的语义并不完全一致，因此 XML 模块采用的是规范化映射，不保证保留原始 XML 的所有结构信息。
