# SCX IO

SCX IO 是一个面向字节流处理的轻量 IO 抽象库。

它提供了 `ByteInput`、`ByteOutput`、`ByteSupplier`、`ByteConsumer`、`ByteChunk`、`ByteIndexer` 等基础接口，用来在 Java 原生 `InputStream` / `OutputStream` 之外，表达更明确的字节读取、字节写入、零拷贝视图、边界查找、缓存回放、分段读取、gzip 包装和流式传输语义。

SCX IO 本身不是文件系统库，也不是 NIO 框架。它的重点是把“字节来源”“字节读取器”“字节输出器”“字节消费者”“字节匹配器”这些角色拆开，让上层协议解析、HTTP 处理、压缩包装、分块传输等逻辑可以复用同一套字节流抽象。

当前版本为 `0.4.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-io</artifactId>
    <version>0.4.0</version>
</dependency>
```

## 基本概念

SCX IO 中最核心的概念包括：

```text
ByteChunk       byte[] 上的一个只读区间视图
ByteSupplier    字节块生产者
ByteConsumer    字节块消费者
ByteInput       字节输入接口
ByteOutput      字节输出接口
ByteIndexer     字节匹配器
ScxIO           创建、适配、传输和 gzip 工具类
```

它们之间的关系可以简单理解为：

```text
ByteSupplier
    ↓
ByteInput
    ↓
read / peek / skip / indexOf / readUntil
    ↓
ByteConsumer 或 byte[]

ByteOutput
    ↑
write / flush / close
    ↑
ByteConsumer / ScxIO.transferTo(...)
```

更具体一点：

```text
ByteSupplier 负责从上游拉取 ByteChunk
ByteInput    负责在 ByteSupplier 之上提供读取、预读、跳过和查找能力
ByteConsumer 负责消费 ByteInput 交付的 ByteChunk
ByteOutput   负责立即写出 ByteChunk
ByteIndexer  负责在 ByteChunk 流中查找字节模式
ScxIO        负责把这些组件组合起来
```

## 快速开始

从 `byte[]` 创建 `ByteInput`：

```java
import dev.scx.io.ScxIO;

var input = ScxIO.createByteInput("hello world".getBytes());

var bytes = input.readFully(5);

System.out.println(new String(bytes));
```

输出：

```text
hello
```

读取全部内容：

```java
var input = ScxIO.createByteInput(
    "hello ".getBytes(),
    "world".getBytes()
);

var bytes = input.readAll();

System.out.println(new String(bytes));
```

输出：

```text
hello world
```

查找并读取到指定边界：

```java
var input = ScxIO.createByteInput("abc\r\ndef".getBytes());

var line = input.readUntil("\r\n".getBytes());

System.out.println(new String(line));
```

输出：

```text
abc
```

写入到内存输出：

```java
import dev.scx.io.output.ByteArrayByteOutput;

var output = new ByteArrayByteOutput();

output.write("hello".getBytes());
output.write(" world".getBytes());

System.out.println(new String(output.bytes()));
```

输出：

```text
hello world
```

从输入传输到输出：

```java
import dev.scx.io.ScxIO;
import dev.scx.io.output.ByteArrayByteOutput;

var input = ScxIO.createByteInput("hello world".getBytes());
var output = new ByteArrayByteOutput();

long n = ScxIO.transferToAll(input, output);

System.out.println(n);
System.out.println(new String(output.bytes()));
```

输出：

```text
11
hello world
```

## ByteChunk

`ByteChunk` 表示 `byte[]` 上的一个区间视图。

它的区间语义是：

```text
[start, end)
```

也就是包含 `start`，不包含 `end`。

结构可以理解为：

```java
public final class ByteChunk {

    public final byte[] bytes;

    public final int start;

    public final int end;

    public final int length;

}
```

示例：

```java
import dev.scx.io.ByteChunk;

var bytes = "hello world".getBytes();

var chunk = ByteChunk.of(bytes, 0, 5);

System.out.println(chunk.length);
System.out.println(chunk.get(0));
System.out.println(chunk.toString());
```

输出类似：

```text
5
104
hello
```

### 创建 ByteChunk

从整个数组创建：

```java
var chunk = ByteChunk.of(bytes);
```

从指定区间创建：

```java
var chunk = ByteChunk.of(bytes, 6, 11);
```

### subChunk

`subChunk(...)` 可以在当前视图上继续创建子视图。

```java
var bytes = "hello world".getBytes();

var chunk = ByteChunk.of(bytes, 0, bytes.length);

var sub = chunk.subChunk(6, 11);

System.out.println(sub.toString());
```

输出：

```text
world
```

需要注意，`subChunk(...)` 不会复制数组。

```text
原 ByteChunk 和子 ByteChunk 共享同一个 byte[]
```

### ByteChunk 不拥有数据

`ByteChunk` 本身只是视图，不拥有底层数据。

```text
ByteChunk = byte[] + start + end + length
```

因此：

1. 多个 `ByteChunk` 可能共享同一个 `byte[]`。
2. `ByteChunk` 不会拷贝数据。
3. 修改底层 `byte[]` 会影响所有引用它的 `ByteChunk`。
4. SCX IO 约定 `ByteChunk` 应被视为只读视图。
5. 使用者不应修改 `ByteChunk.bytes` 中对应区间的数据。

### 边界检查

`ByteChunk` 为了性能不做额外边界检查。

例如：

```java
var chunk = ByteChunk.of(bytes, 0, 5);
```

调用者需要自己保证：

```text
0 <= start <= end <= bytes.length
```

否则可能触发普通数组访问异常，或者产生不可预期行为。

## ByteSupplier

`ByteSupplier` 是字节块生产者。

它负责从上游来源中拉取数据，并以 `ByteChunk` 形式输出。

上游来源可以是：

```text
byte[]
InputStream
File
ByteInput
多个 ByteSupplier
带边界的 ByteInput
缓存包装器
长度限制包装器
```

接口核心方法是：

```java
public interface ByteSupplier extends AutoCloseable {

    ByteChunk get() throws ScxInputException;

    default ByteChunk borrow() throws ScxInputException {
        return get();
    }

    @Override
    default void close() throws ScxInputException {

    }

}
```

### get

`get()` 表示获取下一个数据块。

```java
ByteChunk chunk = supplier.get();
```

返回值语义：

```text
返回 ByteChunk    表示成功获取到一个数据块
返回 null         表示 EOF，也就是没有更多数据
```

`get()` 返回的 `ByteChunk` 具有 owned 语义。

这里的 owned 不是说调用者可以修改底层数组，而是说：

```text
返回的数据视图在后续 get / borrow 调用之后仍然有效
```

适合用于：

1. 缓存。
2. 回放。
3. 异步处理。
4. 跨线程保存。
5. 放入集合。
6. 需要在后续继续引用数据块的场景。

### borrow

`borrow()` 表示借用下一个数据块。

```java
ByteChunk chunk = supplier.borrow();
```

`borrow()` 返回的 `ByteChunk` 具有 borrowed 语义。

也就是说：

```text
返回的数据只保证在下一次 get / borrow 之前有效
```

实现可以复用内部缓冲区来减少分配。

适合用于：

1. 立即消费。
2. 立即写出。
3. 流水线处理。
4. 不保存返回数据的场景。

默认实现是：

```java
default ByteChunk borrow() throws ScxInputException {
    return get();
}
```

也就是说，如果某个实现没有特殊优化，`borrow()` 会退化为 `get()`。

### get 和 borrow 的区别

可以简单理解为：

```text
get       稳定结果，适合保存
borrow    临时结果，适合立即使用
```

例如：

```java
var chunk = supplier.get();

// 可以保存 chunk
cache.add(chunk);
```

而：

```java
var chunk = supplier.borrow();

// 推荐立即使用
output.write(chunk);
```

不应该把 `borrow()` 返回的数据长期保存。

### ByteSupplier 的 EOF

`ByteSupplier` 使用 `null` 表示 EOF。

```java
while (true) {
    var chunk = supplier.get();
    if (chunk == null) {
        break;
    }
    // 使用 chunk
}
```

注意：

```text
ByteSupplier 返回 null 表示结束
ByteInput 读不到数据时通常抛 NoMoreDataException
```

这是两个层级的语义差异。

## ByteInput

`ByteInput` 是面向调用者的字节输入接口。

它在 `ByteSupplier` 之上提供更高层的读取能力。

核心能力包括：

```text
read          读取并移动指针
peek          查看但不移动指针
skip          跳过并移动指针
indexOf       查找字节模式
readUntil     读取到指定模式之前
peekUntil     查看指定模式之前的数据
mark          标记当前位置
close         关闭输入
```

## 创建 ByteInput

最常见的创建方式是通过 `ScxIO`。

### 从 byte[] 创建

```java
import dev.scx.io.ScxIO;

var input = ScxIO.createByteInput("hello".getBytes());
```

也可以传入多个数组：

```java
var input = ScxIO.createByteInput(
    "hello ".getBytes(),
    "world".getBytes()
);
```

多个数组会按顺序读取。

### 从 InputStream 创建

```java
import dev.scx.io.ScxIO;

var input = ScxIO.createByteInput(inputStream);
```

内部会使用 `InputStreamByteSupplier`。

### 从 File 创建

```java
import dev.scx.io.ScxIO;

var input = ScxIO.createByteInput(file);
```

也可以只读取文件的一段：

```java
var input = ScxIO.createByteInput(file, 100, 1024);
```

这里表示从文件偏移 `100` 开始，最多读取 `1024` 字节。

### 从 ByteSupplier 创建

```java
var input = ScxIO.createByteInput(byteSupplier);
```

这种方式适合自定义数据来源。

## read

`read()` 读取单个字节。

```java
byte b = input.read();
```

如果当前没有更多数据，会抛出：

```text
NoMoreDataException
```

如果输入已经关闭，会抛出：

```text
InputAlreadyClosedException
```

### read(maxLength)

`read(maxLength)` 最多读取 `maxLength` 个字节，并返回 `byte[]`。

```java
var bytes = input.read(1024);
```

语义是：

```text
最多读取 maxLength 个字节
可能少于 maxLength
如果 maxLength > 0，则至少读取 1 个字节
如果当前立即 EOF，则抛 NoMoreDataException
```

也就是说，`read(1024)` 并不保证一定返回 1024 个字节。

它适合“读一块数据”的场景。

### readUpTo(length)

`readUpTo(length)` 会尽量读取 `length` 个字节。

```java
var bytes = input.readUpTo(1024);
```

语义是：

```text
尽量读取 length 个字节
如果中途 EOF，可以少于 length
如果 length > 0 且当前立即 EOF，则抛 NoMoreDataException
```

它适合“我希望最多拿到这么多，如果流提前结束也可以接受”的场景。

### readFully(length)

`readFully(length)` 会严格读取 `length` 个字节。

```java
var bytes = input.readFully(1024);
```

语义是：

```text
必须读取 length 个字节
如果中途 EOF，抛 NoMoreDataException
```

它适合协议解析中的固定长度字段。

例如：

```java
var header = input.readFully(8);
```

### 三种读取方式对比

```text
read(maxLength)       宽松读取，可能只读当前已有的一块
readUpTo(length)      尽量读取，只有立即 EOF 才抛异常
readFully(length)     严格读取，不够 length 就抛异常
```

更具体地说：

```text
read          宽松
readUpTo      中等
readFully     严格
```

## 0 长度读取

SCX IO 对 `length = 0` 有明确约定。

```java
input.read(0);
input.readUpTo(0);
input.readFully(0);
```

这些操作都被视为无动作：

```text
立即返回
不读取数据
不移动指针
即使已经 EOF，也不抛 NoMoreDataException
```

这样可以消除“0 字节读取”带来的歧义。

## readAll

`readAll()` 读取剩余全部数据。

```java
var bytes = input.readAll();
```

和 `readFully(...)` 不同，`readAll()` 是请求型方法。

它不会把 EOF 当作错误。

例如，如果流已经结束：

```java
var bytes = input.readAll();
```

返回：

```text
空 byte[]
```

而不是抛出 `NoMoreDataException`。

适合“我只关心剩余内容”的场景。

## ByteConsumer 版本读取

除了直接返回 `byte[]`，`ByteInput` 还支持把数据交给 `ByteConsumer`。

```java
input.read(byteConsumer, 1024);
```

这种方式适合：

1. 低分配处理。
2. 流式转发。
3. 协议解析。
4. 边读边匹配。
5. 提前中断读取。
6. 避免一次性合并大数组。

示例：

```java
import dev.scx.io.consumer.ByteConsumer;

ByteConsumer consumer = chunk -> {
    System.out.println(chunk.length);
    return true;
};

input.readUpTo(consumer, 1024);
```

`ByteConsumer#accept(...)` 返回值表示是否需要更多数据：

```text
true     继续读取
false    停止读取
```

## peek

`peek` 和 `read` 类似，但不会移动读取指针。

```java
var bytes = input.peekFully(5);
```

之后再次读取，仍然可以读到同样的数据：

```java
var a = input.peekFully(5);
var b = input.readFully(5);
```

`a` 和 `b` 内容相同。

常见用法：

```java
var header = input.peekFully(4);

if (isExpected(header)) {
    input.skipFully(4);
}
```

## skip

`skip` 用于跳过数据。

```java
long n = input.skip(1024);
```

和 `read` 类似，`skip` 也分三种：

```text
skip(maxLength)       最多跳过 maxLength
skipUpTo(length)      尽量跳过 length
skipFully(length)     必须跳过 length
```

示例：

```java
input.skipFully(4);

var body = input.readAll();
```

`skipAll()` 会跳过剩余全部数据：

```java
long skipped = input.skipAll();
```

如果已经 EOF，`skipAll()` 返回 `0`。

## indexOf

`indexOf(...)` 用于在当前读取位置之后查找字节。

查找单字节：

```java
var result = input.indexOf((byte) '\n');

System.out.println(result.index);
System.out.println(result.matchedLength);
```

查找字节数组：

```java
var result = input.indexOf("\r\n".getBytes());
```

返回值是 `ByteMatchResult`：

```java
public final class ByteMatchResult {

    public final long index;

    public final int matchedLength;

}
```

其中：

```text
index          匹配位置，相对于当前读取位置
matchedLength  匹配到的模式长度
```

### indexOf 不移动读取指针

`indexOf(...)` 只是查找。

它不会消费数据。

例如：

```java
var input = ScxIO.createByteInput("abc\r\ndef".getBytes());

var result = input.indexOf("\r\n".getBytes());

System.out.println(result.index);

var bytes = input.readFully(3);

System.out.println(new String(bytes));
```

输出：

```text
3
abc
```

### 限制查找范围

可以指定最大查找长度：

```java
var result = input.indexOf("\r\n".getBytes(), 8192);
```

如果在 `maxLength` 范围内找不到，会抛出：

```text
NoMatchFoundException
```

这对协议解析很重要，可以避免无限读取。

例如读取一行时限制最大长度：

```java
var line = input.readUntil("\r\n".getBytes(), 8192);
```

如果超过 8192 字节还没有找到换行，就会抛出 `NoMatchFoundException`。

### 空匹配模式

空模式被视为无动作匹配。

```java
var result = input.indexOf(new byte[]{});
```

结果是：

```text
index = 0
matchedLength = 0
```

## readUntil

`readUntil(...)` 用于读取到指定模式之前。

```java
var input = ScxIO.createByteInput("abc\r\ndef".getBytes());

var line = input.readUntil("\r\n".getBytes());

System.out.println(new String(line));
```

输出：

```text
abc
```

需要注意：

```text
返回结果不包含匹配模式
调用结束后读取指针会跳过匹配模式
```

也就是说，上面读取后，下一次读取会从 `def` 开始。

```java
var rest = input.readAll();

System.out.println(new String(rest));
```

输出：

```text
def
```

### readUntil 和 peekUntil

`peekUntil(...)` 只查看，不移动读取指针。

```java
var line = input.peekUntil("\r\n".getBytes());
```

而 `readUntil(...)` 会消费数据，并跳过匹配模式。

```java
var line = input.readUntil("\r\n".getBytes());
```

区别：

```text
peekUntil    不移动指针，不跳过模式
readUntil    移动指针，并跳过模式
```

## mark

`mark()` 用于标记当前读取位置。

```java
var mark = input.mark();

var bytes = input.readFully(5);

// 回到 mark 位置
mark.reset();
```

示例：

```java
var input = ScxIO.createByteInput("hello world".getBytes());

var mark = input.mark();

var a = input.readFully(5);

mark.reset();

var b = input.readFully(5);

System.out.println(new String(a));
System.out.println(new String(b));
```

输出：

```text
hello
hello
```

需要注意：

1. `mark()` 记录的是当前读取位置。
2. `reset()` 会恢复读取位置。
3. `reset()` 还会重置后续缓存节点的读取位置。
4. 如果 `ByteInput` 已经关闭，`reset()` 会抛出 `InputAlreadyClosedException`。
5. `mark` 依赖 `DefaultByteInput` 内部缓存已拉取的数据，因此适合局部回退，不应滥用为无限制缓存策略。

## ByteOutput

`ByteOutput` 是字节输出接口。

它表示立即写出 / 提交型输出。

核心方法是：

```java
public interface ByteOutput extends AutoCloseable {

    void write(byte b) throws ScxOutputException, OutputAlreadyClosedException;

    void write(ByteChunk b) throws ScxOutputException, OutputAlreadyClosedException;

    void flush() throws ScxOutputException, OutputAlreadyClosedException;

    boolean isClosed();

    @Override
    void close() throws ScxOutputException;

    default void write(byte[] b) throws ScxOutputException, OutputAlreadyClosedException {
        write(ByteChunk.of(b));
    }

}
```

### 立即写出语义

`ByteOutput#write(...)` 接收到的数据只保证在方法调用期间有效。

也就是说，`ByteOutput` 实现不应该在 `write(...)` 返回后继续引用或保存传入的 `ByteChunk`。

如果某个实现需要缓存、聚合、延迟或异步写出，必须在 `write(...)` 内部自行拷贝数据。

这和 `ByteConsumer` 的 retaining 语义不同。

```text
ByteConsumer 可以保存 ByteChunk
ByteOutput 不应该保存 ByteChunk
```

### 写入 byte[]

```java
output.write("hello".getBytes());
```

### 写入 ByteChunk

```java
var bytes = "hello world".getBytes();

output.write(ByteChunk.of(bytes, 0, 5));
```

### flush

```java
output.flush();
```

不同实现的 `flush()` 语义不同。

例如：

1. `OutputStreamByteOutput` 会调用底层 `OutputStream.flush()`。
2. `ByteArrayByteOutput` 中 `flush()` 基本是 no-op，但会检查是否关闭。
3. `NoCloseByteOutput#close()` 会把底层 close 改成 flush。

### close

`close()` 用于关闭输出。

约定：

```text
close 是幂等的
关闭成功后 isClosed 应返回 true
如果关闭时发生异常，不应改变 closed 状态
```

如果关闭后继续写入，会抛出：

```text
OutputAlreadyClosedException
```

## ScxIO

`ScxIO` 是工具入口类。

它提供几类能力：

```text
创建 ByteInput
创建 ByteOutput
ByteInput / InputStream 互转
ByteOutput / OutputStream 互转
gzip 包装和 byte[] gzip
transferTo 数据传输
CacheByteSupplier 创建
NoClose / DrainOnClose 包装
```

## 创建输入和输出

### createByteInput(ByteSupplier)

```java
var input = ScxIO.createByteInput(byteSupplier);
```

### createByteInput(byte[]...)

```java
var input = ScxIO.createByteInput(
    "hello ".getBytes(),
    "world".getBytes()
);
```

### createByteInput(InputStream)

```java
var input = ScxIO.createByteInput(inputStream);
```

### createByteInput(File)

```java
var input = ScxIO.createByteInput(file);
```

### createByteInput(File, offset, length)

```java
var input = ScxIO.createByteInput(file, 100, 1024);
```

### createByteOutput(OutputStream)

```java
var output = ScxIO.createByteOutput(outputStream);
```

如果需要内存输出，可以直接使用：

```java
import dev.scx.io.output.ByteArrayByteOutput;

var output = new ByteArrayByteOutput();
```

## InputStream 适配

SCX IO 可以和 JDK `InputStream` 互相适配。

### ByteInput 转 InputStream

```java
InputStream inputStream = ScxIO.byteInputToInputStream(byteInput);
```

内部使用 `ByteInputInputStream`。

### InputStream 转 ByteInput

```java
ByteInput byteInput = ScxIO.inputStreamToByteInput(inputStream);
```

如果传入的 `InputStream` 本身是 `ByteInputAdapter`，会直接取回原始 `ByteInput`。

否则会创建新的 `DefaultByteInput`。

### ByteInputInputStream

`ByteInputInputStream` 会把 `ByteInput` 映射成标准 `InputStream` 行为。

例如：

```java
InputStream in = ScxIO.byteInputToInputStream(byteInput);

int b = in.read();
```

当 `ByteInput` 遇到 `NoMoreDataException` 时：

```text
InputStream#read() 返回 -1
InputStream#read(byte[], off, len) 返回 -1
InputStream#readNBytes(...) 返回空数组或 0
InputStream#skip(...) 返回 0
InputStream#skipNBytes(...) 抛 EOFException
```

这符合 `InputStream` 的基本习惯。

## OutputStream 适配

### ByteOutput 转 OutputStream

```java
OutputStream outputStream = ScxIO.byteOutputToOutputStream(byteOutput);
```

如果传入的是 `OutputStreamByteOutput`，会直接取回内部 `OutputStream`。

否则会创建 `ByteOutputOutputStream`。

### OutputStream 转 ByteOutput

```java
ByteOutput byteOutput = ScxIO.outputStreamToByteOutput(outputStream);
```

如果传入的 `OutputStream` 本身是 `ByteOutputAdapter`，会直接取回原始 `ByteOutput`。

否则会创建 `OutputStreamByteOutput`。

## transferTo

`ScxIO.transferTo...` 用于把输入数据传输到输出。

### ByteInput 到 ByteOutput

```java
long n = ScxIO.transferToAll(byteInput, byteOutput);
```

读取全部并写出。

也可以限制长度：

```java
long n = ScxIO.transferTo(byteInput, byteOutput, 1024);
```

严格传输指定长度：

```java
long n = ScxIO.transferToFully(byteInput, byteOutput, 1024);
```

尽量传输指定长度：

```java
long n = ScxIO.transferToUpTo(byteInput, byteOutput, 1024);
```

它们和 `ByteInput` 的读取语义对应：

```text
transferTo          对应 read
transferToUpTo      对应 readUpTo
transferToFully     对应 readFully
transferToAll       对应 readAll
```

### ByteInput 到 OutputStream

```java
long n = ScxIO.transferToAll(byteInput, outputStream);
```

也支持：

```java
ScxIO.transferTo(byteInput, outputStream, maxLength);

ScxIO.transferToUpTo(byteInput, outputStream, length);

ScxIO.transferToFully(byteInput, outputStream, length);
```

### ByteSupplier 到 ByteOutput

```java
long n = ScxIO.transferToAll(byteSupplier, byteOutput);
```

这里会直接调用 `byteSupplier.borrow()`，拿到块后写入输出。

如果 `ByteSupplier` 返回空块，会跳过并继续拉取。

### ByteSupplier 到 OutputStream

```java
long n = ScxIO.transferToAll(byteSupplier, outputStream);
```

这适合将底层 `ByteSupplier` 直接转发到 JDK 输出流。

## gzip

SCX IO 提供两类 gzip 能力。

### 包装 ByteInput

```java
var gzipInput = ScxIO.gzipByteInput(byteInput);
```

它会把一个 gzip 压缩数据输入包装成解压后的 `ByteInput`。

示例：

```java
var raw = "hello gzip".getBytes();

var gzipData = ScxIO.gzip(raw);

var gzipInput = ScxIO.createByteInput(gzipData);

var input = ScxIO.gzipByteInput(gzipInput);

var bytes = input.readAll();

System.out.println(new String(bytes));
```

输出：

```text
hello gzip
```

### 包装 ByteOutput

```java
var gzipOutput = ScxIO.gzipByteOutput(byteOutput);
```

写入 `gzipOutput` 的数据会被 gzip 压缩后写到底层 `byteOutput`。

示例：

```java
import dev.scx.io.output.ByteArrayByteOutput;

var rawOutput = new ByteArrayByteOutput();

try (var gzipOutput = ScxIO.gzipByteOutput(rawOutput)) {
    gzipOutput.write("hello gzip".getBytes());
}

var compressed = rawOutput.bytes();
```

### 压缩整个 byte[]

```java
byte[] compressed = ScxIO.gzip(data);
```

### 解压整个 byte[]

```java
byte[] raw = ScxIO.ungzip(compressed);
```

需要注意：

```text
gzip(byte[]) 和 ungzip(byte[]) 适合小数据
大数据推荐使用 gzipByteInput / gzipByteOutput 或 JDK GZIPInputStream / GZIPOutputStream
```

## ByteArrayByteOutput

`ByteArrayByteOutput` 是内存输出实现。

它会把写入的数据复制到内部连续 `byte[]` 中。

示例：

```java
import dev.scx.io.output.ByteArrayByteOutput;

var output = new ByteArrayByteOutput();

output.write("hello".getBytes());
output.write(" ".getBytes());
output.write("world".getBytes());

byte[] bytes = output.bytes();

System.out.println(new String(bytes));
```

输出：

```text
hello world
```

### bytes

```java
byte[] bytes = output.bytes();
```

返回当前已经写入的数据副本。

### size

```java
int size = output.size();
```

返回当前已写入字节数。

### close

关闭后继续写入会抛出：

```text
OutputAlreadyClosedException
```

```java
output.close();

output.write("x".getBytes());
```

## OutputStreamByteOutput

`OutputStreamByteOutput` 把 JDK `OutputStream` 包装为 `ByteOutput`。

```java
var output = new OutputStreamByteOutput(outputStream);
```

写入时会调用：

```java
outputStream.write(...)
```

刷新时会调用：

```java
outputStream.flush()
```

关闭时会调用：

```java
outputStream.close()
```

如果底层抛出 `IOException`，会包装为：

```text
ScxOutputException
```

## LengthBoundedByteOutput

`LengthBoundedByteOutput` 是带长度约束的输出包装器。

```java
var output = new LengthBoundedByteOutput(
    rawOutput,
    minLength,
    maxLength
);
```

它会检查：

```text
写入不能超过 maxLength
close 时写入总长度不能少于 minLength
```

示例：

```java
import dev.scx.io.output.ByteArrayByteOutput;
import dev.scx.io.output.LengthBoundedByteOutput;

var raw = new ByteArrayByteOutput();

var output = new LengthBoundedByteOutput(raw, 3, 5);

output.write("abc".getBytes());

output.close();
```

如果写入超过最大长度：

```java
var output = new LengthBoundedByteOutput(raw, 0, 3);

output.write("abcd".getBytes());
```

会抛出：

```text
ScxOutputException
```

如果关闭时不足最小长度：

```java
var output = new LengthBoundedByteOutput(raw, 3, 5);

output.write("ab".getBytes());

output.close();
```

也会抛出：

```text
ScxOutputException
```

需要注意：

```text
LengthBoundedByteOutput#isClosed() 直接代理底层 ByteOutput
内部 closed 字段只用于 close() 幂等控制
```

## NoCloseByteOutput

`NoCloseByteOutput` 用于隔离底层 close。

```java
var output = new NoCloseByteOutput(rawOutput);
```

它的语义是：

```text
write    转发到底层 ByteOutput
flush    转发到底层 ByteOutput
close    不关闭底层，而是 flush 底层
```

需要注意：

```text
NoCloseByteOutput 自己仍然有 closed 状态
```

也就是说：

```java
var output = new NoCloseByteOutput(rawOutput);

output.close();

output.write("x".getBytes());
```

会抛出：

```text
OutputAlreadyClosedException
```

但底层 `rawOutput` 没有被关闭。

## NullByteOutput

`NullByteOutput` 是空输出。

它会丢弃所有写入的数据。

```java
var output = new NullByteOutput();

output.write("hello".getBytes());

output.flush();

output.close();
```

关闭后继续写入会抛出：

```text
OutputAlreadyClosedException
```

适合：

1. 测试。
2. 统计读取长度。
3. 丢弃输出。
4. 需要一个 no-op 输出端的场景。

## ByteArrayByteSupplier

`ByteArrayByteSupplier` 从一个或多个 `byte[]` 中提供数据。

```java
var supplier = new ByteArrayByteSupplier(
    "hello ".getBytes(),
    "world".getBytes()
);
```

也可以通过 `ScxIO` 间接创建：

```java
var input = ScxIO.createByteInput(
    "hello ".getBytes(),
    "world".getBytes()
);
```

需要注意：

```text
ByteArrayByteSupplier 不会拷贝传入的 byte[]
```

因此调用者需要保证这些数组在读取期间不被外部修改。

## InputStreamByteSupplier

`InputStreamByteSupplier` 从 `InputStream` 中读取数据。

```java
var supplier = new InputStreamByteSupplier(inputStream);
```

默认缓冲区大小是：

```text
8192
```

也可以指定：

```java
var supplier = new InputStreamByteSupplier(inputStream, 4096);
```

如果 `bufferLength <= 0`，会抛出：

```text
IllegalArgumentException
```

### get 和 borrow 的内存行为

`get()` 每次读取时会创建新的 `byte[]`。

这保证返回的数据在后续调用之后仍然有效。

`borrow()` 会复用内部缓冲区。

这可以减少分配，但返回的 `ByteChunk` 只适合立即消费。

## BufferedInputStreamByteSupplier

`BufferedInputStreamByteSupplier` 也从 `InputStream` 中读取数据，但内存策略不同。

```java
var supplier = new BufferedInputStreamByteSupplier(inputStream);
```

区别可以简单理解为：

```text
InputStreamByteSupplier
    get 每次创建 bufferLength 大小的数组

BufferedInputStreamByteSupplier
    内部固定一个 buffer
    get 时只复制实际读取长度到新数组
```

适合场景：

```text
读取长度经常等于 bufferLength       InputStreamByteSupplier 可能更合适
读取长度经常小于 bufferLength       BufferedInputStreamByteSupplier 可能更合适
```

`borrow()` 都会返回复用缓冲区上的视图。

## FileByteSupplier

`FileByteSupplier` 从文件中读取数据。

```java
var supplier = new FileByteSupplier(file);
```

也可以指定缓冲区长度：

```java
var supplier = new FileByteSupplier(file, 4096);
```

也可以只读取文件的一段：

```java
var supplier = new FileByteSupplier(file, offset, length);
```

或者：

```java
var supplier = new FileByteSupplier(file, offset, length, bufferLength);
```

边界规则：

```text
offset >= 0
length >= 0
offset + length <= file.length()
bufferLength > 0
```

不满足时会抛出：

```text
IllegalArgumentException
```

内部使用：

```java
RandomAccessFile
```

读取结束后返回 `null`。

关闭时会关闭底层 `RandomAccessFile`。

## SequenceByteSupplier

`SequenceByteSupplier` 用于把多个 `ByteSupplier` 串联起来。

```java
var supplier = new SequenceByteSupplier(
    new ByteArrayByteSupplier("hello ".getBytes()),
    new ByteArrayByteSupplier("world".getBytes())
);

var input = ScxIO.createByteInput(supplier);

System.out.println(new String(input.readAll()));
```

输出：

```text
hello world
```

当当前 `ByteSupplier` 返回 `null` 时：

```text
SequenceByteSupplier 会关闭当前 supplier
然后切换到下一个 supplier
```

`close()` 会尽可能关闭当前和剩余的所有 supplier。

如果多个 close 都失败，第一个异常会被抛出，其它异常会作为 suppressed exception 附加上去。

## ByteInputByteSupplier

`ByteInputByteSupplier` 把 `ByteInput` 转成 `ByteSupplier`。

```java
var supplier = new ByteInputByteSupplier(byteInput);
```

它会从 `byteInput` 中读取数据块。

如果底层 `ByteInput` 没有更多数据，会返回 `null`。

如果底层 `ByteInput` 已经关闭，会包装成：

```text
ScxInputException
```

这个类适合在需要 `ByteSupplier` 的地方复用已有 `ByteInput`。

## LimitLengthByteSupplier

`LimitLengthByteSupplier` 是带最大长度限制的 `ByteSupplier`。

```java
var supplier = new LimitLengthByteSupplier(byteInput, 1024);
```

它最多从底层 `ByteInput` 中读取 `1024` 字节。

达到限制后返回：

```text
null
```

它本质上是一个带剩余长度计数的 `ByteInputByteSupplier`。

适合：

1. 限制 body 长度。
2. 从一个大输入中截取一段作为子输入。
3. 避免某个读取过程越过协议边界。

## CacheByteSupplier

`CacheByteSupplier` 可以缓存上游 `ByteSupplier` 返回的所有块。

```java
var cacheSupplier = ScxIO.cacheByteSupplier(byteSupplier);
```

读取一次：

```java
var input1 = ScxIO.createByteInput(cacheSupplier);

var bytes1 = input1.readAll();
```

重置后再次读取：

```java
cacheSupplier.reset();

var input2 = ScxIO.createByteInput(cacheSupplier);

var bytes2 = input2.readAll();
```

语义是：

```text
第一次读取时从上游拉取并保存
reset 后可以从缓存开头回放
```

需要注意：

1. 空块也会被缓存。
2. EOF 后会记录完成状态。
3. `reset()` 只重置缓存读取位置。
4. `close()` 会关闭上游 supplier。
5. 缓存会保留已经读取过的 ByteChunk 引用，因此要注意内存占用。

## BoundaryByteSupplier

`BoundaryByteSupplier` 用于从 `ByteInput` 中读取到某个边界之前。

它适合处理分隔符场景，例如：

```text
multipart boundary
协议帧边界
自定义分隔符
读取到某个 marker 之前
```

示例：

```java
import dev.scx.io.ScxIO;
import dev.scx.io.indexer.KMPByteIndexer;
import dev.scx.io.supplier.BoundaryByteSupplier;

var rawInput = ScxIO.createByteInput("abc--boundary--def".getBytes());

var supplier = new BoundaryByteSupplier(
    rawInput,
    new KMPByteIndexer("--boundary--".getBytes()),
    true
);

var input = ScxIO.createByteInput(supplier);

var bytes = input.readAll();

System.out.println(new String(bytes));
```

输出：

```text
abc
```

### keepBoundaryInSource

构造参数中的第三个参数是：

```java
boolean keepBoundaryInSource
```

如果为 `true`：

```text
BoundaryByteSupplier 读取结束后，底层 ByteInput 会保留 boundary
```

例如：

```text
source: abc--boundary--def
result: abc
source remaining: --boundary--def
```

如果为 `false`：

```text
BoundaryByteSupplier 读取结束后，底层 ByteInput 会跳过 boundary
```

例如：

```text
source: abc--boundary--def
result: abc
source remaining: def
```

这两个模式分别适合：

```text
keepBoundaryInSource = true     上层还需要自己处理 boundary
keepBoundaryInSource = false    当前层直接消费 boundary
```

### 跨 chunk 匹配

`BoundaryByteSupplier` 支持边界跨多个 chunk。

例如边界是：

```text
hello
```

但底层分块可能是：

```text
he
llo
```

只要 `ByteIndexer` 支持跨 chunk 状态，边界仍然可以被正确匹配。

测试中也覆盖了边界在开头、结尾、跨 chunk、重复边界、重叠边界等情况。

## DrainOnCloseByteSupplier

`DrainOnCloseByteSupplier` 会在关闭前排空上游。

```java
var supplier = ScxIO.drainOnClose(rawSupplier);
```

关闭时：

```text
先持续 borrow 直到 EOF
再关闭底层 ByteSupplier
```

适合协议场景中“必须读完当前输入段才能复用连接”的情况。

## NoCloseByteSupplier

`NoCloseByteSupplier` 用于隔离底层 close。

```java
var supplier = ScxIO.noClose(rawSupplier);
```

它会转发：

```text
get
borrow
```

但 `close()` 什么都不做。

适合：

1. 临时包装。
2. 不希望关闭底层资源。
3. 多层读取共享同一个上游资源。

## drainOnCloseNoClose

`ScxIO.drainOnCloseNoClose(...)` 同时具备：

```text
close 时排空
但不关闭底层
```

内部等价于：

```java
new DrainOnCloseByteSupplier(new NoCloseByteSupplier(byteSupplier))
```

适合：

```text
关闭当前视图时要消费掉剩余数据
但底层资源还要继续给其它组件使用
```

## NullByteSupplier

`NullByteSupplier` 是空数据源。

```java
var supplier = NullByteSupplier.NULL_BYTE_SUPPLIER;
```

调用：

```java
supplier.get();
```

永远返回：

```text
null
```

也就是立即 EOF。

## ByteConsumer

`ByteConsumer` 是一个支持中断的 `ByteChunk` 消费者。

接口定义可以理解为：

```java
public interface ByteConsumer {

    boolean accept(ByteChunk chunk) throws Throwable;

}
```

返回值语义：

```text
true     还需要更多数据
false    已经满足需求，读取过程可以提前停止
```

示例：

```java
ByteConsumer consumer = chunk -> {
    System.out.println(chunk.length);
    return true;
};

input.readUpTo(consumer, 1024);
```

### retaining 语义

`ByteConsumer` 允许在 `accept(...)` 返回后继续使用或保存传入的 `ByteChunk`。

因此，调用 `ByteConsumer` 的一方必须保证传入的 `ByteChunk` 稳定。

如果无法保证，就必须先拷贝。

这也是为什么 `ByteInput` 在给 consumer 交付数据时，会基于内部缓存节点提供稳定的 `ByteChunk`。

### consumer 异常

如果 `ByteConsumer#accept(...)` 抛出异常，`ByteInput` 会把它包装成：

```text
ScxWrappedException
```

再抛出。

`ScxIO.transferTo(...)` 这类工具方法会根据内部原因再拆回原始异常类型。

## LazyByteArrayByteConsumer

`LazyByteArrayByteConsumer` 用于把接收到的多个 `ByteChunk` 合并成一个 `byte[]`。

它的策略是：

```text
accept 时只保存 ByteChunk 引用
bytes() 时一次性分配并复制
```

示例：

```java
var consumer = new LazyByteArrayByteConsumer();

input.readAll(consumer);

byte[] bytes = consumer.bytes();
```

适合：

```text
accept 次数较少
单个 chunk 较大
最终只调用一次 bytes()
```

不太适合：

```text
chunk 很小
accept 次数很高
频繁调用 bytes()
```

## EagerByteArrayByteConsumer

`EagerByteArrayByteConsumer` 也是把多个 `ByteChunk` 合并成一个 `byte[]`，但策略不同。

它的策略是：

```text
accept 时立即复制到内部连续 byte[]
bytes() 时返回副本
```

示例：

```java
var consumer = new EagerByteArrayByteConsumer();

input.readAll(consumer);

byte[] bytes = consumer.bytes();
```

适合：

```text
小块数据
高频 accept
需要频繁获取最终 bytes()
```

代价是：

```text
随着数据增长，可能发生扩容和数组复制
```

## FillByteArrayByteConsumer

`FillByteArrayByteConsumer` 用于严格填充已有数组。

```java
var target = new byte[1024];

var consumer = new FillByteArrayByteConsumer(target);

input.readFully(consumer, target.length);
```

如果写入超过目标容量，会抛出：

```text
IndexOutOfBoundsException
```

这个类常用于 `InputStream` 适配层。

## ByteOutputByteConsumer

`ByteOutputByteConsumer` 把 `ByteConsumer` 接收到的数据写入 `ByteOutput`。

```java
var consumer = new ByteOutputByteConsumer(byteOutput);

input.readAll(consumer);

System.out.println(consumer.bytesWritten());
```

它适合实现 `ByteInput -> ByteOutput` 传输。

## OutputStreamByteConsumer

`OutputStreamByteConsumer` 把 `ByteConsumer` 接收到的数据写入 `OutputStream`。

```java
var consumer = new OutputStreamByteConsumer(outputStream);

input.readAll(consumer);

System.out.println(consumer.bytesWritten());
```

它适合实现 `ByteInput -> OutputStream` 传输。

## SkipByteConsumer

`SkipByteConsumer` 只统计接收到的字节数，不保存内容。

```java
var consumer = new SkipByteConsumer();

input.readUpTo(consumer, 1024);

System.out.println(consumer.bytesSkipped());
```

`ByteInput#skip...` 系列方法就是基于它实现的。

## ByteChunkByteConsumer

`ByteChunkByteConsumer` 只接收一次 `ByteChunk`。

```java
var consumer = new ByteChunkByteConsumer();

input.read(consumer, Long.MAX_VALUE);

var chunk = consumer.byteChunk();
```

它的 `accept(...)` 会返回 `false`，表示不需要更多数据。

适合从 `ByteInput` 中拿出当前一块数据。

## ByteIndexer

`ByteIndexer` 是字节匹配器。

它支持跨 chunk 匹配，并且是有状态的。

接口核心方法是：

```java
public interface ByteIndexer {

    StatusByteMatchResult indexOf(ByteChunk chunk);

    boolean isEmptyPattern();

    void reset();

}
```

`ByteIndexer` 通常不会直接暴露给最终业务代码，而是被 `ByteInput#indexOf(...)`、`readUntil(...)`、`BoundaryByteSupplier` 等使用。

## StatusByteMatchResult

`ByteIndexer#indexOf(...)` 返回 `StatusByteMatchResult`。

状态有三种：

```text
NO_MATCH        完全未匹配
PARTIAL_MATCH   部分匹配，可能跨 chunk 完成
FULL_MATCH      完全匹配
```

当状态是 `FULL_MATCH` 时，会包含：

```text
index           相对于当前 chunk 的匹配位置
matchedLength   实际匹配长度
```

因为支持跨 chunk 匹配，所以 `index` 可能是负数。

这表示匹配开始位置在之前的 chunk 中。

## SingleByteIndexer

`SingleByteIndexer` 用于查找单个字节。

```java
var indexer = new SingleByteIndexer((byte) '\n');
```

也可以直接使用 `ByteInput` 的便捷方法：

```java
var result = input.indexOf((byte) '\n');
```

## KMPByteIndexer

`KMPByteIndexer` 用 KMP 算法查找字节数组模式。

```java
var indexer = new KMPByteIndexer("\r\n".getBytes());
```

也可以直接使用：

```java
var result = input.indexOf("\r\n".getBytes());
```

`KMPByteIndexer` 支持跨 chunk 匹配。

例如底层分块是：

```text
abc\r
\ndef
```

仍然可以匹配到：

```text
\r\n
```

## BitMaskByteIndexer

`BitMaskByteIndexer` 使用 Shift-And / bitmask 思路查找短模式。

```java
var indexer = new BitMaskByteIndexer("hello".getBytes());
```

限制是：

```text
pattern.length <= 64
```

如果模式长度大于 64，会抛出：

```text
IllegalArgumentException
```

适合短分隔符、短 boundary 等场景。

## LineBreakByteIndexer

`LineBreakByteIndexer` 用于匹配换行。

它同时支持：

```text
\n
\r\n
```

示例：

```java
import dev.scx.io.indexer.LineBreakByteIndexer;

var line = input.readUntil(new LineBreakByteIndexer());
```

匹配结果不包含换行符。

## 异常

SCX IO 中常见异常包括：

```text
ScxInputException             输入端 IO 异常
ScxOutputException            输出端 IO 异常
InputAlreadyClosedException   输入已关闭
OutputAlreadyClosedException  输出已关闭
NoMoreDataException           没有更多数据
NoMatchFoundException         没有找到匹配模式
```

### ScxInputException

`ScxInputException` 表示宏观输入异常。

例如：

1. 底层 `InputStream` 抛出 `IOException`。
2. 文件读取失败。
3. 解压失败。
4. 解码或上游转换失败。
5. 某个包装器发现底层输入状态异常。

构造方法包括：

```java
public ScxInputException(String message)

public ScxInputException(Throwable cause)

public ScxInputException(String message, Throwable cause)
```

### ScxOutputException

`ScxOutputException` 表示宏观输出异常。

例如：

1. 底层 `OutputStream` 抛出 `IOException`。
2. gzip 输出失败。
3. 长度校验失败。
4. 输出包装器发现底层输出状态异常。

构造方法包括：

```java
public ScxOutputException(String message)

public ScxOutputException(Throwable cause)

public ScxOutputException(String message, Throwable cause)
```

### NoMoreDataException

`NoMoreDataException` 表示没有更多可用数据。

它本质上是流结束信号。

在 `ByteInput` 层：

```text
read / readUpTo / readFully 等动作方法在立即 EOF 时会抛 NoMoreDataException
```

在 `ByteSupplier` 层：

```text
EOF 用 null 表示
```

### NoMatchFoundException

`NoMatchFoundException` 表示没有找到匹配的字节序列。

常见于：

```java
input.indexOf(pattern, maxLength);

input.readUntil(pattern, maxLength);
```

如果在最大查找范围内没有找到模式，就会抛出这个异常。

### AlreadyClosedException

`InputAlreadyClosedException` 表示输入已关闭。

`OutputAlreadyClosedException` 表示输出已关闭。

它们都是状态异常，用来定位错误调用。

## 完整示例：按行读取

```java
import dev.scx.io.ScxIO;
import dev.scx.io.exception.NoMoreDataException;
import dev.scx.io.indexer.LineBreakByteIndexer;

public class LineReadDemo {

    public static void main(String[] args) {
        var input = ScxIO.createByteInput("""
            line1
            line2
            line3
            """.getBytes());

        var lineBreak = new LineBreakByteIndexer();

        while (true) {
            try {
                var line = input.readUntil(lineBreak);
                System.out.println(new String(line));
                lineBreak.reset();
            } catch (NoMoreDataException e) {
                break;
            }
        }
    }

}
```

需要注意，`ByteIndexer` 是有状态的。

如果你在循环中复用同一个 `ByteIndexer`，在一些场景下可以在每次处理后调用：

```java
lineBreak.reset();
```

对于 `ByteInput#readUntil(byte[])` 这种便捷方法，内部会为每次调用创建新的 indexer。

## 完整示例：限制读取长度

```java
import dev.scx.io.ScxIO;
import dev.scx.io.supplier.LimitLengthByteSupplier;

public class LimitLengthDemo {

    public static void main(String[] args) {
        var rawInput = ScxIO.createByteInput("hello world".getBytes());

        var supplier = new LimitLengthByteSupplier(rawInput, 5);

        var limitedInput = ScxIO.createByteInput(supplier);

        var bytes = limitedInput.readAll();

        System.out.println(new String(bytes));
    }

}
```

输出：

```text
hello
```

## 完整示例：缓存并回放

```java
import dev.scx.io.ScxIO;

public class CacheDemo {

    public static void main(String[] args) {
        var supplier = ScxIO.cacheByteSupplier(
            ScxIO.createByteInput("hello world".getBytes())
        );

        var input1 = ScxIO.createByteInput(supplier);

        System.out.println(new String(input1.readAll()));

        supplier.reset();

        var input2 = ScxIO.createByteInput(supplier);

        System.out.println(new String(input2.readAll()));
    }

}
```

输出：

```text
hello world
hello world
```

## 完整示例：gzip 压缩和解压

```java
import dev.scx.io.ScxIO;

public class GzipDemo {

    public static void main(String[] args) {
        var raw = "hello gzip".getBytes();

        var compressed = ScxIO.gzip(raw);

        var uncompressed = ScxIO.ungzip(compressed);

        System.out.println(new String(uncompressed));
    }

}
```

输出：

```text
hello gzip
```

## 完整示例：ByteInput 到 ByteOutput

```java
import dev.scx.io.ScxIO;
import dev.scx.io.output.ByteArrayByteOutput;

public class TransferDemo {

    public static void main(String[] args) {
        var input = ScxIO.createByteInput(
            "hello ".getBytes(),
            "world".getBytes()
        );

        var output = new ByteArrayByteOutput();

        long count = ScxIO.transferToAll(input, output);

        System.out.println(count);
        System.out.println(new String(output.bytes()));
    }

}
```

输出：

```text
11
hello world
```

## 完整示例：读取 boundary 前的数据

```java
import dev.scx.io.ScxIO;
import dev.scx.io.indexer.KMPByteIndexer;
import dev.scx.io.supplier.BoundaryByteSupplier;

public class BoundaryDemo {

    public static void main(String[] args) {
        var rawInput = ScxIO.createByteInput("part1--boundary--part2".getBytes());

        var supplier = new BoundaryByteSupplier(
            rawInput,
            new KMPByteIndexer("--boundary--".getBytes()),
            false
        );

        var partInput = ScxIO.createByteInput(supplier);

        var part = partInput.readAll();

        var remaining = rawInput.readAll();

        System.out.println(new String(part));
        System.out.println(new String(remaining));
    }

}
```

输出：

```text
part1
part2
```

如果把 `keepBoundaryInSource` 改为 `true`：

```java
var supplier = new BoundaryByteSupplier(
    rawInput,
    new KMPByteIndexer("--boundary--".getBytes()),
    true
);
```

则剩余数据会是：

```text
--boundary--part2
```

## 设计说明

### 1. ByteChunk 是视图，不是数据所有者

`ByteChunk` 只是 `byte[]` 上的区间视图。

这可以减少复制，但也要求调用者遵守只读约定。

如果需要长期保存数据，并且不确定底层数组是否稳定，应复制成新的 `byte[]`。

### 2. ByteSupplier 和 ByteInput 分层

`ByteSupplier` 是底层数据块来源。

`ByteInput` 是上层读取器。

这样可以把“数据从哪里来”和“如何读取、预读、查找、跳过”分开。

例如：

```text
InputStreamByteSupplier
FileByteSupplier
BoundaryByteSupplier
CacheByteSupplier
SequenceByteSupplier
```

都可以放到 `DefaultByteInput` 下面使用。

### 3. ByteInput 使用异常表达 EOF

`ByteInput` 的动作方法把 EOF 作为明确控制流信号。

例如：

```java
input.read();
```

如果没有更多数据，会抛出：

```text
NoMoreDataException
```

这样可以避免 `-1`、`0`、空数组等多种 EOF 表达混在一起的问题。

### 4. 请求型方法对 EOF 更宽松

`readAll()`、`peekAll()`、`skipAll()` 这类方法通常表示“把剩下的都给我”。

因此它们在 EOF 时返回空结果或 `0`，而不是抛出 `NoMoreDataException`。

### 5. read / readUpTo / readFully 的容忍度不同

三者不是重复 API，而是表达三种不同读取语义：

```text
read          尽快给我一块
readUpTo      尽量给我指定长度
readFully     必须给我指定长度
```

协议解析中推荐优先使用语义最明确的方法。

### 6. ByteConsumer 是 retaining 语义

`ByteConsumer` 可以保存接收到的 `ByteChunk`。

因此调用方要保证传入的 `ByteChunk` 稳定。

这和 `ByteOutput` 不同。

### 7. ByteOutput 是立即写出语义

`ByteOutput` 实现不应保存传入的 `ByteChunk`。

如果需要保存，必须在 `write(...)` 内部复制。

这可以避免调用方传入临时视图后，输出实现延迟使用导致数据损坏。

### 8. get / borrow 区分数据生命周期

`ByteSupplier#get()` 返回稳定视图。

`ByteSupplier#borrow()` 返回临时视图。

这让实现可以在性能和生命周期之间做明确取舍。

### 9. ByteIndexer 是有状态匹配器

`ByteIndexer` 支持跨 chunk 匹配，所以它必须保存部分匹配状态。

这也是为什么它提供：

```java
reset()
```

如果一个 indexer 要在多个独立匹配任务中复用，应在合适时机重置。

### 10. close 通常是幂等的

SCX IO 中多数 `close()` 都按幂等方式设计。

重复 close 不应造成重复释放。

但如果 close 过程中抛出异常，通常不应把对象标记为已关闭。

### 11. SCX IO 不隐藏底层 IO 错误

底层 `IOException` 会被包装成 `ScxInputException` 或 `ScxOutputException`。

适配回 JDK `InputStream` / `OutputStream` 时，如果 cause 本身是 `IOException`，会尽量还原抛出。

## 常见问题

### SCX IO 和 InputStream / OutputStream 有什么区别？

`InputStream` 使用 `-1` 表示 EOF，很多方法的读取语义也比较混合。

SCX IO 把读取语义拆得更明确：

```text
read
readUpTo
readFully
readAll
peek
skip
indexOf
readUntil
```

同时引入 `ByteChunk`，可以更方便地做分块、查找和低拷贝处理。

### ByteInput 读不到数据为什么抛异常？

这是设计选择。

`ByteInput` 的动作方法要求“只要执行了有效读取请求，就必须读到至少一个字节”。

如果立即 EOF，就用 `NoMoreDataException` 明确表示流结束。

### readAll 为什么不抛 NoMoreDataException？

因为 `readAll()` 是请求型方法。

调用者通常只是想要剩余所有数据。

如果没有剩余数据，返回空数组更符合这个方法的使用预期。

### read 和 readUpTo 有什么区别？

`read(maxLength)` 更宽松，它最多只拉取有限次数，可能返回当前已有的一块数据。

`readUpTo(length)` 会尽量读取到指定长度，除非中途 EOF。

### readFully 什么时候用？

当协议要求固定长度字段时使用。

例如：

```java
var header = input.readFully(8);
```

如果不足 8 字节，说明数据不完整，应该抛出异常。

### peek 会移动指针吗？

不会。

`peek` 只是查看数据，不移动读取位置。

### indexOf 会移动指针吗？

不会。

`indexOf` 只是查找。

如果要读取到匹配位置，并跳过匹配模式，使用 `readUntil(...)`。

### readUntil 返回结果包含分隔符吗？

不包含。

并且调用结束后，读取指针会跳过分隔符。

### 如何只查看分隔符前的数据但不消费？

使用：

```java
peekUntil(...)
```

### ByteChunk 可以修改吗？

不应该。

虽然字段是公开的，底层也是普通 `byte[]`，但 SCX IO 的约定是 `ByteChunk` 是只读视图。

修改底层数组属于未定义行为。

### ByteSupplier#get 和 borrow 有什么区别？

`get()` 返回的数据可以在后续调用后继续使用。

`borrow()` 返回的数据只保证在下一次 `get()` / `borrow()` 前有效。

### ByteConsumer 可以保存 ByteChunk 吗？

可以。

`ByteConsumer` 是 retaining 语义。

但它仍然不能修改 `ByteChunk` 的底层数据。

### ByteOutput 可以保存 ByteChunk 吗？

不应该。

`ByteOutput` 是立即写出语义。

如果实现需要保存数据，必须在 `write(...)` 内部复制。

### 如何把 ByteInput 转成 InputStream？

```java
InputStream in = ScxIO.byteInputToInputStream(byteInput);
```

### 如何把 InputStream 转成 ByteInput？

```java
ByteInput input = ScxIO.inputStreamToByteInput(inputStream);
```

### 如何把 ByteOutput 转成 OutputStream？

```java
OutputStream out = ScxIO.byteOutputToOutputStream(byteOutput);
```

### 如何把 OutputStream 转成 ByteOutput？

```java
ByteOutput output = ScxIO.outputStreamToByteOutput(outputStream);
```

### gzip(byte[]) 适合大文件吗？

不适合。

`gzip(byte[])` 和 `ungzip(byte[])` 适合小数据。

大数据应该使用 `gzipByteInput(...)`、`gzipByteOutput(...)` 或 JDK 原生 gzip 流。

### BoundaryByteSupplier 的 keepBoundaryInSource 是什么意思？

表示匹配到 boundary 后，底层输入是否保留 boundary。

```text
true     底层输入仍然可以读到 boundary
false    当前 supplier 会跳过 boundary
```

### CacheByteSupplier 会缓存空块吗？

会。

它会保存上游返回的空块，以保证对上游行为的干预尽可能少。

### SequenceByteSupplier 会自动关闭每个子 supplier 吗？

会。

当某个子 supplier 读到 EOF 后，会关闭它并切换到下一个。

`SequenceByteSupplier#close()` 也会尽可能关闭当前和剩余的 supplier。

### LengthBoundedByteOutput 的 minLength 什么时候检查？

在 `close()` 时检查。

如果写入总长度小于 `minLength`，`close()` 会抛出 `ScxOutputException`。

### LengthBoundedByteOutput 的 maxLength 什么时候检查？

每次 `write(...)` 前检查。

如果本次写入会导致总长度超过 `maxLength`，立即抛出 `ScxOutputException`。

### NoCloseByteOutput 的 close 会做什么？

不会关闭底层输出，而是调用底层 `flush()`。

但 `NoCloseByteOutput` 自己会进入 closed 状态。

### NoCloseByteSupplier 的 close 会做什么？

什么都不做。

它用于隔离底层 supplier 的 close。

### drainOnClose 和 noClose 可以一起用吗？

可以。

使用：

```java
ScxIO.drainOnCloseNoClose(byteSupplier)
```

语义是：

```text
关闭时排空
但不关闭底层
```

### 什么情况下用 ByteIndexer？

当你需要在字节流中查找模式时使用。

例如：

```text
查找换行
查找 HTTP header 结束符
查找 multipart boundary
查找自定义协议帧分隔符
```

### KMPByteIndexer 和 BitMaskByteIndexer 怎么选？

一般模式可以使用 `KMPByteIndexer`。

短模式，并且长度不超过 64，可以使用 `BitMaskByteIndexer`。

如果只是查找单个字节，使用 `SingleByteIndexer`。

### LineBreakByteIndexer 匹配什么？

同时匹配：

```text
\n
\r\n
```

### 关闭 ByteInput 会关闭底层 ByteSupplier 吗？

`DefaultByteInput#close()` 会关闭底层 `ByteSupplier`。

如果不希望关闭底层，可以使用 `NoCloseByteSupplier` 包装。

### 关闭 ByteOutput 会关闭底层 OutputStream 吗？

`OutputStreamByteOutput#close()` 会关闭底层 `OutputStream`。

如果不希望关闭底层，可以使用 `NoCloseByteOutput`。

### SCX IO 是线程安全吗？

默认不应假定线程安全。

`ByteInput`、`ByteOutput`、`ByteSupplier` 通常都有内部状态，例如读取指针、缓存、关闭状态、匹配状态等。

如果需要多线程访问，应由调用方自己同步。

### 什么时候用 SCX IO？

适合下面这些场景：

1. 需要比 `InputStream` 更明确的读取语义。
2. 需要 `peek`、`mark`、`readUntil`。
3. 需要跨 chunk 查找字节模式。
4. 需要处理协议边界。
5. 需要把多个字节来源串起来。
6. 需要缓存并回放输入。
7. 需要 gzip 包装。
8. 需要在 `ByteInput` / `ByteOutput` 和 JDK stream 之间适配。
9. 需要低拷贝的 `ByteChunk` 流式消费。