# SCX

SCX 是一组面向 Java 的轻量库。它们不是一个单体框架，而是一套可以按需组合的基础能力：应用启动、依赖注入、HTTP、WebSocket、数据访问、SQL、序列化、对象映射、定时任务、日志、数组/集合/字符串/IO 等工具。

如果你只是想写一个完整应用，通常从 [SCX App X](./scx-app-x/index.md) 开始；如果你只需要某一项能力，可以直接引入对应库。

[GitHub](https://github.com/scx-projects)

## 怎么选择

### 应用与组合

- [SCX App](./scx-app/index.md)：应用启动底座，负责配置环境、模块编排、DI 容器装配和启动/停止顺序。
- [SCX App X](./scx-app-x/index.md)：在 SCX App 之上组合 Web、HTTP、SQL、定时任务、日志、静态资源、自动扫描等常用应用能力。
- [SCX Parent](./scx-parent/index.md)：SCX 项目的 Maven 父工程，统一 Java 版本、插件、发布、测试和格式化配置。
- [SCX DI](./scx-di/index.md)：轻量依赖注入核心，负责显式注册组件、构造函数注入、字段注入、值注入和循环依赖检测。

### HTTP、Web 和 WebSocket

- [SCX HTTP](./scx-http/index.md)：HTTP 协议抽象，定义请求、响应、Header、URI、Media 读写、错误处理等模型。
- [SCX HTTP X](./scx-http-x/index.md)：基于 TCP 的 HTTP 客户端和服务端实现，提供 `HttpServer`、`HttpClient` 和 HTTP/1.x 读写能力。
- [SCX HTTP Routing](./scx-http-routing/index.md)：HTTP 路由核心，支持请求匹配、路径模板、正则路径、方法匹配和路由表。
- [SCX HTTP Routing X](./scx-http-routing-x/index.md)：路由扩展能力，提供静态文件、单文件、CORS、Range 请求等常见处理器。
- [SCX Web](./scx-web/index.md)：注解式 Web 路由封装，把 Java Controller 方法编译成 HTTP / WebSocket 路由。
- [SCX WebSocket](./scx-websocket/index.md)：WebSocket 协议层实现，处理 frame、message、close、ping/pong 和事件式读写。
- [SCX WebSocket X](./scx-websocket-x/index.md)：把 WebSocket 接入 SCX HTTP X，提供服务端升级请求和客户端连接能力。

### 数据访问与 SQL

- [SCX Data](./scx-data/index.md)：底层无关的数据访问抽象，提供 Repository、Query、FieldPolicy、聚合和锁查询模型。
- [SCX Data X](./scx-data-x/index.md)：SCX Data 的扩展库，提供 `Query`、`FieldPolicy`、`Aggregation` 与 `Node` 互转能力。
- [SCX Data SQL](./scx-data-sql/index.md)：把 SCX Data 的 Repository 抽象落到 SQL/JDBC 上，提供 `SQLRepository`、表映射、字段映射和聚合查询。
- [SCX Data JS](./scx-data-js/index.md)：SCX Data 与 SCX Data X 中 Query、FieldPolicy 相关能力的 JavaScript 版本。
- [SCX SQL](./scx-sql/index.md)：SQL/JDBC 基础库，提供 SQLClient、事务、Dialect、表结构、DDL、类型处理器和结果读取。
- [SCX SQL MySQL](./scx-sql-mysql/index.md)：MySQL 方言和 MySQL 数据库相关适配。
- [SCX JDBC Spy](./scx-jdbc-spy/index.md)：JDBC 代理与 SQL 观察工具，用于记录 SQL、参数、结果和执行耗时。
- [SCX Transaction](./scx-transaction/index.md)：事务抽象，支持事务上下文、事务传播和事务执行包装。

### 序列化、格式和对象映射

- [SCX Node](./scx-node/index.md)：统一的树形数据节点模型，表示 Object、Array、String、Number、Boolean、Null 等结构。
- [SCX Object](./scx-object/index.md)：对象和 Node 之间转换的核心抽象。
- [SCX Object X](./scx-object-x/index.md)：常用对象映射实现，支持 class、record、集合、数组、Map、枚举、时间、路径、UUID、URI 等类型。
- [SCX Format](./scx-format/index.md)：格式读写基础抽象，定义字符串/文件/流与 Node 之间的转换接口。
- [SCX Format JSON](./scx-format-json/index.md)：JSON 与 Node 互转。
- [SCX Format XML](./scx-format-xml/index.md)：XML 与 Node 互转。
- [SCX Serialize](./scx-serialize/index.md)：组合 Object X、JSON、XML，提供对象到 JSON/XML、JSON/XML 到对象的高层入口。

### 定时、日志、终端与系统支持

- [SCX Scheduling](./scx-scheduling/index.md)：调度任务封装，支持一次性任务、固定频率、固定延迟、Cron、过期策略和错误处理。
- [SCX Timer](./scx-timer/index.md)：轻量定时器和超时任务工具。
- [SCX Logging](./scx-logging/index.md)：SCX 日志核心，并适配 JDK Logger、SLF4J、Log4j。
- [SCX ANSI](./scx-ansi/index.md)：ANSI 终端样式和颜色输出。
- [SCX FFI](./scx-ffi/index.md)：基于 Java FFM 的外部函数调用封装，用于绑定 native library、结构体、指针和回调。

### 通用工具

- [SCX Array](./scx-array/index.md)：数组工具，支持基本类型数组和对象数组的互转、拼接、截取、查找、反转、打乱和随机取值。
- [SCX Collection](./scx-collection/index.md)：集合工具，提供 CountMap、MultiMap 和集合便捷操作。
- [SCX Digest](./scx-digest/index.md)：摘要和校验工具，支持 MD5、SHA、CRC 等常见摘要算法。
- [SCX Exception](./scx-exception/index.md)：异常包装工具，尤其用于把 checked exception 转成可传递的 runtime wrapper。
- [SCX Function](./scx-function/index.md)：可抛异常的函数式接口，补齐 JDK 函数式接口不能声明 checked exception 的问题。
- [SCX IO](./scx-io/index.md)：字节输入/输出抽象、内存字节流、文件字节流、压缩、分片、读写工具。
- [SCX Lock](./scx-lock/index.md)：命名锁工具，用于按 key 串行化执行代码。
- [SCX Random](./scx-random/index.md)：随机工具，支持随机字节、随机字符串、权重随机、随机范围等。
- [SCX Reflect](./scx-reflect/index.md)：反射增强模型，处理类型信息、泛型绑定、字段/方法/构造函数视图和 record 构造函数。
- [SCX String](./scx-string/index.md)：字符串工具，包含空白判断、大小写转换、模板、截断、重复等常用操作。
- [SCX TCP](./scx-tcp/index.md)：TCP 客户端和服务端抽象，提供 socket endpoint、连接处理和基础网络能力。

## 常见组合

### 写一个 Web 应用

通常组合：

```text
scx-app-x
scx-web
scx-http-x
scx-http-routing-x
scx-data-sql
scx-sql-mysql
scx-scheduling
scx-logging
```

优先阅读 [SCX App X](./scx-app-x/index.md)，再按需要阅读 Web、SQL、Scheduling 的独立手册。

### 只写 HTTP 服务

如果不需要应用启动编排，可以直接组合：

```text
scx-http-x
scx-http-routing
scx-http-routing-x
```

`scx-http-x` 负责启动服务，`scx-http-routing` 负责路由，`scx-http-routing-x` 提供静态文件、CORS、Range 等常用处理器。

### 只做对象序列化

通常直接用：

```text
scx-serialize
```

它已经组合了对象映射、JSON、XML 和 Node。只有在你需要自定义底层格式或自定义 Node 映射时，才需要深入看 `scx-object-x`、`scx-format-json`、`scx-format-xml`。

### 只做 SQL / Repository

如果希望业务层面向 Repository，可以从：

```text
scx-data-sql
```

开始。它会进一步使用 `scx-data` 的抽象和 `scx-sql` 的 JDBC 能力。

如果需要把 `Query`、`FieldPolicy` 或 `Aggregation` 转成可传输的 Node 结构，可额外组合：

```text
scx-data-x
```

如果你只想手写 SQL 并管理连接、事务、结果映射，则直接看 [SCX SQL](./scx-sql/index.md)。
