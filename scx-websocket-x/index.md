# SCX WebSocket X

SCX WebSocket X 是 `scx-websocket` 和 `scx-http-x` 之间的集成库。

`scx-websocket` 提供 WebSocket 帧、消息、事件和协议处理能力；`scx-http-x` 提供 HTTP/1.1 客户端、服务端和 Upgrade 能力；`scx-websocket-x` 则把两者连接起来，提供 WebSocket 客户端握手、服务端 Upgrade 请求识别、握手校验、101 响应发送，以及升级后的 `ScxWebSocket` 创建。当前版本为 `0.3.0`，依赖 `scx-websocket 0.4.0` 和 `scx-http-x 0.4.0`。([GitHub][1])

它适合这种场景：

```text
你希望使用 scx-http-x 启动 HTTP 服务，
并在某些请求上升级为 WebSocket。

或者你希望使用 scx-http-x 的 HttpClient 发起 WebSocket 握手，
并得到 scx-websocket 的 ScxWebSocket 对象。
```

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-websocket-x</artifactId>
    <version>0.3.0</version>
</dependency>
```

`scx-websocket-x` 会引入 `scx-websocket` 和 `scx-http-x`，但具体项目中通常还会直接使用这两个库中的类型，例如 `HttpServer`、`HttpServerOptions`、`ScxWebSocket` 和 `ScxEventWebSocket`。([GitHub][1])

## 基本概念

SCX WebSocket X 中最常用的概念包括：

```text
WebSocketClient                         WebSocket 客户端
WebSocketOptions                        WebSocket 帧大小和消息大小选项
WebSocketUpgradeRequestFactory          HTTP/1.1 服务端 Upgrade 请求工厂

ScxClientWebSocketHandshakeRequest      客户端 WebSocket 握手请求
ScxClientWebSocketHandshakeResponse     客户端 WebSocket 握手响应
ScxServerWebSocketHandshakeRequest      服务端 WebSocket 握手请求
ScxServerWebSocketHandshakeResponse     服务端 WebSocket 握手响应

ScxClientWebSocketHandshakeRejectedException  客户端握手被拒绝异常
ScxServerWebSocketHandshakeInvalidException   服务端握手请求非法异常
```

核心流程分成两边：

```text
客户端：
WebSocketClient -> handshake request -> handshake response -> ScxWebSocket

服务端：
HttpServer + WebSocketUpgradeRequestFactory
-> ScxServerWebSocketHandshakeRequest
-> upgrade()
-> ScxWebSocket
```

## 快速开始

### 服务端

```java
import dev.scx.http.x.HttpServer;
import dev.scx.http.x.HttpServerOptions;
import dev.scx.websocket.event.ScxEventWebSocket;
import dev.scx.websocket.x.ScxServerWebSocketHandshakeRequest;
import dev.scx.websocket.x.WebSocketUpgradeRequestFactory;

public class WebSocketServerExample {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer(
            new HttpServerOptions()
                .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())
        );

        server.onRequest(request -> {
            if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
                var webSocket = wsRequest.upgrade();

                ScxEventWebSocket
                    .of(webSocket)
                    .onText(text -> {
                        System.out.println("收到消息: " + text);
                        webSocket.send("echo: " + text);
                    })
                    .onClose(closeInfo -> {
                        System.out.println("WebSocket closed: " + closeInfo);
                    })
                    .onError(Throwable::printStackTrace)
                    .start();

                return;
            }

            request.response().send("普通 HTTP 请求");
        });

        server.start(8080);
    }

}
```

测试代码中也是通过 `new HttpServerOptions().addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())` 注册 WebSocket Upgrade 工厂，然后在 `onRequest` 中通过 `instanceof ScxServerWebSocketHandshakeRequest` 判断当前请求是否是 WebSocket 握手请求。([GitHub][2])

### 客户端

```java
import dev.scx.websocket.x.WebSocketClient;

public class WebSocketClientExample {

    public static void main(String[] args) {
        var webSocket = new WebSocketClient()
            .webSocketHandshakeRequest()
            .uri("ws://127.0.0.1:8080/websocket")
            .upgrade();

        webSocket.send("hello");

        var message = webSocket.read();

        System.out.println(new String(message.payloadData()));

        webSocket.sendClose();
        webSocket.close();
    }

}
```

`WebSocketClient#webSocketHandshakeRequest()` 会创建一个 HTTP/1.1 客户端请求，并包装成 `ScxClientWebSocketHandshakeRequest`；调用 `upgrade()` 会完成握手并返回 `ScxWebSocket`。([GitHub][3])

## WebSocketClient

`WebSocketClient` 是客户端入口。

```java
var client = new WebSocketClient();
```

它内部持有一个 `HttpClient` 和一个 `WebSocketOptions`：

```java
var httpClient = client.httpClient();

var options = client.options();
```

构造方式：

```java
new WebSocketClient();

new WebSocketClient(new WebSocketOptions());

new WebSocketClient(httpClient, new WebSocketOptions());
```

`WebSocketClient` 的默认构造函数会创建新的 `HttpClient` 和默认 `WebSocketOptions`；`webSocketHandshakeRequest()` 会基于 `HttpVersion.HTTP_1_1` 创建请求。([GitHub][3])

## 客户端握手请求

客户端通过 `webSocketHandshakeRequest()` 创建握手请求。

```java
var request = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://127.0.0.1:8080/websocket")
    .addHeader("Authorization", "Bearer token");
```

然后可以分两步执行：

```java
var response = request.handshake();

if (response.handshakeAccepted()) {
    var webSocket = response.upgrade();
}
```

也可以一步完成：

```java
var webSocket = request.upgrade();
```

`ScxClientWebSocketHandshakeRequest#upgrade()` 内部就是 `handshake().upgrade()`。握手请求固定使用 `GET` 方法，并且不允许发送请求体；如果尝试自定义 HTTP method 或 body，会抛出 `UnsupportedOperationException`。([GitHub][4])

### 自动设置握手头

客户端执行 `handshake()` 时，会自动设置：

```text
GET
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: <随机生成>
Sec-WebSocket-Version: 13
```

这些逻辑在 `Http1ClientWebSocketHandshakeRequest#handshake()` 中完成。([GitHub][5])

示例：

```java
var response = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://localhost:8080/chat")
    .addHeader("X-Client", "demo")
    .handshake();
```

## 客户端握手响应

客户端握手响应由 `ScxClientWebSocketHandshakeResponse` 表示。

```java
var response = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://localhost:8080/chat")
    .handshake();

boolean ok = response.handshakeAccepted();

var webSocket = response.upgrade();
```

`handshakeAccepted()` 会校验握手是否被接受；`upgrade()` 会在首次调用时校验 `101 Switching Protocols`、`Connection: Upgrade`、`Upgrade: websocket` 和 `Sec-WebSocket-Accept`，然后创建并缓存 `ScxWebSocket`；再次调用 `upgrade()` 会直接返回同一个 `ScxWebSocket` 实例。([GitHub][6])

客户端校验规则包括：

```text
状态码必须是 101 Switching Protocols
Connection 必须是 Upgrade
Upgrade 必须是 websocket
Sec-WebSocket-Accept 必须和客户端 Sec-WebSocket-Key 计算结果一致
```

如果校验失败，会抛出 `ScxClientWebSocketHandshakeRejectedException`。([GitHub][7])

## WebSocketOptions

`WebSocketOptions` 用来设置接收帧大小和接收消息大小。

```java
var options = new WebSocketOptions()
    .maxFrameSize(1024 * 1024)
    .maxMessageSize(8 * 1024 * 1024);

var client = new WebSocketClient(options);
```

默认值：

```text
maxFrameSize   = 16 MB
maxMessageSize = 64 MB
```

这两个值最终会传给底层 `scx-websocket`：`maxFrameSize` 用于 `ScxFrameWebSocket.of(...)`，`maxMessageSize` 用于 `ScxWebSocket.of(...)`。([GitHub][8])

服务端也可以使用相同选项：

```java
var options = new WebSocketOptions()
    .maxFrameSize(1024 * 1024)
    .maxMessageSize(8 * 1024 * 1024);

var server = new HttpServer(
    new HttpServerOptions()
        .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory(options))
);
```

## 服务端 Upgrade

服务端需要把 `WebSocketUpgradeRequestFactory` 注册到 `HttpServerOptions`。

```java
var server = new HttpServer(
    new HttpServerOptions()
        .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())
);
```

`WebSocketUpgradeRequestFactory` 实现了 `Http1UpgradeRequestFactory`，它声明的 upgrade protocol 是 `websocket`。当 `scx-http-x` 的 HTTP/1.1 server 识别到符合 Upgrade 条件的请求时，会通过这个工厂创建 `ScxServerWebSocketHandshakeRequest`。([GitHub][9])

### 判断 WebSocket 请求

```java
server.onRequest(request -> {
    if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
        var webSocket = wsRequest.upgrade();
        webSocket.send("hello");
        return;
    }

    request.response().send("普通 HTTP 请求");
});
```

测试代码中也使用这种方式判断请求类型。([GitHub][2])

## 服务端握手请求

`ScxServerWebSocketHandshakeRequest` 继承自 `ScxHttpServerRequest`，并额外提供：

```java
ScxServerWebSocketHandshakeResponse response();

String secWebSocketKey();

String secWebSocketVersion();

ScxWebSocket upgrade();
```

示例：

```java
if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
    System.out.println(wsRequest.secWebSocketKey());
    System.out.println(wsRequest.secWebSocketVersion());

    var webSocket = wsRequest.upgrade();

    webSocket.send("hello");
}
```

`secWebSocketKey()` 读取 `Sec-WebSocket-Key` 请求头；`secWebSocketVersion()` 读取 `Sec-WebSocket-Version` 请求头；`upgrade()` 是 `response().upgrade()` 的便捷方法。([GitHub][10])

## 服务端握手响应

服务端调用 `upgrade()` 接受握手并完成协议升级。

```java
var webSocket = wsRequest.response().upgrade();
```

或者：

```java
var webSocket = wsRequest.upgrade();
```

首次调用服务端 `upgrade()` 会：

```text
校验 Sec-WebSocket-Version
校验 Sec-WebSocket-Key
设置 Upgrade: websocket
设置 Connection: Upgrade
设置 Sec-WebSocket-Accept
设置状态码 101 Switching Protocols
发送 HTTP 响应
创建并缓存 ScxWebSocket
```

再次调用 `upgrade()` 不会再次产生 IO，而是直接返回同一个 `ScxWebSocket` 实例。服务端响应的 `upgrade()` 和普通 HTTP `send(...)` 是互斥的。([GitHub][11])

## 服务端握手校验

服务端握手校验包括：

```text
Sec-WebSocket-Version 必须存在
Sec-WebSocket-Version 必须是 13
Sec-WebSocket-Key 必须存在
Sec-WebSocket-Key 必须是合法 Base64
Sec-WebSocket-Key 解码后必须是 16 字节
```

如果校验失败，会抛出 `ScxServerWebSocketHandshakeInvalidException`。这个异常实现了 `ScxHttpException`，状态码是 `400 Bad Request`。([GitHub][12])

源码注释也说明，服务端响应不再校验 `Connection` 和 `Upgrade` 头，因为 `Http1ServerConnection` 只有在这两个请求头符合要求时才会调用 `WebSocketUpgradeRequestFactory`。([GitHub][12])

## 使用事件式 WebSocket

`upgrade()` 返回的是 `scx-websocket` 中的 `ScxWebSocket`，所以可以继续包装成 `ScxEventWebSocket`。

```java
import dev.scx.websocket.event.ScxEventWebSocket;

if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
    var eventWebSocket = ScxEventWebSocket.of(wsRequest.upgrade());

    eventWebSocket
        .onText(text -> {
            System.out.println("收到: " + text);
            eventWebSocket.send("echo: " + text);
        })
        .onClose(closeInfo -> {
            System.out.println("关闭: " + closeInfo.code() + " " + closeInfo.reason());
        })
        .onError(Throwable::printStackTrace)
        .start();
}
```

测试代码中服务端就使用 `ScxEventWebSocket.of(wsRequest.upgrade())` 处理文本消息、关闭事件和错误事件。([GitHub][2])

`ScxEventWebSocket#start()` 是阻塞读取循环；如果不希望阻塞当前请求处理线程，可以放到独立线程或虚拟线程中运行。

```java
var eventWebSocket = ScxEventWebSocket.of(wsRequest.upgrade());

Thread.ofVirtual().start(eventWebSocket::start);
```

## 使用消息级 WebSocket

也可以直接使用 `ScxWebSocket` 的消息级 API。

```java
if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
    var webSocket = wsRequest.upgrade();

    while (true) {
        var message = webSocket.read();

        if (message.type() == WebSocketMessageType.CLOSE) {
            break;
        }

        if (message.type() == WebSocketMessageType.TEXT) {
            var text = new String(message.payloadData());
            webSocket.send("echo: " + text);
        }
    }

    webSocket.close();
}
```

测试代码中的 `ManyWebSocketTest` 就展示了服务端直接使用 `webSocket.read()` / `webSocket.send(...)` 处理消息，而客户端再用 `ScxEventWebSocket` 读取回显消息。([GitHub][13])

## 客户端 URI

客户端可以使用 `ws://` URI：

```java
var webSocket = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://127.0.0.1:8080/websocket")
    .upgrade();
```

测试代码中也使用了 `ws://localhost:8080/websocket`。([GitHub][14])

测试中还出现了使用 `http://localhost:8899/...` 发起 WebSocket handshake 的例子，因为握手本质上是 HTTP/1.1 Upgrade 请求；不过面向用户文档中更推荐使用语义更明确的 `ws://`。([GitHub][2])

## 自定义请求头和 Cookie

客户端握手请求继承自 `ScxHttpClientRequest`，并重写了一批方法返回 `ScxClientWebSocketHandshakeRequest`，方便链式调用。

```java
var webSocket = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://127.0.0.1:8080/websocket")
    .addHeader("Authorization", "Bearer token")
    .addHeader("X-Client", "demo")
    .upgrade();
```

支持链式返回的方法包括：

```text
uri(...)
headers(...)
setHeader(...)
addHeader(...)
addCookie(...)
removeCookie(...)
```

这些方法在 `ScxClientWebSocketHandshakeRequest` 中被重写，以保持链式调用的返回类型。([GitHub][4])

## 完整示例：Echo Server + Client

```java
import dev.scx.http.x.HttpServer;
import dev.scx.http.x.HttpServerOptions;
import dev.scx.websocket.WebSocketMessageType;
import dev.scx.websocket.x.ScxServerWebSocketHandshakeRequest;
import dev.scx.websocket.x.WebSocketClient;
import dev.scx.websocket.x.WebSocketUpgradeRequestFactory;

public class EchoExample {

    public static void main(String[] args) throws Exception {
        startServer();

        var webSocket = new WebSocketClient()
            .webSocketHandshakeRequest()
            .uri("ws://127.0.0.1:8080/echo")
            .upgrade();

        webSocket.send("hello");

        var message = webSocket.read();

        if (message.type() == WebSocketMessageType.TEXT) {
            System.out.println(new String(message.payloadData()));
        }

        webSocket.sendClose();
        webSocket.close();
    }

    static void startServer() throws Exception {
        var server = new HttpServer(
            new HttpServerOptions()
                .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())
        );

        server.onRequest(request -> {
            if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
                var webSocket = wsRequest.upgrade();

                while (true) {
                    var message = webSocket.read();

                    if (message.type() == WebSocketMessageType.CLOSE) {
                        break;
                    }

                    if (message.type() == WebSocketMessageType.TEXT) {
                        webSocket.send(new String(message.payloadData()));
                    }
                }

                webSocket.close();
                return;
            }

            request.response().send("not websocket");
        });

        server.start(8080);
    }

}
```

## 完整示例：事件式聊天室骨架

```java
import dev.scx.http.x.HttpServer;
import dev.scx.http.x.HttpServerOptions;
import dev.scx.websocket.event.ScxEventWebSocket;
import dev.scx.websocket.x.ScxServerWebSocketHandshakeRequest;
import dev.scx.websocket.x.WebSocketUpgradeRequestFactory;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class ChatServerExample {

    private static final List<ScxEventWebSocket> clients = new CopyOnWriteArrayList<>();

    public static void main(String[] args) throws Exception {
        var server = new HttpServer(
            new HttpServerOptions()
                .addUpgradeRequestFactory(new WebSocketUpgradeRequestFactory())
        );

        server.onRequest(request -> {
            if (request instanceof ScxServerWebSocketHandshakeRequest wsRequest) {
                var eventWebSocket = ScxEventWebSocket.of(wsRequest.upgrade());

                clients.add(eventWebSocket);

                eventWebSocket
                    .onText(text -> {
                        for (var client : clients) {
                            client.send(text);
                        }
                    })
                    .onClose(closeInfo -> {
                        clients.remove(eventWebSocket);
                    })
                    .onError(error -> {
                        clients.remove(eventWebSocket);
                        error.printStackTrace();
                    })
                    .start();

                return;
            }

            request.response().send("SCX WebSocket X Chat Server");
        });

        server.start(8080);
    }

}
```

`WebSocketServerTest` 里也使用了 `CopyOnWriteArrayList` 保存连接，并在 `onClose` 时移除连接，用来测试是否存在“幽灵连接”。([GitHub][14])

## 异常

### ScxClientWebSocketHandshakeRejectedException

客户端握手响应不符合 WebSocket Upgrade 要求时抛出。

可能原因包括：

```text
状态码不是 101
缺少 Connection: Upgrade
缺少 Upgrade: websocket
缺少 Sec-WebSocket-Accept
Sec-WebSocket-Accept 不正确
```

异常类型是 `RuntimeException`。([GitHub][7])

```java
try {
    var webSocket = new WebSocketClient()
        .webSocketHandshakeRequest()
        .uri("ws://127.0.0.1:8080/websocket")
        .upgrade();
} catch (ScxClientWebSocketHandshakeRejectedException e) {
    System.err.println("握手被拒绝: " + e.getMessage());
}
```

### ScxServerWebSocketHandshakeInvalidException

服务端握手请求不合法时抛出。

可能原因包括：

```text
缺少 Sec-WebSocket-Version
Sec-WebSocket-Version 不是 13
缺少 Sec-WebSocket-Key
Sec-WebSocket-Key 不是合法 Base64
Sec-WebSocket-Key 解码后不是 16 字节
```

该异常实现 `ScxHttpException`，状态码是 `400 Bad Request`。([GitHub][12])

```java
try {
    var webSocket = wsRequest.upgrade();
} catch (ScxServerWebSocketHandshakeInvalidException e) {
    wsRequest.response().statusCode(e.statusCode()).send(e.getMessage());
}
```

## 设计说明

### 1. scx-websocket-x 是集成层

`scx-websocket` 只处理 WebSocket 协议帧、消息和事件；`scx-websocket-x` 处理的是 HTTP/1.1 Upgrade 握手和 `scx-http-x` 集成。`WebSocketClient` 依赖 `HttpClient`，`WebSocketUpgradeRequestFactory` 依赖 `Http1UpgradeRequestFactory`。([GitHub][3])

### 2. 升级后 TCP 连接由 WebSocket 独占

服务端 `upgrade()` 发送 `101 Switching Protocols` 后，会基于同一个 connection endpoint 创建 `ScxWebSocket`。源码注释也说明：一旦 WebSocket 升级响应发送成功，整个 TCP 将会被 WebSocket 独占。([GitHub][12])

### 3. upgrade() 是幂等的

客户端和服务端的 `upgrade()` 都会缓存创建出的 `ScxWebSocket`。首次调用会完成校验、升级和创建；再次调用不会产生新的 IO，而是返回同一个实例。([GitHub][6])

### 4. 客户端握手请求固定为 GET 且没有请求体

WebSocket 协议握手要求使用 GET 和空请求体，所以 `ScxClientWebSocketHandshakeRequest` 屏蔽了自定义 method 和 body send。([GitHub][4])

### 5. WebSocketOptions 只控制底层 WebSocket 的大小限制

`WebSocketOptions` 本身只包含 `maxFrameSize` 和 `maxMessageSize`。它不负责超时、重连、压缩、子协议、认证或代理等高级能力。([GitHub][8])

## 常见问题

### scx-websocket-x 和 scx-websocket 有什么区别？

`scx-websocket` 是 WebSocket 协议层，负责帧、消息、事件、close、ping/pong 等；`scx-websocket-x` 是 HTTP 集成层，负责通过 `scx-http-x` 发起或接受 HTTP/1.1 WebSocket Upgrade，并返回 `ScxWebSocket`。([GitHub][1])

### 服务端为什么要注册 WebSocketUpgradeRequestFactory？

因为 `scx-http-x` 需要通过 Upgrade 工厂把普通 HTTP 请求识别并包装成 `ScxServerWebSocketHandshakeRequest`。注册后，业务代码才能在 `onRequest` 中通过 `instanceof ScxServerWebSocketHandshakeRequest` 判断 WebSocket 握手请求。([GitHub][9])

### 客户端可以直接拿到 ScxWebSocket 吗？

可以。最简单方式是：

```java
var webSocket = new WebSocketClient()
    .webSocketHandshakeRequest()
    .uri("ws://localhost:8080/websocket")
    .upgrade();
```

`upgrade()` 会发送握手、校验响应，并返回 `ScxWebSocket`。([GitHub][4])

### `handshakeAccepted()` 和 `upgrade()` 有什么区别？

`handshakeAccepted()` 只校验响应是否符合 WebSocket 握手要求，并返回 boolean；`upgrade()` 会在校验通过后创建并返回 `ScxWebSocket`，校验失败则抛出异常。([GitHub][6])

### 服务端可以拒绝 WebSocket 握手吗？

可以。不要调用 `upgrade()`，而是按普通 HTTP 响应返回即可。

```java
if (!authorized) {
    wsRequest.response().statusCode(HttpStatusCode.UNAUTHORIZED).send("Unauthorized");
    return;
}

var webSocket = wsRequest.upgrade();
```

服务端 `upgrade()` 和普通 HTTP `send(...)` 是互斥的；一旦发送普通 HTTP 响应，就不应再升级。([GitHub][11])

### 默认最大帧和消息大小是多少？

默认最大帧大小是 16 MB，默认最大消息大小是 64 MB。([GitHub][8])

[1]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/test/java/dev/scx/websocket/x/test/WebSocketClientTest.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/WebSocketClient.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/ScxClientWebSocketHandshakeRequest.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/Http1ClientWebSocketHandshakeRequest.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/ScxClientWebSocketHandshakeResponse.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/Http1ClientWebSocketHandshakeResponse.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/WebSocketOptions.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/WebSocketUpgradeRequestFactory.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/ScxServerWebSocketHandshakeRequest.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/ScxServerWebSocketHandshakeResponse.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/main/java/dev/scx/websocket/x/Http1ServerWebSocketHandshakeResponse.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/test/java/dev/scx/websocket/x/test/ManyWebSocketTest.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-websocket-x/master/src/test/java/dev/scx/websocket/x/test/WebSocketServerTest.java "raw.githubusercontent.com"
