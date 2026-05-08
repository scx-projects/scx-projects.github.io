# SCX TCP

SCX TCP 是一个极简 TCP 抽象库。

它提供了非常薄的一层 TCP Server 和 TCP Client 封装，用来简化 `ServerSocket`、`Socket` 的创建、监听、连接和回调处理。

SCX TCP 本身不是网络框架，也不是协议框架。它不理解 HTTP、WebSocket、MQTT 或其它任何应用层协议。

它只负责：

```text
TCP Server 绑定端口
TCP Server 接受连接
TCP Server 将 Socket 交给用户处理器
TCP Client 连接远程地址
少量 TLS 辅助能力
```

当前版本为 `0.2.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-tcp</artifactId>
    <version>0.2.0</version>
</dependency>
```

SCX TCP 依赖 `scx-function`。

这个依赖主要用于 `onConnect(...)` 回调，使连接处理器可以直接抛出受检异常，而不需要强行包装成 `RuntimeException`。

## 基本概念

SCX TCP 中最核心的概念包括：

```text
ScxTCPServer       TCP Server 抽象接口
TCPServer          ScxTCPServer 的默认实现
TCPServerOptions   TCP Server 构造选项
ScxTCPClient       TCP Client 抽象接口
TCPClient          ScxTCPClient 的默认实现
TLS                TLS 辅助接口
TLSHelper          SSLContext 创建工具
```

它们之间的关系可以简单理解为：

```text
TCPServer
    ↓
绑定 ServerSocket
    ↓
循环 accept()
    ↓
每个 Socket 交给 onConnect 处理器
    ↓
每个连接使用一个虚拟线程处理
```

客户端部分更简单：

```text
TCPClient
    ↓
new Socket()
    ↓
socket.connect(endpoint, timeout)
    ↓
返回 Socket
```

也就是说：

```text
TCPServer 负责接收连接
TCPClient 负责建立连接
Socket 的读写和关闭由用户自己负责
```

## 快速开始

### 创建 TCP Server

```java
import dev.scx.tcp.TCPServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;

var tcpServer = new TCPServer();

tcpServer.onConnect(socket -> {
    System.out.println("客户端连接了: " + socket.getRemoteSocketAddress());

    try (socket) {
        var reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream())
        );

        while (true) {
            var line = reader.readLine();

            if (line == null) {
                break;
            }

            System.out.println(socket.getRemoteSocketAddress() + " : " + line);
        }
    }

    System.out.println("连接结束");
});

tcpServer.start(8899);

System.out.println("已监听端口号: " + tcpServer.localAddress().getPort());
```

这里需要注意：

1. `onConnect(...)` 必须在 `start(...)` 前设置。
2. `start(...)` 只负责开始监听。
3. 每个连接会交给 `onConnect(...)` 处理。
4. `Socket` 的读写和关闭由 `onConnect(...)` 内部自己负责。
5. `TCPServer` 不会替用户关闭已经交付出去的 `Socket`。

### 创建 TCP Client

```java
import dev.scx.tcp.TCPClient;

import java.net.InetSocketAddress;

var tcpClient = new TCPClient();

try (var socket = tcpClient.connect(new InetSocketAddress("127.0.0.1", 8899))) {
    var out = socket.getOutputStream();

    out.write("hello\r\n".getBytes());
    out.write("world\r\n".getBytes());
}
```

带连接超时：

```java
var socket = tcpClient.connect(
    new InetSocketAddress("127.0.0.1", 8899),
    3000
);
```

超时时间单位来自 JDK `Socket#connect(SocketAddress, int)`，也就是毫秒。

## ScxTCPServer

`ScxTCPServer` 是 TCP Server 抽象接口。

接口可以理解为：

```java
public interface ScxTCPServer {

    ScxTCPServer onConnect(Function1Void connectHandler);

    void start(SocketAddress localAddress) throws IOException;

    void stop();

    InetSocketAddress localAddress();

    default void start(int port) throws IOException {
        start(new InetSocketAddress(port));
    }

}
```

它的职责非常明确：

```text
绑定端口
等待 TCP 连接
连接建立后，把 Socket 交给用户处理器
```

它不负责：

```text
连接池
连接状态管理
连接数量统计
连接超时控制
协议解析
协议防护
流量控制
应用层调度
优雅关闭已建立连接
```

## TCPServer

`TCPServer` 是 `ScxTCPServer` 的默认实现。

创建方式：

```java
var server = new TCPServer();
```

带选项：

```java
var options = new TCPServerOptions()
    .backlog(256);

var server = new TCPServer(options);
```

内部使用的是 JDK `ServerSocket`。

启动时大致流程是：

```text
1. 检查 server 是否已经运行
2. 检查是否设置了 connectHandler
3. 创建 ServerSocket
4. bind(localAddress, backlog)
5. 标记 running = true
6. 创建监听线程
7. 监听线程循环 accept()
8. 每个连接交给一个虚拟线程处理
```

## onConnect

`onConnect(...)` 用于设置连接处理器。

```java
server.onConnect(socket -> {
    // 处理 socket
});
```

连接处理器接收的是 JDK 原生 `Socket`。

```java
Socket
```

SCX TCP 不包装这个 `Socket`，也不替它定义额外协议。

因此你可以直接使用：

```java
socket.getInputStream()
socket.getOutputStream()
socket.getRemoteSocketAddress()
socket.getLocalSocketAddress()
socket.close()
```

示例：

```java
server.onConnect(socket -> {
    try (socket) {
        var input = socket.getInputStream();
        var output = socket.getOutputStream();

        var bytes = input.readNBytes(1024);

        output.write(bytes);
    }
});
```

## connectHandler 可以抛异常

`onConnect(...)` 接收的是 `Function1Void`。

因此连接处理器可以直接抛出受检异常。

```java
server.onConnect(socket -> {
    try (socket) {
        var input = socket.getInputStream();

        if (input.read() == -1) {
            throw new IOException("客户端已关闭");
        }
    }
});
```

不需要写成：

```java
throw new RuntimeException(e);
```

这也是 `scx-function` 在这里的作用。

## connectHandler 异常处理

如果 `connectHandler` 抛出异常，`TCPServer` 不会吞掉异常，也不会额外提供 `onError` 回调。

当前实现会把异常显式移交给当前线程的 `UncaughtExceptionHandler`。

也就是说，语义上尽量接近：

```text
这个连接处理线程自然失败
```

这样做的目的有几个：

1. 不在 TCP 接受器这一层解释业务异常。
2. 不引入第二条异常处理路径。
3. 不替用户记录、包装或吞掉连接处理异常。
4. 保留最原始的异常信息。
5. 让用户可以在 `onConnect(...)` 内部自行决定是否捕获异常。

如果需要自定义异常处理，可以在 `onConnect(...)` 内部自己写 `try/catch`。

```java
server.onConnect(socket -> {
    try (socket) {
        handle(socket);
    } catch (Throwable e) {
        logError(socket, e);
    }
});
```

## Socket 生命周期

`TCPServer` 把 `Socket` 交给 `onConnect(...)` 后，就不再管理这个连接。

因此推荐在处理器内部使用 `try-with-resources`：

```java
server.onConnect(socket -> {
    try (socket) {
        // 读写 socket
    }
});
```

如果不关闭 `Socket`，连接可能一直占用系统资源。

需要特别注意：

```text
TCPServer.stop() 不会关闭已经交给 connectHandler 的 Socket
```

因为已经建立的连接属于用户处理器。

TCP Server 层无法知道什么叫“优雅关闭连接”。

这个语义只能由协议层或业务层定义。

## start

`start(...)` 用于启动 TCP Server。

按端口启动：

```java
server.start(8899);
```

按地址启动：

```java
server.start(new InetSocketAddress("127.0.0.1", 8899));
```

也可以绑定到任意可用端口：

```java
server.start(0);

int port = server.localAddress().getPort();
```

如果 server 已经运行，再次调用 `start(...)` 会抛出：

```java
IllegalStateException
```

如果没有设置 `connectHandler`，调用 `start(...)` 也会抛出：

```java
IllegalStateException
```

## localAddress

`localAddress()` 返回当前 server 绑定的本地地址。

```java
InetSocketAddress address = server.localAddress();

System.out.println(address.getHostString());
System.out.println(address.getPort());
```

常见用途是获取自动分配端口：

```java
server.start(0);

int port = server.localAddress().getPort();
```

如果 server 没有启动，调用 `localAddress()` 会抛出：

```java
IllegalStateException
```

## stop

`stop()` 用于停止 TCP Server。

```java
server.stop();
```

它的语义非常明确：

```text
停止接受新的 TCP 连接
不干涉已经建立的连接
```

当前实现大致会：

```text
1. running = false
2. 关闭 ServerSocket
3. accept() 因 ServerSocket 关闭而退出
4. 等待监听线程结束
5. 清空 serverSocket 和 listenThread
```

如果 server 本来没有运行，调用 `stop()` 会直接返回。

```java
server.stop();
server.stop();
```

第二次调用不会做任何事情。

## stop 不等于关闭所有连接

`stop()` 只停止监听。

它不会关闭已经交付给 `onConnect(...)` 的连接。

例如：

```java
server.onConnect(socket -> {
    try (socket) {
        Thread.sleep(60_000);
    }
});
```

如果一个连接已经进入处理器，此时调用：

```java
server.stop();
```

这个连接处理器仍然会继续运行，直到它自己结束。

这是刻意设计。

因为已经建立的连接属于协议层或业务层，TCP 接受器不应该越权干涉。

## TCPServerOptions

`TCPServerOptions` 承载 TCP Server 构造期的少量配置。

当前唯一选项是：

```text
backlog
```

创建：

```java
var options = new TCPServerOptions();
```

设置 backlog：

```java
var options = new TCPServerOptions()
    .backlog(256);
```

默认值是：

```text
128
```

复制构造：

```java
var copy = new TCPServerOptions(options);
```

## backlog

`backlog` 会传给：

```java
ServerSocket#bind(SocketAddress endpoint, int backlog)
```

它表示服务端 socket 的 listen 队列长度提示。

示例：

```java
var server = new TCPServer(
    new TCPServerOptions().backlog(512)
);
```

需要注意：

1. `backlog` 的实际行为由操作系统决定。
2. 它不是最大连接数。
3. 它不是并发限制。
4. 它不是防护机制。
5. 它只影响底层监听队列。

如果需要连接数限制、限流或协议级超时，应在更高层实现。

## 并发模型

`TCPServer` 的并发模型非常简单：

```text
一个监听线程
多个连接处理虚拟线程
```

监听线程：

```text
Thread.ofPlatform()
```

每个连接处理线程：

```text
Thread.ofVirtual()
```

也就是说：

```text
监听 accept 使用平台线程
处理连接使用虚拟线程
```

每次成功 `accept()` 后，都会创建一个新的虚拟线程执行 `connectHandler`。

```text
accept()
    ↓
Thread.ofVirtual().start(() -> handle(socket))
```

## 为什么每个连接一个虚拟线程

SCX TCP 的定位是极简 TCP 接受器。

使用虚拟线程可以让每个连接仍然写成阻塞式代码。

例如：

```java
server.onConnect(socket -> {
    try (socket) {
        var reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream())
        );

        while (true) {
            var line = reader.readLine();

            if (line == null) {
                break;
            }

            handleLine(line);
        }
    }
});
```

代码保持同步、直接、易读。

SCX TCP 不提供事件循环，也不提供异步回调模型。

## 虚拟线程不是受管资源

SCX TCP 不跟踪虚拟线程。

它不会统计：

```text
当前连接数
当前虚拟线程数
已完成连接数
失败连接数
```

也不会主动中断虚拟线程。

因此如果你需要这些能力，应在 `onConnect(...)` 内部自己实现。

示例：

```java
var activeConnections = new AtomicInteger();

server.onConnect(socket -> {
    activeConnections.incrementAndGet();

    try (socket) {
        handle(socket);
    } finally {
        activeConnections.decrementAndGet();
    }
});
```

## 为什么没有 maxConcurrentConnections

SCX TCP 刻意不提供 `maxConcurrentConnections`。

原因是：

```text
TCPServer 能干涉的连接已经是三次握手成功后的连接
```

它无法防止半连接攻击。

真正的 TCP 层防护通常属于：

```text
操作系统
内核参数
防火墙
负载均衡
云厂商安全服务
```

协议层防护则属于：

```text
HTTP 层
WebSocket 层
自定义协议层
业务层
```

例如 Slowloris 这类攻击，需要在协议层或业务层处理超时和读取策略。

TCP 接受器本身不应该理解这些协议语义。

## 为什么没有连接超时管理

SCX TCP 不提供连接级读取超时或写入超时管理。

因为 TCP Server 只负责把 `Socket` 交给用户。

如果需要读取超时，可以在处理器中使用：

```java
socket.setSoTimeout(10_000);
```

示例：

```java
server.onConnect(socket -> {
    try (socket) {
        socket.setSoTimeout(10_000);

        var reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream())
        );

        var line = reader.readLine();

        if (line != null) {
            handleLine(line);
        }
    }
});
```

连接超时、读取超时、协议超时都应该由使用者根据协议需要自行设置。

## 为什么没有 onError

`ScxTCPServer` 没有 `onError` 回调。

TCP Server 层面的异常大致分两类。

第一类是系统异常。

例如：

```text
accept 异常
ServerSocket 被强制关闭
系统资源耗尽
```

这类异常通常没有足够上下文让用户处理。

当前实现会记录日志并关闭监听。

第二类是用户处理器异常。

例如：

```text
onConnect 中读取失败
业务处理失败
协议解析失败
```

这些异常最适合由用户在 `onConnect(...)` 内部处理。

如果额外提供 `onError`，反而容易让异常处理路径变复杂。

因此当前设计是：

```text
系统异常内部记录
处理器异常移交 UncaughtExceptionHandler
自定义处理请在 onConnect 内部完成
```

## ScxTCPClient

`ScxTCPClient` 是 TCP Client 抽象接口。

接口可以理解为：

```java
public interface ScxTCPClient {

    Socket connect(SocketAddress endpoint, int timeout) throws IOException;

    default Socket connect(SocketAddress endpoint) throws IOException {
        return connect(endpoint, 0);
    }

}
```

它只负责建立 TCP 连接。

它不负责：

```text
协议握手
请求发送
响应读取
连接池
重连
心跳
超时策略
```

这些都属于上层逻辑。

## TCPClient

`TCPClient` 是 `ScxTCPClient` 的默认实现。

实现非常直接：

```java
public final class TCPClient implements ScxTCPClient {

    @Override
    public Socket connect(SocketAddress endpoint, int timeout) throws IOException {
        var tcpSocket = new Socket();
        tcpSocket.connect(endpoint, timeout);
        return tcpSocket;
    }

}
```

也就是说，`TCPClient` 只是 `Socket#connect(...)` 的结构化封装。

## 连接远程地址

```java
var client = new TCPClient();

var socket = client.connect(
    new InetSocketAddress("127.0.0.1", 8899)
);
```

使用完后应该关闭：

```java
try (var socket = client.connect(new InetSocketAddress("127.0.0.1", 8899))) {
    var out = socket.getOutputStream();
    out.write("hello\r\n".getBytes());
}
```

带连接超时：

```java
try (var socket = client.connect(
    new InetSocketAddress("127.0.0.1", 8899),
    3000
)) {
    // 使用 socket
}
```

## Echo Server 示例

下面是一个完整 echo server。

```java
import dev.scx.tcp.TCPServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;

public class EchoServer {

    public static void main(String[] args) throws Exception {
        var server = new TCPServer();

        server.onConnect(socket -> {
            try (socket) {
                var reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
                );

                var writer = new PrintWriter(
                    socket.getOutputStream(),
                    true
                );

                while (true) {
                    var line = reader.readLine();

                    if (line == null) {
                        break;
                    }

                    writer.println("echo: " + line);
                }
            }
        });

        server.start(8899);

        System.out.println("Echo server started: " + server.localAddress());
    }

}
```

## Echo Client 示例

```java
import dev.scx.tcp.TCPClient;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.InetSocketAddress;

public class EchoClient {

    public static void main(String[] args) throws Exception {
        var client = new TCPClient();

        try (var socket = client.connect(new InetSocketAddress("127.0.0.1", 8899))) {
            var reader = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );

            var writer = new PrintWriter(
                socket.getOutputStream(),
                true
            );

            writer.println("hello");

            var response = reader.readLine();

            System.out.println(response);
        }
    }

}
```

输出：

```text
echo: hello
```

## 多客户端示例

下面示例会启动多个虚拟线程客户端，同时连接同一个 server。

```java
import dev.scx.tcp.TCPClient;

import java.net.InetSocketAddress;

for (int j = 0; j < 10; j = j + 1) {
    Thread.ofVirtual().start(() -> {
        var tcpClient = new TCPClient();

        try (var socket = tcpClient.connect(new InetSocketAddress("127.0.0.1", 8899))) {
            var out = socket.getOutputStream();

            for (int i = 0; i < 10000; i = i + 1) {
                out.write((i + "\r\n").getBytes());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
}
```

这个示例展示的是：

```text
TCPServer 每个连接一个虚拟线程
TCPClient 返回原生 Socket
具体写入逻辑由用户处理
```

## TLS

`scx-tcp` 提供了一个轻量 TLS 辅助接口。

```java
import dev.scx.tcp.tls.TLS;
```

它主要包装：

```text
SSLContext
SSLServerSocketFactory
SSLSocketFactory
```

接口可以理解为：

```java
public interface TLS {

    static TLS of(SSLContext sslContext);

    static TLS of(Path path, String password);

    static TLS ofDefault();

    static TLS ofTrustAny();

    SSLContext sslContext();

    SSLServerSocketFactory serverSocketFactory();

    SSLSocketFactory socketFactory();

    default SSLSocket upgradeToTLS(Socket tcpSocket) throws IOException;

}
```

需要注意：

```text
TCPServer 默认使用的是普通 ServerSocket
TLS 只是辅助能力，不会自动改变 TCPServer 的行为
```

如果你需要 TLS，需要在合适的位置显式使用 TLS 相关能力。

## TLS.of

如果你已经有 `SSLContext`，可以直接创建 `TLS`。

```java
SSLContext sslContext = ...;

TLS tls = TLS.of(sslContext);
```

然后可以获取：

```java
tls.sslContext();
tls.socketFactory();
tls.serverSocketFactory();
```

## TLS.of(Path, password)

可以从 keystore 文件创建 TLS。

```java
import dev.scx.tcp.tls.TLS;

import java.nio.file.Path;

var tls = TLS.of(
    Path.of("server.p12"),
    "changeit"
);
```

内部大致流程是：

```text
1. 从 path 和 password 创建 KeyStore
2. 创建 KeyManagerFactory
3. 创建 TrustManagerFactory
4. 创建 SSLContext("TLS")
5. 初始化 SSLContext
6. 返回 TLSImpl
```

这种方式适合服务端或需要指定证书的场景。

## TLS.ofDefault

`TLS.ofDefault()` 会使用系统默认信任证书创建 TLS。

```java
var tls = TLS.ofDefault();

var socketFactory = tls.socketFactory();
```

这更常用于客户端访问公开可信证书服务。

例如：

```java
try (var socket = tls.socketFactory().createSocket("example.com", 443)) {
    // 使用 SSLSocket
}
```

## TLS.ofTrustAny

`TLS.ofTrustAny()` 会创建一个忽略证书校验的 `SSLContext`。

```java
var tls = TLS.ofTrustAny();
```

它适合测试环境，例如自签名证书测试。

不推荐用于生产环境。

```text
TrustAny 会跳过服务端证书验证
生产环境使用会带来严重安全风险
```

## upgradeToTLS

`upgradeToTLS(...)` 可以把已有 `Socket` 包装为 `SSLSocket`。

```java
var tls = TLS.ofDefault();

try (var tcpSocket = new TCPClient().connect(address)) {
    var sslSocket = tls.upgradeToTLS(tcpSocket);

    sslSocket.startHandshake();

    // 使用 sslSocket 读写
}
```

需要注意：

1. `upgradeToTLS(...)` 只是把已有 socket 包装为 `SSLSocket`。
2. 是否调用 `startHandshake()` 由调用方决定。
3. client mode / server mode 等 TLS 行为仍然应由调用方根据场景配置。
4. 它不会自动让 `TCPServer` 变成 TLS Server。

## TLS Client 示例

下面是一个普通 TLS client 示例。

```java
import dev.scx.tcp.tls.TLS;

var tls = TLS.ofDefault();

try (var socket = tls.socketFactory().createSocket("example.com", 443)) {
    var sslSocket = (SSLSocket) socket;

    sslSocket.startHandshake();

    var out = sslSocket.getOutputStream();
    var in = sslSocket.getInputStream();

    out.write("""
        GET / HTTP/1.1\r
        Host: example.com\r
        Connection: close\r
        \r
        """.getBytes());

    var bytes = in.readAllBytes();

    System.out.println(new String(bytes));
}
```

如果是测试自签名证书：

```java
var tls = TLS.ofTrustAny();
```

但生产环境不应使用 `ofTrustAny()`。

## TLS ServerSocketFactory 示例

如果需要自己创建 `SSLServerSocket`，可以使用：

```java
import dev.scx.tcp.tls.TLS;

import java.nio.file.Path;

var tls = TLS.of(Path.of("server.p12"), "changeit");

try (var serverSocket = tls.serverSocketFactory().createServerSocket(8443)) {
    while (true) {
        var socket = serverSocket.accept();

        Thread.ofVirtual().start(() -> {
            try (socket) {
                // 处理 TLS socket
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    }
}
```

这段代码没有使用 `TCPServer`。

因为当前 `TCPServer` 固定使用普通 `ServerSocket`。

如果需要完整 TLS Server，可以直接使用 `TLS#serverSocketFactory()` 自己构建。

## 设计边界

SCX TCP 的设计边界非常严格。

它只做 TCP 接受器应该做的事。

```text
绑定端口
accept
交付 Socket
停止 accept
```

它不做：

```text
连接管理
协议解析
连接池
读写封装
心跳
重试
限流
统计
超时策略
异常回调
业务调度
```

这些能力应该放在更高层。

例如：

```text
scx-http-x        可以基于 scx-tcp 实现 HTTP Server
自定义协议层      可以基于 scx-tcp 处理协议帧
业务层            可以决定连接生命周期和资源管理
```

## 与 SCX HTTP X 的关系

`scx-tcp` 是更底层的 TCP 能力。

上层 HTTP 实现可以基于它构建。

可以理解为：

```text
scx-tcp
    提供 Socket accept / connect

scx-http-x
    基于 TCP Socket 实现 HTTP/1.1 读写、路由、SSE、Upgrade 等能力
```

因此，如果你需要写 HTTP 服务，通常不直接使用 `scx-tcp`。

如果你要实现自定义 TCP 协议、协议网关、简单 socket 工具，才适合直接使用 `scx-tcp`。

## 与 Socket 的关系

SCX TCP 没有隐藏 JDK `Socket`。

这是刻意设计。

因为 TCP 层最重要的对象就是：

```java
Socket
```

把它交给用户后，用户可以完整控制：

```text
输入流
输出流
超时
关闭
半关闭
TCP 参数
协议解析
错误处理
```

例如：

```java
socket.setSoTimeout(10_000);
socket.setTcpNoDelay(true);
socket.shutdownOutput();
```

这些都可以直接使用 JDK 原生 API。

## 资源管理建议

### Server

Server 应在应用关闭时调用：

```java
server.stop();
```

示例：

```java
var server = new TCPServer();

Runtime.getRuntime().addShutdownHook(new Thread(server::stop));
```

### Socket

连接处理器内部应关闭 socket：

```java
server.onConnect(socket -> {
    try (socket) {
        handle(socket);
    }
});
```

客户端也应关闭 socket：

```java
try (var socket = client.connect(address)) {
    // 使用 socket
}
```

## 错误处理建议

在简单程序中，可以让处理器异常交给默认线程异常处理器。

```java
server.onConnect(socket -> {
    try (socket) {
        handle(socket);
    }
});
```

在生产程序中，更推荐在处理器内部捕获并记录。

```java
server.onConnect(socket -> {
    try (socket) {
        handle(socket);
    } catch (Throwable e) {
        logger.log(ERROR, "连接处理失败: " + socket.getRemoteSocketAddress(), e);
    }
});
```

原因是：

```text
TCPServer 不理解业务语义
只有 onConnect 内部知道这个异常应该如何处理
```

## 完整示例：可停止的服务

```java
import dev.scx.tcp.TCPServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.concurrent.CountDownLatch;

public class StoppableServer {

    public static void main(String[] args) throws Exception {
        var server = new TCPServer();

        server.onConnect(socket -> {
            try (socket) {
                var reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
                );

                while (true) {
                    var line = reader.readLine();

                    if (line == null) {
                        break;
                    }

                    System.out.println(line);
                }
            }
        });

        server.start(8899);

        Runtime.getRuntime().addShutdownHook(new Thread(server::stop));

        System.out.println("server started at " + server.localAddress());

        new CountDownLatch(1).await();
    }

}
```

## 完整示例：带超时的协议读取

```java
import dev.scx.tcp.TCPServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;

public class TimeoutServer {

    public static void main(String[] args) throws Exception {
        var server = new TCPServer();

        server.onConnect(socket -> {
            try (socket) {
                socket.setSoTimeout(10_000);

                var reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
                );

                var line = reader.readLine();

                if (line == null) {
                    return;
                }

                System.out.println("收到: " + line);
            }
        });

        server.start(8899);
    }

}
```

这里的超时策略属于协议层或业务层，因此放在 `onConnect(...)` 内部。

## 完整示例：简单文本协议

```java
import dev.scx.tcp.TCPServer;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;

public class TextProtocolServer {

    public static void main(String[] args) throws Exception {
        var server = new TCPServer();

        server.onConnect(socket -> {
            try (socket) {
                var reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
                );

                var writer = new PrintWriter(
                    socket.getOutputStream(),
                    true
                );

                while (true) {
                    var line = reader.readLine();

                    if (line == null) {
                        break;
                    }

                    if (line.equals("PING")) {
                        writer.println("PONG");
                    } else if (line.equals("QUIT")) {
                        writer.println("BYE");
                        break;
                    } else {
                        writer.println("UNKNOWN");
                    }
                }
            }
        });

        server.start(8899);
    }

}
```

客户端：

```java
import dev.scx.tcp.TCPClient;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.InetSocketAddress;

public class TextProtocolClient {

    public static void main(String[] args) throws Exception {
        var client = new TCPClient();

        try (var socket = client.connect(new InetSocketAddress("127.0.0.1", 8899))) {
            var reader = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );

            var writer = new PrintWriter(
                socket.getOutputStream(),
                true
            );

            writer.println("PING");

            System.out.println(reader.readLine());

            writer.println("QUIT");

            System.out.println(reader.readLine());
        }
    }

}
```

## 设计说明

### 1. SCX TCP 是 TCP 接受器，不是服务器框架

`ScxTCPServer` 的职责边界严格等同于一个结构化 accept 循环。

它不是：

```text
服务器框架
连接管理器
协议框架
任务调度器
安全网关
```

它只是更方便地写：

```java
while (true) {
    var socket = serverSocket.accept();
    handle(socket);
}
```

### 2. Socket 所有权会转移给用户

一旦连接被 accept，`Socket` 就会传给 `connectHandler`。

此后，连接生命周期由用户处理。

```text
读
写
关闭
超时
异常处理
协议解析
```

都属于用户逻辑。

### 3. stop 只停止 accept

`stop()` 不关闭已建立连接。

因为 TCP 接受器无法知道已建立连接应该如何结束。

例如 HTTP、WebSocket、数据库协议、自定义协议都有不同的关闭语义。

### 4. 不提供 onError 是刻意设计

`onError` 会制造第二条异常通道。

系统异常通常无法恢复。

业务异常应该在 `onConnect(...)` 内部处理。

因此 SCX TCP 不暴露 `onError`。

### 5. 不做连接数量限制

连接数量限制不属于这个层次。

如果要防护 TCP 层攻击，应使用操作系统、防火墙或基础设施。

如果要防护协议层攻击，应在协议处理器中实现。

### 6. 阻塞式写法配合虚拟线程

SCX TCP 的默认模型是：

```text
一个连接一个虚拟线程
```

这让用户可以写阻塞式 I/O 代码，而不需要事件循环。

### 7. TLS 是辅助能力

TLS 模块提供 SSLContext 和 socket factory 的创建辅助。

但它不会自动改变 `TCPServer` 的工作方式。

普通 `TCPServer` 仍然是普通 TCP Server。

## 常见问题

### SCX TCP 是网络框架吗？

不是。

它只是 TCP Server / TCP Client 的极简封装。

### TCPServer 会解析协议吗？

不会。

它只把 `Socket` 交给 `onConnect(...)`。

协议解析由用户自己做。

### 每个连接用什么线程处理？

每个连接使用一个新的虚拟线程处理。

监听 accept 使用平台线程。

### TCPServer 会关闭客户端 Socket 吗？

不会。

`Socket` 交给 `onConnect(...)` 后，由用户负责关闭。

推荐写法：

```java
server.onConnect(socket -> {
    try (socket) {
        handle(socket);
    }
});
```

### stop 会关闭已建立连接吗？

不会。

`stop()` 只停止接受新连接。

已经交给处理器的连接继续运行。

### 可以获取监听端口吗？

可以。

```java
int port = server.localAddress().getPort();
```

这对 `start(0)` 很有用。

### start 前必须设置 onConnect 吗？

是的。

否则会抛出 `IllegalStateException`。

### 可以重复 start 吗？

不可以。

已经运行时再次调用 `start(...)` 会抛出 `IllegalStateException`。

### stop 可以重复调用吗？

可以。

如果 server 没有运行，`stop()` 直接返回。

### backlog 是最大连接数吗？

不是。

它只是传给 `ServerSocket#bind(...)` 的 listen backlog。

实际含义和效果由操作系统决定。

### TCPServer 支持最大连接数限制吗？

不支持。

如果需要限制连接数，应在 `onConnect(...)` 或更高层实现。

### TCPServer 支持连接超时吗？

不直接支持。

可以在 `onConnect(...)` 中对 `Socket` 设置：

```java
socket.setSoTimeout(...)
```

### TCPClient 支持连接超时吗？

支持。

```java
client.connect(address, timeout)
```

不传 timeout 时使用默认值：

```text
0
```

### TCPClient 会自动重连吗？

不会。

重连属于上层策略。

### TCPClient 有连接池吗？

没有。

每次 `connect(...)` 返回一个新的 `Socket`。

### 连接处理器可以抛 checked exception 吗？

可以。

因为 `onConnect(...)` 使用的是 `Function1Void`。

### connectHandler 抛异常会怎样？

异常会被移交给当前线程的 `UncaughtExceptionHandler`。

如果需要自定义处理，应在 `onConnect(...)` 内部捕获。

### SCX TCP 会记录连接处理异常吗？

不会主动解释或处理用户处理器异常。

系统级 accept 异常会由内部日志记录。

### SCX TCP 支持 TLS 吗？

提供 TLS 辅助工具。

但 `TCPServer` 本身默认仍然是普通 TCP Server。

如果需要 TLS，需要显式使用 `TLS` 的 `SSLSocketFactory` 或 `SSLServerSocketFactory`。

### TLS.ofTrustAny 可以用于生产吗？

不应该。

它会忽略证书验证，只适合测试环境。

### upgradeToTLS 会自动握手吗？

不会明确替你完成所有 TLS 流程。

通常你应根据场景调用：

```java
sslSocket.startHandshake();
```

并配置 client/server mode 等参数。

### SCX TCP 适合什么场景？

适合：

```text
自定义 TCP 协议
简单 socket 工具
本地 TCP 测试
作为更高层协议库的底层 accept/connect 能力
```

不适合直接承担：

```text
HTTP 框架
WebSocket 框架
连接池
负载均衡
限流系统
安全网关
复杂协议服务器
```

### 为什么设计得这么少？

因为 TCP 层只应该做 TCP 层的事。

一旦加入协议语义、连接统计、限流、错误回调、优雅关闭等能力，这个库就会变成服务器框架。

SCX TCP 刻意保持最小边界，让上层模块自己定义语义。