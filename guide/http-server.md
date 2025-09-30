# 一个简单的 Http 服务器示例

首先引入 scx-http-x 依赖

```xml
<dependency>
    <groupId>cool.scx</groupId>
    <artifactId>scx-http-x</artifactId>
    <version>{version}</version>
</dependency>
```

然后编写如下代码

```java
var httpServer = new HttpServer();
httpServer.onRequest(request -> {
    request.response().send("Hello World");
});
httpServer.start(8899);
```

运行访问 http://localhost:8899 即可 实现简单的 http 服务器

### 路由

路由本质上只是一个特殊的 requestHandler


```java
var httpServer = new HttpServer();

var router = Router.of();

//这是一个通用的 路由
router.route().path("/*").handler(ctx -> {
    System.out.println(ctx.request().path());
    //转发到下一路由
    ctx.next();
});

// 这是一个普通的路由
router.route().path("/hello").method(GET).handler(ctx -> {
    ctx.response().send("hello");
});

// 我们可以通过抛出 Http 异常的方式
router.route().path("/401").method(GET).handler(c -> {
    throw new UnauthorizedException();
});

//也可以手动控制
router.route().path("/403").method(GET).handler(c -> {
    c.response().status(403).send("无权限");
});

httpServer.onRequest(router);

httpServer.start(8899);
```
