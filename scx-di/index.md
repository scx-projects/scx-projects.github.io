# SCX DI

SCX DI 是一个轻量的 Java 依赖注入核心库。

它提供显式注册 Component、按名称或类型获取 Component、构造函数注入、字段注入、值注入、单例 / 多例作用域、循环依赖检测、组件可验证性检查和自定义依赖解析器等能力。

SCX DI 不做包扫描，不提供 `@Component` / `@Service` 一类的自动发现机制，也没有复杂的生命周期模型。它的定位不是完整应用框架，而是一个小而清晰的 DI 组装核心。

[GitHub](https://github.com/scx-projects/scx-di)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-di</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX DI 中最常用的概念包括：

```text
ComponentContainer                 Component 容器接口
ComponentContainerBuilder          Component 容器构建器
ComponentDefinition                Component 定义信息
DependencyResolver                 依赖解析器
DependencyResolverContext          依赖解析上下文
DependencyResolutionIntent         依赖解析意图
DependencyPoint                    依赖点
ConstructorParameterDependencyPoint 构造函数参数依赖点
FieldDependencyPoint               字段依赖点
InjectAnnotationDependencyResolver 处理 @Inject
ValueAnnotationDependencyResolver  处理 @Value
ValueResolver                      值解析器
@Inject                            依赖注入注解
@Value                             值注入注解
```

最核心的使用方式是：

```java
var container = ComponentContainer.builder()
    .addDependencyResolver(new InjectAnnotationDependencyResolver())
    .registerComponentType("userRepository", UserRepository.class)
    .registerComponentType("userService", UserService.class)
    .build();

UserService userService = container.getComponent(UserService.class);
```

`ComponentContainer` 是对外入口。容器通过 `ComponentContainer.builder()` 创建，构建完成后可按名称或类型获取 Component。

按类型获取时，容器会查找所有 `type.isAssignableFrom(componentType)` 的 Component：

```text
没有匹配项    -> NoSuchComponentException
存在多个匹配项 -> NoUniqueComponentException
唯一匹配项    -> 返回该 Component
```

## 快速开始

### 1. 定义 Component

```java
import dev.scx.di.annotation.Inject;

public class UserService {

    @Inject
    public UserRepository userRepository;

    public String getUserName(Long id) {
        return userRepository.findNameById(id);
    }

}
```

```java
public class UserRepository {

    public String findNameById(Long id) {
        return "Tom";
    }

}
```

### 2. 创建容器并注册 Component

```java
import dev.scx.di.ComponentContainer;
import dev.scx.di.dependency_resolver.InjectAnnotationDependencyResolver;

public class Main {

    public static void main(String[] args) throws Exception {
        var container = ComponentContainer.builder()
            .addDependencyResolver(new InjectAnnotationDependencyResolver())
            .registerComponentType("userRepository", UserRepository.class)
            .registerComponentType("userService", UserService.class)
            .build();

        UserService userService = container.getComponent(UserService.class);

        System.out.println(userService.getUserName(1L));
    }

}
```

`InjectAnnotationDependencyResolver` 负责处理 `@Inject`。对于构造函数参数，它还会把没有标注任何注解的参数作为候选依赖尝试解析；对于字段，只有标注 `@Inject` 的字段才会由它处理。

## 创建容器

SCX DI 通过 builder 创建容器：

```java
var builder = ComponentContainer.builder();
```

添加依赖解析器：

```java
builder.addDependencyResolver(new InjectAnnotationDependencyResolver());
```

如果需要 `@Value`：

```java
builder.addDependencyResolver(
    new ValueAnnotationDependencyResolver(valueResolver)
);

builder.addDependencyResolver(new InjectAnnotationDependencyResolver());
```

构建容器：

```java
ComponentContainer container = builder.build();
```

`build()` 之后得到的是 `ComponentContainer`。注册信息和依赖解析器会被复制到容器内部，后续使用容器进行获取、按类型查找和验证。

## 注册 Component

### 注册类

```java
containerBuilder.registerComponentType("userService", UserService.class);
```

默认等价于：

```java
containerBuilder.registerComponentType(
    "userService",
    UserService.class,
    true,
    true
);
```

含义是：

```text
isSingleton = true
injectField = true
```

也就是注册为单例，并启用字段注入。

### 注册多例 Component

```java
containerBuilder.registerComponentType("task", Task.class, false);
```

`isSingleton = false` 时，每次获取都会重新创建实例。

```java
Task t1 = container.getComponent(Task.class);
Task t2 = container.getComponent(Task.class);

System.out.println(t1 == t2); // false
```

### 关闭字段注入

```java
containerBuilder.registerComponentType(
    "userService",
    UserService.class,
    true,
    false
);
```

这种情况下，容器只会通过构造函数创建对象，不会再执行字段初始化器。

### 注册已有实例

```java
var repository = new UserRepository();

containerBuilder.registerComponent("userRepository", repository);
```

已有实例会被注册为单例。默认情况下，注册已有实例不会进行字段注入。

如果希望给已有实例也执行字段注入：

```java
containerBuilder.registerComponent("service", service, true);
```

这表示容器会直接使用传入的实例，并在首次获取时对它执行字段初始化。

### 名称不能重复

```java
containerBuilder.registerComponentType("userService", UserService.class);

// 再次注册同名 Component 会抛出 DuplicateComponentNameException
containerBuilder.registerComponentType("userService", OtherService.class);
```

同一个 builder 中 Component name 必须唯一。

## 获取 Component

### 按类型获取

```java
UserService service = container.getComponent(UserService.class);
```

如果没有匹配类型，抛出 `NoSuchComponentException`。如果存在多个可赋值给该类型的 Component，抛出 `NoUniqueComponentException`。

### 按名称获取

```java
Object service = container.getComponent("userService");
```

如果名称不存在，抛出 `NoSuchComponentException`。

### 按名称和类型获取

```java
UserService service = container.getComponent(
    "userService",
    UserService.class
);
```

如果名称不存在，或者名称存在但类型不匹配，都会抛出 `NoSuchComponentException`。

### 获取所有 Component 定义

```java
Map<String, ComponentDefinition> definitions = container.componentDefinitions();
```

`ComponentDefinition` 是一个 record：

```java
public record ComponentDefinition(
    String componentName,
    Class<?> componentType,
    boolean isSingleton
) {
}
```

可以通过它查看当前容器中的 Component 名称、类型和是否单例：

```java
for (var definition : container.componentDefinitions().values()) {
    System.out.println(definition.componentName());
    System.out.println(definition.componentType());
    System.out.println(definition.isSingleton());
}
```

返回的 definitions 是只读视图。

### 验证所有 Component

```java
container.verifyComponents();
```

`verifyComponents()` 会主动获取容器中的每一个 Component。

```text
对于 singleton，会创建并缓存实例。
对于 prototype，会创建一个临时实例并丢弃。
```

这个方法适合在应用启动阶段提前暴露注册错误、依赖缺失、构造失败、字段注入失败和不可解决循环依赖等问题。

## 构造函数注入

SCX DI 创建类 Component 时，会选择一个 public 构造函数来创建对象。

```java
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

}
```

只要注册了 `InjectAnnotationDependencyResolver`，构造参数即使没有显式标注 `@Inject`，也会被作为候选依赖按类型解析。

### 多个构造函数时使用 `@Inject`

如果一个类有多个 public 构造函数，需要在期望使用的构造函数上标注 `@Inject`：

```java
import dev.scx.di.annotation.Inject;

public class UserService {

    public UserService() {
    }

    @Inject
    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    private final UserRepository repository;

}
```

构造函数选择规则是：

```text
1. 只使用 public 构造函数
2. 没有 public 构造函数，抛出 NoSuchConstructorException
3. 只有一个 public 构造函数，直接使用它
4. 多个 public 构造函数时，必须且只能有一个 public 构造函数标注 @Inject
5. 多个 public 构造函数都标注 @Inject，抛出 NoUniqueConstructorException
6. 多个 public 构造函数但没有任何 @Inject，抛出 NoUniqueConstructorException
```

### 构造函数中按名称注入

```java
public class UserService {

    private final UserRepository repository;

    public UserService(@Inject("primaryRepository") UserRepository repository) {
        this.repository = repository;
    }

}
```

构造参数上的 `@Inject("name")` 会按名称和类型查找 Component；没有指定名称时，按类型查找。

> 注意：当前版本中 `@Inject` 的 `value` 类型是 `String[]`。实际解析时只使用第一个值。因此推荐只写一个名称，例如 `@Inject("primaryRepository")`。

## 字段注入

字段注入由 `FieldInjectComponentInitializer` 完成。

当 `injectField = true` 时，它会遍历 Component 类型上的所有字段，并把满足条件的字段作为依赖点：

```text
public
非 final
非 static
```

```java
public class UserService {

    @Inject
    public UserRepository repository;

}
```

字段是否真正被设置，取决于依赖解析器是否能处理该字段依赖点。

在只使用内置解析器时：

```text
@Inject 字段由 InjectAnnotationDependencyResolver 处理
@Value 字段由 ValueAnnotationDependencyResolver 处理
没有 @Inject / @Value 的普通字段不会被内置解析器处理
```

如果某个依赖点没有任何匹配的 resolver，解析结果为 `null`。字段初始化器只会设置非 `null` 值，因此该字段会保持原值。

### 按名称注入字段

```java
public class UserService {

    @Inject("primaryRepository")
    public UserRepository repository;

}
```

当 `@Inject` 指定名称时，字段会通过名称和字段类型查找 Component；否则按字段类型查找。

### final / static / 非 public 字段不会被字段注入

```java
public class UserService {

    @Inject
    public final UserRepository repository = null;

}
```

`final` 字段会被跳过。

```java
public class UserService {

    @Inject
    public static UserRepository repository;

}
```

`static` 字段会被跳过。

```java
public class UserService {

    @Inject
    private UserRepository repository;

}
```

非 public 字段会被跳过。SCX DI 不做 private 字段反射注入。

## `@Inject`

`@Inject` 可以标注在：

```text
字段
构造函数参数
构造函数
```

用法：

```java
@Inject
public UserRepository repository;
```

```java
@Inject("primaryRepository")
public UserRepository repository;
```

```java
@Inject
public UserService(UserRepository repository) {
}
```

字段和构造参数上的 `@Inject` 可以指定 Component name。

构造函数上的 `@Inject` 只用于在多个 public 构造函数中标记首选构造函数，构造函数上的 `value` 会被忽略。

## `@Value`

`@Value` 用于从外部配置中注入值。

```java
import dev.scx.di.annotation.Value;

public class AppConfig {

    @Value("app.name")
    public String appName;

}
```

构造参数也可以使用：

```java
public class AppConfig {

    public final int port;

    public AppConfig(@Value("server.port") int port) {
        this.port = port;
    }

}
```

`@Value` 本身只保存 key。SCX DI 不内置配置源，真正如何根据 key 取值由 `ValueResolver` 决定。

当前版本中，`ValueResolver` 是 `ValueAnnotationDependencyResolver` 的内部接口：

```java
public interface ValueResolver {

    Object resolveValue(String key, TypeInfo targetType) throws Exception;

}
```

`resolveValue(key, targetType)` 会收到注解中的 key 和目标类型。缺失值、`null`、类型转换等语义都由实现类自行决定。

### 使用 Map 作为配置源

```java
import dev.scx.di.dependency_resolver.ValueAnnotationDependencyResolver;
import dev.scx.reflect.TypeInfo;

import java.util.Map;

public class MapValueResolver implements ValueAnnotationDependencyResolver.ValueResolver {

    private final Map<String, Object> values;

    public MapValueResolver(Map<String, Object> values) {
        this.values = values;
    }

    @Override
    public Object resolveValue(String key, TypeInfo targetType) throws Exception {
        if (!values.containsKey(key)) {
            throw new IllegalStateException("Missing value: " + key);
        }
        return values.get(key);
    }

}
```

注册：

```java
containerBuilder.addDependencyResolver(
    new ValueAnnotationDependencyResolver(
        new MapValueResolver(Map.of(
            "app.name", "demo",
            "server.port", 8080
        ))
    )
);
```

## 依赖解析器

SCX DI 的依赖注入过程由 `DependencyResolver` 驱动。

```java
public interface DependencyResolver {

    DependencyResolutionIntent match(DependencyPoint dependencyPoint);

    Object resolve(DependencyPoint dependencyPoint,
                   DependencyResolverContext context) throws Exception;

}
```

依赖解析器不再区分 `matchConstructorArgument` / `matchFieldValue` 这类方法。新版统一以 `DependencyPoint` 表示依赖点。

内置依赖点类型包括：

```text
ConstructorParameterDependencyPoint  构造函数参数依赖点
FieldDependencyPoint                 字段依赖点
```

`DependencyResolverContext` 提供从容器继续获取 Component 的能力：

```java
public interface DependencyResolverContext {

    Object getComponent(String name);

    <T> T getComponent(Class<T> type);

    <T> T getComponent(String name, Class<T> type);

}
```

### 解析意图

`DependencyResolutionIntent` 有三个值：

```text
NOT_APPLICABLE  不能处理
CANDIDATE       可以处理
REQUIRED        必须处理
```

选择规则：

```text
1. 如果正好有一个 REQUIRED，使用它
2. 如果有多个 REQUIRED，抛出 DependencyResolutionException
3. 如果没有 REQUIRED，但正好有一个 CANDIDATE，使用它
4. 如果有多个 CANDIDATE，抛出 DependencyResolutionException
5. 如果没有任何匹配，返回 null
```

这个规则保证了解析器之间不会因为注册顺序而静默覆盖。多个强制解析器或多个候选解析器同时匹配同一个依赖点，都会被视为歧义。

### 内置解析器

SCX DI 提供两个常用解析器：

```java
new InjectAnnotationDependencyResolver()
```

```java
new ValueAnnotationDependencyResolver(valueResolver)
```

`InjectAnnotationDependencyResolver` 处理 `@Inject`，并承担构造函数参数的默认按类型注入。

`ValueAnnotationDependencyResolver` 处理 `@Value`，并把实际取值逻辑委托给 `ValueResolver`。

## 自定义依赖解析器

可以实现 `DependencyResolver` 来支持自定义注解或自定义来源。

例如支持 `@CurrentUser`：

```java
import dev.scx.di.dependency_point.ConstructorParameterDependencyPoint;
import dev.scx.di.dependency_point.DependencyPoint;
import dev.scx.di.dependency_point.FieldDependencyPoint;
import dev.scx.di.dependency_resolver.DependencyResolutionIntent;
import dev.scx.di.dependency_resolver.DependencyResolver;
import dev.scx.di.dependency_resolver.DependencyResolverContext;

public class CurrentUserResolver implements DependencyResolver {

    @Override
    public DependencyResolutionIntent match(DependencyPoint dependencyPoint) {
        if (dependencyPoint instanceof ConstructorParameterDependencyPoint p) {
            return p.parameter().findAnnotation(CurrentUser.class) != null
                ? DependencyResolutionIntent.REQUIRED
                : DependencyResolutionIntent.NOT_APPLICABLE;
        }
        if (dependencyPoint instanceof FieldDependencyPoint f) {
            return f.field().findAnnotation(CurrentUser.class) != null
                ? DependencyResolutionIntent.REQUIRED
                : DependencyResolutionIntent.NOT_APPLICABLE;
        }
        return DependencyResolutionIntent.NOT_APPLICABLE;
    }

    @Override
    public Object resolve(DependencyPoint dependencyPoint,
                          DependencyResolverContext context) {
        return CurrentUserHolder.get();
    }

}
```

注册：

```java
containerBuilder.addDependencyResolver(new CurrentUserResolver());
```

自定义解析器适合处理配置、上下文对象、运行时状态、外部资源、特殊注解等。

## 支持的 Component 类型

通过 `registerComponentType(...)` 注册类时，SCX DI 会检查 Component class 是否可用于反射创建。

不支持：

```text
接口
注解
枚举
抽象类
非 static 成员类
没有 public 构造函数的类
```

支持：

```text
普通 public 构造函数类
static 成员类
record
```

SCX DI 只使用 public 构造函数创建 Component。

## 单例和多例

### 单例

默认注册类就是单例：

```java
containerBuilder.registerComponentType("userService", UserService.class);
```

单例 Component 第一次获取时创建，之后返回同一个实例。

单例内部有四个状态：

```text
NULL         未开始
CREATING     正在创建
INITIALIZING 正在初始化
READY        完全就绪
```

字段注入循环依赖能够被解决，依赖的就是单例在 `INITIALIZING` 阶段可以提前返回早期对象。

如果创建或初始化失败，单例状态会回滚到 `NULL`，并丢弃当前脏对象。

### 多例

```java
containerBuilder.registerComponentType("task", Task.class, false);
```

多例 Component 每次获取都会重新创建：

```java
Task t1 = container.getComponent(Task.class);
Task t2 = container.getComponent(Task.class);

System.out.println(t1 == t2); // false
```

多例不会缓存实例，也不会暴露早期对象。

## 循环依赖

SCX DI 内置循环依赖检测，并支持一部分字段注入循环依赖。

循环依赖检测基于依赖链：

```text
ComponentFrame
DependencyPointFrame
ComponentFrame
DependencyPointFrame
...
```

进入 Component 时，容器会检查当前依赖链中是否已经出现同一个 Component 类型。如果出现，就根据循环链判断是否可解决。

### 可以解决的循环依赖

单例之间的字段注入循环可以解决：

```java
public class A {
    @Inject
    public B b;
}

public class B {
    @Inject
    public A a;
}
```

只要循环链中没有构造函数依赖点，并且循环链中存在至少一个单例，SCX DI 就认为该字段循环可以解决。

原因是字段注入发生在对象创建之后。单例对象在 `INITIALIZING` 阶段可以提前暴露自身，从而打破字段循环。

### 不可解决的循环依赖

构造函数循环依赖不可解决：

```java
public class A {
    public A(B b) {
    }
}

public class B {
    public B(A a) {
    }
}
```

构造函数 + 字段注入混合形成的循环也不可解决：

```java
public class A {
    public A(B b) {
    }
}

public class B {
    @Inject
    public A a;
}
```

全部都是多例的字段循环也不可解决：

```java
containerBuilder.registerComponentType("a", A.class, false);
containerBuilder.registerComponentType("b", B.class, false);
```

不可解决循环会抛出 `ComponentCreationException`，并生成可视化循环链错误信息。

### 循环依赖判断规则

```text
1. 循环链中只要出现构造函数参数依赖点，就不可解决
2. 如果循环链全部是字段依赖，并且存在至少一个 singleton，则可以解决
3. 如果循环链全部是字段依赖，但全部都是 prototype，则不可解决
```

## 异常

常见异常包括：

```text
NoSuchComponentException        找不到 Component
NoUniqueComponentException      按类型查找时存在多个匹配 Component
DuplicateComponentNameException Component 名称重复
IllegalComponentClassException  Component class 不合法
NoSuchConstructorException      找不到可用 public 构造函数
NoUniqueConstructorException    构造函数选择不唯一
ComponentCreationException      Component 创建、依赖解析或字段注入失败
DependencyResolutionException   依赖解析器选择冲突
```

异常类位于 `dev.scx.di.exception` 包下。

## 完整示例

```java
import dev.scx.di.ComponentContainer;
import dev.scx.di.annotation.Inject;
import dev.scx.di.annotation.Value;
import dev.scx.di.dependency_resolver.InjectAnnotationDependencyResolver;
import dev.scx.di.dependency_resolver.ValueAnnotationDependencyResolver;
import dev.scx.reflect.TypeInfo;

import java.util.Map;

public class DIExample {

    public static void main(String[] args) throws Exception {
        var container = ComponentContainer.builder()
            .addDependencyResolver(
                new ValueAnnotationDependencyResolver(
                    new MapValueResolver(Map.of(
                        "app.name", "demo-app",
                        "server.port", 8080
                    ))
                )
            )
            .addDependencyResolver(new InjectAnnotationDependencyResolver())
            .registerComponentType("config", AppConfig.class)
            .registerComponentType("userRepository", UserRepository.class)
            .registerComponentType("userService", UserService.class)
            .build();

        container.verifyComponents();

        UserService service = container.getComponent(UserService.class);

        System.out.println(service.hello(1L));
    }

    public static class AppConfig {

        @Value("app.name")
        public String appName;

        public final int port;

        public AppConfig(@Value("server.port") int port) {
            this.port = port;
        }

    }

    public static class UserRepository {

        public String findNameById(Long id) {
            return "Tom";
        }

    }

    public static class UserService {

        public final AppConfig config;

        @Inject
        public UserRepository repository;

        public UserService(AppConfig config) {
            this.config = config;
        }

        public String hello(Long id) {
            return "app=" + config.appName
                + ", port=" + config.port
                + ", user=" + repository.findNameById(id);
        }

    }

    public static class MapValueResolver implements ValueAnnotationDependencyResolver.ValueResolver {

        private final Map<String, Object> values;

        public MapValueResolver(Map<String, Object> values) {
            this.values = values;
        }

        @Override
        public Object resolveValue(String key, TypeInfo targetType) {
            if (!values.containsKey(key)) {
                throw new IllegalStateException("Missing value: " + key);
            }
            return values.get(key);
        }

    }

}
```

## 设计说明

### 1. SCX DI 不做自动扫描

当前公开 API 以手动注册为主：

```java
containerBuilder.registerComponentType("name", ComponentClass.class);
containerBuilder.registerComponent("name", instance);
```

仓库中没有类似 `@Component`、`@Service`、包扫描或自动发现 API。

### 2. 容器创建使用 builder

新版对外使用方式是：

```java
ComponentContainer container = ComponentContainer.builder()
    .addDependencyResolver(new InjectAnnotationDependencyResolver())
    .registerComponentType("name", ComponentClass.class)
    .build();
```

不需要直接 new 默认容器实现。

### 3. 构造函数注入默认按类型，字段注入需要 resolver 匹配

构造函数参数即使没有 `@Inject`，也会被 `InjectAnnotationDependencyResolver` 作为候选依赖解析。

字段初始化器会遍历 public、非 final、非 static 字段，但字段是否被注入由 resolver 决定。使用内置 resolver 时，只有 `@Inject` 和 `@Value` 字段会被处理。

### 4. 字段注入不处理 private / final / static 字段

SCX DI 不做 private 字段反射注入，也不会修改 final 字段或 static 字段。

### 5. 构造函数中不允许拿到半成品对象

只要循环链中存在构造函数参数依赖点，就被视为不可解决。构造函数中拿到的依赖应该永远是完整对象，而不是半成品对象。

### 6. `@Value` 的实际来源由用户决定

SCX DI 不内置配置文件读取，也不规定类型转换规则。`@Value` 只声明 key，`ValueResolver` 决定如何解析。

### 7. 依赖解析器是唯一扩展点

SCX DI 的扩展核心是 `DependencyResolver`。

如果要支持新的注解、新的上下文对象、新的值来源或特殊依赖规则，应通过实现 `DependencyResolver` 完成，而不是扩展容器本身。

## 常见问题

### 为什么没有自动包扫描？

SCX DI 当前定位是轻量 DI 核心，公开 API 是手动注册 Component class 或已有实例，没有提供包扫描能力。

### 为什么按接口类型获取时报 `NoUniqueComponentException`？

因为多个 Component 的类型都可以赋值给该接口。按类型获取要求结果唯一；如果有多个实现，请按名称获取，或者在注入点使用 `@Inject("componentName")`。

### 为什么我的字段没有被注入？

使用内置 resolver 时，字段注入需要满足：

```text
字段是 public
字段不是 final
字段不是 static
字段标注了 @Inject 或 @Value
注册该 Component 时 injectField = true
容器中添加了对应的 DependencyResolver
```

例如只写了 `@Inject`，但没有添加 `InjectAnnotationDependencyResolver`，字段不会被成功解析。

### 为什么多个 public 构造函数会报错？

多个 public 构造函数时，容器无法自动判断应该使用哪一个。请在唯一一个期望使用的 public 构造函数上标注 `@Inject`。

如果没有任何 `@Inject`，或者多个 public 构造函数都标注了 `@Inject`，都会抛出 `NoUniqueConstructorException`。

### 为什么构造函数循环依赖不能解决？

SCX DI 不允许构造函数拿到半成品对象。循环链中只要出现构造函数参数依赖点，就会被视为不可解决循环依赖，并抛出 `ComponentCreationException`。

### 单例字段循环为什么可以工作？

字段注入发生在对象创建之后。对于单例 Component，容器可以在字段注入过程中提前暴露正在初始化的对象，从而打破字段循环。

### `@Value` 会自动做类型转换吗？

SCX DI 自身不做值转换。`@Value` 的 key 会交给 `ValueResolver`，解析结果是什么、是否转换类型、缺失值是否抛异常，都由 `ValueResolver` 实现决定。

### 没有匹配的 DependencyResolver 会怎样？

如果某个依赖点没有任何 resolver 匹配，解析结果为 `null`。

对于字段注入，`null` 不会写入字段，字段保持原值。

对于构造函数参数，`null` 会作为参数值传入构造函数。如果目标参数是 primitive，最终可能在反射调用阶段失败。

### `verifyComponents()` 和直接 `getComponent(...)` 有什么区别？

`getComponent(...)` 只获取指定 Component。

`verifyComponents()` 会遍历容器中所有 Component，并尝试逐个获取。它适合用于启动阶段提前检查整个容器是否可用。
