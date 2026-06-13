# SCX Web

SCX Web 是一个轻量的 Java Web 路由封装库。

它基于 SCX HTTP Routing，将普通 Java 对象中的方法编译成 HTTP / WebSocket 路由，让你可以用注解声明 Controller、用方法参数接收请求数据、用返回值直接生成响应。

当前仓库版本为 `0.7.0`，项目依赖 `scx-http-routing-x`、`scx-websocket-x` 和 `scx-serialize`。

[GitHub](https://github.com/scx-projects/scx-web)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-web</artifactId>
    <version>0.7.0</version>
</dependency>
```

SCX Web 不是一个完整的“应用框架”，它主要负责把带有 `@Routes` / `@Route` 注解的类编译成 `Route`。实际启动 HTTP 服务时，通常需要配合 `scx-http` / `scx-http-x` 和 `scx-http-routing` 使用。典型接入方式是通过 `HttpServer` + `Router` + `new ScxWeb().routes(...)` 的方式接入。

## 快速开始

下面是一个最小 Controller：

```java
import dev.scx.web.annotation.QueryParam;
import dev.scx.web.annotation.Route;
import dev.scx.web.annotation.Routes;

import java.util.Map;

@Routes("/api")
public class UserController {

    @Route("/hello")
    public Object hello() {
        return Map.of("message", "Hello SCX Web");
    }

    @Route("/user")
    public String user(@QueryParam(required = false) String id) {
        return "user id = " + id;
    }

}
```

然后在启动阶段把 Controller 编译成路由，并注册到 `Router`：

```java
import dev.scx.http.routing.Router;
import dev.scx.http.x.HttpServer;
import dev.scx.web.ScxWeb;

import static dev.scx.web.error_handler.DefaultWebErrorHandler.DEFAULT_WEB_ERROR_HANDLER;

public class Main {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer();
        var router = Router.of();

        var scxWeb = new ScxWeb();
        var routes = scxWeb.routes(new UserController());

        for (var route : routes) {
            router.route(route.priority(), route);
        }

        server
            .onRequest(router)
            .onError(DEFAULT_WEB_ERROR_HANDLER)
            .start(8080);
    }

}
```

访问：

```text
GET http://127.0.0.1:8080/api/hello
GET http://127.0.0.1:8080/api/user?id=1001
```

完整示例会展示 `@Route`、`@QueryParam`、普通对象返回、文件下载、内联二进制响应和异常响应等用法。

## 路由声明

### `@Routes`

`@Routes` 用在类上，表示这个类可以被 SCX Web 注册为路由类，同时可以为类中的方法提供路径前缀。

```java
@Routes("/api/users")
public class UserController {
}
```

### `@Route`

`@Route` 用在方法上，表示这个方法会被暴露为一个路由处理方法。它可以声明路径、HTTP 方法、优先级、是否使用绝对路径，以及路由类型。

```java
import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.method.HttpMethod.POST;

@Routes("/api/users")
public class UserController {

    @Route(value = "/list", methods = GET)
    public Object list() {
        return List.of();
    }

    @Route(value = "/create", methods = POST)
    public Object create(@Body User user) {
        return user;
    }

}
```

### 路径拼接规则

SCX Web 不会自动修正路径字符串。类级 `@Routes` 和方法级 `@Route` 会按字面值直接拼接。规则是：不会自动补 `/`、删 `/`、合并 `//`。

```java
@Routes("/api")
@Route("/user")
// => /api/user

@Routes("/get_by")
@Route("_id")
// => /get_by_id

@Routes("/api/")
@Route("/order")
// => /api//order，不会被自动改成 /api/order
```

如果希望方法路由忽略类级前缀，可以使用 `absolute = true`：

```java
@Routes("/api")
public class DemoController {

    @Route(value = "/health", absolute = true)
    public String health() {
        return "OK";
    }

}
```

上面的最终路径是：

```text
/health
```

而不是：

```text
/api/health
```

## 路由方法规则

路由方法必须满足：

```text
必须是 public
不能是 static
必须声明在当前注册类自身中
```

SCX Web 只扫描注册类自身直接声明的方法，不扫描父类继承来的 `@Route` 方法。这是有意的设计：框架不把 Java 继承解释成 HTTP API 继承，目的是让每个 Controller 的对外 API 暴露面局部可见，避免父类改动意外暴露多个子类接口。

推荐写法：

```java
@Routes("/users")
public class UserController {

    @Route("/list")
    public Object list() {
        return userService.list();
    }

}
```

不推荐依赖父类中的 `@Route`：

```java
public abstract class BaseController {

    @Route("/list")
    public Object list() {
        return List.of();
    }

}

@Routes("/users")
public class UserController extends BaseController {
    // 父类的 @Route 不会因为注册 UserController 而自动暴露
}
```

## 路由优先级

`@Route(priority = ...)` 可以指定路由优先级。数值越小，优先级越高。排序规则大致为：

```text
1. priority 小的优先
2. 精确路径优先于通配路径
3. 路径参数数量少的优先
4. HTTP 方法更具体的优先
```

例如：

```java
@Route(value = "/users/list", priority = 0)
public Object list() {
    return List.of();
}

@Route(value = "/users/:id", priority = 10)
public Object detail(@PathCapture String id) {
    return id;
}
```

SCX Web 编译出的 `ScxWebRoute` 本身带有 `priority()`，注册到 `Router` 时应保留这个优先级：

```java
for (var route : new ScxWeb().routes(new UserController())) {
    router.route(route.priority(), route);
}
```

注册路由时保留 `priority()`，就能让这些排序规则正常生效。

## 参数绑定

SCX Web 的一个重要原则是：普通业务参数必须显式声明来源。它不会根据参数名、参数类型、请求内容、path、query 或 body 自动猜测参数来源。默认兜底参数处理器会在无法确定参数来源时直接报错。

可用参数来源包括：

```java
@PathCapture
@QueryParam
@QueryParams
@Body
@BodyField
@Part
```

少量上下文类型参数可以不加注解，例如 `RoutingContext`、`ScxHttpServerRequest`、`ScxHttpServerResponse`、headers、cookies、body input，以及 WebSocket 握手请求 / 响应对象。

### 路径参数：`@PathCapture`

从路径模板中的命名捕获读取值。

```java
@Routes("/users")
public class UserController {

    @Route("/:id")
    public Object detail(@PathCapture String id) {
        return Map.of("id", id);
    }

}
```

也可以显式指定捕获名：

```java
@Route("/:user_id")
public Object detail(@PathCapture("user_id") String id) {
    return Map.of("id", id);
}
```

`@PathCapture` 默认使用方法参数名作为捕获名称；显式传值时使用注解中的名称。

### 查询参数：`@QueryParam`

从 URL query 中读取单个参数。

```java
@Route("/search")
public Object search(
    @QueryParam String keyword,
    @QueryParam(required = false) Integer page
) {
    return Map.of(
        "keyword", keyword,
        "page", page
    );
}
```

请求示例：

```text
GET /search?keyword=scx&page=1
```

`@QueryParam` 默认使用参数名作为 query 名称，`required` 默认为 `true`；缺少必填参数会抛出 400 类异常。

### 查询对象：`@QueryParams`

把整个 query parameters 转成对象。

```java
public record PageQuery(String keyword, Integer page, Integer size) {
}

@Route("/search")
public Object search(@QueryParams PageQuery query) {
    return query;
}
```

请求示例：

```text
GET /search?keyword=scx&page=1&size=20
```

`@QueryParams` 会把 query parameters 解析成对象结构，并按照参数类型进行转换。

### 请求体：`@Body`

把整个请求体解析后绑定到参数。

```java
public record CreateUserRequest(String name, Integer age) {
}

@Route(value = "/users", methods = POST)
public Object create(@Body CreateUserRequest body) {
    return body;
}
```

支持的结构化内容类型包括：

```text
application/json
application/xml
application/x-www-form-urlencoded
multipart/form-data
```

其中 `multipart/form-data` 的 `@Body` 只包含表单字段，不包含文件 part；如果要读取文件或原始 multipart part，请使用 `@Part`。`text/plain`、`application/octet-stream` 等原始或二进制内容类型不会通过 `@Body` 结构化解析。

### 请求体字段：`@BodyField`

从结构化 body 的某个字段中取值。

```java
@Route(value = "/login", methods = POST)
public Object login(
    @BodyField String username,
    @BodyField String password
) {
    return Map.of("username", username);
}
```

请求体：

```json
{
  "username": "scx",
  "password": "123456"
}
```

也可以显式指定字段名：

```java
@BodyField("user_name")
String username
```

`@BodyField` 默认使用参数名作为字段名，`required` 默认为 `true`。

### Multipart：`@Part`

从 `multipart/*` 请求体中按名称读取 part。当前支持绑定到：

```java
MultiPartPart
MultiPartPart[]
```

示例：

```java
import dev.scx.http.media.multi_part.MultiPartPart;
import dev.scx.web.annotation.Part;

@Route(value = "/upload", methods = POST)
public Object upload(@Part("file") MultiPartPart file) {
    return Map.of("filename", file.filename());
}
```

多个文件：

```java
@Route(value = "/upload-many", methods = POST)
public Object uploadMany(@Part("files") MultiPartPart[] files) {
    return Map.of("count", files.length);
}
```

`@Part` 默认使用参数名作为 part 名称，`required` 默认为 `true`。如果绑定到单个 `MultiPartPart` 但实际匹配到多个 part，会抛出参数转换异常。

### 上下文参数

以下类型可以直接作为路由方法参数，不需要绑定注解：

```java
RoutingContext
ScxHttpServerRequest
ScxHttpServerResponse
ScxServerWebSocketHandshakeRequest
ScxServerWebSocketHandshakeResponse
ScxHttpHeaders
ByteInput
Cookies
```

示例：

```java
@Route("/headers")
public Object headers(ScxHttpHeaders headers) {
    return headers;
}

@Route("/raw")
public void raw(ScxHttpServerRequest request) {
    request.response().send("raw response");
}
```

这些上下文类型由默认的 `ContextParameterHandlerBuilder` 处理。

## 返回值处理

SCX Web 会根据方法返回值选择返回值处理器。默认注册了：

```text
NullReturnValueHandler
StringReturnValueHandler
WebResultReturnValueHandler
LastReturnValueHandler
```

其中 `null` 会发送空响应，`String` 会直接发送字符串，`WebResult` 会调用对应结果对象的 `apply` 方法，其他对象会进入兜底处理器，按 `Accept` 协商 JSON 或 XML，默认 JSON。

### 返回普通对象

```java
@Route("/user")
public Object user() {
    return Map.of("name", "scx");
}
```

默认会返回 JSON；如果请求 `Accept` 明确协商 XML，也可以返回 XML。

### 返回字符串

```java
@Route("/text")
public String text() {
    return "Hello SCX Web";
}
```

### 返回 JSON

```java
import dev.scx.web.result.Json;

@Route("/json")
public Object json() {
    return Json.of(Map.of("ok", true));
}
```

`Json.of(...)` 会设置 `application/json; charset=utf-8` 并使用 SCX Serialize 转 JSON。

### 返回 XML

```java
import dev.scx.web.result.Xml;

@Route("/xml")
public Object xml() {
    return Xml.of(Map.of("ok", true));
}
```

`Xml.of(...)` 会设置 `application/xml; charset=utf-8` 并使用 SCX Serialize 转 XML。

### 返回 HTML

```java
import dev.scx.web.result.Html;

@Route("/page")
public Object page() {
    return Html.of("<h1>Hello SCX Web</h1>");
}
```

`Html.of(...)` 会设置 `text/html; charset=utf-8`。

### 文件下载

```java
import dev.scx.web.result.Binary;

import java.io.File;

@Route("/download")
public Object download() {
    return Binary.download(new File("report.pdf"));
}
```

也可以直接发送字节数组：

```java
@Route("/download-text")
public Object downloadText() {
    return Binary.download("hello".getBytes(), "hello.txt");
}
```

`Binary.download(...)` 会设置附件形式的 `Content-Disposition`，并根据文件名推断 `Content-Type`；`Binary.inline(...)` 则用于浏览器内联预览，例如 PDF、图片、文本等。

### 重定向

```java
import dev.scx.web.result.Redirect;

@Route("/old")
public Object oldPage() {
    return Redirect.ofPermanent("/new");
}

@Route("/login-success")
public Object loginSuccess() {
    return Redirect.ofSeeOther("/home");
}
```

可用工厂方法包括：

```java
Redirect.ofTemporary(location)
Redirect.ofPermanent(location)
Redirect.ofMovedPermanently(location)
Redirect.ofFound(location)
Redirect.ofSeeOther(location)
```

`Redirect` 会设置 `Location` 响应头和对应状态码。

## 拦截器

可以通过 `ScxWeb#interceptor(...)` 设置全局拦截器。拦截器支持：

```java
preHandle(...)
postHandle(...)
```

`preHandle` 在路由方法执行前调用；如需中断执行，可以直接抛出异常。`postHandle` 在路由方法执行后、结果写回客户端前调用，可以替换返回值。

`ScxWeb` 当前只维护一个 `Interceptor` 槽位。这个设计不是限制只能有一个拦截逻辑，而是规定框架只接受一个统一的拦截入口；如果需要鉴权、日志、异常记录等多个步骤，应在自己的 `Interceptor` 实现内部组合这些步骤。`ScxWeb` 本身不维护拦截器列表，也不定义列表顺序、短路、异常传播或 `postHandle` 回放规则。

```java
var scxWeb = new ScxWeb();

scxWeb.interceptor(new Interceptor() {

    @Override
    public void preHandle(RoutingContext ctx, ScxWebRoute route) throws Exception {
        System.out.println("before: " + route);
    }

    @Override
    public Object postHandle(RoutingContext ctx, ScxWebRoute route, Object result) throws Exception {
        System.out.println("after: " + route);
        return result;
    }

});
```

如果路由方法返回 `void`，`postHandle` 收到的 `result` 是 `null`。

## 错误处理

SCX Web 提供了默认错误处理器：

```java
import static dev.scx.web.error_handler.DefaultWebErrorHandler.DEFAULT_WEB_ERROR_HANDLER;

server.onError(DEFAULT_WEB_ERROR_HANDLER);
```

默认错误处理器会根据客户端 `Accept` 返回 HTML 或 JSON 错误信息；如果异常实现了 `ScxHttpException`，会使用异常自身的 HTTP 状态码，否则默认是 500。开发模式下还会把堆栈信息写入响应。

参数解析相关错误，例如请求体解析失败、参数转换失败、必填参数缺失，都会映射为 `400 Bad Request`。

## WebSocket 升级路由

`@Route` 的 `kind` 可以设置为 `WEBSOCKET_UPGRADE`，用于处理 WebSocket 升级请求。此时 `methods` 不参与匹配语义。

```java
import dev.scx.web.annotation.Route;
import dev.scx.web.annotation.Routes;

import static dev.scx.web.annotation.Route.RouteKind.WEBSOCKET_UPGRADE;

@Routes("/ws")
public class WebSocketController {

    @Route(value = "/chat", kind = WEBSOCKET_UPGRADE)
    public void chat(ScxServerWebSocketHandshakeRequest request,
                     ScxServerWebSocketHandshakeResponse response) {
        // 在这里处理 WebSocket 握手
    }

}
```

## 扩展参数处理器

如果你希望支持自定义参数来源，例如 Header、Cookie、Session、认证用户等，可以实现 `ParameterHandlerBuilder` 并注册到 `ScxWeb`。

```java
public class CurrentUserParameterHandlerBuilder implements ParameterHandlerBuilder {

    @Override
    public ParameterHandler tryBuild(ParameterInfo parameter) {
        if (parameter.parameterType().rawClass() != CurrentUser.class) {
            return null;
        }

        return requestInfo -> {
            var token = requestInfo.routingContext()
                .request()
                .headers()
                .get("Authorization");

            return loadUserByToken(token);
        };
    }

}
```

注册：

```java
var scxWeb = new ScxWeb()
    .addParameterHandlerBuilder(0, new CurrentUserParameterHandlerBuilder());
```

`addParameterHandlerBuilder(index, builder)` 可以把处理器插到默认处理链前面；如果所有处理器都不能处理某个普通参数，兜底处理器会抛出“无法确定参数来源”的错误。

## 扩展返回值处理器

如果你希望支持自定义返回值类型，可以实现 `ReturnValueHandler`。

```java
public final class CsvReturnValueHandler implements ReturnValueHandler {

    @Override
    public boolean canHandle(Object returnValue) {
        return returnValue instanceof CsvResult;
    }

    @Override
    public void handle(Object returnValue,
                       ScxHttpServerRequest request,
                       ScxWeb scxWeb) throws Exception {
        var csv = (CsvResult) returnValue;

        request.response()
            .contentType(/* text/csv */)
            .send(csv.content());
    }

}
```

注册：

```java
var scxWeb = new ScxWeb()
    .addReturnValueHandler(0, new CsvReturnValueHandler());
```

返回值处理器会按注册顺序尝试匹配；最后兜底处理器会把普通对象序列化为 JSON 或 XML。

## 设计约定

### 1. 路由声明必须局部可见

SCX Web 不扫描父类中的 `@Route`。想暴露 HTTP endpoint，必须在当前 Controller 类中显式声明 `@Route`。这样可以避免父类修改导致多个子类意外新增接口。

### 2. 参数来源必须显式

除了少数上下文类型参数，普通业务参数必须使用注解说明来源。SCX Web 不会自动猜测 `String id` 是来自 path、query、body 还是其他位置。

### 3. 路径不会被自动归一化

`@Routes("/api/") + @Route("/user")` 的结果就是 `/api//user`。框架不会替用户合并斜杠。

### 4. Controller 负责声明 HTTP 边界，业务复用放到 Service

推荐把 HTTP 接口声明留在 Controller 当前类中，把复用逻辑放到 service、repository、helper 或普通父类方法中。父类可以复用实现，但不要期待父类路由自动成为子类 API。

## 完整示例

```java
import dev.scx.http.media.multi_part.MultiPartPart;
import dev.scx.web.annotation.*;
import dev.scx.web.result.Binary;
import dev.scx.web.result.Html;
import dev.scx.web.result.Json;
import dev.scx.web.result.Redirect;

import java.io.File;
import java.util.List;
import java.util.Map;

import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.method.HttpMethod.POST;

@Routes("/api")
public class DemoController {

    @Route(value = "/hello", methods = GET)
    public String hello() {
        return "Hello SCX Web";
    }

    @Route(value = "/users/:id", methods = GET)
    public Object user(@PathCapture String id) {
        return Map.of("id", id, "name", "scx");
    }

    @Route(value = "/search", methods = GET)
    public Object search(@QueryParam String keyword,
                         @QueryParam(required = false) Integer page) {
        return Map.of("keyword", keyword, "page", page);
    }

    @Route(value = "/users", methods = POST)
    public Object create(@Body CreateUserRequest body) {
        return Json.of(body);
    }

    @Route(value = "/login", methods = POST)
    public Object login(@BodyField String username,
                        @BodyField String password) {
        return Map.of("username", username);
    }

    @Route(value = "/upload", methods = POST)
    public Object upload(@Part("file") MultiPartPart file) {
        return Map.of("filename", file.filename());
    }

    @Route("/html")
    public Object html() {
        return Html.of("<h1>Hello SCX Web</h1>");
    }

    @Route("/download")
    public Object download() {
        return Binary.download(new File("demo.txt"));
    }

    @Route("/go")
    public Object redirect() {
        return Redirect.ofFound("/api/hello");
    }

    public record CreateUserRequest(String name, Integer age) {
    }

}
```

启动：

```java
import dev.scx.http.routing.Router;
import dev.scx.http.x.HttpServer;
import dev.scx.web.ScxWeb;

import static dev.scx.web.error_handler.DefaultWebErrorHandler.DEFAULT_WEB_ERROR_HANDLER;

public class Main {

    public static void main(String[] args) throws Exception {
        var server = new HttpServer();
        var router = Router.of();

        var scxWeb = new ScxWeb();
        var routes = scxWeb.routes(new DemoController());

        for (var route : routes) {
            router.route(route.priority(), route);
        }

        server
            .onRequest(router)
            .onError(DEFAULT_WEB_ERROR_HANDLER)
            .start(8080);

        System.out.println("SCX Web started at http://127.0.0.1:8080");
    }

}
```

## 常见问题

### 为什么我的参数没有被绑定？

普通参数必须显式声明来源。例如下面这样会报错：

```java
@Route("/users/:id")
public Object user(String id) {
    return id;
}
```

应该改为：

```java
@Route("/users/:id")
public Object user(@PathCapture String id) {
    return id;
}
```

### 为什么父类里的 `@Route` 没生效？

SCX Web 只扫描当前注册类自身声明的方法。父类可以放业务复用逻辑，但父类中的 `@Route` 不会自动成为子类路由。

### 为什么 `/api//user` 没被自动改成 `/api/user`？

SCX Web 不做路径归一化。路径模板按注解字面值拼接。请在 `@Routes` 和 `@Route` 中自己保持斜杠风格一致。

### 普通对象返回的是 JSON 还是 XML？

默认是 JSON。如果请求头 `Accept` 可以协商到 XML，则兜底返回值处理器可以返回 XML。
