# SCX WebSocket

SCX WebSocket 是一个轻量的 WebSocket 协议处理库。

它提供帧级 WebSocket、消息级 WebSocket、事件式 WebSocket、关闭信息、握手辅助方法和协议异常等基础能力。它不负责启动 HTTP 服务，也不负责完整的 HTTP Upgrade 流程；它更像一个基于 `ByteEndpoint` 的 WebSocket 协议层工具。当前版本为 `0.4.0`，依赖 `scx-io`、`scx-random` 和 `scx-digest`。([GitHub][1])

## 安装

### Maven

```xml id="d3v2y4"
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-websocket</artifactId>
    <version>0.4.0</version>
</dependency>
```

## 基本概念

SCX WebSocket 中最常用的概念包括：

```text id="80h6mp"
ScxFrameWebSocket       帧级 WebSocket 接口
ScxWebSocket            消息级 WebSocket 接口
ScxEventWebSocket       事件式 WebSocket 接口
WebSocketFrame          WebSocket 帧
WebSocketMessage        WebSocket 消息
WebSocketOpCode         帧操作码
WebSocketMessageType    消息类型
ScxWebSocketCloseInfo   关闭信息
WebSocketHandshakeHelper 握手辅助工具
WebSocketIOException    底层 IO 异常
WebSocketProtocolException 协议异常
WebSocketInvalidStateException 状态异常
```

库中有三个使用层次：

```text id="jl2e6u"
ScxFrameWebSocket   处理单个 WebSocket frame
ScxWebSocket        处理完整 WebSocket message，会自动处理分片、ping/pong、close
ScxEventWebSocket   在 ScxWebSocket 之上提供 onText/onBinary/onClose 事件风格 API
```

`ScxFrameWebSocket` 是帧级接口，只保证单帧读写的协议合法性，不做消息重组和连接状态管理；`ScxWebSocket` 是消息级接口，会组合分片、分帧发送，并处理 close 握手逻辑。([GitHub][2]) ([GitHub][3])

## 快速开始

下面示例假设你已经有一个 `ByteEndpoint`。`ByteEndpoint` 来自 `scx-io`，代表一组字节输入 / 输出端点。

### 消息级 WebSocket

```java id="zgavk1"
import dev.scx.io.endpoint.ByteEndpoint;
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.WebSocketMessageType;

import java.nio.charset.StandardCharsets;

public class WebSocketExample {

    public void run(ByteEndpoint endpoint) {
        // true 表示当前端是 client；false 表示当前端是 server
        ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true);

        webSocket.send("hello");

        while (true) {
            var message = webSocket.read();

            if (message.type() == WebSocketMessageType.TEXT) {
                var text = new String(message.payloadData(), StandardCharsets.UTF_8);
                System.out.println(text);
            }

            if (message.type() == WebSocketMessageType.CLOSE) {
                break;
            }
        }

        webSocket.close();
    }

}
```

`ScxWebSocket.of(endpoint, isClient)` 会基于 `ByteEndpoint` 创建帧级 WebSocket，再包装为消息级 WebSocket；默认最大消息大小是 64 MB。([GitHub][3])

## ByteEndpoint

SCX WebSocket 不直接依赖 `Socket`、`Channel` 或某个 HTTP Server。它通过 `ByteEndpoint` 读写字节。

测试代码中用普通 `Socket` 包装了一个最小 `ByteEndpoint`：([GitHub][4])

```java id="quu0hh"
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

```java id="hbh82v"
ScxWebSocket clientWebSocket = ScxWebSocket.of(endpoint, true);

ScxWebSocket serverWebSocket = ScxWebSocket.of(endpoint, false);
```

这个参数会影响 WebSocket 掩码规则。客户端发送帧时需要加掩码；服务端发送帧时不加掩码。读取时也会按当前端角色校验对端发来的帧是否符合掩码规则。([GitHub][5]) ([GitHub][6])

## 消息级 API：ScxWebSocket

`ScxWebSocket` 是推荐的常规使用入口。

创建：

```java id="h7qz72"
ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true);
```

指定最大消息大小：

```java id="ml1jz5"
ScxWebSocket webSocket = ScxWebSocket.of(endpoint, true, 1024 * 1024);
```

从已有帧级 WebSocket 创建：

```java id="9cwq94"
ScxFrameWebSocket frameWebSocket = ScxFrameWebSocket.of(endpoint, true);

ScxWebSocket webSocket = ScxWebSocket.of(frameWebSocket);
```

`ScxWebSocket` 提供：

```java id="bijlhy"
WebSocketMessage read();

void send(WebSocketMessage message);

void send(String textMessage);

void send(byte[] binaryMessage);

void sendPing(byte[] data);

void sendPong(byte[] data);

void sendClose(ScxWebSocketCloseInfo closeInfo);

void sendClose(int code, String reason);

void sendClose();

void close();
```

这些方法都定义在 `ScxWebSocket` 接口中。([GitHub][3])

### 发送文本消息

```java id="rnomss"
webSocket.send("hello websocket");
```

`send(String)` 会把字符串按 UTF-8 编码成 `TEXT` 消息。([GitHub][3])

### 发送二进制消息

```java id="15g953"
webSocket.send(new byte[]{1, 2, 3});
```

### 读取消息

```java id="kreg88"
var message = webSocket.read();

switch (message.type()) {
    case TEXT -> {
        var text = new String(message.payloadData(), StandardCharsets.UTF_8);
        System.out.println(text);
    }
    case BINARY -> {
        byte[] data = message.payloadData();
    }
    case CLOSE -> {
        webSocket.close();
    }
    case PING, PONG -> {
        // 可按需处理
    }
}
```

`WebSocketMessage` 是一个 record，包含 `WebSocketMessageType type` 和 `byte[] payloadData`，并要求两者都不能为 `null`。([GitHub][7])

### 消息类型

```java id="76qrhm"
TEXT
BINARY
CLOSE
PING
PONG
```

这些类型定义在 `WebSocketMessageType` 中。([GitHub][8])

## Ping / Pong

发送 ping：

```java id="9oqhtc"
webSocket.sendPing("ping".getBytes(StandardCharsets.UTF_8));
```

发送 pong：

```java id="51jmt7"
webSocket.sendPong("pong".getBytes(StandardCharsets.UTF_8));
```

消息级 `ScxWebSocket` 在读取到 `PING` 帧时，会立即回一个 `PONG`，payload 与 ping 相同。([GitHub][9])

```java id="ejqv2f"
var message = webSocket.read();

if (message.type() == WebSocketMessageType.PING) {
    // ScxWebSocket 内部已经自动回了 PONG
}
```

## Close

### 正常关闭

```java id="6tym61"
webSocket.sendClose();
webSocket.close();
```

`sendClose()` 默认发送 `NORMAL_CLOSE`。([GitHub][3])

### 指定关闭码和原因

```java id="wil7dr"
webSocket.sendClose(1000, "normal close");
```

或者：

```java id="a62mik"
import dev.scx.websocket.close_info.ScxWebSocketCloseInfo;

webSocket.sendClose(
    ScxWebSocketCloseInfo.of(1000, "normal close")
);
```

`ScxWebSocketCloseInfo` 支持 `of(code)`、`of(code, reason)`、`empty()`、`ofPayload(payload)` 和 `toPayload()`。([GitHub][10])

### 关闭码常量

`WebSocketCloseInfo` 提供常见关闭信息：

```java id="wdukfk"
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

这些枚举包含关闭码和默认 reason。([GitHub][11])

### close() 的含义

`close()` 是资源级关闭，不等同于发送 WebSocket CLOSE 帧。它会立即关闭底层连接并释放资源，不发送任何 WebSocket 帧，也不等待对端响应。若需要按协议优雅关闭，应先发送 CLOSE 帧，再调用 `close()` 释放资源。([GitHub][2])

推荐写法：

```java id="7kg37s"
try {
    webSocket.sendClose();
} finally {
    webSocket.close();
}
```

## 关闭信息 payload

关闭信息可以和 payload 互转：

```java id="piqzl9"
var closeInfo = ScxWebSocketCloseInfo.of(1000, "done");

byte[] payload = closeInfo.toPayload();

var parsed = ScxWebSocketCloseInfo.ofPayload(payload);
```

测试代码中也验证了 `ScxWebSocketCloseInfo.of(...).toPayload()` 和 `ofPayload(...)` 可以互相还原。([GitHub][12])

关闭 payload 规则：

```text id="6htnm4"
空 payload        -> empty close info
长度小于 2 的 payload -> 非法
前 2 字节         -> close code
剩余字节          -> UTF-8 reason
reason 最大长度   -> 123 字节
```

这些规则由 `ScxWebSocketCloseInfoHelper` 实现。([GitHub][13])

## 事件式 API：ScxEventWebSocket

如果你更喜欢事件回调风格，可以使用 `ScxEventWebSocket`。

```java id="0ob6n1"
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

`ScxEventWebSocket` 本质上是 `ScxWebSocket` 的另一种使用风格，支持 `onText`、`onBinary`、`onPing`、`onPong`、`onClose`、`onError`、`send(...)`、`start()` 和 `close()`。`start()` 是阻塞方法，调用后会持续读取消息并触发回调。([GitHub][14])

### 在独立线程中启动

```java id="62ufg8"
Thread.ofVirtual().start(eventWebSocket::start);

eventWebSocket.send("hello");
```

测试代码中就是把 `ScxEventWebSocket#start` 放到虚拟线程中运行，然后继续发送消息。([GitHub][15])

### 使用回调执行器

```java id="qa5nuw"
Executor executor = command -> Thread.ofVirtual().start(command);

ScxEventWebSocket eventWebSocket = ScxEventWebSocket.of(
    messageWebSocket,
    executor
);
```

`ScxEventWebSocket.of(messageWebSocket, callbackExecutor)` 可以传入回调执行器，避免回调之间互相阻塞。([GitHub][14])

### 事件式 close 行为

`ScxEventWebSocket#close()` 会关闭底层 message WebSocket，从而打断 `start()` 中阻塞的读取流程，并通过异常路径间接触发 `onClose`。实现中收到 CLOSE 消息时，也会调用 `onClose`、关闭连接并停止循环。([GitHub][16])

## 帧级 API：ScxFrameWebSocket

`ScxFrameWebSocket` 是更底层的帧级接口。通常只有在你需要直接控制 WebSocket frame 时才使用它。

```java id="3ro4xe"
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

`ScxFrameWebSocket` 只负责单帧协议合法性，不做状态管理，不做消息分片重组，也不管理 close / ping / pong 的连接状态。([GitHub][2])

### 指定最大帧大小

```java id="tpdzdh"
ScxFrameWebSocket frameWebSocket = ScxFrameWebSocket.of(
    endpoint,
    true,
    1024 * 1024
);
```

默认最大帧大小是 16 MB，并且只约束接收帧。([GitHub][2])

### WebSocketFrame

```java id="xipxn4"
public record WebSocketFrame(
    WebSocketOpCode opCode,
    byte[] payloadData,
    boolean fin
) {
}
```

`WebSocketFrame` 要求 `opCode` 和 `payloadData` 都不能为 `null`。([GitHub][17])

### WebSocketOpCode

```java id="cq10je"
CONTINUATION
TEXT
BINARY
CLOSE
PING
PONG
```

每个 `WebSocketOpCode` 都有对应的整数 code，并提供 `of(code)` 和 `find(code)` 方法。`of(code)` 找不到时抛出异常，`find(code)` 找不到时返回 `null`。([GitHub][18])

## 分片消息

消息级 `ScxWebSocket` 会自动处理分片消息。

当读取到非 final 的 `TEXT` 或 `BINARY` 帧时，它会进入分片聚合状态；后续读取 `CONTINUATION` 帧，直到 final continuation 到达后合并成一个完整消息。聚合后的消息类型使用起始帧的 opcode。([GitHub][9])

```java id="adcg63"
// 对调用方来说，read() 返回的是完整 message
WebSocketMessage message = webSocket.read();
```

非法分片会抛出 `WebSocketProtocolException`，例如：

```text id="o1rvsp"
分片过程中又收到新的 TEXT / BINARY 帧
没有分片进行中却收到 CONTINUATION 帧
聚合后的消息超过 maxMessageSize
```

这些规则由 `ScxWebSocketImpl#readFrameUntilLast()` 和 `appendFragment(...)` 实现。([GitHub][9])

## 最大大小限制

有两个大小限制：

```text id="6nc1cu"
ScxFrameWebSocket 最大帧大小     默认 16 MB，只约束接收帧
ScxWebSocket 最大消息大小        默认 64 MB，只约束接收消息聚合后大小
```

`ScxFrameWebSocket` 在读取协议帧头后检查 payloadLength 是否超过 `maxWebSocketFrameSize`；`ScxWebSocket` 在分片聚合时检查总 payload 是否超过 `maxMessageSize`。([GitHub][5]) ([GitHub][9])

## 协议校验

SCX WebSocket 会做基础 WebSocket 协议校验。

帧级校验包括：

```text id="g9w2l3"
opcode 必须合法
rsv1 / rsv2 / rsv3 必须为 false
客户端和服务端 mask 规则必须正确
控制帧必须 fin = true
控制帧 payload 长度必须 <= 125
close frame payload 长度不能为 1
```

这些校验由 `ScxFrameWebSocketImplHelper#fromProtocolFrame(...)` 和 `toProtocolFrame(...)` 完成。([GitHub][6])

底层 `WebSocketProtocolFrameHelper` 只负责二进制结构的读写，不做语义校验、不做掩码处理、不管理状态、不做分片重组；它不是直接面向业务使用的安全 WebSocket 实现。([GitHub][19])

## 握手辅助

`WebSocketHandshakeHelper` 提供两个静态方法：

```java id="m8y0zc"
String key = WebSocketHandshakeHelper.generateSecWebSocketKey();

String accept = WebSocketHandshakeHelper.computeSecWebSocketAccept(key);
```

`generateSecWebSocketKey()` 会生成客户端握手使用的 `Sec-WebSocket-Key`；`computeSecWebSocketAccept(...)` 会根据 `Sec-WebSocket-Key` 计算服务端响应中的 `Sec-WebSocket-Accept`。([GitHub][20])

注意：当前 `scx-websocket` 只提供握手辅助方法，并没有提供完整的 HTTP Upgrade 客户端或服务端封装。仓库的 `handshake` 包中只有 `WebSocketHandshakeHelper`。([GitHub][21])

服务端握手响应示意：

```java id="c7ftpl"
String accept = WebSocketHandshakeHelper.computeSecWebSocketAccept(secWebSocketKey);

// HTTP/1.1 101 Switching Protocols
// Upgrade: websocket
// Connection: Upgrade
// Sec-WebSocket-Accept: <accept>
```

真正的 HTTP 请求读取、响应写出和连接升级，需要由上层 HTTP 模块或业务代码处理。

## 异常

### WebSocketIOException

`WebSocketIOException` 表示底层 I/O 已不可继续使用，例如对端关闭连接、本地输入 / 输出已关闭、网络或传输错误。出现该异常后，连接生命周期结束，调用 `close()` 释放资源是安全且幂等的。([GitHub][22])

```java id="tznb75"
try {
    var message = webSocket.read();
} catch (WebSocketIOException e) {
    webSocket.close();
}
```

### WebSocketProtocolException

`WebSocketProtocolException` 表示协议异常，例如帧过大、掩码错误、非法分片等。它包含一个 `closeCode()`，可以用于给对端发送 close frame。([GitHub][23])

```java id="zw4b83"
try {
    webSocket.read();
} catch (WebSocketProtocolException e) {
    webSocket.sendClose(e.closeCode(), e.getMessage());
    webSocket.close();
}
```

### WebSocketInvalidStateException

`WebSocketInvalidStateException` 表示当前连接状态不允许执行某个操作，例如已经发送 close frame 后继续发送普通消息。([GitHub][24])

```java id="zgodqj"
webSocket.sendClose();

// 之后再发送普通消息会抛出 WebSocketInvalidStateException
webSocket.send("after close");
```

`ScxWebSocketImpl#send(...)` 中也明确检查了：发送过 close 后，不允许再发送非 close 消息；收到 close 后，也不允许再发送非 close 消息。([GitHub][9])

## 完整示例：Socket + 消息级 WebSocket

下面示例演示一个非常简化的裸 socket WebSocket 通信。实际浏览器 WebSocket 场景还需要先完成 HTTP Upgrade 握手。

```java id="ufpcna"
import dev.scx.io.ByteInput;
import dev.scx.io.ByteOutput;
import dev.scx.io.ScxIO;
import dev.scx.io.endpoint.ByteEndpoint;
import dev.scx.websocket.ScxWebSocket;
import dev.scx.websocket.WebSocketMessageType;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

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

                for (int i = 0; i < 10; i++) {
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

            if (message.type() == WebSocketMessageType.CLOSE) {
                System.out.println("closed");
                break;
            }

            if (message.type() == WebSocketMessageType.TEXT) {
                System.out.println(new String(message.payloadData(), StandardCharsets.UTF_8));
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

测试代码中也展示了用 `ServerSocket` / `Socket` 创建 `ByteEndpoint`，再使用 `ScxFrameWebSocket`、`ScxWebSocket` 和 `ScxEventWebSocket` 的方式。([GitHub][4])

## 完整示例：事件式 WebSocket

```java id="vt69qd"
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

`ScxEventWebSocketImpl#start()` 会循环调用 `messageWebSocket.read()`，根据消息类型调用对应回调；如果发生异常，会先调用 `onError`，再根据异常类型构造 close info，调用 `onClose`，尝试发送 close frame，并终止连接。([GitHub][16])

## 设计说明

### 1. 分层明确

SCX WebSocket 分成三层：

```text id="q1xvwq"
WebSocketProtocolFrameHelper 只做 RFC6455 二进制结构读写
ScxFrameWebSocket            做单帧协议合法性校验
ScxWebSocket                 做消息聚合、ping/pong、close 状态
ScxEventWebSocket            做事件回调封装
```

源码注释也强调：底层 protocol frame helper 只解析结构，不判断语义；帧级接口只保证单帧合法性，不做状态管理；消息级接口才处理分片和 close 逻辑。([GitHub][19])

### 2. close() 是资源级终止

`close()` 不是 WebSocket 协议层的 closing handshake，而是立即关闭底层连接。优雅关闭需要先 `sendClose(...)`，再 `close()`。([GitHub][2])

### 3. 消息级 API 会自动处理控制帧

`ScxWebSocket#read()` 读到 `PING` 会自动回 `PONG`；读到 `CLOSE` 会记录 close received 并尝试回 close。([GitHub][9])

### 4. 发送 close 后禁止普通消息

`ScxWebSocket` 发送 close 后，只允许再次发送 close，普通 `TEXT`、`BINARY`、`PING`、`PONG` 都会触发状态异常。收到 close 后，也不允许再发送非 close 消息。([GitHub][9])

### 5. 当前库不提供完整 HTTP Server 集成

`scx-websocket` 处理的是 WebSocket 协议帧和消息，不负责 HTTP 服务、路由或 Upgrade 请求绑定。握手包中只有 `WebSocketHandshakeHelper`，它只生成 key 和计算 accept。([GitHub][21])

## 常见问题

### `isClient` 应该传什么？

当前端是 WebSocket 客户端时传 `true`；当前端是 WebSocket 服务端时传 `false`。这个值会决定发送帧是否加掩码，以及接收帧时如何校验对端掩码规则。([GitHub][5])

### `close()` 会发送 close frame 吗？

不会。`close()` 是资源级终止，会直接关闭底层连接，不发送任何 WebSocket 帧。需要优雅关闭时，先调用 `sendClose()`，再调用 `close()`。([GitHub][2])

### `ScxFrameWebSocket` 和 `ScxWebSocket` 该用哪个？

普通业务建议使用 `ScxWebSocket`，它返回完整消息，并自动处理分片、ping/pong 和 close。只有需要直接操作 WebSocket frame 时，才使用 `ScxFrameWebSocket`。([GitHub][2])

### `ScxEventWebSocket#start()` 会阻塞吗？

会。接口注释明确说明，`start()` 需要在回调设置完成后调用，并且是阻塞方法。需要异步运行时，把它放到独立线程或虚拟线程中。([GitHub][14])

### 是否支持浏览器 WebSocket？

协议层支持，但你还需要上层 HTTP 服务完成 Upgrade 握手，并把升级后的字节流包装成 `ByteEndpoint`。当前库只提供 `WebSocketHandshakeHelper`，不提供完整 HTTP Server 集成。([GitHub][21])

### 默认最大消息大小是多少？

`ScxWebSocket` 默认最大消息大小是 64 MB；`ScxFrameWebSocket` 默认最大帧大小是 16 MB。它们都只约束接收端。([GitHub][3])

[1]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/ScxFrameWebSocket.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/ScxWebSocket.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/test/java/dev/scx/websocket/test/FrameWebSocketTest.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/ScxFrameWebSocketImpl.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/ScxFrameWebSocketImplHelper.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/WebSocketMessage.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/WebSocketMessageType.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/ScxWebSocketImpl.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/close_info/ScxWebSocketCloseInfo.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/close_info/WebSocketCloseInfo.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/test/java/dev/scx/websocket/test/WebSocketCloseInfoTest.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/close_info/ScxWebSocketCloseInfoHelper.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/event/ScxEventWebSocket.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/test/java/dev/scx/websocket/test/EventWebSocketTest.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/event/ScxEventWebSocketImpl.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/WebSocketFrame.java "raw.githubusercontent.com"
[18]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/WebSocketOpCode.java "raw.githubusercontent.com"
[19]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/frame/WebSocketProtocolFrameHelper.java "raw.githubusercontent.com"
[20]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/handshake/WebSocketHandshakeHelper.java "raw.githubusercontent.com"
[21]: https://github.com/scx-projects/scx-websocket/tree/master/src/main/java/dev/scx/websocket/handshake "scx-websocket/src/main/java/dev/scx/websocket/handshake at master · scx-projects/scx-websocket · GitHub"
[22]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/exception/WebSocketIOException.java "raw.githubusercontent.com"
[23]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/exception/WebSocketProtocolException.java "raw.githubusercontent.com"
[24]: https://raw.githubusercontent.com/scx-projects/scx-websocket/master/src/main/java/dev/scx/websocket/exception/WebSocketInvalidStateException.java "raw.githubusercontent.com"
