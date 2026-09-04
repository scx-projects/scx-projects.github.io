# SCX Format XML

SCX Format XML 是 `scx-format` 的 XML 实现模块。

它负责在 XML 文本和 `scx-node` 的 `Node` 数据模型之间互相转换。

由于 XML 和 JSON/Node 的数据模型并不完全一致，所以 SCX Format XML 明确面向 data-centric XML，也就是数据型 XML。

它不试图完整保留文档型 XML 的所有结构信息。

[GitHub](https://github.com/scx-projects/scx-format-xml)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-format-xml</artifactId>
    <version>0.10.0</version>
</dependency>
```

`scx-format-xml` 依赖：

```text
scx-format
woodstox-core
```

因此普通使用场景中，只需要引入 `scx-format-xml`。

## 基本概念

SCX Format XML 中最核心的概念包括：

```text
XmlNodeConverter          XML 和 Node 之间的转换器
XmlNodeConverterOptions   XML 转换选项
XmlElementConverter       XML 文本和 Element 之间的转换器
ElementNodeConverter      Element 和 Node 之间的转换器
Element                   XML 元素模型接口
TagElement                XML 标签元素
TextElement               XML 文本元素
Attribute                 XML 属性
```

它们之间的关系可以简单理解为：

```text
XML 文本
    ↓
XmlElementConverter
    ↓
Element
    ↓
ElementNodeConverter
    ↓
Node

Node
    ↓
ElementNodeConverter
    ↓
Element
    ↓
XmlElementConverter
    ↓
XML 文本
```

也就是说：

```text
XmlElementConverter 负责 XML 语法读写
ElementNodeConverter 负责 XML 结构和 Node 结构之间的规范化映射
XmlNodeConverter 负责把两步组合成统一入口
```

## 为什么需要 Element 层

JSON 和 Node 的结构很接近。

但是 XML 和 Node 差异很大。

XML 有：

```text
根标签
属性
子元素
文本节点
CDATA
自闭合标签
同名子标签
mixed content
```

Node 有：

```text
ObjectNode
ArrayNode
ValueNode
NullNode
```

两者不是一一对应的。

所以 SCX Format XML 中间增加了 `Element` 层。

```text
XML 文本 <-> Element <-> Node
```

这样可以把问题拆开：

```text
XML 语法解析问题       交给 XmlElementConverter
XML 到 Node 映射问题   交给 ElementNodeConverter
```

## 快速开始

把 XML 转成 `Node`：

```java
import dev.scx.format.xml.XmlNodeConverter;
import dev.scx.format.xml.XmlNodeConverterOptions;
import dev.scx.node.Node;

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

把 `Node` 输出为 XML：

```java
String xml = converter.nodeToFormatString(
    node,
    new XmlNodeConverterOptions().rootName("user")
);
```

## 默认转换器

`XmlNodeConverter` 提供了默认实例：

```java
public static final XmlNodeConverter DEFAULT_XML_NODE_CONVERTER
```

可以这样使用：

```java
import static dev.scx.format.xml.XmlNodeConverter.DEFAULT_XML_NODE_CONVERTER;

var node = DEFAULT_XML_NODE_CONVERTER.formatToNode(
    xml,
    new XmlNodeConverterOptions()
);
```

也可以自己创建：

```java
var converter = new XmlNodeConverter();
```

## XmlNodeConverter

`XmlNodeConverter` 是 `FormatNodeConverter<XmlNodeConverterOptions>` 的实现。

它把 XML 转 Node 的过程分成两步：

```text
XML -> Element -> Node
```

把 Node 转 XML 的过程也分成两步：

```text
Node -> Element -> XML
```

常见方法包括：

```java
Node formatToNode(String text, XmlNodeConverterOptions options);

Node formatToNode(byte[] bytes, Charset charset, XmlNodeConverterOptions options);

Node formatToNode(Reader reader, XmlNodeConverterOptions options)
        throws IOException;

Node formatToNode(InputStream inputStream, Charset charset, XmlNodeConverterOptions options)
        throws IOException;

Node formatToNode(File file, Charset charset, XmlNodeConverterOptions options)
        throws IOException;

String nodeToFormatString(Node node, XmlNodeConverterOptions options);

byte[] nodeToFormatBytes(Node node, Charset charset, XmlNodeConverterOptions options);

void nodeToFormat(Node node, Writer writer, XmlNodeConverterOptions options)
        throws IOException;

void nodeToFormat(Node node, OutputStream outputStream, Charset charset, XmlNodeConverterOptions options)
        throws IOException;

File nodeToFormatFile(Node node, File file, Charset charset, XmlNodeConverterOptions options)
        throws IOException;
```

## XmlElementConverter

`XmlElementConverter` 只负责：

```text
XML 文本 <-> Element
```

它不关心 `Node`。

可以单独使用它读取 XML 元素结构：

```java
var element = DEFAULT_XML_NODE_CONVERTER
    .xmlElementConverter()
    .formatToElement(
        """
        <user>
            <name>Tom</name>
            <age>18</age>
        </user>
        """,
        new XmlNodeConverterOptions()
    );
```

也可以把 `Element` 输出回 XML：

```java
String xml = DEFAULT_XML_NODE_CONVERTER
    .xmlElementConverter()
    .elementToFormatString(
        element,
        new XmlNodeConverterOptions()
    );
```

这适合你只想处理 XML 结构，而不想转成 `Node` 的场景。

## Element 模型

SCX Format XML 使用一个简单的 XML 元素模型。

```java
public sealed interface Element permits TagElement, TextElement {

}
```

也就是说，`Element` 只有两种实现：

```text
TagElement      标签元素
TextElement     文本元素
```

### TagElement

`TagElement` 表示 XML 标签。

它包含：

```text
tagName          标签名
useSelfClosing   是否使用自闭合标签
attributes       属性列表
children         子元素列表
```

示例：

```java
import dev.scx.format.xml.element.TagElement;
import dev.scx.format.xml.element.TextElement;

var user = new TagElement("user", false);

user.addAttribute("id", "123");

var name = new TagElement("name", false);
name.add(new TextElement("Tom"));

user.add(name);
```

对应 XML：

```xml
<user id="123">
    <name>Tom</name>
</user>
```

### TextElement

`TextElement` 表示 XML 文本。

```java
import dev.scx.format.xml.element.TextElement;

var text = new TextElement("hello");
```

对应 XML 文本内容：

```text
hello
```

### Attribute

`Attribute` 是一个 record。

```java
public record Attribute(String name, String value) {

}
```

属性值始终是字符串。

## XmlNodeConverterOptions

`XmlNodeConverterOptions` 是 XML 转换选项。

默认值包括：

```text
maxNestingDepth   200
maxChildCount     5000
maxStringLength   20000000
rootName          root
itemName          item
```

可以链式配置：

```java
var options = new XmlNodeConverterOptions()
    .rootName("user")
    .itemName("item")
    .maxNestingDepth(200)
    .maxChildCount(5000)
    .maxStringLength(20_000_000);
```

## rootName

`rootName` 用于 Node 转 XML 时的根标签名称。

默认值是：

```text
root
```

示例：

```java
String xml = converter.nodeToFormatString(
    node,
    new XmlNodeConverterOptions().rootName("user")
);
```

如果 `node` 是：

```json
{
  "name": "Tom",
  "age": 18
}
```

输出大致为：

```xml
<user>
    <name>Tom</name>
    <age>18</age>
</user>
```

如果设置为：

```java
new XmlNodeConverterOptions().rootName("data")
```

根标签就是：

```xml
<data>
    ...
</data>
```

`rootName` 不允许为 `null`。

## itemName

`itemName` 用于顶层数组或无字段名上下文的数组元素。

默认值是：

```text
item
```

例如顶层数组：

```java
var array = new ArrayNode();

array.add("a");
array.add("b");
array.add("c");

String xml = converter.nodeToFormatString(
    array,
    new XmlNodeConverterOptions()
);
```

输出大致为：

```xml
<root>
    <item>a</item>
    <item>b</item>
    <item>c</item>
</root>
```

修改 `itemName`：

```java
new XmlNodeConverterOptions().itemName("value")
```

输出大致为：

```xml
<root>
    <value>a</value>
    <value>b</value>
    <value>c</value>
</root>
```

`itemName` 不允许为 `null`。

## maxNestingDepth

`maxNestingDepth` 用于限制最大嵌套深度。

默认值是：

```text
200
```

示例：

```java
var options = new XmlNodeConverterOptions()
    .maxNestingDepth(100);
```

如果 XML 或 Node 嵌套超过限制，会抛出异常。

这个限制同时服务于：

```text
防止过深 XML 解析
防止过深 Element -> Node 转换
防止过深 Node -> Element 转换
防止递归序列化导致栈溢出
```

`maxNestingDepth` 不能小于 `0`。

## maxChildCount

`maxChildCount` 用于限制单个元素的最大子元素数量。

默认值是：

```text
5000
```

它同时作用于：

```text
子元素数量
属性数量
```

示例：

```java
var options = new XmlNodeConverterOptions()
    .maxChildCount(1000);
```

`maxChildCount` 不能小于 `0`。

## maxStringLength

`maxStringLength` 用于限制最大字符串长度。

默认值是：

```text
20000000
```

它同时作用于：

```text
属性值
文本内容
```

示例：

```java
var options = new XmlNodeConverterOptions()
    .maxStringLength(1_000_000);
```

`maxStringLength` 不能小于 `0`。

## XML 安全限制

SCX Format XML 使用 Woodstox 作为 XML 读写实现。

读取 XML 时会设置一些安全限制。

包括：

```text
最大元素深度
最大文本长度
最大属性长度
每个元素最大子元素数量
每个元素最大属性数量
禁用 DTD
禁用外部实体
```

其中 DTD 和外部实体被关闭：

```text
XMLInputFactory.SUPPORT_DTD = false
XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES = false
```

这样可以避免 XML 外部实体相关风险。

## XML -> Element

XML 解析为 `Element` 时，会保留：

```text
标签名
属性名
属性值
子标签
非空白文本
CDATA 文本
自闭合标签信息
```

示例：

```xml
<user id="123">
    <name>Tom</name>
    <age>18</age>
</user>
```

会得到类似结构：

```text
TagElement("user")
    attributes:
        id = "123"
    children:
        TagElement("name")
            TextElement("Tom")
        TagElement("age")
            TextElement("18")
```

纯空白文本节点会被忽略。

例如缩进和换行不会生成有效 `TextElement`。

## XML -> Node 映射规则

XML 到 Node 的映射由 `ElementNodeConverter#elementToNode(...)` 完成。

它面向 data-centric XML。

### 1. 纯空白文本忽略

元素内部的纯空白文本会被忽略。

例如：

```xml
<user>

</user>
```

不会把缩进和换行当作有效文本数据。

### 2. 文本元素转 StringNode

如果标签没有属性，也没有子标签，那么它会作为文本元素处理。

```xml
<name>Tom</name>
```

转换为：

```json
"Tom"
```

需要注意，XML 文本没有数字、布尔等原生类型。

因此：

```xml
<age>18</age>
```

转换后是字符串：

```json
"18"
```

不是数字：

```json
18
```

### 3. 自闭合标签转 NullNode

自闭合标签表示 `null`。

```xml
<name/>
```

转换为：

```json
null
```

### 4. 空标签转空字符串

非自闭合空标签表示空字符串。

```xml
<name></name>
```

转换为：

```json
""
```

### 5. 有属性或子元素的标签转 ObjectNode

如果元素包含属性或子元素，则转换为 `ObjectNode`。

```xml
<user name="Tom">
    <age>18</age>
</user>
```

转换为：

```json
{
  "name": "Tom",
  "age": "18"
}
```

### 6. 属性作为字段

XML 属性会作为对象字段。

```xml
<user name="Tom" age="18"/>
```

转换为：

```json
{
  "name": "Tom",
  "age": "18"
}
```

属性值始终是字符串。

### 7. 子元素作为字段

子元素也会作为对象字段。

```xml
<user>
    <name>Tom</name>
    <age>18</age>
</user>
```

转换为：

```json
{
  "name": "Tom",
  "age": "18"
}
```

### 8. 属性和子元素来源统一

属性和子元素都会成为字段。

如果同名，就按重复字段处理。

```xml
<user name="Tom">
    <name>Jerry</name>
</user>
```

转换为：

```json
{
  "name": ["Tom", "Jerry"]
}
```

### 9. 重复字段合并为数组

重复标签会自动合并为 `ArrayNode`。

```xml
<user>
    <tag>java</tag>
    <tag>scx</tag>
</user>
```

转换为：

```json
{
  "tag": ["java", "scx"]
}
```

### 10. 同名字段允许异构数组

同名字段合并出的数组可以包含不同类型。

```xml
<root>
    <x>123</x>
    <x/>
    <x>
        <y>1</y>
    </x>
</root>
```

转换为：

```json
{
  "x": [
    "123",
    null,
    {
      "y": "1"
    }
  ]
}
```

### 11. 不支持 mixed content

如果一个元素同时包含有效文本和属性或子元素，会被视为 mixed content。

例如：

```xml
<p>hello <b>world</b></p>
```

或者：

```xml
<p id="1">hello</p>
```

都会在转换为 `Node` 时抛出 `FormatToNodeException`。

原因是 SCX Format XML 面向 data-centric XML，而不是文档型 XML。

mixed content 无法稳定映射为 `ObjectNode` / `StringNode` 而不损失语义。

## Node -> XML 映射规则

Node 到 XML 的映射由 `ElementNodeConverter#nodeToElement(...)` 完成。

### 1. 总是生成根标签

无论输入 Node 是什么，输出 XML 都会有根标签。

根标签名称由 `rootName` 控制。

默认是：

```text
root
```

### 2. ValueNode 转文本标签

非 null 标量节点会转换成文本标签。

例如：

```json
"Tom"
```

输出：

```xml
<root>Tom</root>
```

数字和布尔值会使用字符串形式输出：

```json
123
```

输出：

```xml
<root>123</root>
```

```json
true
```

输出：

```xml
<root>true</root>
```

### 3. NullNode 转自闭合标签

```json
null
```

输出：

```xml
<root/>
```

### 4. ObjectNode 转子元素

对象字段会转换为子元素。

```json
{
  "name": "Tom",
  "age": 18
}
```

输出：

```xml
<root>
    <name>Tom</name>
    <age>18</age>
</root>
```

需要注意，字段不会转换为属性。

Node -> XML 时统一使用子元素。

### 5. 对象字段中的 ArrayNode 转重复同名子元素

```json
{
  "tag": ["java", "scx"]
}
```

输出：

```xml
<root>
    <tag>java</tag>
    <tag>scx</tag>
</root>
```

也就是说，对象字段中的数组表示重复同名标签。

### 6. 顶层 ArrayNode 使用 itemName

```json
["a", "b", "c"]
```

输出：

```xml
<root>
    <item>a</item>
    <item>b</item>
    <item>c</item>
</root>
```

其中 `item` 由 `itemName` 控制。

### 7. 嵌套数组不会扁平化

```json
[1, [2]]
```

输出结构类似：

```xml
<root>
    <item>1</item>
    <item>
        <item>2</item>
    </item>
</root>
```

嵌套数组会继续递归转换。

### 8. 空字段名非法

XML 标签名不能是空字符串。

如果 `ObjectNode` 中存在空字符串字段名，会抛出 `NodeToFormatException`。

```java
var node = new ObjectNode();

node.put("", "123");

converter.nodeToFormatString(node, new XmlNodeConverterOptions());
```

会抛出异常。

## XML 往返不是原始结构保真

SCX Format XML 的目标是稳定的 data-centric XML 映射，不是保留原始 XML 文档结构。

因此下面这些信息可能不会保留：

```text
属性来源信息
属性顺序
子元素顺序的某些语义
CDATA 和普通文本的区别
空对象和空字符串的区别
空数组字段
数字/布尔原始类型
注释
处理指令
DTD
命名空间前缀细节
```

例如：

```xml
<user name="Tom"/>
```

转成 Node：

```json
{
  "name": "Tom"
}
```

再输出 XML：

```xml
<root>
    <name>Tom</name>
</root>
```

属性不会还原为属性，而是变成子元素。

这是设计选择，不是 bug。

## XML 数值会字符串化

XML 文本天然是字符串。

因此 XML -> Node 时：

```xml
<age>18</age>
```

得到的是：

```json
"18"
```

而不是：

```json
18
```

Node -> XML -> Node 后，非 null 标量也会字符串化。

例如：

```json
{
  "age": 18,
  "active": true
}
```

经过 XML 往返后会变成：

```json
{
  "age": "18",
  "active": "true"
}
```

如果需要恢复数字或布尔类型，应在后续对象绑定阶段处理类型转换。

## 空对象和空数组

由于 XML 没有直接表达“空对象”和“空数组”的通用语义，部分结构经过 XML 往返会退化。

例如空对象：

```json
{}
```

输出类似：

```xml
<root></root>
```

再读回来可能是：

```json
""
```

空数组：

```json
[]
```

输出类似：

```xml
<root></root>
```

再读回来也可能是：

```json
""
```

对象中的空数组字段可能无法保留。

```json
{
  "a": []
}
```

输出时 `a` 没有任何子元素可生成，因此字段可能丢失。

这属于 XML 和 Node 语义不完全一致导致的限制。

## CDATA

CDATA 会作为普通文本处理。

```xml
<msg><![CDATA[hi&hello]]></msg>
```

转换为：

```json
"hi&hello"
```

再次输出 XML 时，不保证仍然使用 CDATA。

可能输出为普通转义文本：

```xml
<root>hi&amp;hello</root>
```

也就是说，CDATA 只是输入语法形式，不会作为独立节点类型保留。

## 特殊字符

XML 中的特殊字符会由 XML writer 正确转义。

例如文本中包含：

```text
< > & " '
```

输出 XML 时会根据 XML 规则转义。

读取回来后，会还原成原始文本值。

## 使用 XmlElementConverter

如果你只需要 XML 结构，而不需要转成 Node，可以直接使用 `XmlElementConverter`。

```java
import static dev.scx.format.xml.XmlNodeConverter.DEFAULT_XML_NODE_CONVERTER;

var element = DEFAULT_XML_NODE_CONVERTER
    .xmlElementConverter()
    .formatToElement(
        """
        <user id="1">
            <name>Tom</name>
        </user>
        """,
        new XmlNodeConverterOptions()
    );
```

输出：

```java
String xml = DEFAULT_XML_NODE_CONVERTER
    .xmlElementConverter()
    .elementToFormatString(
        element,
        new XmlNodeConverterOptions()
    );
```

这种方式更接近 XML 本身。

它可以保留属性和标签结构。

## 使用 ElementNodeConverter

如果你已经有 `Element`，可以单独转换为 `Node`。

```java
var options = new XmlNodeConverterOptions();

var node = new ElementNodeConverter(options)
    .elementToNode(element);
```

也可以把 `Node` 转成 `Element`。

```java
var element = new ElementNodeConverter(options)
    .nodeToElement(node);
```

这适合需要自定义 XML 结构处理流程的场景。

## Reader / InputStream / File

从字符流读取：

```java
try (var reader = new FileReader(file, StandardCharsets.UTF_8)) {
    Node node = converter.formatToNode(
        reader,
        new XmlNodeConverterOptions()
    );
}
```

从字节流读取：

```java
try (var inputStream = new FileInputStream(file)) {
    Node node = converter.formatToNode(
        inputStream,
        StandardCharsets.UTF_8,
        new XmlNodeConverterOptions()
    );
}
```

从文件读取：

```java
Node node = converter.formatToNode(
    file,
    StandardCharsets.UTF_8,
    new XmlNodeConverterOptions()
);
```

## 输出到 Writer / OutputStream / File

输出到字符流：

```java
try (var writer = new FileWriter(file, StandardCharsets.UTF_8)) {
    converter.nodeToFormat(
        node,
        writer,
        new XmlNodeConverterOptions().rootName("data")
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
        new XmlNodeConverterOptions().rootName("data")
    );
}
```

输出到文件：

```java
converter.nodeToFormatFile(
    node,
    file,
    StandardCharsets.UTF_8,
    new XmlNodeConverterOptions().rootName("data")
);
```

## 异常处理

XML 解析或 XML 到 Node 映射失败时，会抛出 `FormatToNodeException`。

```java
try {
    Node node = converter.formatToNode(xml, options);
} catch (FormatToNodeException e) {
    // XML -> Node 失败
}
```

Node 输出为 XML 失败时，会抛出 `NodeToFormatException`。

```java
try {
    String xml = converter.nodeToFormatString(node, options);
} catch (NodeToFormatException e) {
    // Node -> XML 失败
}
```

如果使用 Reader、InputStream、Writer、OutputStream 或 File，还可能出现 `IOException`。

## 完整示例

```java
import dev.scx.format.xml.XmlNodeConverter;
import dev.scx.format.xml.XmlNodeConverterOptions;
import dev.scx.node.ArrayNode;
import dev.scx.node.ObjectNode;

import static dev.scx.node.NullNode.NULL;

public class XmlFormatDemo {

    public static void main(String[] args) {
        var converter = new XmlNodeConverter();

        var user = new ObjectNode();

        user.put("id", 12345);
        user.put("name", "小明");
        user.put("active", true);
        user.put("updated_at", NULL);

        var tags = new ArrayNode();
        tags.add("程序员");
        tags.add("摄影师");
        tags.add("旅行者");

        user.put("tag", tags);

        var options = new XmlNodeConverterOptions()
            .rootName("user")
            .itemName("item");

        String xml = converter.nodeToFormatString(user, options);

        System.out.println(xml);

        var node2 = converter.formatToNode(xml, new XmlNodeConverterOptions());

        System.out.println(node2);
    }

}
```

输出结构大致类似：

```xml
<user>
    <id>12345</id>
    <name>小明</name>
    <active>true</active>
    <updated_at/>
    <tag>程序员</tag>
    <tag>摄影师</tag>
    <tag>旅行者</tag>
</user>
```

再次读回 Node 后，非 null 标量会变成字符串：

```json
{
  "id": "12345",
  "name": "小明",
  "active": "true",
  "updated_at": null,
  "tag": ["程序员", "摄影师", "旅行者"]
}
```

## 设计说明

### 1. SCX Format XML 面向 data-centric XML

它适合处理这种 XML：

```xml
<user>
    <name>Tom</name>
    <age>18</age>
</user>
```

不适合处理这种文档型 XML：

```xml
<p>Hello <b>world</b>!</p>
```

后者属于 mixed content。

### 2. 不支持 mixed content

mixed content 无法稳定映射到 `Node`。

所以遇到 mixed content 时直接抛出异常。

这比做模糊转换更可靠。

### 3. 属性和子元素统一为字段

XML -> Node 时：

```text
属性
子元素
```

都会映射为 `ObjectNode` 字段。

这让后续对象绑定更简单。

但也意味着属性来源信息不会保留。

### 4. 重复字段自然映射为数组

XML 中同名子元素很常见。

```xml
<tag>java</tag>
<tag>scx</tag>
```

映射为：

```json
{
  "tag": ["java", "scx"]
}
```

这是 XML 到 Node 的核心规则之一。

### 5. Node -> XML 不生成属性

Node 输出为 XML 时，统一使用子元素。

这样可以避免猜测某个字段应该是属性还是子元素。

如果你需要精确控制属性，应使用 `XmlElementConverter` 和 `TagElement` 手动构造 XML 结构。

### 6. XML 往返是规范化映射

SCX Format XML 追求的是：

```text
data-centric XML 与 Node 之间的稳定规范化映射
```

而不是：

```text
原始 XML 文档完全保真
```

因此不要用它做 XML 文档编辑器或格式保持工具。

### 7. 空结构存在语义限制

XML 很难区分：

```text
空字符串
空对象
空数组
空标签
```

所以空对象、空数组等结构经过 XML 往返时可能退化。

如果你需要完全保留这些差异，XML 可能不是合适的中间格式，或者需要定义额外标记规则。

### 8. 安全默认值

读取 XML 时会关闭 DTD 和外部实体。

这适合大多数数据型 XML 场景。

如果你需要 DTD 或外部实体，应另行实现专用 XML 解析逻辑。

## 常见问题

### SCX Format XML 是 XML 文档处理库吗？

不是。

它是 XML 和 `Node` 之间的转换库，面向 data-centric XML。

### 支持 mixed content 吗？

不支持。

例如：

```xml
<p>Hello <b>world</b></p>
```

会在 XML -> Node 阶段抛出 `FormatToNodeException`。

### XML 属性会保留吗？

XML -> Element 时会保留。

XML -> Node 时，属性会变成 `ObjectNode` 字段。

Node -> XML 时，不会恢复为属性，而是统一输出为子元素。

### 为什么数字读回来变成字符串？

因为 XML 文本本身没有 JSON 那样的数字类型。

```xml
<age>18</age>
```

读回来是：

```json
"18"
```

类型恢复应交给后续对象绑定阶段处理。

### 自闭合标签表示什么？

自闭合标签表示 `null`。

```xml
<name/>
```

对应：

```json
null
```

### 空标签表示什么？

非自闭合空标签表示空字符串。

```xml
<name></name>
```

对应：

```json
""
```

### 重复标签如何处理？

合并为数组。

```xml
<tag>java</tag>
<tag>scx</tag>
```

对应：

```json
{
  "tag": ["java", "scx"]
}
```

### 属性和子元素同名怎么办？

也按重复字段处理，合并为数组。

```xml
<root x="1">
    <x>2</x>
</root>
```

对应：

```json
{
  "x": ["1", "2"]
}
```

### 顶层数组怎么输出？

使用 `rootName` 作为根标签，数组元素使用 `itemName`。

```xml
<root>
    <item>1</item>
    <item>2</item>
</root>
```

### 可以修改根标签名吗？

可以。

```java
new XmlNodeConverterOptions().rootName("user")
```

### 可以修改数组元素标签名吗？

可以。

```java
new XmlNodeConverterOptions().itemName("value")
```

### 空对象能保留吗？

不能完全保留。

空对象输出为 XML 后，再读回来可能变成空字符串。

### 空数组能保留吗？

不能完全保留。

空数组没有子元素可输出，因此经过 XML 往返可能丢失语义。

### CDATA 会保留吗？

不会保留 CDATA 形式。

CDATA 会被当作普通文本处理。

### 注释会保留吗？

不会。

XML 注释不会映射为 `Node`。

### DTD 支持吗？

默认不支持。

读取时关闭 DTD。

### 外部实体支持吗？

默认不支持。

读取时关闭外部实体。

### XML 命名空间支持吗？

当前映射主要使用 local name。

如果你的 XML 依赖命名空间前缀语义，转换到 Node 后可能无法保留完整命名空间信息。

### 如何只处理 XML 结构，不转成 Node？

使用 `XmlElementConverter`。

```java
var element = converter.xmlElementConverter()
    .formatToElement(xml, options);
```

### 如何手动构造 XML？

使用 `TagElement` 和 `TextElement`。

```java
var root = new TagElement("user", false);
root.addAttribute("id", "1");

var name = new TagElement("name", false);
name.add(new TextElement("Tom"));

root.add(name);
```

然后：

```java
String xml = converter.xmlElementConverter()
    .elementToFormatString(root, options);
```

### XML 解析失败抛什么异常？

通过 `XmlNodeConverter` 读取时，最终会包装为 `FormatToNodeException`。

### Node 输出 XML 失败抛什么异常？

抛 `NodeToFormatException`。

### Reader 会自动关闭吗？

不会。

外部传入资源由调用方关闭。

### 什么时候应该用 SCX Format XML？

适合：

```text
配置 XML
数据交换 XML
结构简单的 data-centric XML
需要转为 Node 再做对象绑定的 XML
```

不适合：

```text
HTML 类文档
富文本 XML
mixed content XML
需要保留注释/DTD/命名空间细节的 XML
需要原样往返的 XML 文档编辑
```
