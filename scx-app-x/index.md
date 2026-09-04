# SCX App X

SCX App X 是基于 `scx-app` 的应用扩展模块集合。

它把一个常见服务端应用需要的配置环境、DI 容器、HTTP Server、Web 路由、SQL Client、静态资源服务、CORS、定时任务、日志配置、自动修表和应用类自动扫描等能力拆成独立的 `ScxAppModule`。SCX App 负责模块图和启动/停止流程，各个 App X 模块通过 `ScxApp.getModule(...)` 获取彼此提供的能力。

SCX App X 提供一组基于 `ScxAppModule` 的开箱即用应用模块，可以按应用需要自由组合。

[GitHub](https://github.com/scx-projects/scx-app-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-app-x</artifactId>
    <version>0.7.0</version>
</dependency>
```

## 快速开始

一个典型的 SCX App X 应用入口如下：

```java
import dev.scx.app.ScxApp;
import dev.scx.app.x.auto_scan.ScxAutoScanModule;
import dev.scx.app.x.component.ScxAppComponentModule;
import dev.scx.app.x.config.ScxAppConfigModule;
import dev.scx.app.x.cors.ScxAppCorsModule;
import dev.scx.app.x.di.ScxAppDIModule;
import dev.scx.app.x.fix_table.ScxAppFixTableModule;
import dev.scx.app.x.http.ScxAppHttpModule;
import dev.scx.app.x.logging.ScxAppLoggingModule;
import dev.scx.app.x.scheduling.ScxAppSchedulingModule;
import dev.scx.app.x.sql.ScxAppSQLModule;
import dev.scx.app.x.static_server.ScxAppStaticServerModule;
import dev.scx.app.x.web.ScxAppWebModule;
import dev.scx.app.x.web.ScxAppWebComponentModule;

public class Main extends ScxAutoScanModule {

    public static void main(String[] args) throws Exception {
        ScxApp.builder()
            .module(new ScxAppConfigModule(Main.class, args))
            .module(new ScxAppLoggingModule())
            .module(new ScxAppComponentModule())
            .module(new ScxAppHttpModule())
            .module(new ScxAppStaticServerModule())
            .module(new ScxAppWebComponentModule())
            .module(new ScxAppWebModule())
            .module(new ScxAppSchedulingModule())
            .module(new ScxAppCorsModule())
            .module(new ScxAppSQLModule())
            .module(new ScxAppFixTableModule())
            .module(new ScxAppDIModule())
            .module(new Main())
            .run();
    }

}
```

典型入口使用 `ScxAppConfigModule` 接收 `mainClass` 和 `args`，使用 `ScxAppDIModule` 创建 DI 容器。应用模块继承 `ScxAutoScanModule` 后，会扫描当前模块类所在包及其子包下的所有类，并把这些类加入 `ScxApp` 的 candidates。

## 基本模块

SCX App X 中最常用的模块包括：

```text
ScxAppConfigModule        根据 mainClass 和 args 创建 ScxEnvironment
ScxAppDIModule            构建并校验 DI 容器，提供 @Inject / @Value
ScxAppLoggingModule       根据配置初始化 scx-logging
ScxAppComponentModule     将 @Component 标记的 candidate 注册到 DI 容器
ScxAppHttpModule          创建 Router 和 HttpServer，并监听端口
ScxAppWebComponentModule  将 @Routes 标记的 candidate 注册到 DI 容器
ScxAppWebModule           将 @Routes 控制器编译为 Web 路由并注册到 Router
ScxAppStaticServerModule  根据配置注册静态资源路由
ScxAppCorsModule          注册全局 CORS 路由
ScxAppSQLModule           创建 SQLClient；存在 DI 模块时注册到 DI 容器
ScxAppFixTableModule      根据 @Table 实体自动创建或补齐数据表
ScxAppSchedulingModule    扫描 DI 容器中的 @Scheduled 方法并启动定时任务
ScxAutoScanModule         扫描当前模块类所在包及子包下的候选类
```

这些模块都通过 `ScxAppModuleDefinition` 表达自己的存在依赖和启动顺序。各模块持有自己创建的运行时能力；其他模块使用 `scxApp.getModule(...)` 获取模块后，可以访问其 `environment()`、`componentContainer()`、`router()`、`sqlClient()` 等接口。

## 配置模块

### ScxAppConfigModule

`ScxAppConfigModule` 提供应用配置环境。它接收应用主类和启动参数，在 `start` 阶段根据主类的 JVM CodeSource 计算应用根目录，并建立 `ScxEnvironment`：

```java
ScxApp.builder()
    .module(new ScxAppConfigModule(Main.class, args))
    .module(new Main())
    .run();
```

其他模块通过 `ScxApp` 获取配置模块，再访问环境：

```java
var configModule = scxApp.getModule(ScxAppConfigModule.class);
var environment = configModule.environment();
```

`environment()` 只有在 `ScxAppConfigModule` 启动后才可用。需要配置的模块通常会 `require(ScxAppConfigModule.class)` 并 `startAfter(ScxAppConfigModule.class)`。

### 默认配置文件

`ScxAppConfigModule` 内置了一个默认配置：

```text
scx.config = AppRoot:scx-config.json
```

也就是说，如果没有通过命令行参数覆盖配置路径，配置模块会尝试从应用根目录下的 `scx-config.json` 读取配置。

### 配置优先级

`ScxEnvironment` 支持多个配置源。配置源按添加顺序保存，但读取时会从最后一个配置源开始查找；因此越晚添加的配置源优先级越高。

默认构建流程中配置源大致是：

1. 默认配置源。
2. JSON 文件配置源。
3. 命令行参数配置源。

所以实际优先级是：

```text
命令行参数 > scx-config.json > 默认配置
```

### 读取配置

`ScxEnvironment` 提供字符串路径读取和类型化读取：

```java
var node = environment.get("server.port");

Integer port = environment.get("server.port", Integer.class);
Integer portWithDefault = environment.get("server.port", Integer.class, 8080);
```

环境内部使用对象转换器把配置节点转换成目标类型，并内置了 `ConfiguredPath` 和 `ConfiguredSize` 等配置类型的映射能力。

### 配置源

SCX App X 的配置源统一实现 `ScxConfigSource`。配置源最终都会提供一个树形 `ObjectNode`，点号路径会被看成树形路径的一层。

内置配置源包括：

```text
MapConfigSource       从 Map 读取配置
ArgsConfigSource      从命令行参数读取配置
JsonFileConfigSource  从 JSON 文件读取配置
```

`MapConfigSource` 适合提供默认值：

```java
var source = MapConfigSource.of(Map.of(
    "server.port", 8080,
    "app.name", "demo"
));
```

上面的 `server.port` 会被转换成类似下面的树形结构：

```json
{
  "server": {
    "port": 8080
  },
  "app": {
    "name": "demo"
  }
}
```

`ArgsConfigSource` 读取 `--key=value` 形式的参数：

```text
--server.port=8080
--app.name=demo
--scx.config=AppRoot:config/dev.json
```

只有以 `--` 开头并且包含 `=` 的参数会进入配置源。`--debug` 这种没有值的参数不会被当作配置项。

`JsonFileConfigSource` 读取 JSON 配置文件，并要求文件内容必须是 JSON Object：

```json
{
  "server": {
    "port": 8080
  }
}
```

如果文件不存在、读取失败、JSON 格式错误，或者根节点不是 Object，创建配置源时会抛出 `ScxConfigSourceException`。

### AppRoot 路径

`ConfiguredPath` 用于读取配置中的路径值。它支持 `AppRoot:` 前缀：

```json
{
  "upload": {
    "dir": "AppRoot:uploads"
  }
}
```

读取：

```java
ConfiguredPath uploadDir = environment.get("upload.dir", ConfiguredPath.class);
Path path = uploadDir.path();
```

`AppRoot:` 会被解析为应用根目录。应用根目录来自传给 `ScxAppConfigModule` 的 `mainClass` 的 JVM CodeSource：开发环境通常是 classes 目录，发布环境通常是 jar 所在目录。

如果配置值不以 `AppRoot:` 开头，则按普通文件系统路径解析，并最终转成绝对、规范化后的路径。

### 大小配置

`ConfiguredSize` 用于读取文件大小、缓存大小、上传限制等配置：

```json
{
  "upload": {
    "max-size": "10MB"
  }
}
```

读取：

```java
ConfiguredSize maxSize = environment.get("upload.max-size", ConfiguredSize.class);
long bytes = maxSize.size();
```

支持的单位包括：

```text
B
KB
MB
GB
TB
```

不写单位时按字节处理。`1KB` 表示 `1024`，`1MB` 表示 `1024 * 1024`。

### CodeSource

`ScxCodeSource` 是 SCX App X 配置模块对 JVM CodeSource 的封装。它表示某个 class 来自哪里。

常见情况：

```text
开发环境  classes 目录
发布环境  jar 文件
```

`ScxAppConfigModule` 会通过传入的 `mainClass` 创建 `ScxCodeSource`，再用它计算应用根目录。你通常不需要直接使用 `ScxCodeSource`，但理解它有助于判断 `AppRoot:` 到底指向哪里。

## DI 模块

### ScxAppDIModule

`ScxAppDIModule` 提供 DI 容器。模块创建时会准备 `ComponentContainerBuilder` 并注册 `@Inject` 解析器；在 `start` 阶段构建并校验最终的 `ComponentContainer`。

如果应用中同时存在 `ScxAppConfigModule`，DI 模块会在配置模块之后启动，并额外注册 `@Value` 解析器以及当前 `ScxEnvironment` 实例。因此配置模块对 DI 是可选能力，而不是 SCX App 的硬依赖。

需要向 DI 容器注册组件的模块，应在 DI 模块之前启动并操作 builder：

```java
@Override
public ScxAppModuleDefinition define() {
    return ScxAppModuleDefinition.of()
        .require(ScxAppDIModule.class)
        .startBefore(ScxAppDIModule.class);
}

@Override
public void start(ScxApp scxApp) {
    var diModule = scxApp.getModule(ScxAppDIModule.class);
    diModule.componentContainerBuilder()
        .registerComponentType(UserService.class.getName(), UserService.class);
}
```

DI 模块启动完成后，消费者可以通过：

```java
var diModule = scxApp.getModule(ScxAppDIModule.class);
var componentContainer = diModule.componentContainer();
```

访问构建并校验后的容器。`componentContainerBuilder()` 仅在 DI 模块启动前可用，`componentContainer()` 从 DI 模块启动完成后可用。`ScxAppModule` 通过 `ScxApp.getModule(...)` 进行模块级能力访问；需要进入 DI 容器的对象应显式注册到 `ComponentContainerBuilder`。

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

只要 `Main extends ScxAutoScanModule`，`controller`、`service`、`entity` 等子包里的类都会成为候选类。candidate 本身只是应用级候选类；是否进入 DI 容器、是否被 Web 或修表模块使用，由对应的 App X 模块决定。

如果需要排除或补充候选类，可以在自己的模块中重写 `configureCandidates(...)`。这个方法接收一个可变 `List<Class<?>>`，你可以原地 `removeIf(...)` / `add(...)`，也可以返回一个新的列表。

## 组件扫描

### `@Component`

`@Component` 用于标识一个需要注册到组件容器中的类。

```java
import dev.scx.app.x.component.Component;

@Component
public class UserService {
}
```

### ScxAppComponentModule

`ScxAppComponentModule` 要求应用中存在 `ScxAppDIModule`，并在 DI 模块之前启动。它会遍历 `ScxApp.candidates()`，把带有 `@Component` 注解的 candidate type 注册到 `ScxAppDIModule` 的 `ComponentContainerBuilder` 中；真正的容器构建和校验由 `ScxAppDIModule` 完成。

```java
ScxApp.builder()
    .module(new ScxAppComponentModule())
    .module(new ScxAppDIModule())
    .module(new Main())
    .run();
```

这个模块通常需要和 `ScxAutoScanModule`、`ScxAppDIModule` 一起使用：`ScxAutoScanModule` 负责提供候选类，`ScxAppComponentModule` 负责注册 `@Component` 类型，`ScxAppDIModule` 最后构建并校验 DI 容器。

## HTTP 模块

### ScxAppHttpModule

`ScxAppHttpModule` 负责创建 `Router`、`HttpServerOptions` 和 `HttpServer`。它要求存在 `ScxAppConfigModule` 并在配置模块之后启动；在 `start` 阶段读取 HTTP 配置、创建 HTTP Server、打印路由信息并监听端口；在 `stop` 阶段停止 HTTP Server。`Router` 在模块构造后即可通过 `router()` 获取。

```java
ScxApp.builder()
    .module(new ScxAppConfigModule(Main.class, args))
    .module(new ScxAppHttpModule())
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

其他模块通过 `ScxApp.getModule(...)` 获取 `ScxAppHttpModule`，再访问它暴露的 `router()` 和 `httpServer()`：

```java
var httpModule = scxApp.getModule(ScxAppHttpModule.class);
var router = httpModule.router();
var server = httpModule.httpServer();
```

这也是 Web、CORS 和静态资源模块接入 HTTP Server 的方式。

## Web 路由模块

### ScxAppWebModule

`ScxAppWebModule` 用于把 `@Routes` 控制器接入 HTTP Router。它要求 `ScxAppHttpModule` 和 `ScxAppDIModule` 都存在，并在 DI 模块之后、HTTP 模块之前启动。启动时它遍历 candidates，找到所有 `@Routes` 类型，再从 DI 容器取得对应组件实例，用 `ScxWeb` 编译成路由并注册到 HTTP 模块的 Router。

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
    .module(new ScxAppConfigModule(Main.class, args))
    .module(new ScxAppDIModule())
    .module(new ScxAppHttpModule())
    .module(new ScxAppWebComponentModule())
    .module(new ScxAppWebModule())
    .module(new Main())
    .run();
```

`ScxAppWebModule` 声明了 `require(ScxAppHttpModule.class)`、`require(ScxAppDIModule.class)`、`startAfter(ScxAppDIModule.class)` 和 `startBefore(ScxAppHttpModule.class)`，因此控制器实例会先由 DI 容器准备好，Web 路由完成注册后 HTTP Server 才开始监听端口。

### Controller 构造器注入

控制器需要先成为 DI 组件，通常注册 `ScxAppWebComponentModule` 即可。这样控制器就可以通过构造器注入 Service：

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

`ScxAppStaticServerModule` 依赖 `ScxAppConfigModule` 和 `ScxAppHttpModule`。它在配置模块之后读取 `scx.static-servers`，把每一项转换成一条路由注册到 HTTP Router，并要求在 HTTP 模块启动之前完成注册。

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

`ScxAppCorsModule` 依赖 `ScxAppConfigModule` 和 `ScxAppHttpModule`。它在配置模块之后读取 CORS 配置，创建一个全局 `CorsHandler`，并以优先级 `-10000` 注册到 HTTP Router，因此它会很早参与请求处理。

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

`ScxAppSQLModule` 负责创建并暴露 `SQLClient`。它要求 `ScxAppConfigModule` 存在，在配置模块之后读取数据库连接配置，创建 `JDBCConnectionInfo`，注册默认 SQL 类型处理器，并额外注册一个基于 JSON 的对象类型处理器，最后通过 HikariCP 包装 DataSource。

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

如果应用中存在 `ScxAppDIModule`，SQL 模块会在 DI 模块之前启动，并把创建好的 `SQLClient` 注册到 `ComponentContainerBuilder`。因此使用 DI 时，业务组件可以直接通过构造器接收：

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

`ScxAppFixTableModule` 用于根据 `@Table` 实体自动创建或补齐数据库表结构。它依赖 `ScxAppSQLModule` 和 `ScxAppConfigModule`，如果应用中存在 `ScxAppHttpModule`，它会在 HTTP 模块之前启动，这样应用真正对外提供服务前就可以先完成表结构检查。

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

`ScxAppSchedulingModule` 要求 `ScxAppDIModule` 存在并在 DI 模块之后启动。它通过 `ScxAppDIModule.componentContainer()` 扫描所有组件，查找带有 `@Scheduled` 或 `@ScheduledList` 的方法，然后通过 `ScxScheduling.cron()` 启动任务。应用停止时，它会取消所有已启动的 `ScheduleHandle`。

被 `@Scheduled` 标记的方法必须满足：

```text
必须是 public
不能是 static
不能有参数
```

如果不满足这些规则，模块启动时会抛出异常。

## 日志模块

### ScxAppLoggingModule

`ScxAppLoggingModule` 要求 `ScxAppConfigModule` 存在，并在配置模块之后的 `start` 阶段读取日志配置，更新 `ScxLogging` 的 root config，以及按名称或正则表达式匹配的 logger config。

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

每条 `loggers` 配置必须且只能设置 `name` 或 `regex`：`name` 按 logger 名称精确匹配，`regex` 会通过 `Pattern.compile(...)` 编译后匹配。两者都未设置或同时设置时，模块会在启动阶段抛出 `IllegalArgumentException`。

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
    .module(new ScxAppConfigModule(Main.class, args))
    .module(new ScxAppLoggingModule())
    .module(new ScxAppComponentModule())
    .module(new ScxAppHttpModule())
    .module(new ScxAppStaticServerModule())
    .module(new ScxAppWebComponentModule())
    .module(new ScxAppWebModule())
    .module(new ScxAppSchedulingModule())
    .module(new ScxAppCorsModule())
    .module(new ScxAppSQLModule())
    .module(new ScxAppFixTableModule())
    .module(new ScxAppDIModule())
    .module(new Main())
    .run();
```

这里的注册顺序主要是为了可读性。实际启动顺序会由 `scx-app` 根据模块的 `require`、`startBefore` 和 `startAfter` 重新计算。

几个关键关系：

```text
ScxAppDIModule           startAfter ScxAppConfigModule (Config 可选)
ScxAppComponentModule    require ScxAppDIModule, startBefore ScxAppDIModule
ScxAppSQLModule          require ScxAppConfigModule, startAfter ScxAppConfigModule, startBefore ScxAppDIModule
ScxAppWebComponentModule require ScxAppDIModule, startBefore ScxAppDIModule
ScxAppWebModule          require ScxAppHttpModule + ScxAppDIModule, startAfter DI, startBefore HTTP
ScxAppSchedulingModule   require ScxAppDIModule, startAfter ScxAppDIModule
ScxAppStaticServerModule require ScxAppHttpModule + ScxAppConfigModule, startBefore HTTP
ScxAppCorsModule         require ScxAppHttpModule + ScxAppConfigModule, startBefore HTTP
ScxAppFixTableModule     require ScxAppSQLModule + ScxAppConfigModule, startAfter SQL, startBefore HTTP
```

因此，配置会先于依赖它的模块建立；需要向 DI 注册类型或实例的模块会在 DI 容器构建前完成注册；Web、CORS、静态资源和修表会在 HTTP Server 监听端口之前完成。

## 常见问题

### 为什么我的 `@Component` 没有被注入？

通常需要同时满足三个条件：

1. 这个类被 `ScxAutoScanModule` 扫描进 candidates。
2. 应用注册了 `ScxAppComponentModule`，把 `@Component` 类型注册到 DI builder。
3. 应用注册了 `ScxAppDIModule`，最终构建并校验 DI 容器。

如果类不在应用模块类所在包或子包下，`ScxAutoScanModule` 默认不会扫描到它。可以把入口模块放在业务根包，或者重写 `configureCandidates(...)` 补充候选类。

### 为什么我的 `@Routes` Controller 没有生效？

需要确认五点：

1. Controller 类被 `ScxAutoScanModule` 扫描进 candidates。
2. Controller 已进入 DI 容器；典型做法是注册 `ScxAppWebComponentModule`。
3. 应用注册了 `ScxAppDIModule`。
4. 应用注册了 `ScxAppWebModule`。
5. 应用注册了 `ScxAppHttpModule`（HTTP 模块本身还需要 `ScxAppConfigModule`）。

`ScxAppWebModule` 会从 candidates 中查找 `@Routes` 类型，再从构建完成的 DI 容器取得对应实例并注册路由。`@Routes` 类型需要通过组件模块或其他方式注册到 DI 容器。

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
