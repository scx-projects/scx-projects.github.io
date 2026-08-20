# SCX App

SCX App 是一个 Java 应用启动编排库。

它提供统一的应用入口、模块定义、候选类汇总、模块依赖与启动顺序解析，以及应用停止流程。常用应用模块可以参阅 [SCX App X](../scx-app-x/index.md)。

[GitHub](https://github.com/scx-projects/scx-app)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-app</artifactId>
    <version>0.10.0</version>
</dependency>
```

SCX App 更适合作为其他 SCX 模块的启动底座。你可以把 Web、HTTP、SQL、Scheduler、静态资源、服务发现等功能都拆成 `ScxAppModule`，然后由 `ScxApp` 统一完成模块定义汇总、依赖校验、候选类收集、启动顺序调度和停止流程。

## 快速开始

下面是一个最小应用入口：

```java
import dev.scx.app.ScxApp;

public class Main {

    public static void main(String[] args) throws Exception {
        ScxApp.builder()
                .module(new HelloModule())
                .run();
    }

}
```

一个最小模块可以这样写：

```java
import dev.scx.app.ScxApp;
import dev.scx.app.ScxAppModule;

public class HelloModule implements ScxAppModule {

    private final HelloService helloService = new HelloService();

    @Override
    public void start(ScxApp app) {
        System.out.println(helloService.hello());
    }

    @Override
    public void stop(ScxApp app) {
        System.out.println("HelloModule stopped");
    }

    public HelloService helloService() {
        return helloService;
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

这个例子里，`HelloModule` 持有并暴露 `HelloService`。如果另一个模块需要这项能力，可以在自己的 `start(ScxApp)` 中通过 `app.getModule(HelloModule.class)` 获取 `HelloModule`。

## 核心概念

### ScxApp

`ScxApp` 表示一个已经构建好的应用运行时。它提供以下能力：

- `candidates()`：获取所有模块定义汇总后的候选类。
- `run()`：启动应用。
- `shutdown()`：停止应用。
- `getModule(...)`：按模块类获取已经注册的模块；不存在时返回 `null`。

`candidates()` 会在所有模块完成 `define()` 后统一收集，因此在模块的 `start(ScxApp)` 阶段可以安全访问；在候选类尚未收集前调用会抛出 `IllegalStateException`。

### ScxAppBuilder

`ScxApp.builder()` 会返回默认构建器，用于收集模块并构建应用：

```java
ScxApp app = ScxApp.builder()
        .module(new AModule(), new BModule())
        .build();

app.run();
```

也可以直接调用 `run()`：

```java
ScxApp app = ScxApp.builder()
        .module(new AModule(), new BModule())
        .run();
```

### ScxAppModule

`ScxAppModule` 是 SCX App 启动流程中的动作单元。它不是传统意义上的“生命周期对象”，而是一个“启动动作节点”。

一个模块的典型职责包括：

- 在 `define()` 中提供候选类。
- 声明和其他模块的存在依赖与启动顺序关系。
- 在 `start` 阶段执行自己的启动动作，并通过 `ScxApp` 访问其他模块。
- 把自己创建的资源作为模块能力暴露给其他模块。
- 在 `stop` 阶段释放资源。

SCX App 会先调用所有模块的 `define()`，汇总所有 `ScxAppModuleDefinition`，校验模块依赖、解析启动顺序并收集 candidates，然后再按计算出的顺序调用每个模块的 `start(ScxApp)`。应用停止时，会按成功启动模块的反向顺序调用 `stop(ScxApp)`。

### ScxAppModuleDefinition

`ScxAppModuleDefinition` 是模块在 `define()` 阶段返回给应用的声明。它可以包含：

- `candidate(...)`：添加候选类。
- `startBefore(...)`：声明当前模块要早于某些模块启动。
- `startAfter(...)`：声明当前模块要晚于某些模块启动。
- `require(...)`：声明当前模块依赖某些模块存在。

示例：

```java
@Override
public ScxAppModuleDefinition define() {
    return ScxAppModuleDefinition.of()
            .candidate(UserService.class, OrderService.class)
            .startBefore(HttpModule.class)
            .require(ConfigModule.class);
}
```

`candidate(...)` 用于把类加入应用级候选集合，这些类可以通过 `ScxApp.candidates()` 统一获取。

## 启动流程

`DefaultScxApp.run()` 的主要流程如下：

1. 打印 SCX App Banner。
2. 调用所有模块的 `define()`，得到模块定义。
3. 根据模块定义校验依赖并解析模块启动顺序。
4. 收集候选类。
5. 按解析后的顺序启动模块。
6. 注册 JVM Shutdown Hook。
7. 打印启动耗时。

这意味着所有模块的定义和 candidates 都会在任意模块 `start` 之前准备完成。模块可以在自己的 `start` 阶段创建运行时资源，并通过模块公开的方法向其他模块提供能力。

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
    public ScxAppModuleDefinition define() {
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

### 模块之间怎么访问彼此提供的能力?

模块可以把自己创建的资源保存在成员字段中，并通过方法暴露。其他模块在 `start(ScxApp)` 中使用 `app.getModule(ModuleType.class)` 获取模块：

```java
var sqlModule = app.getModule(SqlModule.class);
var sqlClient = sqlModule.sqlClient();
```

`getModule(...)` 按具体模块类查找；模块不存在时返回 `null`。如果某个模块必须存在，应同时在 `define()` 中使用 `require(...)` 声明依赖。

### 为什么 `startBefore` / `startAfter` 指向不存在的模块不会报错?

因为它们表达的是“如果目标模块存在，则需要满足这个顺序关系”。这样模块可以声明与可选模块之间的关系，而不强制目标模块一定存在。如果确实要求目标模块必须存在，应使用 `require(...)`。

### 为什么模块类不能重复注册?

模块顺序解析以模块类作为节点标识。重复注册同一个模块类会让图关系变得不明确，因此默认实现会直接拒绝重复模块类。
