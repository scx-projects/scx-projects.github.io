# SCX App

SCX App 是一个轻量的 Java 应用启动编排库。

它本身不直接实现 Web Server、SQL Client 或 Scheduler，而是提供统一的应用入口、配置环境、模块定义、DI 容器装配、模块启动顺序解析以及应用停止流程。

[GitHub](https://github.com/scx-projects/scx-app)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-app</artifactId>
    <version>0.6.0</version>
</dependency>
```

SCX App 更适合作为其他 SCX 模块的启动底座。你可以把 Web、HTTP、SQL、Scheduler、静态资源、服务发现等功能都拆成 `ScxAppModule`，然后由 `ScxApp` 统一完成初始化、依赖收集、DI 容器构建和启动顺序调度。

## 快速开始

下面是一个最小应用入口：

```java
import dev.scx.app.ScxApp;

public class Main {

    public static void main(String[] args) throws Exception {
        ScxApp.builder()
                .mainClass(Main.class)
                .args(args)
                .module(new HelloModule())
                .run();
    }

}
```

一个最小模块可以这样写：

```java
import dev.scx.app.ScxApp;
import dev.scx.app.ScxAppModule;
import dev.scx.app.ScxAppModuleDefinition;
import dev.scx.app.environment.ScxEnvironment;

public class HelloModule implements ScxAppModule {

    @Override
    public ScxAppModuleDefinition init(ScxEnvironment environment) throws Exception {
        return ScxAppModuleDefinition.of()
                .componentInstance(new HelloService());
    }

    @Override
    public void start(ScxApp app) throws Exception {
        var helloService = app.getComponent(HelloService.class);
        System.out.println(helloService.hello());
    }

    @Override
    public void stop(ScxApp app) throws Exception {
        System.out.println("HelloModule stopped");
    }

}
```

```java
public class HelloService {

    public String hello() {
        return "Hello SCX App";
    }

}
```

这个例子里，`HelloModule` 在 `init` 阶段把 `HelloService` 作为组件实例交给应用；应用完成 DI 容器构建后，再调用模块的 `start` 方法。此时可以通过 `app.getComponent(HelloService.class)` 取出组件。

## 核心概念

### ScxApp

`ScxApp` 表示一个已经构建好的应用运行时。它提供以下能力：

- `environment()`：获取应用配置环境。
- `candidates()`：获取全部组件候选类。
- `componentContainer()`：获取 DI 组件容器。
- `run()`：启动应用。
- `shutdown()`：停止应用。
- `getComponent(...)`：从 DI 容器中获取组件。

`candidates()` 和 `componentContainer()` 只有在应用启动后才可用。如果应用还没有启动，默认实现会抛出 `IllegalStateException`。

### ScxAppBuilder

`ScxApp.builder()` 会返回默认构建器。构建器主要负责收集启动参数、主类和模块：

```java
ScxApp app = ScxApp.builder()
        .mainClass(Main.class)
        .args(args)
        .module(new AModule(), new BModule())
        .build();

app.run();
```

也可以直接调用 `run()`：

```java
ScxApp app = ScxApp.builder()
        .mainClass(Main.class)
        .args(args)
        .module(new AModule(), new BModule())
        .run();
```

默认构建器要求必须设置 `mainClass`，否则构建时会抛出异常。`mainClass` 会用于推导应用代码源和应用根目录。

### ScxAppModule

`ScxAppModule` 是 SCX App 启动流程中的动作单元。它不是传统意义上的“生命周期对象”，而是一个“启动动作节点”。

一个模块的典型职责包括：

- 读取配置。
- 提供组件候选类。
- 提供组件实例。
- 定义组件选择规则。
- 声明和其他模块的启动顺序关系。
- 在 `start` 阶段执行自己的启动动作。
- 在 `stop` 阶段释放资源。

SCX App 会先调用所有模块的 `init()`，汇总所有 `ScxAppModuleDefinition`，构建 DI 容器并解析模块启动顺序，然后再按计算出的顺序调用每个模块的 `start(ScxApp)`。应用停止时，会按成功启动模块的反向顺序调用 `stop(ScxApp)`。

### ScxAppModuleDefinition

`ScxAppModuleDefinition` 是模块在 `init` 阶段返回给应用的声明。它可以包含：

- `candidate(...)`：添加组件候选类。
- `componentSelector(...)`：添加组件选择器。
- `componentInstance(...)`：添加已经创建好的组件实例。
- `startBefore(...)`：声明当前模块要早于某些模块启动。
- `startAfter(...)`：声明当前模块要晚于某些模块启动。
- `require(...)`：声明当前模块依赖某些模块存在。

示例：

```java
@Override
public ScxAppModuleDefinition init(ScxEnvironment environment) throws Exception {
    return ScxAppModuleDefinition.of()
            .candidate(UserService.class, OrderService.class)
            .componentSelector(c -> c.getPackageName().startsWith("com.example.app"))
            .componentInstance(new ManualComponent())
            .startBefore(HttpModule.class)
            .require(ConfigModule.class);
}
```

## 启动流程

`DefaultScxApp.run()` 的主要流程如下：

1. 打印 SCX App Banner。
2. 调用所有模块的 `init(environment)`，得到模块定义。
3. 根据模块定义解析模块启动顺序。
4. 收集组件候选类。
5. 创建 DI 组件容器。
6. 校验 DI 组件容器。
7. 按解析后的顺序启动模块。
8. 注册 JVM Shutdown Hook。
9. 打印启动耗时。

这意味着模块的 `start` 执行时，DI 容器已经构建并校验完成，模块可以安全地通过 `ScxApp` 获取运行时能力。

## 配置环境

### 默认配置文件

默认构建器内置了一个默认配置：

```text
scx.config = AppRoot:scx-config.json
```

也就是说，如果没有通过命令行参数覆盖配置路径，SCX App 会尝试从应用根目录下的 `scx-config.json` 读取配置。

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

SCX App 的配置源统一实现 `ScxConfigSource`。配置源最终都会提供一个树形 `ObjectNode`，点号路径会被看成树形路径的一层。

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

`AppRoot:` 会被解析为应用根目录。应用根目录来自 `mainClass` 的 JVM CodeSource：开发环境通常是 classes 目录，发布环境通常是 jar 所在目录。

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

`ScxCodeSource` 是 SCX App 对 JVM CodeSource 的封装。它表示某个 class 来自哪里。

常见情况：

```text
开发环境  classes 目录
发布环境  jar 文件
```

默认构建器会通过 `mainClass` 创建 `ScxCodeSource`，再用它计算应用根目录。你通常不需要直接使用 `ScxCodeSource`，但理解它有助于判断 `AppRoot:` 到底指向哪里。

## DI 容器

SCX App 使用 `scx-di` 构建组件容器。创建容器时会注册：

- `@Inject` 依赖解析器。
- `@Value` 配置值解析器。
- 所有模块实例。
- 模块定义中声明的组件实例。
- 根据组件选择器从候选类中筛选出的组件类型。

因此模块既可以直接提供已经创建好的实例，也可以提供候选类和选择器，让 SCX App 统一注册组件类型。

一个常见模式是：

```java
@Override
public ScxAppModuleDefinition init(ScxEnvironment environment) throws Exception {
    return ScxAppModuleDefinition.of()
            .candidate(UserController.class, UserService.class)
            .componentSelector(c -> c.getPackageName().startsWith("com.example"));
}
```

如果组件需要读取配置，可以配合 `@Value`。`ValueAnnotationDependencyResolver` 会通过 `environment.get(...)` 读取配置值。

## 模块顺序

模块顺序由 `startBefore`、`startAfter` 和 `require` 共同决定。

### startBefore

```java
ScxAppModuleDefinition.of()
        .startBefore(HttpModule.class);
```

表示当前模块必须早于 `HttpModule` 启动。

### startAfter

```java
ScxAppModuleDefinition.of()
        .startAfter(WebModule.class);
```

表示当前模块必须晚于 `WebModule` 启动。

### require

```java
ScxAppModuleDefinition.of()
        .require(ConfigModule.class);
```

表示当前模块要求 `ConfigModule` 必须存在。如果被 `require` 的模块不存在，应用启动会失败。

### 顺序规则

SCX App 的模块顺序规则是：

- `startBefore(B)`：当前模块要在 `B` 之前启动。
- `startAfter(B)`：当前模块要在 `B` 之后启动。
- `startBefore` / `startAfter` 指向的模块如果不存在，会被忽略。
- `require` 指向的模块如果不存在，会启动失败。
- 没有顺序关系的模块，保持用户注册顺序。
- 同一个模块类不能重复注册。
- 模块不能依赖自身。
- 如果存在循环依赖，应用启动失败。

例如：

```java
public class WebRouteModule implements ScxAppModule {

    @Override
    public ScxAppModuleDefinition init(ScxEnvironment environment) throws Exception {
        return ScxAppModuleDefinition.of()
                .startBefore(HttpServerModule.class);
    }

}
```

这个例子表达的是：Web 路由注册必须发生在 HTTP Server 监听端口之前。

## 为什么不是多阶段生命周期

SCX App 的设计重点不是增加更多生命周期阶段，而是把启动过程看成一个有向图。

例如下面这些需求：

```text
Web 路由注册必须早于 HTTP listen
CORS 注册必须早于 HTTP listen
SQLClient 创建必须早于 HTTP listen
服务发现注册必须晚于 HTTP listen
```

它们本质上不是“还需要一个新的生命周期方法”，而是“启动动作之间存在先后关系”。因此 SCX App 推荐把不同启动时机拆成不同模块节点，再用 `startBefore` / `startAfter` 表达局部顺序。

也就是说：

```text
新时机 != 新生命周期方法
新时机 = 新模块节点 + startBefore / startAfter
```

这个设计可以避免 `prepare()`、`started()`、`ready()`、`afterReady()` 等全局阶段不断膨胀。

## 停止流程

应用停止时，SCX App 会按已经成功启动模块的反向顺序调用 `stop(ScxApp)`。如果某个模块在启动过程中抛出异常，SCX App 会停止已经成功启动的模块，然后继续抛出原始异常。

停止阶段的异常会被忽略，以尽量保证后续模块仍然有机会执行自己的停止逻辑。

## 典型模块拆分方式

假设一个完整应用需要 Web、HTTP、SQL 和 Scheduler，可以拆成下面几个模块：

```text
SqlModule
WebRouteModule
StaticServerModule
HttpServerModule
JobRegisterModule
SchedulerModule
```

然后通过顺序关系表达启动图：

```java
ScxAppModuleDefinition.of()
        .startBefore(HttpServerModule.class);
```

```java
ScxAppModuleDefinition.of()
        .startAfter(JobRegisterModule.class);
```

注册时可以保持可读的业务顺序：

```java
ScxApp.builder()
        .mainClass(Main.class)
        .args(args)
        .module(
                new SqlModule(),
                new WebRouteModule(),
                new StaticServerModule(),
                new HttpServerModule(),
                new JobRegisterModule(),
                new SchedulerModule()
        )
        .run();
```

最终实际启动顺序由 SCX App 根据模块定义计算。

## 常见问题

### 为什么 `mainClass` 必须设置?

默认构建器会用 `mainClass` 推导应用代码源和应用根目录。应用根目录又会影响 `AppRoot:...` 这类配置路径的解析，所以必须显式设置主类。

### 为什么 `startBefore` / `startAfter` 指向不存在的模块不会报错?

因为它们表达的是“如果目标模块存在，则需要满足这个顺序关系”。这样模块可以声明与可选模块之间的关系，而不强制目标模块一定存在。如果确实要求目标模块必须存在，应使用 `require(...)`。

### 为什么模块类不能重复注册?

模块顺序解析以模块类作为节点标识。重复注册同一个模块类会让图关系变得不明确，因此默认实现会直接拒绝重复模块类。

### 什么时候用 `componentInstance`，什么时候用 `candidate`?

如果对象必须由模块自己创建，或者创建过程依赖特殊逻辑，可以用 `componentInstance(...)`。

如果对象可以由 DI 容器创建，可以把类放入 `candidate(...)`，再用 `componentSelector(...)` 选择它们。
