# SCX HTTP

SCX HTTP 是一个轻量的 HTTP 抽象库。

它定义了 HTTP Client、HTTP Server、请求、响应、Header、URI、Method、Status Code、Media 读写、错误处理等基础接口和模型。SCX HTTP 本身不负责创建 TCP 连接，也不直接启动 HTTP 服务；它更像一组协议层抽象，供 `scx-http-x`、`scx-http-routing`、`scx-web`、`scx-websocket-x` 等上层或实现库复用。当前版本为 `0.5.0`。([GitHub][1])

## 安装

### Maven

```xml id="p5o1ef"
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-http</artifactId>
    <version>0.5.0</version>
</dependency>
```

## 基本概念

SCX HTTP 中最常用的概念包括：

```text id="0fvlv6"
ScxHttpClient             HTTP 客户端抽象
ScxHttpClientRequest      HTTP 客户端请求构建器
ScxHttpClientResponse     HTTP 客户端响应
ScxHttpServer             HTTP 服务端抽象
ScxHttpServerRequest      HTTP 服务端请求
ScxHttpServerResponse     HTTP 服务端响应

ScxHttpReceived           已接收的 HTTP 消息事实
ScxHttpSender             一次 HTTP 发送动作
ScxHttpMediaReceived      带便捷 media 读取能力的 received
ScxHttpMediaSender        带便捷 media 发送能力的 sender

ScxHttpHeaders            只读 Header
ScxHttpHeadersWritable    可写 Header
ScxURI                    URI 抽象
ScxMediaType              Media Type 抽象
Parameters                多值参数集合
```

其中 `ScxHttpReceived` 表示“HTTP 消息头已经接收完成之后的只读事实”，包含 `headers()`、`bodyLength()` 和 `body()`；`ScxHttpSender` 表示一次不可逆的 HTTP 发送提交动作，核心方法是 `send(bodyWriter)`。([GitHub][2]) ([GitHub][3])

## Client 抽象

`ScxHttpClient` 只定义一个能力：创建请求。

```java id="l0j1yq"
ScxHttpClientRequest request(HttpVersion... httpVersions);
```

`httpVersions` 表示客户端支持的 HTTP 版本，空列表表示由实现自动协商。([GitHub][4])

示例：

```java id="m3g6ja"
ScxHttpClient client = ...;

ScxHttpClientResponse response = client
    .request()
    .uri("http://localhost:8080/hello")
    .method("GET")
    .send();

String body = response.asString();
```

`ScxHttpClientRequest` 更接近一个 Builder，而不是一个已经存在的请求事实对象。它可以设置 method、uri、headers，并通过 `send(...)` 提交请求。([GitHub][5])

## Server 抽象

`ScxHttpServer` 定义服务端最小能力：

```java id="lcn76r"
ScxHttpServer onRequest(Function1Void<ScxHttpServerRequest> requestHandler);

ScxHttpServer onError(ScxHttpServerErrorHandler errorHandler);

void start(SocketAddress localAddress) throws IOException;

void stop();

InetSocketAddress localAddress();

default void start(int port) throws IOException;
```

`onRequest(...)` 设置请求处理器，`onError(...)` 设置错误处理器，`start(...)` 启动服务，`stop()` 停止服务。([GitHub][6])

示例：

```java id="ggl2za"
ScxHttpServer server = ...;

server.onRequest(request -> {
    request.response().send("Hello SCX HTTP");
});

server.start(8080);
```

`ScxHttpServer` 的源码注释也规定了错误处理原则：接收请求或执行用户处理器时发生错误，应调用错误处理器；如果错误处理器自身也抛异常，不应递归调用错误处理器。([GitHub][6])

## 服务端请求

`ScxHttpServerRequest` 表示服务端已经接收到的请求。

```java id="g6nzcc"
ScxHttpServerResponse response();

ScxHttpMethod method();

ScxURI uri();

HttpVersion version();

PeerInfo remotePeer();

PeerInfo localPeer();

String path();

Parameters query();

String getQuery(String name);
```

它继承了 `ScxHttpMediaReceived` 和 `EasyHttpHeadersReader`，因此可以直接读取 body、headers、content type、cookies 等信息。([GitHub][7])

示例：

```java id="msx0sn"
server.onRequest(request -> {
    System.out.println(request.method());
    System.out.println(request.path());
    System.out.println(request.getQuery("id"));
    System.out.println(request.headers().encode());

    String body = request.autoDecode().asString();

    request.response().send("OK");
});
```

## 服务端响应

`ScxHttpServerResponse` 表示服务端响应发送器。

```java id="tktfq0"
ScxHttpServerRequest request();

ScxHttpStatusCode statusCode();

ScxHttpServerResponse statusCode(ScxHttpStatusCode statusCode);

default ScxHttpServerResponse statusCode(int statusCode);
```

它继承了 `ScxHttpMediaSender` 和 `EasyHttpHeadersWriter`，所以可以设置状态码、Header、Cookie、Content-Type，并发送字符串、字节、文件、表单、multipart、SSE 等内容。([GitHub][8])

示例：

```java id="m4njmb"
request.response()
    .statusCode(200)
    .contentType(MediaType.TEXT_PLAIN)
    .send("Hello");
```

## 发送内容

`ScxHttpMediaSender` 提供一组便捷 `send(...)` 方法：

```java id="k24qz6"
send();

send(ByteInput byteInput);

send(InputStream inputStream);

send(byte[] bytes);

send(String str);

send(String str, Charset charset);

send(File file);

send(File file, long offset, long length);

send(FormParams formParams);

send(MultiPart multiPart);

sendEventStream();
```

当响应还没有设置 `Content-Type` 时，`send(MediaWriter)` 会根据 `MediaWriter#mediaType()` 自动设置。([GitHub][9])

示例：

```java id="zhhb27"
request.response().send("hello");

request.response().send(new byte[]{1, 2, 3});

request.response().send(new File("demo.txt"));
```

### Gzip 发送

```java id="krgc3x"
request.response()
    .gzip()
    .send("compressed response");
```

`gzip()` 会包装 sender，并设置 `Content-Encoding: gzip`。重复对同一层 sender 调用 `gzip()` 是幂等的。([GitHub][9])

## 读取内容

`ScxHttpMediaReceived` 提供一组便捷 `as...` 方法：

```java id="dmem0n"
as(MediaReader<T, X> mediaReader);

asBytes();

asString();

asString(Charset charset);

asFile(File file);

asFormParams();

asMultiPart();

asEventStream();
```

这些方法基于 `body()` 读取内容；`MediaReader` 的约定是：读取会消费 `ByteInput`，并在读取完成后关闭它。([GitHub][10]) ([GitHub][11])

示例：

```java id="5ed2o8"
String body = request.asString();

byte[] bytes = request.asBytes();

FormParams form = request.asFormParams();
```

### 自动解码

```java id="u7m83a"
String body = request.autoDecode().asString();
```

`autoDecode()` 会根据 `Content-Encoding` 自动解码。目前源码中只支持 `gzip`；不支持的编码会抛出 `UnsupportedMediaTypeException`。([GitHub][10])

### 缓存 body

```java id="lwazid"
var cached = request.cache();

String a = cached.asString();
String b = cached.asString();
```

`cache()` 用于把一次性 body 转换成可重复读取的缓存视图。`ScxHttpReceived` 本身的 `body()` 通常是一次性的、不可回溯的；需要多次读取时，应通过装饰器实现。([GitHub][2])

## Header

### 创建 Header

```java id="eflvzu"
ScxHttpHeadersWritable headers = ScxHttpHeaders.of();

headers
    .set("X-Token", "abc")
    .contentType(MediaType.APPLICATION_JSON)
    .contentLength(100);
```

`ScxHttpHeaders` 是只读 Header，`ScxHttpHeadersWritable` 是可写 Header；它们都基于 `Parameters`，一个 Header 名称可以对应多个值。([GitHub][12]) ([GitHub][13])

### 读取 Header

```java id="brvkc3"
String token = request.getHeader("X-Token");

ScxMediaType contentType = request.contentType();

Long contentLength = request.contentLength();

Cookies cookies = request.cookies();

Accept accept = request.accept();
```

`EasyHttpHeadersReader` 提供了 `getHeader`、`contentType`、`contentLength`、`contentEncoding`、`contentDisposition`、`cookies`、`setCookies`、`accept` 等快捷方法。([GitHub][14])

### 写入 Header

```java id="u935b3"
response
    .setHeader("X-Server", "scx")
    .contentType(MediaType.TEXT_HTML)
    .addSetCookie(Cookie.of("token", "abc"));
```

`EasyHttpHeadersWriter` 提供了 `setHeader`、`addHeader`、`removeHeader`、`contentType`、`contentLength`、`contentEncoding`、`contentDisposition`、`cookies`、`addCookie`、`addSetCookie` 等快捷方法。([GitHub][15])

### 从字符串解析 Header

```java id="f154fj"
var headers = ScxHttpHeaders.parse("""
Content-Type: text/plain
X-Test: 1
""");
```

`ScxHttpHeaders.parse(...)` 支持 `\r\n` 和 `\n` 分隔；`parseStrict(...)` 只支持标准 `\r\n` 分隔。([GitHub][12])

## URI

`ScxURI` 是 HTTP URI 抽象。它内部保存的是未编码的原始字符串，编码只在 `encode(true)` 或 `toURI()` 时发生。([GitHub][16])

```java id="q229u8"
ScxURI uri = ScxURI.of("http://example.com/中文路径?a=空 格");

System.out.println(uri.path());
System.out.println(uri.getQuery("a"));
System.out.println(uri.encode(true));
```

创建空 URI 并逐步设置：

```java id="m7hb6c"
ScxURIWritable uri = ScxURI.of()
    .scheme("http")
    .host("localhost")
    .port(8080)
    .path("/hello")
    .addQuery("name", "Tom");
```

`ScxURIWritable` 支持设置 scheme、host、port、path、fragment，以及增删 query 参数。([GitHub][17])

## Parameters

`Parameters` 表示一组 HTTP 参数项，一个 name 可以对应多个 value。它专为 HTTP 场景设计，区分只读和可写版本。([GitHub][18])

```java id="8gwhye"
ParametersWritable<String, String> params = Parameters.of();

params
    .add("a", "1")
    .add("a", "2")
    .set("b", "3");

System.out.println(params.get("a"));     // 第一个值
System.out.println(params.getAll("a"));  // 所有值
```

`ParametersWritable` 提供 `set`、`add`、`remove`、`clear`。([GitHub][19])

## Method

标准 HTTP 方法由 `HttpMethod` 枚举提供：

```java id="fwf1pk"
GET
POST
PUT
DELETE
PATCH
HEAD
OPTIONS
CONNECT
TRACE
```

`ScxHttpMethod.of(String)` 会优先返回标准 `HttpMethod`，如果不是标准方法，则使用通用实现保存该方法名。HTTP Method 是大小写敏感的。([GitHub][20]) ([GitHub][21])

```java id="ri8k6w"
request.method(HttpMethod.POST);

request.method("CUSTOM");
```

## Status Code

标准状态码由 `HttpStatusCode` 枚举提供，例如：

```java id="9jvv7y"
OK
CREATED
NO_CONTENT
BAD_REQUEST
NOT_FOUND
METHOD_NOT_ALLOWED
INTERNAL_SERVER_ERROR
```

`ScxHttpStatusCode.of(int)` 会优先返回标准 `HttpStatusCode`，如果不是标准状态码，则使用通用实现保存该状态码。([GitHub][22]) ([GitHub][23])

```java id="px6bop"
response.statusCode(HttpStatusCode.OK);

response.statusCode(418);
```

## Media Type

`ScxMediaType` 表示 media type 的结构事实，也就是 `type/subtype + params`。它不是 `Content-Type`，因为 `Content-Type` 只是 Header 名称，真正的值语义是 Media Type。([GitHub][24])

```java id="tkl8i4"
ScxMediaType json = MediaType.APPLICATION_JSON;

ScxMediaType custom = ScxMediaType
    .of("application", "custom")
    .param("charset", "utf-8");
```

常用枚举包括：

```text id="yqkojc"
TEXT_PLAIN
TEXT_HTML
TEXT_EVENT_STREAM
APPLICATION_JSON
APPLICATION_XML
APPLICATION_OCTET_STREAM
APPLICATION_X_WWW_FORM_URLENCODED
MULTIPART_FORM_DATA
IMAGE_PNG
IMAGE_JPEG
VIDEO_MP4
```

这些常用类型定义在 `MediaType` 枚举中。([GitHub][25])

## FormParams

`FormParams` 表示 `application/x-www-form-urlencoded` 表单参数。

```java id="i5bazn"
FormParamsWritable form = FormParams.of()
    .add("name", "Tom")
    .add("age", "18");

response.send(form);
```

也可以从编码后的字符串解析：

```java id="vtfdet"
FormParams form = FormParams.parse("name=Tom&age=18");
```

`FormParams` 继承自 `Parameters`，并提供 `encode()`。([GitHub][26])

## Multipart

`MultiPart` 表示 `multipart/form-data` 内容。

```java id="y7j9mw"
MultiPartWritable multiPart = MultiPart.of();

// 添加 part 的具体 API 可结合 MultiPartWritable / MultiPartPart 使用
response.send(multiPart);
```

`MultiPart.of()` 会自动生成一个随机 boundary；也可以手动传入 boundary。([GitHub][27])

读取 multipart：

```java id="og5e1x"
try (var stream = request.asMultiPart()) {
    for (var part : stream) {
        System.out.println(part.headers().contentDisposition());
        byte[] bytes = part.body().readAllBytes();
    }
}
```

`MultiPartStream` 是流式读取器，迭代下一个 part 时会自动消费上一个未消费完的 part 内容；part header 使用严格模式解析。([GitHub][28])

## Server-Sent Events

服务端发送 SSE：

```java id="iqvg9w"
var eventStream = request.response().sendEventStream();

try (eventStream) {
    eventStream.send(
        SseEvent.of("hello")
            .id("1")
            .event("message")
            .comment("comment")
    );
}
```

`sendEventStream()` 会创建 `ServerEventStreamMediaWriter` 并返回 `ServerEventStream`。`ServerEventStream#send(...)` 会把 `SseEvent` 编码为 SSE 文本格式并 flush。([GitHub][9]) ([GitHub][29])

客户端读取 SSE：

```java id="plxe8v"
var eventStream = response.asEventStream();

EventClientEventStream
    .of(eventStream)
    .onEvent(event -> {
        System.out.println(event.event() + " " + event.data());
    })
    .start();
```

`ClientEventStream` 支持读取 `event`、`data`、`id`、`retry` 和注释行；`EventClientEventStream` 提供简单的事件回调封装。([GitHub][30]) ([GitHub][31])

## 错误处理

`ScxHttpException` 是一个标记接口。异常只要实现它，就可以被 `ScxHttpServerErrorHandler` 识别为 HTTP 响应异常。([GitHub][32])

```java id="w5se2m"
public class NotFoundException extends RuntimeException implements ScxHttpException {

    @Override
    public ScxHttpStatusCode statusCode() {
        return HttpStatusCode.NOT_FOUND;
    }

}
```

`ScxHttpServerErrorHandler` 的约定是：如果异常实现了 `ScxHttpException`，应使用其 `statusCode()` 生成响应；否则生成 500 响应。([GitHub][33])

错误阶段分为：

```text id="3p3qlq"
SYSTEM  系统阶段，例如解析 HTTP 头失败
USER    用户阶段，例如用户处理器抛异常
```

`ErrorPhase` 只包含这两个值。([GitHub][34])

## 设计说明

### 1. scx-http 是抽象层

`scx-http` 定义 HTTP 领域模型和接口，不负责具体 TCP / Socket 实现。真正可运行的实现由 `scx-http-x` 提供。

### 2. Received 是事实，Sender 是动作

`ScxHttpReceived` 表示已经接收到的 HTTP message head 事实，读取 body 不改变协议状态；`ScxHttpSender` 表示一次协议级提交动作，`send(...)` 一旦开始就是不可逆的提交。([GitHub][2]) ([GitHub][3])

### 3. Body 通常是一次性的

`body()` 返回的是顺序字节输入，通常不可回溯。需要多次读取时，使用 `cache()`。

### 4. MediaReader / MediaWriter 是扩展点

`MediaReader` 负责把 body 读成对象；`MediaWriter` 负责把对象写成 body，并可提供 media type。([GitHub][11]) ([GitHub][35])