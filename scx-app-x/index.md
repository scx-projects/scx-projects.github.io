# SCX App X

SCX App X 是基于 `scx-app` 的应用扩展模块集合。

它把一个常见服务端应用需要的 HTTP Server、Web 路由、SQL Client、静态资源服务、CORS、定时任务、日志配置、自动修表和应用类自动扫描等能力拆成独立的 `ScxAppModule`，再交给 SCX App 统一完成配置读取、组件装配和启动顺序编排。

SCX App X 不是替代 `scx-app` 的启动器，而是为 `scx-app` 提供一组开箱即用的模块。

[GitHub](https://github.com/scx-projects/scx-app-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-app-x</artifactId>
    <version>0.6.0</version>
</dependency>
```

## 快速开始

一个典型的 SCX App X 应用入口如下：

```java
import dev.scx.app.ScxApp;
import dev.scx.app.x.component.ScxAppComponentModule;
import dev.scx.app.x.cors.ScxAppCorsModule;
import dev.scx.app.x.fix_table.ScxAppFixTableModule;
import dev.scx.app.x.http.ScxAppHttpModule;
import dev.scx.app.x.logging.ScxAppLoggingModule;
import dev.scx.app.x.scheduling.ScxAppSchedulingModule;
import dev.scx.app.x.sql.ScxAppSQLModule;
import dev.scx.app.x.static_server.ScxAppStaticServerModule;
import dev.scx.app.x.auto_scan.ScxAutoScanModule;
import dev.scx.app.x.web.ScxAppWebModule;

public class Main extends ScxAutoScanModule {

    public static void main(String[] args) throws Exception {
        ScxApp.builder()
            .module(new ScxAppLoggingModule())
            .module(new ScxAppComponentModule())
            .module(new ScxAppHttpModule())
            .module(new ScxAppStaticServerModule())
            .module(new ScxAppWebModule())
            .module(new ScxAppSchedulingModule())
            .module(new ScxAppCorsModule())
            .module(new ScxAppSQLModule())
            .module(new ScxAppFixTableModule())
            .module(new Main())
            .mainClass(Main.class)
            .args(args)
            .run();
    }

}
```

典型入口也是这种组合方式：先注册 SCX App X 提供的功能模块，再把应用自身模块加入应用。应用模块继承 `ScxAutoScanModule` 后，会扫描当前模块类所在包及其子包下的所有类，并把这些类加入 `ScxApp` 的 candidates。

## 基本模块

SCX App X 中最常用的模块包括：

```text
ScxAppLoggingModule       根据配置初始化 scx-logging
ScxAppComponentModule     将 @Component 标记的类注册为 DI 组件
ScxAppHttpModule          创建 Router 和 HttpServer，并监听端口
ScxAppWebModule           将 @Routes 控制器编译为 Web 路由并注册到 Router
ScxAppStaticServerModule  根据配置注册静态资源路由
ScxAppCorsModule          注册全局 CORS 路由
ScxAppSQLModule           创建 SQLClient，并注册到 DI 容器
ScxAppFixTableModule      根据 @Table 实体自动创建或补齐数据表
ScxAppSchedulingModule    扫描 @Scheduled 方法并启动定时任务
ScxAutoScanModule         扫描当前模块类所在包及子包下的候选类
```

这些模块都通过 `ScxAppModuleDefinition` 表达自己的依赖和启动顺序。例如 Web、静态资源和 CORS 都要求在 HTTP Server 真正监听端口之前完成路由注册，因此它们都会 `require(ScxAppHttpModule.class)` 并 `startBefore(ScxAppHttpModule.class)`。

## 配置示例

SCX App X 主要通过 `scx-config.json` 配置。下面是一个常见配置：

```json
{
  "scx": {
    "http": {
      "port": 8888,
      "max-payload-size": "16MB",
      "use-development-error-page": true,
      "ssl": {
        "enabled": false,
        "path": "AppRoot:ssl/my_keystore.jks",
        "password": "123456"
      }
    },
    "sql": {
      "url": "jdbc:mysql://127.0.0.1:3306/scx_app_x",
      "username": "root",
      "password": "root",
      "parameters": [
        "allowMultiQueries=true",
        "rewriteBatchedStatements=true",
        "createDatabaseIfNotExist=true"
      ],
      "spy": {
        "enabled": true,
        "style": "rendered-sql"
      }
    },
    "cors": {
      "allowed-origin": "*",
      "allow-credentials": true
    },
    "fix-table": {
      "enabled": false
    },
    "logging": {
      "default": {
        "level": "error",
        "type": "console",
        "path": "AppRoot:logs",
        "trace": false
      },
      "loggers": [
        {
          "name": "ScxJdbcSpy",
          "level": "debug",
          "type": "console",
          "path": "AppRoot:sql-logs",
          "trace": false
        }
      ]
    },
    "static-servers": [
      {
        "host": null,
        "type": "static-files",
        "route": "/*",
        "path": "AppRoot:public"
      },
      {
        "host": null,
        "type": "single-file",
        "route": "/app/*",
        "path": "AppRoot:public/index.html"
      }
    ]
  }
}
```

示例配置里的 `scx-config.json` 展示了 HTTP、SQL、CORS、修表、日志和静态资源服务的完整配置结构。

## 用户模块扫描

### ScxAutoScanModule

`ScxAutoScanModule` 是应用模块基类。它会根据当前模块类定位代码源，然后扫描当前模块类所在包及其子包下的所有 class，并把这些 class 作为 candidates 返回给 SCX App。

```java
public class Main extends ScxAutoScanModule {

}
```

这意味着你通常只需要把入口模块放在业务包根目录下：

```text
com.example.app.Main
com.example.app.controller.UserController
com.example.app.service.UserService
com.example.app.entity.User
```

只要 `Main extends ScxAutoScanModule`，`controller`、`service`、`entity` 等子包里的类都会成为候选类。是否真的进入 DI 容器，则由各个组件选择器决定。

如果需要排除或补充候选类，可以在自己的模块中重写 `configureCandidates(...)`。这个方法接收一个可变 `List<Class<?>>`，你可以原地 `removeIf(...)` / `add(...)`，也可以返回一个新的列表。

## 组件扫描

### `@Component`

`@Component` 用于标识一个类需要注入到容器中。

```java
import dev.scx.app.x.component.Component;

@Component
public class UserService {
}
```

### ScxAppComponentModule

`ScxAppComponentModule` 会添加一个组件选择器：只要 candidate class 上存在 `@Component` 注解，就会被选入 DI 容器。启动时它会打印当前已加载的 Component 数量。

```java
ScxApp.builder()
    .module(new ScxAppComponentModule())
    .module(new Main())
    .mainClass(Main.class)
    .args(args)
    .run();
```

这个模块通常需要和 `ScxAutoScanModule` 一起使用：`ScxAutoScanModule` 负责提供候选类，`ScxAppComponentModule` 负责从候选类里挑出 `@Component`。

## HTTP 模块

### ScxAppHttpModule

`ScxAppHttpModule` 负责创建 `Router`、`HttpServerOptions` 和 `HttpServer`。它在 `init` 阶段读取 HTTP 配置，创建 HTTP Server，并把请求处理器设置为内部的 Router；在 `start` 阶段打印路由信息、监听端口并打印服务器地址；在 `stop` 阶段停止 HTTP Server。

```java
ScxApp.builder()
    .module(new ScxAppHttpModule())
    .mainClass(Main.class)
    .args(args)
    .run();
```

常用配置：

```text
scx.http.port                       默认 8888
scx.http.max-payload-size           默认 16MB
scx.http.use-development-error-page 默认 false
scx.http.ssl.enabled                默认 false
scx.http.ssl.path                   SSL keystore 路径
scx.http.ssl.password               SSL keystore 密码
```

如果启用 SSL，`scx.http.ssl.path` 和 `scx.http.ssl.password` 都不能为空。HTTP 模块会用 `TLS.of(path, password)` 创建 TLS 配置。

```json
{
  "scx": {
    "http": {
      "ssl": {
        "enabled": true,
        "path": "AppRoot:ssl/my_keystore.jks",
        "password": "123456"
      }
    }
  }
}
```

HTTP 模块还会默认注册 `WebSocketUpgradeRequestFactory`，因此可以配合 `scx-websocket-x` 和 `ScxAppWebModule` 暴露 WebSocket 升级路由。

### 获取 Router 和 HttpServer

其他模块可以通过 DI 容器获取 `ScxAppHttpModule`，再访问它暴露的 `router()` 和 `httpServer()`：

```java
var httpModule = scxApp.getComponent(ScxAppHttpModule.class);
var router = httpModule.router();
var server = httpModule.httpServer();
```

这也是 Web、CORS 和静态资源模块接入 HTTP Server 的方式。

## Web 路由模块

### ScxAppWebModule

`ScxAppWebModule` 用于把 `@Routes` 控制器接入 HTTP Router。它会在 `init` 阶段添加一个组件选择器，让所有带 `@Routes` 注解的 candidate 成为 DI 组件；在 `start` 阶段遍历 candidates，取出所有 `@Routes` 组件，用 `ScxWeb` 编译成路由，再注册到 HTTP 模块的 Router。

```java
import dev.scx.web.annotation.Route;
import dev.scx.web.annotation.Routes;

import static dev.scx.http.method.HttpMethod.GET;

@Routes("/api/hello")
public class HelloController {

    @Route(value = "", methods = GET)
    public Object hello() {
        return "Hello SCX App X";
    }

}
```

接入模块：

```java
ScxApp.builder()
    .module(new ScxAppHttpModule())
    .module(new ScxAppWebModule())
    .module(new Main())
    .mainClass(Main.class)
    .args(args)
    .run();
```

`ScxAppWebModule` 声明了 `require(ScxAppHttpModule.class)` 和 `startBefore(ScxAppHttpModule.class)`，因此 Web 路由会先注册，HTTP Server 再开始监听端口。

### Controller 构造器注入

控制器本身也是 DI 组件，所以可以通过构造器注入 Service：

```java
@Routes("/api/apple")
public class AppleController {

    private final AppleService service;

    public AppleController(AppleService service) {
        this.service = service;
    }

    @Route(value = "", methods = GET)
    public Object list() {
        return service.find();
    }

}
```

示例里的 `AppleController` 展示了基于 `@Routes`、`@Route`、`@PathCapture` 和 `@Body` 的 CRUD Controller 写法。

## 静态资源服务

### ScxAppStaticServerModule

`ScxAppStaticServerModule` 从 `scx.static-servers` 中读取静态资源服务配置，并把每一项转换成一条路由注册到 HTTP Router。它也要求在 HTTP 模块启动之前完成注册。

配置项结构：

```text
host   可选；为空时匹配任意 Host
type   可选；静态资源服务类型，默认为 static-files
route  路径模板，例如 /* 或 /app/*
path   文件或目录路径，支持 AppRoot: 等配置路径
```

示例：

```json
{
  "scx": {
    "static-servers": [
      {
        "type": "static-files",
        "route": "/*",
        "path": "AppRoot:public"
      },
      {
        "type": "single-file",
        "route": "/app/*",
        "path": "AppRoot:public/index.html"
      }
    ]
  }
}
```

`static-files` 会使用 `StaticFilesHandler` 处理目录资源；`single-file` 会使用 `SingleFileHandler` 始终返回同一个文件，适合 SPA 前端应用的 history fallback。

`type` 支持以下值 (不区分大小写) :

```text
STATIC-FILES / STATIC_FILES / DIRECTORY / DIR / D
SINGLE-FILE / SINGLE_FILE / FILE / F
```

静态资源路由使用较靠后的优先级 `999999`，这样业务 API 路由通常会先于静态资源路由匹配。

## CORS 模块

### ScxAppCorsModule

`ScxAppCorsModule` 会创建一个全局 `CorsHandler`，并以优先级 `-10000` 注册到 HTTP Router，因此它会很早参与请求处理。

配置：

```text
scx.cors.allowed-origin    默认 *
scx.cors.allow-credentials 默认 false
```

示例：

```json
{
  "scx": {
    "cors": {
      "allowed-origin": "*",
      "allow-credentials": true
    }
  }
}
```

默认允许的方法:

```text
使用 Reflect 策略, 按预检请求中的 Access-Control-Request-Method 动态响应
```

默认允许的请求头:

```text
使用 Reflect 策略, 按预检请求中的 Access-Control-Request-Headers 动态响应
```

默认暴露的响应头:

```text
Content-Disposition
```

当 `allowed-origin` 配置为 `*` 时，模块不会把它当作普通通配符响应，而是使用 Reflect 策略回显请求来源，这样可以和 `allow-credentials=true` 正确配合。

## SQL 模块

### ScxAppSQLModule

`ScxAppSQLModule` 负责创建并注入 `SQLClient`。它会读取数据库连接配置，创建 `JDBCConnectionInfo`，注册默认 SQL 类型处理器，并额外注册一个基于 JSON 的对象类型处理器。最后通过 HikariCP 包装 DataSource。

配置：

```text
scx.sql.url             JDBC URL
scx.sql.username        用户名
scx.sql.password        密码
scx.sql.parameters      JDBC 参数数组，默认 []
scx.sql.spy.enabled     是否启用 ScxJdbcSpy，默认 false
scx.sql.spy.style       Spy 日志样式，默认 RENDERED_SQL
```

示例：

```json
{
  "scx": {
    "sql": {
      "url": "jdbc:mysql://127.0.0.1:3306/scx_app_x",
      "username": "root",
      "password": "root",
      "parameters": [
        "allowMultiQueries=true",
        "rewriteBatchedStatements=true",
        "createDatabaseIfNotExist=true"
      ],
      "spy": {
        "enabled": true,
        "style": "rendered-sql"
      }
    }
  }
}
```

SQL 模块会把创建好的 `SQLClient` 作为 component instance 注入到容器里。因此业务组件可以直接通过构造器接收：

```java
@Component
public class UserService {

    private final SQLClient sqlClient;

    public UserService(SQLClient sqlClient) {
        this.sqlClient = sqlClient;
    }

}
```

如果开启 `scx.sql.spy.enabled`，DataSource 会被 `ScxJdbcSpy.spy(...)` 包装，并使用 `LoggingDataSourceListener` 输出 SQL 日志。日志格式由 `scx.sql.spy.style` 控制；未配置时默认使用 `RENDERED_SQL`。

`style` 支持以下值 (不区分大小写) :

```text
RENDERED_SQL / RENDERED-SQL / RENDERED / R
SQL_AND_PARAMETERS / SQL-AND-PARAMETERS / PARAMETERS / PARAMETER / P
```

其中 `RENDERED_SQL` 输出渲染后的 SQL，`SQL_AND_PARAMETERS` 输出 SQL 和参数。

### 对象 JSON 映射

SCX App X 额外注册了 `ObjectSQLHandlerFactory`。当字段类型没有更具体的 SQL 类型处理器时，对象会通过 `ScxSerialize.toJson(...)` 写入字符串，读取时再通过 `ScxSerialize.fromJson(json, typeInfo)` 反序列化。

这让实体类中直接包含普通对象、数组或枚举字段时，可以更方便地存成 JSON 字段或字符串字段。示例里的 `CarOwner owner` 和 `String[] tags` 就是这种使用方式。

## 自动修表模块

### ScxAppFixTableModule

`ScxAppFixTableModule` 用于根据 `@Table` 实体自动创建或补齐数据库表结构。它依赖 `ScxAppSQLModule`，并要求在 HTTP 模块之前启动，这样应用真正对外提供服务前就可以先完成表结构检查。

配置：

```text
scx.fix-table.enabled 默认 false
```

启用：

```json
{
  "scx": {
    "fix-table": {
      "enabled": true
    }
  }
}
```

启动流程大致为：

1. 从 `ScxAppSQLModule` 获取 `SQLClient`。
2. 读取 `scx.fix-table.enabled`。
3. 检查 DataSource 是否可连接。
4. 从 candidates 中找出所有标注了 `@Table` 的实体类。
5. 对比当前数据库中的同名表和实体映射出来的新表。
6. 如果表不存在，则创建表。
7. 如果表存在，则只添加缺失的列和索引。

修表逻辑是保守策略：当前只自动添加不存在的列和索引，不会自动删除列、删除索引或修改已有列定义。`TableSupport.diffTable(...)` 会计算需要添加和删除的列、索引，但 `getFixTableSQLs(...)` 只生成添加列和添加索引的 DDL。

## 定时任务模块

### `@Scheduled`

`@Scheduled` 用在方法上，目前支持 cron 表达式。它是可重复注解，同一个方法可以标注多个 `@Scheduled`。

```java
import dev.scx.app.x.component.Component;
import dev.scx.app.x.scheduling.Scheduled;

@Component
public class DemoTasks {

    @Scheduled(cron = "*/1 * * * * ?")
    public void everySecond() {
        System.out.println("tick");
    }

}
```

### ScxAppSchedulingModule

`ScxAppSchedulingModule` 在启动时扫描 DI 容器中的所有组件，查找带有 `@Scheduled` 或 `@ScheduledList` 的方法，然后通过 `ScxScheduling.cron()` 启动任务。应用停止时，它会取消所有已启动的 `ScheduleHandle`。

被 `@Scheduled` 标记的方法必须满足：

```text
必须是 public
不能是 static
不能有参数
```

如果不满足这些规则，模块启动时会抛出异常。

## 日志模块

### ScxAppLoggingModule

`ScxAppLoggingModule` 会在 `init` 阶段读取日志配置，并更新 `ScxLogging` 的 root config，以及按名称或正则表达式匹配的 logger config。

默认配置：

```text
scx.logging.default.level 默认 error
scx.logging.default.type  默认 console
scx.logging.default.path  默认 AppRoot:logs
scx.logging.default.trace 默认 false
```

日志规则配置：

```json
{
  "scx": {
    "logging": {
      "loggers": [
        {
          "name": "ScxJdbcSpy",
          "level": "debug",
          "type": "console",
          "path": "AppRoot:sql-logs",
          "trace": false
        },
        {
          "regex": "dev[.]scx[.]app[.]service[.].*",
          "level": "info",
          "type": "console",
          "trace": false
        }
      ]
    }
  }
}
```

每条 `loggers` 配置必须且只能设置 `name` 或 `regex`：`name` 按 logger 名称精确匹配，`regex` 会通过 `Pattern.compile(...)` 编译后匹配。两者都未设置或同时设置时，模块会在初始化阶段抛出 `IllegalArgumentException`。

`level` 支持以下值 (不区分大小写) :

```text
OFF / O
ERROR / E
WARN / WARNING / W
INFO / I
DEBUG / D
TRACE / T
ALL / A
```

`type` 支持以下值 (不区分大小写) :

```text
CONSOLE / C
FILE / F
BOTH / B
```

当日志规则没有配置 `path` 时，会回退使用默认 logging path。

## 模块启动顺序建议

一个完整应用通常可以按下面顺序注册模块：

```java
ScxApp.builder()
    .module(new ScxAppLoggingModule())
    .module(new ScxAppComponentModule())
    .module(new ScxAppHttpModule())
    .module(new ScxAppStaticServerModule())
    .module(new ScxAppWebModule())
    .module(new ScxAppSchedulingModule())
    .module(new ScxAppCorsModule())
    .module(new ScxAppSQLModule())
    .module(new ScxAppFixTableModule())
    .module(new Main())
    .mainClass(Main.class)
    .args(args)
    .run();
```

这里的注册顺序主要是为了可读性。实际启动顺序会由 `scx-app` 根据模块的 `require`、`startBefore` 和 `startAfter` 重新计算。

几个关键关系：

```text
ScxAppWebModule          require ScxAppHttpModule, startBefore ScxAppHttpModule
ScxAppStaticServerModule require ScxAppHttpModule, startBefore ScxAppHttpModule
ScxAppCorsModule         require ScxAppHttpModule, startBefore ScxAppHttpModule
ScxAppFixTableModule     require ScxAppSQLModule, startAfter ScxAppSQLModule, startBefore ScxAppHttpModule
```

因此，路由注册、CORS 注册、静态资源注册和修表都会在 HTTP Server 监听端口之前完成。

## 常见问题

### 为什么我的 `@Component` 没有被注入？

通常需要同时满足两个条件：

1. 这个类被 `ScxAutoScanModule` 扫描进 candidates。
2. 应用注册了 `ScxAppComponentModule`，让 `@Component` 类进入 DI 容器。

如果类不在应用模块类所在包或子包下，`ScxAutoScanModule` 默认不会扫描到它。可以把入口模块放在业务根包，或者重写 `configureCandidates(...)` 补充候选类。

### 为什么我的 `@Routes` Controller 没有生效？

需要确认三点：

1. Controller 类被 `ScxAutoScanModule` 扫描进 candidates。
2. 应用注册了 `ScxAppWebModule`。
3. 应用注册了 `ScxAppHttpModule`。

`ScxAppWebModule` 会把 `@Routes` 类选入容器并注册路由，但它依赖 HTTP 模块提供 Router。

### 为什么自动修表没有执行？

`ScxAppFixTableModule` 默认关闭，需要显式配置：

```json
{
  "scx": {
    "fix-table": {
      "enabled": true
    }
  }
}
```

同时，必须注册 `ScxAppSQLModule`，数据库连接必须可用，实体类必须被扫描进 candidates 并标注 `@Table`。

### 自动修表会不会删除列？

不会。当前策略只会在表不存在时创建表，或者在表存在时添加缺失的列和索引，不会自动删除列、删除索引或修改已有字段定义。

### `@Scheduled` 方法为什么启动报错？

被 `@Scheduled` 标记的方法必须是 `public`、非 `static`、无参数。定时任务模块会在启动阶段检查这些规则。

### 为什么 `allowed-origin="*"` 还能和 `allow-credentials=true` 一起用？

因为 CORS 模块不会把 `*` 直接作为普通 wildcard 响应，而是使用 reflect 策略回显请求中的 Origin。这样响应头中实际返回的是具体 Origin，而不是字面量 `*`，因此可以和 credentials 场景配合。

### 静态资源路由为什么没有匹配？

检查 `scx.static-servers` 中的 `type`、`route` 和 `path` 是否正确配置。
