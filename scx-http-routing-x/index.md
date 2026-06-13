# SCX HTTP Routing X

SCX HTTP Routing X 是 `scx-http-routing` 的扩展处理器库。

它提供了一组常用的 HTTP Routing handler，用来补充路由核心本身没有内置的能力，例如 CORS、静态文件服务、单文件返回、Range 请求、HTTP 缓存验证、Cache-Control 构造等。

SCX HTTP Routing X 本身不是 HTTP Server，也不是路由核心。HTTP 请求、响应、路由匹配和 `RoutingContext` 来自 `scx-http` 与 `scx-http-routing`。本模块只是提供可以挂载到路由上的常用处理器。

当前版本为 `0.3.0`。

[GitHub](https://github.com/scx-projects/scx-http-routing-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-http-routing-x</artifactId>
    <version>0.3.0</version>
</dependency>
```

`scx-http-routing-x` 依赖：

```text
scx-http-routing
```

因此在普通使用场景中，只需要引入 `scx-http-routing-x`，就可以同时获得 `scx-http-routing` 的路由核心能力。

## 基本概念

SCX HTTP Routing X 主要包含三类 handler：

```text
CorsHandler          CORS 处理器
StaticFilesHandler   目录型静态文件处理器
SingleFileHandler    单文件处理器
```

同时包含几组辅助对象：

```text
AllowOrigin          CORS Origin 策略
AllowMethods         CORS Method 策略
AllowHeaders         CORS Header 策略
ExposeHeaders        CORS Expose Header 策略

CacheControl         Cache-Control 响应头构造器
Range                Range 请求头模型
ContentRange         Content-Range 响应头模型
HttpDateHelper       HTTP-date 编解码工具
HttpValidator        ETag / Last-Modified 验证器
StaticFilesSupport   静态文件响应工具
```

它们之间的关系可以简单理解为：

```text
Router
    ↓
route(..., CorsHandler)
route(..., StaticFilesHandler)
route(..., SingleFileHandler)
    ↓
处理 CORS / 静态文件 / 单文件响应
```

静态文件相关流程可以理解为：

```text
StaticFilesHandler / SingleFileHandler
    ↓
定位文件
    ↓
设置 Cache-Control
    ↓
StaticFilesSupport.serveFile(...)
    ↓
ETag / Last-Modified / If-None-Match / If-Modified-Since
    ↓
StaticFilesSupport.sendFile(...)
    ↓
Range / Content-Range / 206 / 416 / 完整文件响应
```

## 快速开始

### CORS

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.cors.CorsHandler;
import dev.scx.http.routing.x.cors.allow_origin.AllowOrigin;

var router = Router.of();

router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
        .allowCredentials(true)
);
```

这里使用较小的 route order，让 CORS 尽可能靠前执行。

### 静态文件目录

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.static_files.StaticFilesHandler;

import java.nio.file.Path;

var router = Router.of();

router.route(
    "/*",
    StaticFilesHandler.of(Path.of("./public"))
);
```

如果请求：

```text
/assets/app.js
```

并且路由模板是：

```text
/*
```

那么 `*` 捕获会被转换成相对路径，然后从 `./public` 目录下查找对应文件。

### 单文件

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.single_file.SingleFileHandler;

import java.nio.file.Path;

var router = Router.of();

router.route(
    "/*",
    SingleFileHandler.of(Path.of("./public/index.html"))
);
```

这常用于 SPA fallback。

所有匹配到该路由的 `GET` / `HEAD` 请求，都会尝试返回同一个文件。

### 手动发送文件

```java
import dev.scx.http.routing.x.static_files.StaticFilesSupport;

import java.io.File;

router.route("/download", ctx -> {
    var file = new File("./files/demo.zip");

    StaticFilesSupport.sendFile(file, ctx.request());
});
```

如果希望支持 ETag / Last-Modified / 304：

```java
router.route("/image", ctx -> {
    var file = new File("./files/image.png");

    StaticFilesSupport.serveFile(file, ctx.request());
});
```

## CorsHandler

`CorsHandler` 是 CORS 处理器。

它实现了：

```java
Function1Void<RoutingContext, Throwable>
```

因此可以直接挂载到 `Router` 上。

```java
router.route(
    -10000,
    CorsHandler.of()
);
```

默认配置是：

```text
allowOrigin       reflect
allowMethods      reflect
allowHeaders      reflect
exposeHeaders     none
allowCredentials  true
maxAgeSeconds     null
```

也就是说，默认情况下：

1. 如果请求中有 `Origin`，默认把该 Origin 原样写回 `Access-Control-Allow-Origin`。
2. 预检请求中的 method 默认采用反射策略。
3. 预检请求中的 headers 默认采用反射策略。
4. 普通 CORS 响应不暴露额外 headers。
5. 允许 credentials，并写入 `Access-Control-Allow-Credentials: true`。
6. 不设置 `Access-Control-Max-Age`。

需要注意，默认配置会反射请求 Origin。生产环境如果只允许指定域名，应显式配置 `allowOrigin(AllowOrigin.of(...))`；如果要完全禁止 CORS，应使用 `AllowOrigin.ofNone()`。

## CORS 基本使用

允许指定 Origin：

```java
import dev.scx.http.routing.x.cors.CorsHandler;
import dev.scx.http.routing.x.cors.allow_origin.AllowOrigin;

router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
);
```

允许多个 Origin：

```java
router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of(
            "https://a.example.com",
            "https://b.example.com"
        ))
);
```

允许任意 Origin：

```java
router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.ofWildcard())
);
```

允许 credentials：

```java
router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
        .allowCredentials(true)
);
```

需要注意，`allowCredentials(true)` 不能和任何 wildcard 策略组合使用。

## CORS 执行流程

`CorsHandler` 的执行流程大致如下：

```text
1. 读取请求头 Origin
2. 如果 Origin 不存在，说明不是 CORS 请求，直接 context.next()
3. 使用 allowOrigin 校验 Origin
4. 如果 Origin 不允许，直接 context.next()
5. 写入 Access-Control-Allow-Origin
6. 写入 Vary: Origin
7. 如果 allowCredentials=true，写入 Access-Control-Allow-Credentials: true
8. 判断是否是预检请求
9. 预检请求直接返回 204
10. 非预检请求写入 Access-Control-Expose-Headers 后继续 context.next()
```

预检请求的判断条件是：

```text
请求方法是 OPTIONS
并且存在 Access-Control-Request-Method 请求头
```

也就是说：

```text
OPTIONS + Access-Control-Request-Method
```

才会被当作 CORS preflight。

## 非 CORS 请求

如果请求中没有：

```text
Origin
```

`CorsHandler` 不会修改响应，也不会结束请求。

它会直接调用：

```java
context.next();
```

这意味着普通同源请求不会受到 CORS handler 的影响。

## Origin 不允许时

如果请求中存在 `Origin`，但 `allowOrigin` 返回 `null`，表示该 Origin 不允许。

此时 `CorsHandler` 不会返回错误，也不会设置 CORS 头。

它会继续：

```java
context.next();
```

也就是说，是否因为没有 CORS 响应头而被浏览器拦截，是浏览器侧行为，不是服务端主动返回 403。

## 预检请求

对于预检请求，`CorsHandler` 会直接生成响应。

示例请求：

```http
OPTIONS /api/user HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

如果配置允许，则响应类似：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Vary: Origin
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type, Authorization
```

如果设置了 `maxAgeSeconds`：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.of("https://example.com"))
    .maxAgeSeconds(3600L);
```

则会额外设置：

```http
Access-Control-Max-Age: 3600
```

预检请求处理完成后不会继续调用 `context.next()`。

## 普通 CORS 请求

对于普通 CORS 请求，`CorsHandler` 会设置基础 CORS 响应头，然后继续路由链。

```text
Access-Control-Allow-Origin
Vary
Access-Control-Allow-Credentials
Access-Control-Expose-Headers
```

例如：

```java
import dev.scx.http.routing.x.cors.expose_headers.ExposeHeaders;

import static dev.scx.http.headers.HttpHeaderName.CONTENT_RANGE;
import static dev.scx.http.headers.HttpHeaderName.ETAG;

router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
        .exposeHeaders(ExposeHeaders.of(ETAG, CONTENT_RANGE))
);
```

响应会包含：

```http
Access-Control-Expose-Headers: ETag, Content-Range
```

然后继续：

```java
context.next();
```

## AllowOrigin

`AllowOrigin` 用于决定请求中的 `Origin` 是否允许。

接口可以理解为：

```java
public sealed interface AllowOrigin {

    static ListAllowOrigin of(String... origins)

    static WildcardAllowOrigin ofWildcard()

    static NoneAllowOrigin ofNone()

    String allowedOrigin(String origin);

}
```

返回值语义：

```text
返回 String    表示允许，并把该值写入 Access-Control-Allow-Origin
返回 null      表示不允许
```

### 指定 Origin 列表

```java
import dev.scx.http.routing.x.cors.allow_origin.AllowOrigin;

var allowOrigin = AllowOrigin.of(
    "https://a.example.com",
    "https://b.example.com"
);
```

匹配规则是精确匹配。

```text
Origin: https://a.example.com    允许
Origin: https://b.example.com    允许
Origin: https://c.example.com    不允许
```

`ListAllowOrigin` 会：

1. 拒绝 `null` 数组。
2. 拒绝空数组。
3. 拒绝 `null` 元素。
4. trim 每个 origin。
5. 拒绝空白 origin。
6. 拒绝 `"*"`。
7. 去重。
8. 保留首次出现顺序。

如果想使用通配符，应该使用：

```java
AllowOrigin.ofWildcard()
```

而不是：

```java
AllowOrigin.of("*")
```

### 任意 Origin

```java
var allowOrigin = AllowOrigin.ofWildcard();
```

它总是返回：

```text
*
```

因此响应头是：

```http
Access-Control-Allow-Origin: *
```

### 不允许任何 Origin

```java
var allowOrigin = AllowOrigin.ofNone();
```

它总是返回：

```text
null
```

如果要让某个 `CorsHandler` 完全不放行跨域请求，可以把 `allowOrigin` 设置为这个策略。

## AllowMethods

`AllowMethods` 用于决定预检响应中的：

```text
Access-Control-Allow-Methods
```

接口可以理解为：

```java
public sealed interface AllowMethods {

    static ListAllowMethods of(ScxHttpMethod... methods)

    static ReflectAllowMethods ofReflect()

    static WildcardAllowMethods ofWildcard()

    String allowedMethods(String requestMethodString);

}
```

### 指定 Method 列表

```java
import dev.scx.http.routing.x.cors.allow_methods.AllowMethods;

import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.method.HttpMethod.POST;

var allowMethods = AllowMethods.of(GET, POST);
```

生成结果类似：

```text
GET, POST
```

`ListAllowMethods` 会：

1. 拒绝 `null` 数组。
2. 拒绝空数组。
3. 拒绝 `null` method。
4. trim method value。
5. 拒绝空白 method。
6. 拒绝 `"*"`。
7. 拒绝包含 `,` 的 method。
8. 去重。
9. 使用 `", "` 拼接。

需要注意，`ListAllowMethods` 不会校验请求中的 `Access-Control-Request-Method` 是否在列表中。

它只返回配置好的 methods 字符串。

### 反射 Method

```java
var allowMethods = AllowMethods.ofReflect();
```

它会直接返回请求中的：

```text
Access-Control-Request-Method
```

例如请求头是：

```http
Access-Control-Request-Method: POST
```

则响应头是：

```http
Access-Control-Allow-Methods: POST
```

这是默认配置。

### 任意 Method

```java
var allowMethods = AllowMethods.ofWildcard();
```

它返回：

```text
*
```

因此响应头是：

```http
Access-Control-Allow-Methods: *
```

## AllowHeaders

`AllowHeaders` 用于决定预检响应中的：

```text
Access-Control-Allow-Headers
```

接口可以理解为：

```java
public sealed interface AllowHeaders {

    static ListAllowHeaders of(ScxHttpHeaderName... headerNames)

    static ReflectAllowHeaders ofReflect()

    static WildcardAllowHeaders ofWildcard()

    String allowedHeaders(String requestHeadersString);

}
```

返回值语义：

```text
返回 String    写入 Access-Control-Allow-Headers
返回 null      不写入 Access-Control-Allow-Headers
```

### 指定 Header 列表

```java
import dev.scx.http.routing.x.cors.allow_headers.AllowHeaders;

import static dev.scx.http.headers.HttpHeaderName.AUTHORIZATION;
import static dev.scx.http.headers.HttpHeaderName.CONTENT_TYPE;

var allowHeaders = AllowHeaders.of(CONTENT_TYPE, AUTHORIZATION);
```

生成结果类似：

```text
Content-Type, Authorization
```

`ListAllowHeaders` 会：

1. 拒绝 `null` 数组。
2. 拒绝空数组。
3. 拒绝 `null` header。
4. trim header value。
5. 拒绝空白 header。
6. 拒绝 `"*"`。
7. 拒绝包含 `,` 的 header。
8. 去重。
9. 使用 `", "` 拼接。

### 反射 Header

```java
var allowHeaders = AllowHeaders.ofReflect();
```

它会直接返回请求中的：

```text
Access-Control-Request-Headers
```

例如请求头是：

```http
Access-Control-Request-Headers: Content-Type, Authorization
```

则响应头是：

```http
Access-Control-Allow-Headers: Content-Type, Authorization
```

这是默认配置。

如果请求中没有 `Access-Control-Request-Headers`，反射策略会返回 `null`，此时不会设置 `Access-Control-Allow-Headers`。

### 任意 Header

```java
var allowHeaders = AllowHeaders.ofWildcard();
```

它返回：

```text
*
```

因此响应头是：

```http
Access-Control-Allow-Headers: *
```

## ExposeHeaders

`ExposeHeaders` 用于决定普通 CORS 响应中的：

```text
Access-Control-Expose-Headers
```

接口可以理解为：

```java
public sealed interface ExposeHeaders {

    static ListExposeHeaders of(ScxHttpHeaderName... headerNames)

    static WildcardExposeHeaders ofWildcard()

    static NoneExposeHeaders ofNone()

    String exposedHeaders();

}
```

返回值语义：

```text
返回 String    写入 Access-Control-Expose-Headers
返回 null      不写入 Access-Control-Expose-Headers
```

### 指定暴露 Header

```java
import dev.scx.http.routing.x.cors.expose_headers.ExposeHeaders;

import static dev.scx.http.headers.HttpHeaderName.CONTENT_RANGE;
import static dev.scx.http.headers.HttpHeaderName.ETAG;

var exposeHeaders = ExposeHeaders.of(ETAG, CONTENT_RANGE);
```

生成结果类似：

```text
ETag, Content-Range
```

`ListExposeHeaders` 的校验规则和 `ListAllowHeaders` 类似。

### 不暴露额外 Header

```java
var exposeHeaders = ExposeHeaders.ofNone();
```

它返回：

```text
null
```

因此不会设置：

```http
Access-Control-Expose-Headers
```

这是默认配置。

### 暴露任意 Header

```java
var exposeHeaders = ExposeHeaders.ofWildcard();
```

它返回：

```text
*
```

因此响应头是：

```http
Access-Control-Expose-Headers: *
```

## allowCredentials

`allowCredentials(true)` 会让响应包含：

```http
Access-Control-Allow-Credentials: true
```

示例：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.of("https://example.com"))
    .allowCredentials(true);
```

需要注意，当前实现禁止下面这些组合：

```text
allowCredentials=true + WildcardAllowOrigin
allowCredentials=true + WildcardAllowMethods
allowCredentials=true + WildcardAllowHeaders
allowCredentials=true + WildcardExposeHeaders
```

如果这样配置，会抛出 `IllegalArgumentException`。

例如：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.ofWildcard())
    .allowCredentials(true);
```

这是非法配置。

推荐写法是显式列出允许的 Origin：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.of("https://example.com"))
    .allowCredentials(true);
```

## maxAgeSeconds

`maxAgeSeconds(...)` 用于设置预检响应中的：

```text
Access-Control-Max-Age
```

示例：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.of("https://example.com"))
    .maxAgeSeconds(3600L);
```

预检响应会包含：

```http
Access-Control-Max-Age: 3600
```

如果传入 `null`，表示不设置该响应头。

如果传入小于 `0` 的值，会抛出 `IllegalArgumentException`。

## 完整 CORS 示例

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.cors.CorsHandler;
import dev.scx.http.routing.x.cors.allow_headers.AllowHeaders;
import dev.scx.http.routing.x.cors.allow_methods.AllowMethods;
import dev.scx.http.routing.x.cors.allow_origin.AllowOrigin;
import dev.scx.http.routing.x.cors.expose_headers.ExposeHeaders;

import static dev.scx.http.headers.HttpHeaderName.AUTHORIZATION;
import static dev.scx.http.headers.HttpHeaderName.CONTENT_RANGE;
import static dev.scx.http.headers.HttpHeaderName.CONTENT_TYPE;
import static dev.scx.http.headers.HttpHeaderName.ETAG;
import static dev.scx.http.method.HttpMethod.DELETE;
import static dev.scx.http.method.HttpMethod.GET;
import static dev.scx.http.method.HttpMethod.POST;
import static dev.scx.http.method.HttpMethod.PUT;

var router = Router.of();

router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
        .allowMethods(AllowMethods.of(GET, POST, PUT, DELETE))
        .allowHeaders(AllowHeaders.of(CONTENT_TYPE, AUTHORIZATION))
        .exposeHeaders(ExposeHeaders.of(ETAG, CONTENT_RANGE))
        .allowCredentials(true)
        .maxAgeSeconds(3600L)
);
```

## StaticFilesHandler

`StaticFilesHandler` 是目录型静态文件处理器。

创建方式：

```java
import dev.scx.http.routing.x.static_files.StaticFilesHandler;

import java.nio.file.Path;

var handler = StaticFilesHandler.of(Path.of("./public"));
```

挂载方式：

```java
router.route(
    "/*",
    StaticFilesHandler.of(Path.of("./public"))
);
```

或者挂载到某个前缀下：

```java
router.route(
    "/assets/*",
    StaticFilesHandler.of(Path.of("./assets"))
);
```

需要注意，`StaticFilesHandler` 依赖当前路由提供 `*` 捕获。

因此它应该挂载在：

```text
/*
/assets/*
/public/*
```

这类带尾部通配符的路径模板上。

如果当前路由没有 `*` 捕获，运行时会抛出：

```text
IllegalStateException
```

## StaticFilesHandler 处理流程

`StaticFilesHandler` 的处理流程大致如下：

```text
1. 只处理 GET 和 HEAD
2. 非 GET / HEAD 直接 context.next()
3. 读取 pathMatch 中的 * 捕获
4. 把捕获转换为相对路径
5. root.resolve(relativePath).normalize()
6. 检查目标路径是否仍然在 root 内
7. 读取文件属性
8. 如果是常规文件，发送文件
9. 如果是目录且 URL 没有 / 结尾，重定向到带 / 的路径
10. 如果是目录且有 index.html，发送 index.html
11. 找不到文件则 context.next()
```

## 路径安全

`StaticFilesHandler` 会把 root 转换为：

```java
root.toAbsolutePath().normalize()
```

请求路径也会经过：

```java
root.resolve(relativePath).normalize()
```

然后检查：

```java
target.startsWith(root)
```

如果目标路径不在 root 内，会直接抛出：

```text
NotFoundException
```

不会继续 `context.next()`。

这可以阻止类似下面这种路径逃逸：

```text
/../../etc/passwd
```

## 目录 index.html

如果目标路径是目录，`StaticFilesHandler` 会尝试返回该目录下的：

```text
index.html
```

例如：

```text
public/docs/index.html
```

请求：

```text
/docs/
```

会尝试返回：

```text
public/docs/index.html
```

如果 `index.html` 不存在，则继续：

```java
context.next();
```

## 目录重定向

如果目标路径是目录，但请求路径没有以 `/` 结尾，会返回永久重定向。

例如：

```text
/docs
```

如果 `public/docs` 是目录，则响应会设置：

```http
Location: /docs/
```

状态码是：

```text
308 Permanent Redirect
```

这样可以避免目录下相对资源加载错误。

## cacheControl

`StaticFilesHandler` 支持设置 `Cache-Control`。

```java
import dev.scx.http.routing.x.static_files.cache_control.CacheControl;

router.route(
    "/*",
    StaticFilesHandler.of(Path.of("./public"))
        .cacheControl(CacheControl.of("public", "max-age=3600"))
);
```

响应会包含：

```http
Cache-Control: public, max-age=3600
```

如果不设置 `cacheControl`，则不会写入 `Cache-Control` 响应头。

## SingleFileHandler

`SingleFileHandler` 是单文件处理器。

它和 `StaticFilesHandler` 的区别是：

```text
StaticFilesHandler    根据请求路径从目录中查找文件
SingleFileHandler     总是尝试返回同一个文件
```

创建方式：

```java
import dev.scx.http.routing.x.single_file.SingleFileHandler;

import java.nio.file.Path;

var handler = SingleFileHandler.of(Path.of("./public/index.html"));
```

挂载方式：

```java
router.route(
    "/*",
    SingleFileHandler.of(Path.of("./public/index.html"))
);
```

常用于 SPA fallback。

## SingleFileHandler 处理流程

`SingleFileHandler` 的处理流程大致如下：

```text
1. 只处理 GET 和 HEAD
2. 非 GET / HEAD 直接 context.next()
3. 读取构造时传入的固定文件
4. 如果文件不存在或不是常规文件，context.next()
5. 如果设置了 cacheControl，写入 Cache-Control
6. 使用 StaticFilesSupport.serveFile(...) 发送文件
```

需要注意，`SingleFileHandler` 不会根据请求路径查找文件。

只要路由匹配，它就会尝试返回构造时指定的那个文件。

## SPA fallback 示例

常见挂载顺序是：

```java
router.route("/api/hello", ctx -> {
    ctx.request().response().send("hello");
});

router.route(
    "/*",
    StaticFilesHandler.of(Path.of("./public"))
);

router.route(
    "/*",
    SingleFileHandler.of(Path.of("./public/index.html"))
);
```

大致语义是：

```text
/api/hello        先走 API
/assets/app.js    静态文件存在则返回静态文件
/dashboard        静态文件不存在，最后返回 index.html
```

这样可以支持前端路由。

## StaticFilesSupport

`StaticFilesSupport` 是静态文件响应工具类。

它提供两类主要方法：

```java
sendFile(...)

serveFile(...)
```

区别是：

```text
sendFile     处理文件发送和 Range
serveFile    在 sendFile 基础上增加 ETag / Last-Modified / 304
```

### sendFile

`sendFile(...)` 用于发送文件，并处理 Range 请求。

```java
StaticFilesSupport.sendFile(file, request);
```

它会：

1. 检查目标是否是常规文件。
2. 设置 `Accept-Ranges: bytes`。
3. 如果没有 `Range` 请求头，发送完整文件。
4. 如果有 `Range` 请求头，解析 Range。
5. Range 非法时返回 `416 Range Not Satisfiable`。
6. Range 不满足时返回 `416 Range Not Satisfiable`。
7. Range 有效时返回 `206 Partial Content`。
8. 设置 `Content-Range`。
9. 发送文件指定区间。

### serveFile

`serveFile(...)` 用于发送带缓存验证能力的文件。

```java
StaticFilesSupport.serveFile(file, request);
```

它会：

1. 根据文件大小和最后修改时间创建 `ETag`。
2. 根据文件最后修改时间创建 `Last-Modified`。
3. 检查 `If-None-Match`。
4. 检查 `If-Modified-Since`。
5. 如果未修改，返回 `304 Not Modified`。
6. 如果已修改，继续调用 `sendFile(...)`。

也就是说：

```text
serveFile = HTTP 缓存验证 + sendFile
```

## sendFile 和 serveFile 的选择

如果只是文件下载，不关心浏览器缓存：

```java
StaticFilesSupport.sendFile(file, request);
```

如果希望浏览器能使用缓存验证：

```java
StaticFilesSupport.serveFile(file, request);
```

`StaticFilesHandler` 和 `SingleFileHandler` 内部使用的是：

```java
StaticFilesSupport.serveFile(...)
```

因此它们默认支持：

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
304 Not Modified
Range
206 Partial Content
416 Range Not Satisfiable
```

## ETag

`StaticFilesSupport` 创建的 ETag 格式是：

```text
"size-lastModifiedMillis"
```

例如：

```text
"12345-1710000000000"
```

其中：

```text
size                 文件大小
lastModifiedMillis   文件最后修改时间的毫秒值
```

响应会包含：

```http
ETag: "12345-1710000000000"
```

如果请求头：

```http
If-None-Match: "12345-1710000000000"
```

和当前 ETag 完全相等，则返回：

```http
304 Not Modified
```

## Last-Modified

`StaticFilesSupport` 会根据文件最后修改时间设置：

```http
Last-Modified
```

时间会按 HTTP-date 格式输出，并截断到秒。

例如：

```http
Last-Modified: Fri, 08 May 2026 12:30:00 GMT
```

如果请求中没有 `If-None-Match`，但存在：

```http
If-Modified-Since
```

则会解析该时间。

如果文件最后修改时间不晚于 `If-Modified-Since`，则返回：

```http
304 Not Modified
```

如果 `If-Modified-Since` 解析失败，则认为没有有效缓存条件，继续发送文件。

## If-None-Match 优先级

缓存验证时，`If-None-Match` 优先于 `If-Modified-Since`。

也就是说：

```text
如果存在 If-None-Match，则只比较 ETag
如果不存在 If-None-Match，但存在 If-Modified-Since，则比较 Last-Modified
```

这和当前实现的判断顺序一致。

## Range

`Range` 表示请求头：

```http
Range: bytes=...
```

支持三种形式：

```text
bytes=start-end
bytes=start-
bytes=-suffix
```

示例：

```java
import dev.scx.http.routing.x.static_files.range.Range;

var r1 = Range.parse("bytes=0-499");

var r2 = Range.parse("bytes=500-");

var r3 = Range.parse("bytes=-500");
```

编码：

```java
System.out.println(r1.encode());
System.out.println(r2.encode());
System.out.println(r3.encode());
```

输出：

```text
bytes=0-499
bytes=500-
bytes=-500
```

## Range 解析规则

`Range.parse(...)` 的规则包括：

1. 忽略两端空白。
2. `bytes=` 大小写不敏感。
3. `start` 和 `end` 两侧可以有空白。
4. 支持多段 Range，但只取第一段。
5. `start` 和 `end` 不能同时为空。
6. `start` 不能小于 `0`。
7. `end` 不能小于 `0`。
8. 如果 `start` 和 `end` 都存在，则 `start <= end`。
9. 非法格式抛出 `IllegalRangeException`。

合法示例：

```text
bytes=0-499
bytes=500-
bytes=-500
 bytes=0-499 
BYTES=0-499
bytes=0 - 499
bytes=0-1,4-5
```

其中多段 Range：

```text
bytes=0-1,4-5
```

只取第一段：

```text
bytes=0-1
```

非法示例：

```text
bytes
bytes 0-499
items=0-10
bytes=0
bytes=-
bytes=--500
bytes=abc-def
bytes=500-499
```

## ContentRange

`ContentRange` 表示响应头：

```http
Content-Range
```

支持两种形式：

```text
bytes start-end/size
bytes */size
```

普通范围：

```java
import dev.scx.http.routing.x.static_files.content_range.ContentRange;

var contentRange = ContentRange.of(0, 499, 1000);

System.out.println(contentRange.encode());
```

输出：

```text
bytes 0-499/1000
```

不满足范围：

```java
var contentRange = ContentRange.ofUnsatisfied(1000);

System.out.println(contentRange.encode());
```

输出：

```text
bytes */1000
```

## ContentRange 校验规则

`ContentRange` 会校验：

```text
size >= 0
普通范围时 start 和 end 都不能为 null
普通范围时 start >= 0
普通范围时 end >= 0
普通范围时 start <= end
普通范围时 end < size
不满足范围时 start 和 end 都必须为 null
```

如果不满足，会抛出：

```text
IllegalArgumentException
```

解析失败时会抛出：

```text
IllegalContentRangeException
```

## Range 到 ContentRange

`StaticFilesSupport.resolveContentRange(...)` 用于把请求中的 `Range` 转成可响应的 `ContentRange`。

假设文件大小是：

```text
1000
```

请求：

```text
bytes=0-499
```

结果：

```text
bytes 0-499/1000
```

请求：

```text
bytes=500-
```

结果：

```text
bytes 500-999/1000
```

请求：

```text
bytes=-100
```

结果：

```text
bytes 900-999/1000
```

请求：

```text
bytes=2000-
```

结果：

```text
bytes */1000
```

也就是范围不满足。

### end 超出文件大小

如果请求：

```text
bytes=0-999999
```

但文件大小只有：

```text
1000
```

则实际范围会被裁剪为：

```text
bytes 0-999/1000
```

### suffix 大于文件大小

如果请求：

```text
bytes=-2000
```

但文件大小只有：

```text
1000
```

则返回整个文件范围：

```text
bytes 0-999/1000
```

### 空文件

如果文件大小是：

```text
0
```

任何 Range 都会被视为不满足：

```text
bytes */0
```

## 206 Partial Content

当 Range 合法且可以满足时，`sendFile(...)` 会返回：

```http
206 Partial Content
Content-Range: bytes start-end/size
Accept-Ranges: bytes
```

然后只发送文件中对应区间的数据。

这对视频拖动、断点续传、浏览器分段加载等场景很重要。

## 416 Range Not Satisfiable

当 Range 解析失败或范围不满足时，`sendFile(...)` 会返回：

```http
416 Range Not Satisfiable
Content-Range: bytes */size
```

例如文件大小是 `1000`，请求：

```http
Range: bytes=2000-
```

响应会包含：

```http
Content-Range: bytes */1000
```

## CacheControl

`CacheControl` 用于构造 `Cache-Control` 响应头。

创建方式：

```java
import dev.scx.http.routing.x.static_files.cache_control.CacheControl;

var cacheControl = CacheControl.of("public", "max-age=3600");
```

编码结果：

```java
System.out.println(cacheControl.encode());
```

输出：

```text
public, max-age=3600
```

### CacheControl 校验规则

`CacheControl.of(...)` 会：

1. 拒绝 `null` 数组。
2. 拒绝空数组。
3. 拒绝 `null` directive。
4. trim 每个 directive。
5. 拒绝空白 directive。
6. 拒绝包含 `,` 的 directive。
7. 去重。
8. 保留首次出现顺序。
9. 使用 `", "` 拼接。

示例：

```java
var cacheControl = CacheControl.of(
    " public ",
    "max-age=3600",
    "public"
);
```

结果：

```text
public, max-age=3600
```

## 常见 Cache-Control 示例

短缓存：

```java
CacheControl.of("public", "max-age=60")
```

长期缓存：

```java
CacheControl.of("public", "max-age=31536000", "immutable")
```

禁止缓存：

```java
CacheControl.of("no-store")
```

每次使用前重新验证：

```java
CacheControl.of("no-cache")
```

静态文件处理器中使用：

```java
router.route(
    "/assets/*",
    StaticFilesHandler.of(Path.of("./assets"))
        .cacheControl(CacheControl.of("public", "max-age=31536000", "immutable"))
);
```

## HttpDateHelper

`HttpDateHelper` 用于 HTTP-date 编解码。

解析：

```java
import dev.scx.http.routing.x.static_files.http_date.HttpDateHelper;

var instant = HttpDateHelper.parse("Fri, 08 May 2026 12:30:00 GMT");
```

编码：

```java
var value = HttpDateHelper.encode(instant);
```

编码时会：

1. 截断到秒。
2. 使用 UTC。
3. 使用 RFC 1123 日期格式。

解析失败时会抛出：

```text
IllegalHttpDateException
```

## HttpValidator

`HttpValidator` 是一个简单 record：

```java
public record HttpValidator(
    String etag,
    Instant lastModified
) {

}
```

`StaticFilesSupport.createValidator(...)` 会根据文件属性创建它。

```java
var validator = StaticFilesSupport.createValidator(attr);
```

然后用于判断：

```java
StaticFilesSupport.checkNotModified(validator, request);
```

## 非 GET / HEAD 请求

`StaticFilesHandler` 和 `SingleFileHandler` 都只处理：

```text
GET
HEAD
```

如果请求方法不是 `GET` 或 `HEAD`，它们会直接：

```java
context.next();
```

因此它们不会拦截 `POST`、`PUT`、`DELETE` 等请求。

## HEAD 请求

`StaticFilesHandler` 和 `SingleFileHandler` 会把 `HEAD` 请求交给 `StaticFilesSupport.serveFile(...)`。

具体是否发送响应体，由底层 `ScxHttpServerResponse#send(...)` 对 `HEAD` 的处理语义决定。

在 handler 层，它和 `GET` 一样会走静态文件定位、缓存验证和 Range 处理流程。

## 写出异常降噪

在发送完整文件或 Range 文件时，如果底层写出阶段抛出的包装异常原因是：

```text
ScxOutputException
```

`StaticFilesSupport` 会直接返回，不再继续向外抛出。

这是为了处理浏览器在视频拖动、跳转、取消下载时主动中断连接的场景。

其它异常仍然会继续抛出。

## 完整示例：静态资源 + CORS

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.cors.CorsHandler;
import dev.scx.http.routing.x.cors.allow_origin.AllowOrigin;
import dev.scx.http.routing.x.static_files.StaticFilesHandler;
import dev.scx.http.routing.x.static_files.cache_control.CacheControl;

import java.nio.file.Path;

var router = Router.of();

router.route(
    -10000,
    CorsHandler.of()
        .allowOrigin(AllowOrigin.of("https://example.com"))
);

router.route(
    "/assets/*",
    StaticFilesHandler.of(Path.of("./assets"))
        .cacheControl(CacheControl.of(
            "public",
            "max-age=31536000",
            "immutable"
        ))
);
```

## 完整示例：文件下载

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.static_files.StaticFilesSupport;

import java.io.File;

var router = Router.of();

router.route("/download", ctx -> {
    var file = new File("./files/demo.zip");

    StaticFilesSupport.sendFile(file, ctx.request());
});
```

这个示例会支持：

```text
完整文件响应
Range 请求
206 Partial Content
416 Range Not Satisfiable
Accept-Ranges: bytes
```

但不会自动加入：

```text
ETag
Last-Modified
304 Not Modified
```

如果需要这些能力，应使用：

```java
StaticFilesSupport.serveFile(file, ctx.request());
```

## 完整示例：图片响应并支持缓存验证

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.static_files.StaticFilesSupport;

import java.io.File;

var router = Router.of();

router.route("/image", ctx -> {
    var file = new File("./files/image.png");

    StaticFilesSupport.serveFile(file, ctx.request());
});
```

这个示例会支持：

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
304 Not Modified
Range
206 Partial Content
416 Range Not Satisfiable
```

## 完整示例：SPA

```java
import dev.scx.http.routing.Router;
import dev.scx.http.routing.x.single_file.SingleFileHandler;
import dev.scx.http.routing.x.static_files.StaticFilesHandler;
import dev.scx.http.routing.x.static_files.cache_control.CacheControl;

import java.nio.file.Path;

var router = Router.of();

router.route("/api/hello", ctx -> {
    ctx.request().response().send("hello");
});

router.route(
    "/assets/*",
    StaticFilesHandler.of(Path.of("./public/assets"))
        .cacheControl(CacheControl.of(
            "public",
            "max-age=31536000",
            "immutable"
        ))
);

router.route(
    "/*",
    StaticFilesHandler.of(Path.of("./public"))
);

router.route(
    "/*",
    SingleFileHandler.of(Path.of("./public/index.html"))
        .cacheControl(CacheControl.of("no-cache"))
);
```

这个路由组合适合：

```text
/api/hello          API
/assets/app.js      静态资源，长期缓存
/favicon.ico        普通静态文件
/dashboard          前端路由，返回 index.html
/settings/profile   前端路由，返回 index.html
```

## 方法总览

### CorsHandler

```java
static CorsHandler of()

CorsHandler allowOrigin(AllowOrigin allowOrigin)

CorsHandler allowMethods(AllowMethods allowMethods)

CorsHandler allowHeaders(AllowHeaders allowHeaders)

CorsHandler exposeHeaders(ExposeHeaders exposeHeaders)

CorsHandler allowCredentials(boolean allowCredentials)

CorsHandler maxAgeSeconds(Long maxAgeSeconds)
```

### AllowOrigin

```java
static ListAllowOrigin of(String... origins)

static WildcardAllowOrigin ofWildcard()

static NoneAllowOrigin ofNone()

String allowedOrigin(String origin)
```

### AllowMethods

```java
static ListAllowMethods of(ScxHttpMethod... methods)

static ReflectAllowMethods ofReflect()

static WildcardAllowMethods ofWildcard()

String allowedMethods(String requestMethodString)
```

### AllowHeaders

```java
static ListAllowHeaders of(ScxHttpHeaderName... headerNames)

static ReflectAllowHeaders ofReflect()

static WildcardAllowHeaders ofWildcard()

String allowedHeaders(String requestHeadersString)
```

### ExposeHeaders

```java
static ListExposeHeaders of(ScxHttpHeaderName... headerNames)

static WildcardExposeHeaders ofWildcard()

static NoneExposeHeaders ofNone()

String exposedHeaders()
```

### StaticFilesHandler

```java
static StaticFilesHandler of(Path root)

StaticFilesHandler cacheControl(CacheControl cacheControl)
```

### SingleFileHandler

```java
static SingleFileHandler of(Path file)

SingleFileHandler cacheControl(CacheControl cacheControl)
```

### StaticFilesSupport

```java
static void sendFile(
    File target,
    ScxHttpServerRequest request
)
```

```java
static void sendFile(
    File target,
    BasicFileAttributes attr,
    ScxHttpServerRequest request
)
```

```java
static void serveFile(
    File target,
    ScxHttpServerRequest request
)
```

```java
static void serveFile(
    File target,
    BasicFileAttributes attr,
    ScxHttpServerRequest request
)
```

```java
static ContentRange resolveContentRange(
    Range range,
    long size
)
```

```java
static HttpValidator createValidator(
    BasicFileAttributes attr
)
```

```java
static boolean checkNotModified(
    HttpValidator httpValidator,
    ScxHttpServerRequest request
)
```

### CacheControl

```java
static CacheControl of(String... directives)

String encode()
```

### Range

```java
new Range(Long start, Long end)

static Range parse(String rangeStr)

String encode()
```

### ContentRange

```java
new ContentRange(Long start, Long end, Long size)

static ContentRange of(long start, long end, long size)

static ContentRange ofUnsatisfied(long size)

static ContentRange parse(String contentRangeStr)

boolean isUnsatisfied()

String encode()
```

### HttpDateHelper

```java
static Instant parse(String v)

static String encode(Instant instant)
```

### HttpValidator

```java
public record HttpValidator(
    String etag,
    Instant lastModified
) {

}
```

## 设计说明

### 1. SCX HTTP Routing X 是路由扩展，不是路由核心

`scx-http-routing` 提供路由匹配、`Router`、`RoutingContext` 等核心能力。

`scx-http-routing-x` 提供的是可以挂载到路由上的常用 handler。

也就是说：

```text
scx-http-routing      负责路由
scx-http-routing-x    负责常用扩展处理器
```

### 2. Handler 都是 Function1Void

`CorsHandler`、`StaticFilesHandler`、`SingleFileHandler` 都可以直接作为 routing handler 使用。

因为它们都实现了：

```java
Function1Void<RoutingContext, Throwable>
```

所以可以这样挂载：

```java
router.route("/*", StaticFilesHandler.of(Path.of("./public")));
```

### 3. CORS 不主动拒绝请求

当 Origin 不允许时，`CorsHandler` 不会主动返回 403。

它只是不给响应添加 CORS 头，并继续路由链。

真正的跨域阻断通常发生在浏览器端。

### 4. CORS 预检请求会直接结束

如果请求是 CORS 预检请求，并且 Origin 允许，`CorsHandler` 会直接返回：

```text
204
```

不会继续调用 `context.next()`。

### 5. credentials 禁止 wildcard

当前实现明确禁止：

```text
allowCredentials=true
```

和任何 wildcard 策略组合。

这可以避免产生容易误用的跨域响应配置。

### 6. StaticFilesHandler 必须依赖 * 捕获

`StaticFilesHandler` 不直接读取完整 path 来映射文件，而是依赖当前路由模板中的 `*` 捕获。

这让它可以自然挂载在不同前缀下。

```java
router.route("/assets/*", StaticFilesHandler.of(Path.of("./assets")));
```

### 7. 路径归一化后必须仍在 root 内

静态文件处理器会检查最终目标路径是否仍然位于 root 下。

越界请求直接抛出 `NotFoundException`。

这是静态文件服务中非常重要的安全边界。

### 8. SingleFileHandler 适合 SPA fallback

`SingleFileHandler` 不根据请求路径查找文件。

它只返回一个固定文件。

因此它适合放在路由链最后，用来处理前端路由。

### 9. serveFile 比 sendFile 多一层缓存验证

`sendFile(...)` 负责文件发送和 Range。

`serveFile(...)` 负责 ETag / Last-Modified / 304，然后再调用 `sendFile(...)`。

使用静态文件 handler 时，内部用的是 `serveFile(...)`。

### 10. Range 多段只取第一段

当前 `Range.parse(...)` 支持多段格式，但只取第一段。

例如：

```text
bytes=0-1,4-5
```

会被解析为：

```text
bytes=0-1
```

它不会返回 multipart/byteranges 响应。

### 11. HTTP-date 使用 RFC 1123

`HttpDateHelper` 使用 `DateTimeFormatter.RFC_1123_DATE_TIME` 解析和编码 HTTP-date。

编码时使用 UTC，并截断到秒。

### 12. CacheControl 是简单字符串构造器

`CacheControl` 不理解每个 directive 的具体 HTTP 语义。

它只负责：

```text
校验 directive
去重
拼接成 Cache-Control 头值
```

例如：

```java
CacheControl.of("public", "max-age=3600")
```

生成：

```text
public, max-age=3600
```

## 常见问题

### SCX HTTP Routing X 是 HTTP Server 吗？

不是。

它只是 `scx-http-routing` 的扩展 handler 库。

HTTP Server 能力来自 `scx-http` / `scx-http-x` 等模块。

### StaticFilesHandler 为什么必须挂在 `/*` 这种路由上？

因为它需要从 `RoutingContext#pathMatch()` 中读取 `*` 捕获。

如果路由没有提供 `*` 捕获，它不知道应该把哪一段路径映射到文件系统中。

### StaticFilesHandler 找不到文件会怎样？

会调用：

```java
context.next();
```

也就是说，它会把请求交给后续路由继续处理。

### 路径越界也会 next 吗？

不会。

如果解析后的目标路径不在 root 内，会直接抛出 `NotFoundException`。

这是为了防止路径穿越攻击。

### StaticFilesHandler 会处理 POST 吗？

不会。

它只处理：

```text
GET
HEAD
```

其它方法直接 `context.next()`。

### 目录请求为什么会重定向到 `/` 结尾？

为了避免目录下相对资源路径解析错误。

例如请求：

```text
/docs
```

会重定向到：

```text
/docs/
```

### 目录下没有 index.html 会怎样？

会调用：

```java
context.next();
```

### SingleFileHandler 和 StaticFilesHandler 有什么区别？

`StaticFilesHandler` 根据请求路径在目录中找文件。

`SingleFileHandler` 总是尝试返回同一个固定文件。

### SingleFileHandler 找不到文件会怎样？

会调用：

```java
context.next();
```

### sendFile 和 serveFile 有什么区别？

`sendFile(...)` 处理文件发送和 Range。

`serveFile(...)` 在 `sendFile(...)` 前增加 ETag / Last-Modified / 304 处理。

### 静态文件 handler 支持 Range 吗？

支持。

`StaticFilesHandler` 和 `SingleFileHandler` 内部都调用 `StaticFilesSupport.serveFile(...)`，而 `serveFile(...)` 会继续调用支持 Range 的 `sendFile(...)`。

### Range 非法会返回什么？

返回：

```text
416 Range Not Satisfiable
```

并设置：

```http
Content-Range: bytes */size
```

### Range 多段会返回 multipart 吗？

不会。

当前实现只取第一段 Range。

### 支持 ETag 吗？

支持。

`serveFile(...)` 会根据文件大小和最后修改时间生成 ETag。

### 支持 Last-Modified 吗？

支持。

`serveFile(...)` 会设置 `Last-Modified`。

### If-None-Match 和 If-Modified-Since 哪个优先？

`If-None-Match` 优先。

如果存在 `If-None-Match`，就不会再使用 `If-Modified-Since` 判断。

### Cache-Control 默认会设置吗？

不会。

只有调用：

```java
cacheControl(...)
```

之后才会设置 `Cache-Control`。

### CacheControl.of 可以传逗号吗？

不可以。

每个 directive 不能包含 `,`。

应该这样传：

```java
CacheControl.of("public", "max-age=3600")
```

而不是：

```java
CacheControl.of("public, max-age=3600")
```

### CorsHandler 默认允许所有跨域吗？

是。当前 `CorsHandler.of()` 默认使用 reflect origin，并启用 credentials。

也就是说，请求里有 `Origin` 时，默认会把这个 Origin 写回 `Access-Control-Allow-Origin`，并写入 `Access-Control-Allow-Credentials: true`。

生产环境通常应该显式收窄：

```java
CorsHandler.of()
    .allowOrigin(AllowOrigin.of("https://example.com"));
```

### CORS 中 allowMethods 默认是什么？

默认是 reflect。

也就是把请求中的 `Access-Control-Request-Method` 原样写回 `Access-Control-Allow-Methods`。

### CORS 中 allowHeaders 默认是什么？

默认是 reflect。

也就是把请求中的 `Access-Control-Request-Headers` 原样写回 `Access-Control-Allow-Headers`。

如果请求没有这个头，则不写 `Access-Control-Allow-Headers`。

### CORS 中 exposeHeaders 默认是什么？

默认是 none。

也就是不设置 `Access-Control-Expose-Headers`。

### allowCredentials 可以和 wildcard Origin 一起用吗？

不可以。

当前实现会直接抛出 `IllegalArgumentException`。

### Origin 不允许时会返回 403 吗？

不会。

`CorsHandler` 会继续 `context.next()`，但不会添加 CORS 响应头。

浏览器会根据缺失的 CORS 响应头自行拦截。

### 为什么建议 CORS 路由尽量靠前？

因为 CORS 响应头通常应该尽早设置，尤其是预检请求需要尽早处理并直接返回 204。

示例：

```java
router.route(-10000, CorsHandler.of().allowOrigin(...));
```

### 什么时候用 SCX HTTP Routing X？

适合下面这些场景：

1. 需要给 `scx-http-routing` 增加 CORS。
2. 需要快速挂载静态文件目录。
3. 需要 SPA fallback。
4. 需要支持 Range 文件响应。
5. 需要支持 ETag / Last-Modified / 304。
6. 需要手动在 handler 中返回文件。
7. 需要构造 Cache-Control。
8. 需要解析或编码 Range / Content-Range。
