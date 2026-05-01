# SCX HTTP Routing

SCX HTTP Routing 是一个轻量的 HTTP 路由匹配库。

它提供 `Router`、`Route`、`RoutingContext`、`PathMatcher`、`MethodMatcher`、`RequestMatcher` 和 `RouteTable` 等基础组件，用来把 `ScxHttpServerRequest` 分发到对应的处理函数。它本身不负责启动 HTTP 服务，而是作为 `scx-http` 请求处理链中的路由层使用。当前版本为 `0.2.0`，依赖 `scx-http`。([GitHub][1])

## 安装

### Maven

```xml id="k49lnj"
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-http-routing</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX HTTP Routing 中最常用的概念包括：

```text id="f4d9yu"
Router             路由入口，方便接入 HTTP Server
Route              一条路由，由请求匹配器、路径匹配器、方法匹配器和 handler 组成
RoutingContext     当前路由上下文，包含 request、pathMatch、data 和 next()
PathMatcher        路径匹配器
MethodMatcher      HTTP 方法匹配器
RequestMatcher     请求对象匹配器
RouteTable         路由表
PriorityRouteTable 带优先级的路由表
RoutingInput       路由输入视图，当前只包含 path
RoutingPath        路由路径对象
```

`Router` 是一个便捷入口类，用于把路由系统接入 HTTP Server；源码注释也明确说明，它不是路由系统的核心抽象，真正的路由语义由 `Route`、`RouteTable`、`RequestMatcher`、`PathMatcher` 和 `MethodMatcher` 决定。([GitHub][2])

## 快速开始

```java id="y9wko1"
import dev.scx.http.routing.Router;

public class Main {

    public static void main(String[] args) {
        var router = Router.of();

        router.get("/hello", ctx -> {
            ctx.request().response().send("hello");
        });

        router.get("/users/:id", ctx -> {
            var id = ctx.pathMatch().capture("id");
            ctx.request().response().send("user id = " + id);
        });

        // 交给 scx-http 的 server 使用
        // httpServer.onRequest(router);
    }

}
```

测试代码中也展示了这种 DSL 风格：`router.get("/hello", ...)`、`router.get("/path-params/:id", ...)`、`router.post("/405", ...)` 和 `router.mount("/api/v2/*", ...)`。([GitHub][3])

## Router

`Router` 是最常用的入口。

```java id="ubadz2"
Router router = Router.of();
```

它本身实现了 `Function1Void`，可以作为 HTTP Server 的 request handler 使用。处理请求时，`Router#apply(...)` 会根据 `request.path()` 创建 `RoutingInput`，创建共享 `data`，再创建 `RoutingContext` 并调用 `next()` 开始路由分发。([GitHub][2])

```java id="g0kmvq"
httpServer.onRequest(router);
```

### 注册路由

```java id="w8dmm4"
router.get("/hello", ctx -> {
    ctx.request().response().send("hello");
});

router.post("/users", ctx -> {
    ctx.request().response().send("create user");
});

router.put("/users/:id", ctx -> {
    ctx.request().response().send("update user");
});

router.delete("/users/:id", ctx -> {
    ctx.request().response().send("delete user");
});
```

`Router` 提供 `get`、`post`、`put`、`delete` 等快捷方法，也提供更通用的 `route(...)` 方法。快捷方法底层会创建 `Route.of(PathMatcher.ofTemplate(template), MethodMatcher.of(method), handler)`。([GitHub][2])

### 匹配任意 HTTP 方法

```java id="6lobnn"
router.route("/ping", ctx -> {
    ctx.request().response().send("pong");
});
```

`route(String template, handler)` 会匹配任意 HTTP method 和指定路径模板。([GitHub][2])

### 匹配任意路径和任意方法

```java id="qzg3o8"
router.route(ctx -> {
    System.out.println("请求进入路由系统");
    ctx.next();
});
```

`route(handler)` 会使用 `PathMatcher.any()` 和 `MethodMatcher.any()`，因此可以用作全局前置处理器或兜底处理器。([GitHub][2])

## 路由优先级

注册路由时可以指定 priority：

```java id="8rwy2y"
router.route(-1, "/*", ctx -> {
    System.out.println("before all: " + ctx.pathMatch().capture("*"));
    ctx.next();
});

router.get("/hello", ctx -> {
    ctx.request().response().send("hello");
});
```

`PriorityRouteTable` 的优先级规则是：数字越小越先匹配；相同 priority 按注册顺序匹配。默认添加路由时 priority 为 `0`。([GitHub][4])

内部实现使用有序列表保存 `PriorityRouteEntry`，添加路由时会插入到对应优先级区间的末尾，因此相同 priority 能保持注册顺序。([GitHub][5])

## RoutingContext

路由 handler 接收的是 `RoutingContext`。

```java id="2k8wi8"
router.get("/users/:id", ctx -> {
    var request = ctx.request();
    var id = ctx.pathMatch().capture("id");

    request.response().send("id = " + id);
});
```

`RoutingContext` 提供：

```java id="dzgmyx"
ScxHttpServerRequest request();

RoutingInput routingInput();

PathMatch pathMatch();

Map data();

void next();

<T> T request(Class<T> requestType);
```

其中 `request()` 是原始请求，`routingInput()` 是当前路由输入视图，`pathMatch()` 是当前路由路径匹配结果，`data()` 是当前路由链共享数据，`next()` 用于继续匹配下一条路由。([GitHub][6])

### 继续匹配下一条路由

```java id="7igdtp"
router.route(-1, "/*", ctx -> {
    System.out.println("进入: " + ctx.routingInput().path().value());
    ctx.next();
});

router.get("/hello", ctx -> {
    ctx.request().response().send("hello");
});
```

`next()` 会继续从当前路由表中查找后续候选路由。匹配过程依次检查 `RequestMatcher`、`PathMatcher`、`MethodMatcher`；匹配成功后执行当前路由 handler 并返回。([GitHub][7])

如果没有任何路径匹配，会抛出 `NotFoundException`；如果存在路径匹配但方法不匹配，会抛出 `MethodNotAllowedException`。([GitHub][7])

### 共享 data

```java id="04ejtw"
router.route(-1, "/*", ctx -> {
    ctx.data().put("startTime", System.nanoTime());
    ctx.next();
});

router.get("/hello", ctx -> {
    var startTime = ctx.data().get("startTime");
    ctx.request().response().send("hello");
});
```

`Router#apply(...)` 会为一次请求创建一个 `HashMap` 作为 data；子路由挂载时也会共享同一个 request 和同一个 data。([GitHub][2])

## 路径模板

最常用的路径匹配器是模板路径匹配器。

```java id="fjqfjt"
router.get("/users/:id", ctx -> {
    var id = ctx.pathMatch().capture("id");
    ctx.request().response().send(id);
});
```

模板必须以 `/` 开头，由 `/` 分隔的 segment 组成。支持三类 segment：

```text id="6f4ad8"
静态段    例如 /users/list
参数段    例如 /users/:id
通配段    例如 /assets/*
```

`TemplatePathMatcher` 的源码注释说明：静态段必须严格相等；参数段 `:name` 匹配且仅匹配一个路径段；通配段 `*` 匹配剩余的零个或多个路径段，并且必须是模板最后一段。([GitHub][8])

### 静态路径

```java id="8ri1vb"
router.get("/hello", ctx -> {
    ctx.request().response().send("hello");
});
```

### 参数路径

```java id="laknvo"
router.get("/users/:id/books/:bookId", ctx -> {
    var userId = ctx.pathMatch().capture("id");
    var bookId = ctx.pathMatch().capture("bookId");

    ctx.request().response().send(userId + " - " + bookId);
});
```

参数也可以按捕获顺序读取：

```java id="i43jhx"
router.get("/users/:id/books/:bookId", ctx -> {
    var userId = ctx.pathMatch().capture(0);
    var bookId = ctx.pathMatch().capture(1);
});
```

`PathMatch` 支持按 index 读取捕获值，也支持按名称读取捕获值；不存在的 index 或 name 返回 `null`。([GitHub][9])

### 通配路径

```java id="ll76cf"
router.get("/assets/*", ctx -> {
    var rest = ctx.pathMatch().capture("*");
    ctx.request().response().send("assets path = " + rest);
});
```

通配段 `*` 会捕获剩余路径，而且捕获值是原始请求路径的子串，不做归一化。源码注释给出的语义是：`/api/*` 匹配 `/api` 时 `* = ""`，匹配 `/api/` 时 `* = "/"`，匹配 `/api/user` 时 `* = "/user"`。([GitHub][8])

测试代码中也验证了 `/a/b/*` 对 `/a/b`、`/a/b/`、`/a/b/c/d` 的捕获结果。([GitHub][10])

## 子路由挂载

`Router#mount(...)` 可以把一个子路由挂载到指定前缀下。

```java id="int30k"
router.mount("/api/v2/*", api -> {
    api.get("/users", ctx -> {
        ctx.request().response().send("user list");
    });

    api.get("/orders", ctx -> {
        ctx.request().response().send("order list");
    });
});
```

`mount` 的模板必须是尾部带 `*` 的纯静态模板，例如 `/*` 或 `/api/*`；不允许包含 `:name` 这种参数段。匹配成功后，会把 `*` 捕获到的剩余路径交给子路由继续匹配；父子路由共享同一个 request 和同一个 data。([GitHub][2])

当剩余路径为空字符串时，`MountedRouterHandler` 会先把它归一化为 `/`，再交给子路由；子路由中的 `next()` 只会在子路由自身范围内继续，不会自动回到父路由继续匹配后续路由。([GitHub][2])

### 挂载已有 Router

```java id="5vmt1d"
var api = Router.of();

api.get("/users", ctx -> {
    ctx.request().response().send("users");
});

router.mount("/api/*", api);
```

### 指定挂载优先级

```java id="yk5it4"
router.mount(-1, "/api/*", api -> {
    api.get("/health", ctx -> {
        ctx.request().response().send("ok");
    });
});
```

`Router` 同时提供 `mount(template, childRouter)`、`mount(priority, template, childRouter)`、`mount(template, childRouterBuilder)` 和 `mount(priority, template, childRouterBuilder)`。([GitHub][2])

## PathMatcher

如果快捷 DSL 不够用，可以直接创建 `Route` 并指定路径匹配器。

```java id="z20g0i"
import dev.scx.http.routing.Route;
import dev.scx.http.routing.path_matcher.PathMatcher;
import dev.scx.http.routing.method_matcher.MethodMatcher;

import static dev.scx.http.method.HttpMethod.GET;

router.route(
    Route.of(
        PathMatcher.ofExact("/hello"),
        MethodMatcher.of(GET),
        ctx -> ctx.request().response().send("hello")
    )
);
```

`PathMatcher` 提供：

```java id="2wx0y8"
PathMatcher.any();

PathMatcher.ofExact("/hello");

PathMatcher.ofTemplate("/users/:id");

PathMatcher.ofRegex("/users/(\\d+)");
```

`PathMatcher#match(...)` 匹配失败返回 `null`，匹配成功返回 `PathMatch`；`PathMatcher.ofExact(...)` 和 `PathMatcher.ofTemplate(...)` 都要求路径以 `/` 开头。([GitHub][11])

### 精确路径匹配

```java id="p2qje2"
var matcher = PathMatcher.ofExact("/hello");
```

`ExactPathMatcher` 要求路径完全相等；不相等时返回 `null`，相等时返回空的 `PathMatch`。([GitHub][12])

### 模板路径匹配

```java id="mk9ue5"
var matcher = PathMatcher.ofTemplate("/users/:id");
var match = matcher.match(RoutingPath.of("/users/100"));

System.out.println(match.capture("id")); // 100
```

模板路径是严格匹配，不会自动修正路径，也不会做宽松匹配。`TemplatePathMatcher` 源码注释明确说明：它采用严格、可预测的匹配语义，不包含隐式规则或自动修正行为。([GitHub][8])

### 正则路径匹配

```java id="4eu6d3"
var matcher = PathMatcher.ofRegex("/users/(\\d+)/books/(\\d+)");
var match = matcher.match(RoutingPath.of("/users/123/books/456"));

System.out.println(match.capture(0)); // 123
System.out.println(match.capture(1)); // 456
```

正则匹配使用 `Pattern.matcher(path.value()).matches()`，因此是全路径匹配，不是查找子串。测试代码也验证了 `/users/(\\d+)` 不会匹配 `/foo/users/123/bar`。([GitHub][13])

命名捕获也可以通过 `capture("name")` 读取。`RegexPathMatcher` 会把 Java 正则中的 named group 从 1-based group number 转成 `PathMatch` 的 0-based index。([GitHub][13])

## MethodMatcher

`MethodMatcher` 用于匹配 HTTP method。

```java id="0wv6yk"
import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.method.HttpMethod.POST;

var matcher = MethodMatcher.of(GET, POST);
```

`MethodMatcher.any()` 匹配任意方法；`MethodMatcher.of(...)` 至少需要一个 method，传入一个 method 时返回单方法匹配器，传入多个时返回多方法匹配器。([GitHub][14])

```java id="77w0z7"
router.route(
    Route.of(
        PathMatcher.ofTemplate("/users"),
        MethodMatcher.of(GET, POST),
        ctx -> {
            ctx.request().response().send("GET or POST");
        }
    )
);
```

`AnyMethodMatcher` 永远返回 `true`；`SingleMethodMatcher` 使用 `equals` 判断方法；`MultiMethodMatcher` 使用集合判断方法是否包含。([GitHub][15])

## RequestMatcher

`RequestMatcher` 用于在路径和方法之外，再根据请求对象本身做匹配。

```java id="8p4ovg"
import dev.scx.http.routing.request_matcher.RequestMatcher;

var route = Route.of(
    RequestMatcher.any(),
    PathMatcher.ofTemplate("/hello"),
    MethodMatcher.any(),
    ctx -> {
        ctx.request().response().send("hello");
    }
);
```

`RequestMatcher` 提供：

```java id="hupfk6"
RequestMatcher.any();

RequestMatcher.typeIs(SomeRequestClass.class);

RequestMatcher.typeNot(SomeRequestClass.class);
```

`typeIs` 使用 `requestType.isInstance(request)` 判断；`typeNot` 则取反。([GitHub][16])

这个机制适合区分不同 request 实现类型，或在扩展场景中按请求对象能力分流。

## Route

`Route` 是一条路由的完整描述。

```java id="00iyry"
Route route = Route.of(
    RequestMatcher.any(),
    PathMatcher.ofTemplate("/users/:id"),
    MethodMatcher.of(GET),
    ctx -> {
        var id = ctx.pathMatch().capture("id");
        ctx.request().response().send(id);
    }
);

router.route(route);
```

`Route` 包含四部分：

```java id="96no1g"
RequestMatcher requestMatcher();

PathMatcher pathMatcher();

MethodMatcher methodMatcher();

Function1Void handler();
```

`Route.of(pathMatcher, methodMatcher, handler)` 会默认使用 `RequestMatcher.any()`；`RouteImpl` 只保存状态，不做行为处理，并且会检查四个组成部分都不能为 `null`。([GitHub][17])

## RouteTable

`RouteTable` 是路由表抽象。

```java id="m7lmc1"
RouteTable table = router.routeTable();
```

`RouteTable#candidates(RoutingInput)` 返回针对当前 `RoutingInput` 的候选路由序列。源码注释说明：实现可以基于 `routingInput` 做安全的预筛选或索引优化，但不得遗漏任何可能匹配成功的路由。([GitHub][18])

默认 `Router` 使用的是 `PriorityRouteTable`：

```java id="7a0zpm"
PriorityRouteTable table = router.routeTable();
```

`PriorityRouteTable` 支持添加路由、移除路由和查看只读快照。`remove(route)` 使用对象身份 `==` 移除所有使用该 `Route` 实例注册的条目，不使用 `equals`。([GitHub][4])

```java id="rcx37p"
Route route = Route.of(
    PathMatcher.ofTemplate("/hello"),
    MethodMatcher.any(),
    ctx -> ctx.request().response().send("hello")
);

router.route(route);

// 之后移除
router.remove(route);
```

## RoutingInput 和 RoutingPath

`RoutingInput` 是路由系统进行匹配和分流时使用的输入视图。它只建模 URI path，不包含 query。源码注释明确说明：query 属于原始请求 URI 的事实属性，不作为可重写的路由视图；如果需要读取或匹配 query，应通过 `request.uri()` 或 `RequestMatcher` 完成。([GitHub][19])

```java id="pqe0gn"
var input = RoutingInput.of("/users/100");
var path = input.path();

System.out.println(path.value());        // /users/100
System.out.println(path.segmentCount()); // 2
System.out.println(path.segment(0));     // users
System.out.println(path.segment(1));     // 100
```

`RoutingPath` 表示已解码后的 path，并按 `/` 切分 segment；实现中使用 `split("/", -1)`，因此会保留尾部空段，例如 `/a/` 的尾部空段不会被丢弃。([GitHub][20])

## 错误语义

路由分发失败时主要有两种结果：

```text id="fb1yrv"
没有任何路径匹配         -> NotFoundException
路径匹配但 HTTP 方法不匹配 -> MethodNotAllowedException
```

匹配逻辑中会先检查请求匹配器，再检查路径匹配器，最后检查方法匹配器；如果某条路由路径匹配但方法不匹配，会先记为 405，但继续尝试后续路由，因为后面可能还有其他路由能匹配成功。([GitHub][7])

## 完整示例

```java id="ejvhmh"
import dev.scx.http.ScxHttpServer;
import dev.scx.http.routing.Router;

import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.routing.Route.of;
import static dev.scx.http.routing.method_matcher.MethodMatcher.any;
import static dev.scx.http.routing.method_matcher.MethodMatcher.of;
import static dev.scx.http.routing.path_matcher.PathMatcher.ofRegex;

public class RoutingExample {

    public static void main(String[] args) throws Exception {
        var router = Router.of();

        // 全局前置处理
        router.route(-10, "/*", ctx -> {
            System.out.println("request path = " + ctx.routingInput().path().value());
            ctx.next();
        });

        // 普通路由
        router.get("/hello", ctx -> {
            ctx.request().response().send("hello");
        });

        // 模板参数
        router.get("/users/:id", ctx -> {
            var id = ctx.pathMatch().capture("id");
            ctx.request().response().send("user id = " + id);
        });

        // 子路由
        router.mount("/api/*", api -> {
            api.get("/health", ctx -> {
                ctx.request().response().send("ok");
            });

            api.get("/users/:id", ctx -> {
                ctx.request().response().send("api user = " + ctx.pathMatch().capture("id"));
            });
        });

        // 正则路由
        router.route(
            of(
                ofRegex("/books/(\\d+)"),
                of(GET),
                ctx -> {
                    var bookId = ctx.pathMatch().capture(0);
                    ctx.request().response().send("book id = " + bookId);
                }
            )
        );

        // 兜底路由
        router.route(1000, ctx -> {
            ctx.request().response().send("fallback");
        });

        // 交给 scx-http server
        // ScxHttpServer server = ...
        // server.onRequest(router);
        // server.start(...);
    }

}
```

## 设计说明

### 1. Router 是便捷入口，不是核心模型

`Router` 负责把请求包装成 `RoutingContext` 并启动分发。真正的匹配语义来自 `Route`、`RouteTable`、`RequestMatcher`、`PathMatcher` 和 `MethodMatcher`。如果你需要完全控制路由装配、生命周期或调度方式，可以绕过 `Router`，直接使用 `RouteTable` 和自定义调度逻辑。([GitHub][2])

### 2. 匹配顺序是显式的

每条路由的匹配顺序是：

```text id="r7uh5p"
RequestMatcher -> PathMatcher -> MethodMatcher -> handler
```

这个顺序体现在 `RoutingContextImpl#next()` 中。([GitHub][7])

### 3. 路由路径视图只包含 path

`RoutingInput` 只包含 path，不包含 query、headers 或 method。query 和 headers 仍然属于原始 request 的事实属性；需要匹配这些信息时，应该通过 `RequestMatcher` 或直接读取 request 完成。([GitHub][19])

### 4. 模板路径严格、可预测

模板路径不会自动修正，也不会做宽松匹配。通配符只允许出现在最后一段；参数名不能为空；参数名不能是 `*`；重复参数名会抛出异常。([GitHub][8])

### 5. 子路由不会自动回到父路由

`mount` 后，子路由中的 `next()` 只在子路由自身范围内继续；它不会自动回到父路由继续匹配父路由后续条目。([GitHub][2])

## 常见问题

### SCX HTTP Routing 会启动 HTTP Server 吗？

不会。它提供的是路由分发层。`Router` 可以作为 `scx-http` 的 request handler 使用，但真正的 HTTP Server 由其他模块或实现提供。`Router#apply(...)` 接收的是 `ScxHttpServerRequest`。([GitHub][2])

### 路径参数怎么读取？

使用 `ctx.pathMatch().capture("name")`：

```java id="6srosp"
router.get("/users/:id", ctx -> {
    var id = ctx.pathMatch().capture("id");
});
```

`PathMatch` 支持按名称和按捕获顺序读取。([GitHub][9])

### 为什么 `/api/*` 匹配 `/api` 时捕获值是空字符串？

这是刻意设计。`*` 捕获的是原始请求路径的剩余子串，不做归一化；因此 `/api` 的剩余部分是 `""`，`/api/` 的剩余部分是 `/`。([GitHub][8])

### 为什么路径匹配了但返回 405？

当某些路由路径匹配成功但 HTTP method 不匹配，并且后续也没有其他路由匹配成功时，`RoutingContextImpl#next()` 会抛出 `MethodNotAllowedException`。([GitHub][7])

### priority 数值越大越优先吗？

不是。数值越小越优先。相同 priority 按注册顺序匹配。([GitHub][4])

### `Router#mount` 的模板可以写 `/api/:version/*` 吗？

不可以。`mount` 要求模板必须是尾部带 `*` 的纯静态模板，不允许包含参数段。([GitHub][2])

[1]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/Router.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/test/java/dev/scx/http/routing/test/DSLTest.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/route_table/PriorityRouteTable.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/route_table/PriorityRouteTableImpl.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/RoutingContext.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/RoutingContextImpl.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/path_matcher/TemplatePathMatcher.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/path_matcher/PathMatch.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/test/java/dev/scx/http/routing/test/TemplatePathMatcherTest.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/path_matcher/PathMatcher.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/path_matcher/ExactPathMatcher.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/path_matcher/RegexPathMatcher.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/method_matcher/MethodMatcher.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/method_matcher/AnyMethodMatcher.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/request_matcher/RequestMatcher.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/Route.java "raw.githubusercontent.com"
[18]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/route_table/RouteTable.java "raw.githubusercontent.com"
[19]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/routing_input/RoutingInput.java "raw.githubusercontent.com"
[20]: https://raw.githubusercontent.com/scx-projects/scx-http-routing/master/src/main/java/dev/scx/http/routing/routing_path/RoutingPath.java "raw.githubusercontent.com"
