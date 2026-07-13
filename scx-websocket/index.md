# SCX WebSocket

SCX WebSocket 是一个轻量的 WebSocket 协议处理库。

它提供帧级 WebSocket、消息级 WebSocket、事件式 WebSocket、关闭信息和协议异常等基础能力。

它不负责启动 HTTP 服务，也不负责完整的 HTTP Upgrade 流程；

它更像一个基于 `ByteEndpoint` 的 WebSocket 协议层工具。

[GitHub](https://github.com/scx-projects/scx-websocket)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-websocket</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX WebSocket 中最常用的概念包括：

```text
ScxFrameWebSocket       帧级 WebSocket 接口
ScxWebSocket            消息级 WebSocket 接口
ScxEventWebSocket       事件式 WebSocket 接口
WebSocketFrame          WebSocket 帧
WebSocketMessage        WebSocket 消息接口
TextMessage / BinaryMessage / PingMessage / PongMessage / CloseMessage 消息实现
WebSocketOpCode         帧操作码
WebSocketCloseInfo      关闭信息
WebSocketIOException    底层 IO 异常
WebSocketProtocolException 协议异常
WebSocketInvalidStateException 状态异常
```

库中有三个使用层次：

```text
ScxFrameWebSocket   处理单个 WebSocket frame
ScxWebSocket        处理完整 WebSocket message，会自动处理分片、ping/pong、close
ScxEventWebSocket   在 ScxWebSocket 之上提供 onText/onBinary/onClose 事件风格 API
```

`ScxFrameWebSocket` 是帧级接口，只保证单帧读写的协议合法性，不做消息重组和连接状态管理；`ScxWebSocket` 是消息级接口，会组合接收到的分片、发送完整消息，并处理 close 握手逻辑。

## 快速开始

下面示例假设你已经有一个 `ByteEndpoint`。`ByteEndpoint` 来自 `scx-io`，代表一组字节输入 / 输出端点。

### 消息级 WebSocket

```java
import dev.scx.io.endpoint.ByteEndpoint;
import dev.scx.websocket.CloseMessage;
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.TextMessage;

public class WebSocketExample {

    public void run(ByteEndpoint endpoint) {
        // true 表示当前端是 client；false 表示当前端是 server
        ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true);

        webSocket.send("hello");

        while (true) {
            var message = webSocket.read();

            if (message instanceof TextMessage textMessage) {
                System.out.println(textMessage.text());
            }

            if (message instanceof CloseMessage) {
                break;
            }
        }

        webSocket.close();
    }

}
```

`ScxWebSocket.of(endpoint, isClient)` 会基于 `ByteEndpoint` 创建帧级 WebSocket，再包装为消息级 WebSocket；默认最大帧大小是 16 MB，分片消息聚合上限是 64 MB。

## ByteEndpoint

SCX WebSocket 不直接依赖 `Socket`、`Channel` 或某个 HTTP Server。它通过 `ByteEndpoint` 读写字节。

可以用普通 `Socket` 包装了一个最小 `ByteEndpoint`：

```java
import dev.scx.io.ByteInput;
import dev.scx.io.ByteOutput;
import dev.scx.io.ScxIO;
import dev.scx.io.endpoint.ByteEndpoint;

import java.io.IOException;
import java.net.Socket;

public class SocketEndpoint implements ByteEndpoint {

    private final Socket socket;
    private final ByteInput in;
    private final ByteOutput out;

    public SocketEndpoint(Socket socket) throws IOException {
        this.socket = socket;
        this.in = ScxIO.createByteInput(socket.getInputStream());
        this.out = ScxIO.createByteOutput(socket.getOutputStream());
    }

    @Override
    public ByteInput in() {
        return in;
    }

    @Override
    public ByteOutput out() {
        return out;
    }

    @Override
    public void close() throws Exception {
        socket.close();
    }

}
```

## 客户端和服务端角色

创建 WebSocket 时需要传入 `isClient`：

```java
ScxWebSocket clientWebSocket = ScxWebSocket.of(endpoint, true);

ScxWebSocket serverWebSocket = ScxWebSocket.of(endpoint, false);
```

这个参数会影响 WebSocket 掩码规则。客户端发送帧时需要加掩码；服务端发送帧时不加掩码。读取时也会按当前端角色校验对端发来的帧是否符合掩码规则。

## 消息级 API：ScxWebSocket

`ScxWebSocket` 是推荐的常规使用入口。

创建：

```java
ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true);
```

指定分片消息聚合上限：

```java
ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true, 1024 * 1024);
```

第三个参数只用于接收分片消息时的聚合大小检查；底层帧大小仍由内部的 `ScxFrameWebSocket` 决定。

从已有帧级 WebSocket 创建：

```java
ScxFrameWebSocket frameWebSocket = ScxFrameWebSocket.of(endpoint, true);

ScxWebSocket webSocket = ScxWebSocket.of(frameWebSocket);

// 同时指定分片消息聚合上限
ScxWebSocket limitedWebSocket = ScxWebSocket.of(
    frameWebSocket,
    1024 * 1024
);
```

`ScxWebSocket` 提供：

```java
WebSocketMessage read();

void send(WebSocketMessage message);

void send(String textMessage);

void send(byte[] binaryMessage);

void sendPing(byte[] data);

void sendPong(byte[] data);

void sendClose(WebSocketCloseInfo closeInfo);

void sendClose(int code, String reason);

void sendClose();

void close();
```

这些方法都定义在 `ScxWebSocket` 接口中。接口不提供 `state()`、`isOpen()` 或 `isClosed()` 等状态查询；应直接调用 `read()` / `send(...)`，并通过返回结果或异常判断当前操作是否成功。

### 发送文本消息

```java
webSocket.send("hello websocket");
```

`send(String)` 会把字符串按 UTF-8 编码成 `TEXT` 消息。每条消息会转换为一个 `fin = true` 的帧发送，当前不会自动拆分为多个 frame。

### 发送二进制消息

```java
webSocket.send(new byte[]{1, 2, 3});
```

### 读取消息

```java
var message = webSocket.read();

switch (message) {
    case TextMessage textMessage -> {
        System.out.println(textMessage.text());
    }
    case BinaryMessage binaryMessage -> {
        byte[] data = binaryMessage.binary();
    }
    case CloseMessage closeMessage -> {
        WebSocketCloseInfo closeInfo = closeMessage.closeInfo();
        webSocket.close();
    }
    case PingMessage pingMessage -> {
        byte[] data = pingMessage.data();
    }
    case PongMessage pongMessage -> {
        byte[] data = pongMessage.data();
    }
}
```

`WebSocketMessage` 是一个密封接口，具体消息由不同的 record 表示。

### 消息类型

```java
TextMessage(String text)
BinaryMessage(byte[] binary)
CloseMessage(WebSocketCloseInfo closeInfo)
PingMessage(byte[] data)
PongMessage(byte[] data)
```

这些类型都实现了 `WebSocketMessage`。`text`、`binary`、`data` 和 `closeInfo` 都不能为 `null`。

## Ping / Pong

发送 ping：

```java
webSocket.sendPing("ping".getBytes(StandardCharsets.UTF_8));
```

发送 pong：

```java
webSocket.sendPong("pong".getBytes(StandardCharsets.UTF_8));
```

消息级 `ScxWebSocket` 在读取到 `PING` 帧时，会尝试立即回一个 `PONG`，payload 与 ping 相同。

```java
var message = webSocket.read();

if (message instanceof PingMessage) {
    // ScxWebSocket 内部已经尝试自动回 PONG
}
```

## Close

### 正常关闭

```java
webSocket.sendClose();
webSocket.close();
```

`sendClose()` 默认发送 `NORMAL_CLOSE`。

### 指定关闭码和原因

```java
webSocket.sendClose(1000, "normal close");
```

或者：

```java
import dev.scx.websocket.WebSocketCloseInfo;

webSocket.sendClose(
    new WebSocketCloseInfo(1000, "normal close")
);
```

`WebSocketCloseInfo(Integer code, String reason)` 是一个包含关闭码和原因的 record。空关闭信息使用 `new WebSocketCloseInfo(null, null)` 表示。

### 关闭码常量

`WebSocketCloseInfo` 提供常见关闭信息：

```java
NORMAL_CLOSE
GOING_AWAY
PROTOCOL_ERROR
CANNOT_ACCEPT
NO_STATUS_CODE
CLOSED_ABNORMALLY
NOT_CONSISTENT
VIOLATED_POLICY
TOO_BIG
NO_EXTENSION
UNEXPECTED_CONDITION
SERVICE_RESTART
TRY_AGAIN_LATER
```

这些常量包含关闭码和默认 reason。

### close() 的含义

`close()` 是资源级关闭，不等同于发送 WebSocket CLOSE 帧。它会立即关闭底层连接并释放资源，不发送任何 WebSocket 帧，也不等待对端响应。若需要按协议优雅关闭，应先发送 CLOSE 帧，再调用 `close()` 释放资源。

推荐写法：

```java
try {
    webSocket.sendClose();
} finally {
    webSocket.close();
}
```

## 关闭信息 payload

`WebSocketCloseInfo` 会由库在 `CloseMessage` 和 CLOSE 帧 payload 之间自动转换：

```java
var message = webSocket.read();

if (message instanceof CloseMessage closeMessage) {
    WebSocketCloseInfo closeInfo = closeMessage.closeInfo();
}
```

关闭 payload 规则：

```text
空 payload        -> code 和 reason 均为 null
长度为 1 的 payload -> 非法
前 2 字节         -> close code
剩余字节          -> UTF-8 reason
reason 最大长度   -> 123 字节
```

这些转换由库内部完成，无需手动处理 payload。

## 事件式 API：ScxEventWebSocket

如果你更喜欢事件回调风格，可以使用 `ScxEventWebSocket`。

```java
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.event.ScxEventWebSocket;

ScxWebSocket messageWebSocket = ScxWebSocket.of(endpoint, true);

ScxEventWebSocket eventWebSocket = ScxEventWebSocket.of(messageWebSocket);

eventWebSocket
    .onText(text -> {
        System.out.println("text = " + text);
    })
    .onBinary(bytes -> {
        System.out.println("binary length = " + bytes.length);
    })
    .onClose(closeInfo -> {
        System.out.println("closed = " + closeInfo);
    })
    .onError(error -> {
        error.printStackTrace();
    });

eventWebSocket.start();
```

`ScxEventWebSocket` 本质上是 `ScxWebSocket` 的另一种使用风格，支持 `onText`、`onBinary`、`onPing`、`onPong`、`onClose`、`onError`、`send(...)`、`start()` 和 `close()`。`start()` 是阻塞方法，调用后会持续读取消息并触发回调。

### 在独立线程中启动

```java
Thread.ofVirtual().start(eventWebSocket::start);

eventWebSocket.send("hello");
```

一个常见流程是把 `ScxEventWebSocket#start` 放到虚拟线程中运行，然后继续发送消息。

### 使用回调执行器

```java
Executor executor = command -> Thread.ofVirtual().start(command);

ScxEventWebSocket eventWebSocket = ScxEventWebSocket.of(
    messageWebSocket,
    executor
);
```

`ScxEventWebSocket.of(messageWebSocket, callbackExecutor)` 可以传入回调执行器，避免回调之间互相阻塞。

### 事件式 close 行为

`ScxEventWebSocket#close()` 会关闭底层 message WebSocket，从而打断 `start()` 中阻塞的读取流程，并通过异常路径间接触发 `onClose`。实现中收到 CLOSE 消息时，也会调用 `onClose`、关闭连接并停止循环。

## 帧级 API：ScxFrameWebSocket

`ScxFrameWebSocket` 是更底层的帧级接口。通常只有在你需要直接控制 WebSocket frame 时才使用它。

```java
import dev.scx.websocket.frame.ScxFrameWebSocket;
import dev.scx.websocket.frame.WebSocketFrame;
import dev.scx.websocket.frame.WebSocketOpCode;

ScxFrameWebSocket frameWebSocket = ScxFrameWebSocket.of(endpoint, true);

frameWebSocket.sendFrame(
    new WebSocketFrame(
        WebSocketOpCode.TEXT,
        "hello".getBytes(StandardCharsets.UTF_8),
        true
    )
);

WebSocketFrame frame = frameWebSocket.readFrame();
```

`ScxFrameWebSocket` 只负责单帧协议合法性，不做状态管理，不做消息分片重组，也不管理 close / ping / pong 的连接状态。

### 指定最大帧大小

```java
ScxFrameWebSocket frameWebSocket = ScxFrameWebSocket.of(
    endpoint,
    true,
    1024 * 1024
);
```

默认最大帧大小是 16 MB，并且只约束接收帧。

### WebSocketFrame

```java
public record WebSocketFrame(
    WebSocketOpCode opCode,
    byte[] payloadData,
    boolean fin
) {
}
```

`WebSocketFrame` 要求 `opCode` 和 `payloadData` 都不能为 `null`。

### WebSocketOpCode

```java
CONTINUATION
TEXT
BINARY
CLOSE
PING
PONG
```

每个 `WebSocketOpCode` 都有对应的整数 code，并提供 `of(code)` 和 `find(code)` 方法。`of(code)` 找不到时抛出异常，`find(code)` 找不到时返回 `null`。

## 分片消息

消息级 `ScxWebSocket` 会自动处理分片消息。

当读取到非 final 的 `TEXT` 或 `BINARY` 帧时，它会进入分片聚合状态；后续读取 `CONTINUATION` 帧，直到 final continuation 到达后合并成一个完整消息。聚合后的消息类型使用起始帧的 opcode。

```java
// 对调用方来说，read() 返回的是完整 message
WebSocketMessage message = webSocket.read();
```

非法分片会抛出 `WebSocketProtocolException`，例如：

```text
分片过程中又收到新的 TEXT / BINARY 帧
没有分片进行中却收到 CONTINUATION 帧
聚合后的消息超过 maxMessageSize
```

这些规则由 `ScxWebSocketImpl#readFrameUntilLast()` 和 `appendFragment(...)` 实现。

## 最大大小限制

有两个大小限制：

```text
ScxFrameWebSocket 最大帧大小       默认 16 MB，约束每个接收帧
ScxWebSocket 分片消息聚合上限      默认 64 MB，只在分片聚合时检查
```

`ScxFrameWebSocket` 在读取协议帧头后检查 `payloadLength` 是否超过 `maxWebSocketFrameSize`；`ScxWebSocket` 只在分片聚合时检查总 payload 是否超过 `maxMessageSize`。单个 `fin = true` 的 `TEXT` / `BINARY` 帧不会再经过 `maxMessageSize` 检查，但仍受底层帧大小限制。两个限制都只约束接收端。

## 协议校验

SCX WebSocket 会做基础 WebSocket 协议校验。

帧级校验包括：

```text
opcode 必须合法
rsv1 / rsv2 / rsv3 必须为 false
客户端和服务端 mask 规则必须正确
控制帧必须 fin = true
控制帧 payload 长度必须 <= 125
close frame payload 长度不能为 1
```

这些校验由 `ScxFrameWebSocketImplHelper#fromProtocolFrame(...)` 和 `toProtocolFrame(...)` 完成。

底层 `WebSocketProtocolFrameHelper` 只负责二进制结构的读写，不做语义校验、不做掩码处理、不管理状态、不做分片重组；它不是直接面向业务使用的安全 WebSocket 实现。

## 异常

### WebSocketIOException

`WebSocketIOException` 表示底层 I/O 已不可继续使用，例如对端关闭连接、本地输入 / 输出已关闭、网络或传输错误。出现该异常后，连接生命周期结束，调用 `close()` 释放资源是安全且幂等的。

```java
try {
    var message = webSocket.read();
} catch (WebSocketIOException e) {
    webSocket.close();
}
```

### WebSocketProtocolException

`WebSocketProtocolException` 表示协议异常，例如帧过大、掩码错误、非法分片等。它包含一个 `closeCode()`，可以用于给对端发送 close frame。

```java
try {
    webSocket.read();
} catch (WebSocketProtocolException e) {
    webSocket.sendClose(e.closeCode(), e.getMessage());
    webSocket.close();
}
```

### WebSocketInvalidStateException

`WebSocketInvalidStateException` 表示当前连接状态不允许执行某个操作，例如已经发送 close frame 后继续发送普通消息。

```java
webSocket.sendClose();

// 之后再发送普通消息会抛出 WebSocketInvalidStateException
webSocket.send("after close");
```

`ScxWebSocketImpl#send(...)` 中也明确检查了：发送过 close 后，不允许再发送非 close 消息；收到 close 后，也不允许再发送非 close 消息。

## 完整示例：Socket + 消息级 WebSocket

下面示例演示一个非常简化的裸 socket WebSocket 通信。实际浏览器 WebSocket 场景还需要先完成 HTTP Upgrade 握手。

```java
import dev.scx.io.ByteInput;
import dev.scx.io.ByteOutput;
import dev.scx.io.ScxIO;
import dev.scx.io.endpoint.ByteEndpoint;
import dev.scx.websocket.CloseMessage;
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.TextMessage;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class MessageWebSocketExample {

    public static void main(String[] args) throws Exception {
        startServer();
        startClient();
    }

    static void startServer() throws IOException {
        var serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress(8899));

        Thread.ofPlatform().start(() -> {
            try {
                var socket = serverSocket.accept();

                // 服务端角色：isClient = false
                var webSocket = ScxWebSocket.of(new SocketEndpoint(socket), false);

                for (int i = 0; i < 10; i = i + 1) {
                    webSocket.send("message " + i);
                }

                webSocket.sendClose();
                webSocket.close();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }

    static void startClient() throws IOException {
        var socket = new Socket();
        socket.connect(new InetSocketAddress(8899));

        // 客户端角色：isClient = true
        var webSocket = ScxWebSocket.of(new SocketEndpoint(socket), true);

        while (true) {
            var message = webSocket.read();

            if (message instanceof CloseMessage) {
                System.out.println("closed");
                break;
            }

            if (message instanceof TextMessage textMessage) {
                System.out.println(textMessage.text());
            }
        }

        webSocket.close();
    }

    static class SocketEndpoint implements ByteEndpoint {

        private final Socket socket;
        private final ByteInput in;
        private final ByteOutput out;

        SocketEndpoint(Socket socket) throws IOException {
            this.socket = socket;
            this.in = ScxIO.createByteInput(socket.getInputStream());
            this.out = ScxIO.createByteOutput(socket.getOutputStream());
        }

        @Override
        public ByteInput in() {
            return in;
        }

        @Override
        public ByteOutput out() {
            return out;
        }

        @Override
        public void close() throws Exception {
            socket.close();
        }

    }

}
```

常见用法包括用 `ServerSocket` / `Socket` 创建 `ByteEndpoint`，再使用 `ScxFrameWebSocket`、`ScxWebSocket` 和 `ScxEventWebSocket` 的方式。

## 完整示例：事件式 WebSocket

```java
import dev.scx.io.endpoint.ByteEndpoint;
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.event.ScxEventWebSocket;

public class EventWebSocketExample {

    public void run(ByteEndpoint endpoint) {
        var messageWebSocket = ScxWebSocket.of(endpoint, true);

        var eventWebSocket = ScxEventWebSocket.of(messageWebSocket);

        eventWebSocket
            .onText(text -> {
                System.out.println("onText: " + text);
            })
            .onBinary(bytes -> {
                System.out.println("onBinary: " + bytes.length);
            })
            .onPing(bytes -> {
                System.out.println("onPing: " + bytes.length);
            })
            .onPong(bytes -> {
                System.out.println("onPong: " + bytes.length);
            })
            .onClose(closeInfo -> {
                System.out.println("onClose: " + closeInfo.code() + " " + closeInfo.reason());
            })
            .onError(error -> {
                error.printStackTrace();
            });

        eventWebSocket.start();
    }

}
```

`ScxEventWebSocketImpl#start()` 会循环调用 `messageWebSocket.read()`，根据消息类型调用对应回调；如果发生异常，会先调用 `onError`，再根据异常类型构造 close info，调用 `onClose`，尝试发送 close frame，并终止连接。

## 设计说明

### 1. 分层明确

SCX WebSocket 分成三层：

```text
WebSocketProtocolFrameHelper 只做 RFC6455 二进制结构读写
ScxFrameWebSocket            做单帧协议合法性校验
ScxWebSocket                 做消息聚合、ping/pong、close 状态
ScxEventWebSocket            做事件回调封装
```

需要注意：底层 protocol frame helper 只解析结构，不判断语义；帧级接口只保证单帧合法性，不做状态管理；消息级接口才处理分片和 close 逻辑。

### 2. close() 是资源级终止

`close()` 不是 WebSocket 协议层的 closing handshake，而是立即关闭底层连接。优雅关闭需要先 `sendClose(...)`，再 `close()`。

### 3. 消息级 API 会自动处理控制帧

`ScxWebSocket#read()` 读到 `PING` 会尝试回复 payload 相同的 `PONG`；读到 `CLOSE` 会记录 close received，并尝试回复 `NORMAL_CLOSE`。

### 4. 发送 close 后禁止普通消息

`ScxWebSocket` 发送 close 后，只允许再次发送 close，普通 `TEXT`、`BINARY`、`PING`、`PONG` 都会触发状态异常。收到 close 后，也不允许再发送非 close 消息。

### 5. 当前库不提供完整 HTTP Server 集成

`scx-websocket` 处理的是 WebSocket 协议帧和消息，不负责 HTTP 服务、路由、握手计算或 Upgrade 请求绑定。HTTP Upgrade 需要由上层模块完成，再把升级后的字节流包装为 `ByteEndpoint`。

## 常见问题

### `isClient` 应该传什么？

当前端是 WebSocket 客户端时传 `true`；当前端是 WebSocket 服务端时传 `false`。这个值会决定发送帧是否加掩码，以及接收帧时如何校验对端掩码规则。

### `close()` 会发送 close frame 吗？

不会。`close()` 是资源级终止，会直接关闭底层连接，不发送任何 WebSocket 帧。需要优雅关闭时，先调用 `sendClose()`，再调用 `close()`。

### `ScxFrameWebSocket` 和 `ScxWebSocket` 该用哪个？

普通业务建议使用 `ScxWebSocket`，它返回完整消息，并自动处理分片、ping/pong 和 close。只有需要直接操作 WebSocket frame 时，才使用 `ScxFrameWebSocket`。

### `ScxEventWebSocket#start()` 会阻塞吗？

会。接口注释明确说明，`start()` 需要在回调设置完成后调用，并且是阻塞方法。需要异步运行时，把它放到独立线程或虚拟线程中。

### 是否支持浏览器 WebSocket？

协议层支持，但你还需要上层 HTTP 服务完成 Upgrade 握手，并把升级后的字节流包装成 `ByteEndpoint`。当前库不提供握手辅助方法或完整 HTTP Server 集成。

### 默认大小限制是多少？

`ScxFrameWebSocket` 默认最大接收帧大小是 16 MB；`ScxWebSocket` 默认分片消息聚合上限是 64 MB。`maxMessageSize` 只在分片聚合时检查，两个限制都只约束接收端。
