# SCX HTTP X

SCX HTTP X 是 `scx-http` 的具体实现库。

它基于 `scx-tcp` 和 Socket 提供可运行的 HTTP Client / Server，实现 HTTP/1.1 请求解析、响应发送、请求体读取、响应体发送、gzip、SSE、代理、TLS、Upgrade 扩展等能力。

[GitHub](https://github.com/scx-projects/scx-http-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-http-x</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX HTTP X 中最常用的概念包括：

```text
HttpServer                 HTTP 服务端实现
HttpServerOptions          服务端配置
HttpClient                 HTTP 客户端实现
HttpClientOptions          客户端配置
HttpClientRequest          客户端请求实现
Http1ServerConnection      HTTP/1.1 服务端连接
Http1ServerRequest         HTTP/1.1 服务端请求
Http1ServerResponse        HTTP/1.1 服务端响应
Http1ClientConnection      HTTP/1.1 客户端连接
Http1ClientResponse        HTTP/1.1 客户端响应
DefaultHttpServerErrorHandler 默认错误处理器
Proxy                      HTTP 代理配置
```

`HttpServer` 实现了 `ScxHttpServer`，`HttpClient` 实现了 `ScxHttpClient`。

## 快速开始：服务端

```java
import dev.scx.http.x.HttpServer;

public class ServerExample {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer();

        server.onRequest(request -> {
            String body = request.autoDecode().asString();

            System.out.println(request.method() + " " + request.uri());
            System.out.println(request.headers().encode());
            System.out.println(body);

            request.response()
                .gzip()
                .send("这是来自服务端的内容");
        });

        server.start(8899);

        System.out.println("HTTP Server started at " + server.localAddress());
    }

}
```

这个例子展示了最常见的服务端流程：创建 `HttpServer`，设置 `onRequest`，读取 `request.autoDecode().asString()`，最后通过 `request.response().gzip().send(...)` 返回响应。

## 快速开始：客户端

```java
import dev.scx.http.x.HttpClient;

import static dev.scx.http.method.HttpMethod.POST;

public class ClientExample {

    public static void main(String[] args) throws Exception {
        var client = new HttpClient();

        var response = client.request()
            .uri("http://localhost:8899/中文路径?a=1&b=2")
            .method(POST)
            .addHeader("X-Test", "1")
            .gzip()
            .send("这是来自客户端的内容");

        String body = response.autoDecode().asString();

        System.out.println(response.statusCode());
        System.out.println(response.headers().encode());
        System.out.println(body);
    }

}
```

客户端的常见流程是：`new HttpClient()`、`client.request()`、设置 URI、Header、POST、gzip，然后 `send(...)` 并读取响应。

## HttpServer

创建服务端：

```java
HttpServer server = new HttpServer();
```

指定配置：

```java
HttpServer server = new HttpServer(
    new HttpServerOptions()
        .backlog(1024)
        .maxRequestLineSize(64 * 1024)
        .maxHeaderSize(128 * 1024)
        .maxPayloadSize(16 * 1024 * 1024)
);
```

设置处理器并启动：

```java
server.onRequest(request -> {
    request.response().send("OK");
});

server.start(8080);
```

停止：

```java
server.stop();
```

`HttpServer` 内部基于 `TCPServer`，连接到来后会配置 socket、处理 TLS、创建 `SocketByteEndpoint`，然后选择 HTTP/1.1 或 HTTP/2 连接处理器。当前 HTTP/2 代码是占位实现，调用会抛 `UnsupportedOperationException`。

## HttpServerOptions

默认服务端配置：

```text
tls = null
enableHttp2 = false
tcpNoDelay = true

HTTP/1.1:
maxRequestLineSize = 64 KB
maxHeaderSize = 128 KB
maxPayloadSize = 16 MB
autoRespond100Continue = true
validateHost = true
```

这些默认值在 `HttpServerOptions` 和 `Http1ServerConnectionOptions` 中定义。

常用配置：

```java
var options = new HttpServerOptions()
    .backlog(1024)
    .tcpNoDelay(true)
    .maxRequestLineSize(64 * 1024)
    .maxHeaderSize(128 * 1024)
    .maxPayloadSize(16 * 1024 * 1024)
    .autoRespond100Continue(true)
    .validateHost(true);
```

注册 Upgrade 工厂：

```java
var options = new HttpServerOptions()
    .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory());
```

`addUpgradeRequestFactory(...)` 会把 `Http1UpgradeRequestFactory` 注册到 HTTP/1.1 连接配置中，用于 WebSocket 等协议升级场景。

## 服务端请求处理

```java
server.onRequest(request -> {
    System.out.println(request.method());
    System.out.println(request.uri());
    System.out.println(request.path());
    System.out.println(request.getQuery("id"));
    System.out.println(request.remotePeer());
    System.out.println(request.localPeer());

    String body = request.autoDecode().asString();

    request.response().send("OK");
});
```

HTTP/1.1 服务端请求由 `Http1ServerRequest` 实现，包含 method、uri、version、headers、bodyLength、body、remotePeer、localPeer 和 response。

## 服务端响应

```java
request.response()
    .statusCode(200)
    .setHeader("X-Server", "scx")
    .send("hello");
```

HTTP/1.1 服务端响应由 `Http1ServerResponse` 实现，默认状态码是 `200 OK`，可以设置 status code、headers 和 reason phrase。

自定义 reason phrase：

```java
((Http1ServerResponse) request.response())
    .reasonPhrase("Everything OK")
    .send("OK");
```

## Keep-Alive 与连接复用

HTTP/1.1 服务端默认会复用连接。一次响应结束后，连接会根据状态进入三种后续状态之一：

```text
Close Connection     关闭底层 socket
Connection Handoff   交给其他协议栈，例如 101 Upgrade 或 CONNECT
Connection Reuse     继续读取下一条 HTTP message
```

`Http1ServerConnection#onResponseEnd(...)` 会检查 `Connection: close`、`CONNECT`、`101 Switching Protocols`、手动 stop 等条件；否则继续 `requestNext()` 处理下一条请求。

## 100 Continue

服务端默认自动响应 `Expect: 100-continue`。

```java
var options = new HttpServerOptions()
    .autoRespond100Continue(true);
```

如果关闭自动响应，body 读取器会变成 `AutoContinueByteSupplier`，由用户读取 body 时触发继续响应。相关逻辑在 `Http1ServerConnection#readRequest()` 中。

## 错误处理

默认错误处理器是 `DefaultHttpServerErrorHandler.DEFAULT_HTTP_SERVER_ERROR_HANDLER`。

它的行为是：

```text
如果异常实现 ScxHttpException，使用异常的 statusCode
否则默认 500 INTERNAL_SERVER_ERROR
500 会记录日志
默认返回 HTML 错误页面
开发模式下包含堆栈信息
```

这些逻辑在 `DefaultHttpServerErrorHandler` 中实现。

自定义错误处理器：

```java
server.onError((throwable, request, phase) -> {
    request.response()
        .statusCode(500)
        .contentType(MediaType.TEXT_PLAIN)
        .send("Internal Server Error");
});
```

错误处理阶段分为 `SYSTEM` 和 `USER`：系统阶段通常是请求解析错误，用户阶段通常是用户处理器抛出的异常。

## HttpClient

创建客户端：

```java
HttpClient client = new HttpClient();
```

指定配置：

```java
HttpClient client = new HttpClient(
    new HttpClientOptions()
        .timeout(10_000)
        .tcpNoDelay(true)
        .maxStatusLineSize(64 * 1024)
        .maxHeaderSize(128 * 1024)
        .maxPayloadSize(16 * 1024 * 1024)
);
```

发送 GET：

```java
var response = client.request()
    .uri("http://localhost:8080/hello")
    .send();

String body = response.asString();
```

发送 POST：

```java
var response = client.request()
    .uri("http://localhost:8080/users")
    .method(HttpMethod.POST)
    .contentType(MediaType.TEXT_PLAIN)
    .send("Tom");
```

`HttpClientRequest` 默认 method 是 `GET`，默认 URI 是空 URI；`send0(...)` 会创建 socket、创建 `SocketByteEndpoint`，再使用 HTTP/1.1 或 HTTP/2 连接发送请求并读取响应。

## HttpClientOptions

默认客户端配置：

```text
tls = TLS.ofDefault()
proxy = null
timeout = 10 秒
enableHttp2 = false
tcpNoDelay = true

HTTP/1.1:
maxStatusLineSize = 64 KB
maxHeaderSize = 128 KB
maxPayloadSize = 16 MB
```

这些默认值在 `HttpClientOptions` 和 `Http1ClientConnectionOptions` 中定义。

示例：

```java
var options = new HttpClientOptions()
    .timeout(5000)
    .tcpNoDelay(true)
    .maxStatusLineSize(64 * 1024)
    .maxHeaderSize(128 * 1024)
    .maxPayloadSize(16 * 1024 * 1024);

var client = new HttpClient(options);
```

## HTTPS / TLS

客户端默认使用系统 TLS：

```java
new HttpClientOptions().tls(TLS.ofDefault());
```

服务端默认没有 TLS，需要显式设置：

```java
var server = new HttpServer(
    new HttpServerOptions()
        .tls(tls)
);
```

`HttpClient` 会根据 URI scheme 判断是否使用 TLS；`HttpServer` 会在连接进入 HTTP 处理器前根据 options 中的 TLS 配置升级 socket。

## HTTP 代理

配置代理：

```java
import dev.scx.http.x.proxy.Proxy;

var client = new HttpClient(
    new HttpClientOptions()
        .proxy(Proxy.of("127.0.0.1", 17890))
);
```

`Proxy.of(host, port)` 会创建代理配置。

HTTP 请求走代理时，客户端会使用代理地址创建 socket；HTTPS 请求走代理时，会先向代理发送 `CONNECT`，收到 `200 OK` 后再在该连接上升级 TLS。

也可以写成一个简单代理服务器示例：普通 HTTP 代理会复制 method、uri、headers 和 body 转发请求；HTTPS 代理通过 CONNECT 建立隧道后直接转发 TCP 流量。

## Gzip

客户端发送 gzip 请求：

```java
var response = client.request()
    .uri("http://localhost:8080")
    .gzip()
    .send("compressed request");
```

服务端返回 gzip 响应：

```java
request.response()
    .gzip()
    .send("compressed response");
```

读取 gzip 内容：

```java
String body = response.autoDecode().asString();
```

`gzip()` 会设置 `Content-Encoding: gzip` 并压缩 body；`autoDecode()` 会根据 `Content-Encoding` 自动解码 gzip。

## SSE

服务端：

```java
server.onRequest(request -> {
    request.response().setHeader("Access-Control-Allow-Origin", "*");

    var eventStream = request.response().sendEventStream();

    try (eventStream) {
        for (int i = 0; i < 100; i = i + 1) {
            eventStream.send(
                SseEvent.of("hello " + i)
                    .id(String.valueOf(i))
                    .event("message")
            );
        }
    }
});
```

客户端：

```java
var eventStream = client.request()
    .uri("http://127.0.0.1:8080/events")
    .send()
    .asEventStream();

EventClientEventStream
    .of(eventStream)
    .onEvent(event -> {
        System.out.println(event.event() + " " + event.data());
    })
    .start();
```

SSE 的典型组合是：服务端用 `sendEventStream()` 发送 `SseEvent`，客户端通过 `asEventStream()` 和 `EventClientEventStream` 接收事件。

## Upgrade 扩展

`HttpServerOptions#addUpgradeRequestFactory(...)` 可以注册 HTTP/1.1 Upgrade 工厂：

```java
var server = new HttpServer(
    new HttpServerOptions()
        .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())
);
```

`Http1ServerConnection#readRequest()` 会检查请求是否是 Upgrade 请求，如果存在匹配的 `Http1UpgradeRequestFactory`，就创建对应的升级请求对象；否则创建普通 `Http1ServerRequest`。

这个机制被 `scx-websocket-x` 用来把 HTTP 请求升级为 WebSocket。

## HTTP/2 状态

`HttpClientOptions` 和 `HttpServerOptions` 都有 `enableHttp2(boolean)`，但当前 `Http2ClientConnection` 和 `Http2ServerConnection` 是占位实现，调用会抛出 `UnsupportedOperationException`。

因此用户文档中建议按 HTTP/1.1 使用：

```java
new HttpClientOptions().enableHttp2(false);

new HttpServerOptions().enableHttp2(false);
```

## 完整示例：HTTP Server + Client

```java
import dev.scx.http.x.HttpClient;
import dev.scx.http.x.HttpServer;

import static dev.scx.http.method.HttpMethod.POST;

public class HttpXExample {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer();

        server.onRequest(request -> {
            var body = request.autoDecode().asString();

            System.out.println("收到请求:");
            System.out.println(request.method() + " " + request.uri());
            System.out.println(request.headers().encode());
            System.out.println(body);

            request.response()
                .gzip()
                .send("服务端响应内容");
        });

        server.start(8899);

        var client = new HttpClient();

        var response = client.request()
            .uri("http://localhost:8899/hello?a=1")
            .method(POST)
            .addHeader("X-Test", "1")
            .gzip()
            .send("客户端请求内容");

        String responseBody = response.autoDecode().asString();

        System.out.println("收到响应:");
        System.out.println(response.statusCode());
        System.out.println(response.headers().encode());
        System.out.println(responseBody);

        server.stop();
    }

}
```

## 完整示例：自定义错误处理

```java
import dev.scx.http.exception.ScxHttpException;
import dev.scx.http.status_code.ScxHttpStatusCode;
import dev.scx.http.x.HttpServer;

import static dev.scx.http.media_type.MediaType.TEXT_PLAIN;
import static dev.scx.http.status_code.HttpStatusCode.NOT_FOUND;

public class ErrorExample {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer();

        server.onRequest(request -> {
            if (request.path().equals("/not-found")) {
                throw new NotFound();
            }

            request.response().send("OK");
        });

        server.onError((throwable, request, phase) -> {
            var status = throwable instanceof ScxHttpException e
                ? e.statusCode()
                : ScxHttpStatusCode.of(500);

            request.response()
                .statusCode(status)
                .contentType(TEXT_PLAIN)
                .send("error: " + status.value());
        });

        server.start(8080);
    }

    static class NotFound extends RuntimeException implements ScxHttpException {

        @Override
        public ScxHttpStatusCode statusCode() {
            return NOT_FOUND;
        }

    }

}
```

## 设计说明

### 1. scx-http-x 是实现层

`scx-http` 定义抽象，`scx-http-x` 提供基于 TCP / Socket 的实现。`HttpServer` 内部使用 `TCPServer`，`HttpClient` 内部使用 `TCPClient` 创建 socket。

### 2. 请求和响应只能发送一次

客户端请求和服务端响应都继承自 `AbstractHttpSender`。它内部使用 `senderStatus` 和锁确保 `send(...)` 只能在 `NOT_SENT` 状态调用一次；重复发送会抛出 `IllegalSenderStateException`。

### 3. 服务端每个连接用虚拟线程处理

`Http1ServerConnection#requestNext()` 会启动虚拟线程处理请求。响应结束后，如果连接可以复用，会继续进入下一次请求读取。

### 4. 连接升级后 HTTP 层停止管理连接

对于 `101 Switching Protocols` 和 `CONNECT 200` 等场景，HTTP/1.1 连接不会继续复用该 socket，而是把连接交给升级协议或隧道逻辑。

### 5. 默认客户端不使用连接池

`HttpClient` 不复用连接池。每个请求会创建连接，发送请求，读取响应。

## 常见问题

### scx-http 和 scx-http-x 有什么区别？

`scx-http` 是抽象层，定义 HTTP 请求、响应、Header、URI、Sender / Received 等模型；`scx-http-x` 是实现层，提供可运行的 `HttpClient` 和 `HttpServer`。

### 为什么响应 `send(...)` 只能调用一次？

因为 `send(...)` 表示一次协议级提交动作，headers 和 body 必须作为整体提交，提交后不可逆。`AbstractHttpSender` 也在实现上限制了重复发送。

### body 可以读取多次吗？

默认不建议。`body()` 是一次性的顺序字节输入。需要多次读取时，使用 `cache()`。

### 默认最大请求体 / 响应体是多少？

HTTP/1.1 客户端和服务端的默认 `maxPayloadSize` 都是 16 MB。

### 现在能用 HTTP/2 吗？

不建议。虽然配置里有 `enableHttp2`，但当前 HTTP/2 client / server connection 是占位实现，调用会抛出 `UnsupportedOperationException`。
