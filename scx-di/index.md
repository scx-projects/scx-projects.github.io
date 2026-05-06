# SCX DI

SCX DI 是一个轻量的 Java 依赖注入容器。

它提供手动注册 Component、按名称或类型获取 Component、构造函数注入、字段注入、值注入、单例 / 多例作用域、循环依赖检测和自定义依赖解析器等能力。SCX DI 不做包扫描，也没有复杂的生命周期模型；它更像一个小而清晰的 DI 核心库。当前版本为 `0.2.0`，依赖 `scx-reflect`。([GitHub][1])

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-di</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX DI 中最常用的概念包括：

```text
ComponentContainer             Component 容器接口
DefaultComponentContainer      默认容器实现
ComponentProvider              Component 提供器
ComponentDependencyResolver    依赖解析器
InjectAnnotationDependencyResolver 处理 @Inject
ValueAnnotationDependencyResolver  处理 @Value
ValueResolver                  值解析器
ComponentResolutionContext     解析上下文，用于检测循环依赖
@Inject                        依赖注入注解
@Value                         值注入注解
```

最核心的使用方式是：

```java
var container = new DefaultComponentContainer();

container.dependencyResolvers().add(new InjectAnnotationDependencyResolver(container));

container.registerComponentClass("userService", UserService.class);
container.registerComponentClass("userRepository", UserRepository.class);

UserService userService = container.getComponent(UserService.class);
```

`DefaultComponentContainer` 内部维护 Component provider 映射和 dependency resolver 列表；按类型获取时会查找所有 `type.isAssignableFrom(provider.componentType())` 的 Component，如果没有匹配会抛出 `NoSuchComponentException`，如果匹配到多个会抛出 `NoUniqueComponentException`。([GitHub][2])

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
import dev.scx.di.DefaultComponentContainer;
import dev.scx.di.dependency_resolver.InjectAnnotationDependencyResolver;

public class Main {

    public static void main(String[] args) throws Exception {
        var container = new DefaultComponentContainer();

        container.dependencyResolvers()
            .add(new InjectAnnotationDependencyResolver(container));

        container.registerComponentClass("userService", UserService.class);
        container.registerComponentClass("userRepository", UserRepository.class);

        UserService userService = container.getComponent(UserService.class);

        System.out.println(userService.getUserName(1L));
    }

}
```

`InjectAnnotationDependencyResolver` 负责处理 `@Inject`，并通过容器继续查找依赖 Component；字段注入只处理标注了 `@Inject` 的字段，构造参数即使没有标注 `@Inject`，也会作为候选依赖尝试解析。([GitHub][3])

## 创建容器

```java
var container = new DefaultComponentContainer();
```

`DefaultComponentContainer` 默认没有任何依赖解析器。你需要按需求添加：

```java
container.dependencyResolvers()
    .add(new InjectAnnotationDependencyResolver(container));
```

如果需要 `@Value`：

```java
container.dependencyResolvers()
    .add(new ValueAnnotationDependencyResolver(valueResolver));

container.dependencyResolvers()
    .add(new InjectAnnotationDependencyResolver(container));
```

`ComponentContainer#dependencyResolvers()` 返回可修改列表；依赖解析时会遍历这个列表，根据每个解析器返回的 `DependencyResolutionIntent` 选择最终解析器。([GitHub][4])

## 注册 Component

### 注册类

```java
container.registerComponentClass("userService", UserService.class);
```

默认等价于：

```java
container.registerComponentClass("userService", UserService.class, true, true);
```

含义是：

```text
singleton = true
injectField = true
```

也就是注册为单例，并启用字段注入。`ComponentContainer` 的默认方法中明确了这组默认值。([GitHub][4])

### 注册多例 Component

```java
container.registerComponentClass("task", Task.class, false);
```

`singleton = false` 时，每次获取都会重新创建实例。

```java
Task t1 = container.getComponent(Task.class);
Task t2 = container.getComponent(Task.class);

System.out.println(t1 == t2); // false
```

测试代码中也验证了多例 Component 每次获取不是同一个对象。([GitHub][5])

### 关闭字段注入

```java
container.registerComponentClass(
    "userService",
    UserService.class,
    true,
    false
);
```

这种情况下，容器只会通过构造函数创建对象，不会再处理 public 字段上的 `@Inject` 或 `@Value`。

### 注册已有实例

```java
var repository = new UserRepository();

container.registerComponent("userRepository", repository);
```

已有实例会被包装为 `InstanceComponentProvider`，始终返回同一个对象，因此是单例。([GitHub][6])

如果希望给已有实例也做字段注入：

```java
container.registerComponent("service", service, true);
```

测试代码中也验证了直接注册已有对象，并开启字段注入后，容器会给这个已有对象注入依赖。([GitHub][7])

### 名称不能重复

```java
container.registerComponentClass("userService", UserService.class);

// 再次注册同名 Component 会抛出 DuplicateComponentNameException
container.registerComponentClass("userService", OtherService.class);
```

`DefaultComponentContainer` 注册时使用 `putIfAbsent`，如果名称已经存在，会抛出 `DuplicateComponentNameException`。([GitHub][2])

## 获取 Component

### 按类型获取

```java
UserService service = container.getComponent(UserService.class);
```

如果没有匹配类型，抛出 `NoSuchComponentException`；如果存在多个可赋值给该类型的 Component，抛出 `NoUniqueComponentException`。([GitHub][2])

### 按名称获取

```java
Object service = container.getComponent("userService");
```

### 按名称和类型获取

```java
UserService service = container.getComponent(
    "userService",
    UserService.class
);
```

如果名称存在但类型不匹配，也会抛出 `NoSuchComponentException`。测试代码中覆盖了“名称不存在”和“名称存在但类型不符合”的情况。([GitHub][7])

### 获取所有 Component 名称

```java
String[] names = container.getComponentNames();
```

`getComponentNames()` 会从 `providers().keySet()` 中返回当前所有注册名称。([GitHub][4])

### 初始化所有 Component

```java
container.initializeComponents();
```

`initializeComponents()` 会遍历所有 provider 并调用 `getComponent(...)`，从而触发单例 Component 的创建和依赖注入。([GitHub][4])

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

构造参数不需要显式标注 `@Inject`，`InjectAnnotationDependencyResolver` 会把未标注的构造参数作为候选依赖解析。([GitHub][3])

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
4. 多个 public 构造函数时，必须且只能有一个构造函数标注 @Inject
5. 多个构造函数都标注 @Inject，抛出 NoUniqueConstructorException
```

这些规则由 `ConstructorInjectComponentProvider#findPreferredConstructor(...)` 实现，测试代码也覆盖了多个构造函数、无 public 构造函数、多个 `@Inject` 构造函数等情况。([GitHub][8])

### 构造函数中按名称注入

```java
public class UserService {

    private final UserRepository repository;

    public UserService(@Inject("primaryRepository") UserRepository repository) {
        this.repository = repository;
    }

}
```

构造参数上的 `@Inject("name")` 会优先按名称和类型查找 Component；没有指定名称时，按类型查找。([GitHub][3])

## 字段注入

字段注入只处理 public、非 final、标注了 `@Inject` 或 `@Value` 的字段。

```java
public class UserService {

    @Inject
    public UserRepository repository;

}
```

`FieldInjectComponentProvider` 会遍历 `classInfo.allFields()`，只处理 public 且非 final 的字段；如果解析出的值为 `null`，则不会设置字段。([GitHub][9])

### 按名称注入字段

```java
public class UserService {

    @Inject("primaryRepository")
    public UserRepository repository;

}
```

当 `@Inject` 指定名称时，字段会通过 `container.getComponent(name, fieldType, context)` 获取；否则通过字段类型获取。([GitHub][3])

### final 字段不会被字段注入

```java
public class UserService {

    @Inject
    public final UserRepository repository = null;

}
```

final 字段会被跳过。测试代码中也验证了 final 字段不会被重复注入，字段仍然保持 `null`。([GitHub][7])

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

`@Inject` 的源码注释说明：字段和参数可以写 `@Inject` 或 `@Inject("componentName")`；构造函数上可以用 `@Inject` 标记该构造函数可用于 DI，但构造函数上的 `value` 参数会被忽略。([GitHub][10])

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

`@Value` 本身只保存 key。SCX DI 不内置配置源，真正如何根据 key 取值由 `ValueResolver` 决定。`ValueResolver#resolveValue(key, targetType)` 会收到注解中的 key 和目标类型，缺失值、`null`、类型转换等语义都由实现类自行决定。([GitHub][11])

### 使用 Map 作为配置源

```java
import dev.scx.di.dependency_resolver.ValueResolver;
import dev.scx.reflect.TypeInfo;

import java.util.Map;

public class MapValueResolver implements ValueResolver {

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
container.dependencyResolvers().add(
    new ValueAnnotationDependencyResolver(
        new MapValueResolver(Map.of(
            "app.name", "demo",
            "server.port", 8080
        ))
    )
);
```

测试代码中的 `MapValueResolver` 就是一个最小实现：从 `Map` 中按 key 取值，取不到时抛异常。([GitHub][12])

## 依赖解析器

SCX DI 的依赖注入过程由 `ComponentDependencyResolver` 驱动。

```java
public interface ComponentDependencyResolver {

    DependencyResolutionIntent matchConstructorArgument(ParameterInfo parameterInfo);

    DependencyResolutionIntent matchFieldValue(FieldInfo fieldInfo);

    Object resolveConstructorArgument(ParameterInfo parameterInfo,
                                      ComponentResolutionContext context) throws Exception;

    Object resolveFieldValue(FieldInfo fieldInfo,
                             ComponentResolutionContext context) throws Exception;

}
```

依赖解析器可以处理构造函数参数，也可以处理字段。([GitHub][13])

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

这个规则由 `ComponentDependencyResolverHelper#chooseWinnerResolver(...)` 实现。([GitHub][14])

### 内置解析器

SCX DI 提供两个常用解析器：

```java
new InjectAnnotationDependencyResolver(container)

new ValueAnnotationDependencyResolver(valueResolver)
```

`InjectAnnotationDependencyResolver` 处理 `@Inject`，并承担构造函数参数的默认按类型注入；`ValueAnnotationDependencyResolver` 处理 `@Value`，并把实际取值逻辑委托给 `ValueResolver`。([GitHub][3])

## 自定义依赖解析器

可以实现 `ComponentDependencyResolver` 来支持自定义注解或自定义来源。

例如支持 `@CurrentUser`：

```java
public class CurrentUserResolver implements ComponentDependencyResolver {

    @Override
    public DependencyResolutionIntent matchConstructorArgument(ParameterInfo parameterInfo) {
        return parameterInfo.findAnnotation(CurrentUser.class) != null
            ? DependencyResolutionIntent.REQUIRED
            : DependencyResolutionIntent.NOT_APPLICABLE;
    }

    @Override
    public DependencyResolutionIntent matchFieldValue(FieldInfo fieldInfo) {
        return fieldInfo.findAnnotation(CurrentUser.class) != null
            ? DependencyResolutionIntent.REQUIRED
            : DependencyResolutionIntent.NOT_APPLICABLE;
    }

    @Override
    public Object resolveConstructorArgument(ParameterInfo parameterInfo,
                                             ComponentResolutionContext context) {
        return CurrentUserHolder.get();
    }

    @Override
    public Object resolveFieldValue(FieldInfo fieldInfo,
                                    ComponentResolutionContext context) {
        return CurrentUserHolder.get();
    }

}
```

注册：

```java
container.dependencyResolvers().add(new CurrentUserResolver());
```

自定义解析器适合处理配置、上下文对象、运行时状态、外部资源、特殊注解等。

## 支持的 Component 类型

通过 `registerComponentClass(...)` 注册类时，SCX DI 会检查 Component class 是否可用于反射创建。

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

`ConstructorInjectComponentProvider#checkClass(...)` 会拒绝接口、注解、枚举、抽象类和非静态成员类；测试代码中也验证了这些非法类型会抛出 `IllegalComponentClassException`，同时验证了 record 可以注册。([GitHub][8])

## 单例和多例

### 单例

默认注册类就是单例：

```java
container.registerComponentClass("userService", UserService.class);
```

单例 Component 第一次获取时创建，之后返回同一个实例。`SingletonComponentProvider` 内部会缓存第一次创建出的对象。([GitHub][15])

### 多例

```java
container.registerComponentClass("task", Task.class, false);
```

多例 Component 每次获取都会重新创建：

```java
Task t1 = container.getComponent(Task.class);
Task t2 = container.getComponent(Task.class);

System.out.println(t1 == t2); // false
```

测试代码中也覆盖了“多例正常依赖”“单例依赖多例”“多例依赖单例”等场景。([GitHub][5])

## 循环依赖

SCX DI 内置循环依赖检测，并支持一部分字段注入循环依赖。

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

只要循环链中存在至少一个单例，且循环链中没有构造函数注入，容器可以通过提前暴露半成品对象解决字段注入循环。`ComponentResolutionContext#isUnsolvableCycle(...)` 中的规则是：构造器循环不可解决；全部都是字段注入时，只要链路中存在任意一个单例就可解决；如果全部都是多例，则不可解决。([GitHub][16])

测试代码中覆盖了单例字段循环、多层字段循环、混合作用域字段循环等场景。([GitHub][17])

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
container.registerComponentClass("a", A.class, false);
container.registerComponentClass("b", B.class, false);
```

这些情况会抛出 `ComponentCreationException`，并生成可视化循环链错误信息。([GitHub][16])

## 异常

常见异常包括：

```text
NoSuchComponentException        找不到 Component
NoUniqueComponentException      按类型查找时存在多个匹配 Component
DuplicateComponentNameException Component 名称重复
IllegalComponentClassException  Component class 不合法
NoSuchConstructorException      找不到可用 public 构造函数
NoUniqueConstructorException    构造函数选择不唯一
ComponentCreationException      Component 创建或注入失败
DependencyResolutionException   依赖解析器选择冲突
```

异常类位于 `dev.scx.di.exception` 包下，仓库中包含上述异常类型。([GitHub][18])

## 完整示例

```java
import dev.scx.di.DefaultComponentContainer;
import dev.scx.di.annotation.Inject;
import dev.scx.di.annotation.Value;
import dev.scx.di.dependency_resolver.InjectAnnotationDependencyResolver;
import dev.scx.di.dependency_resolver.ValueAnnotationDependencyResolver;
import dev.scx.di.dependency_resolver.ValueResolver;
import dev.scx.reflect.TypeInfo;

import java.util.Map;

public class DIExample {

    public static void main(String[] args) throws Exception {
        var container = new DefaultComponentContainer();

        container.dependencyResolvers().add(
            new ValueAnnotationDependencyResolver(
                new MapValueResolver(Map.of(
                    "app.name", "demo-app",
                    "server.port", 8080
                ))
            )
        );

        container.dependencyResolvers().add(
            new InjectAnnotationDependencyResolver(container)
        );

        container.registerComponentClass("config", AppConfig.class);
        container.registerComponentClass("userRepository", UserRepository.class);
        container.registerComponentClass("userService", UserService.class);

        container.initializeComponents();

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

    public static class MapValueResolver implements ValueResolver {

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

测试代码中也使用了 `DefaultComponentContainer`、`InjectAnnotationDependencyResolver`、`ValueAnnotationDependencyResolver` 和 `MapValueResolver` 组合完成 `@Inject` 与 `@Value` 注入。([GitHub][5])

## 设计说明

### 1. SCX DI 不做自动扫描

当前公开 API 以手动注册为主：

```java
container.registerComponentClass("name", ComponentClass.class);
container.registerComponent("name", instance);
```

仓库中没有类似 `@Component`、`@Service`、包扫描或自动发现 API。`ComponentContainer` 的核心能力就是注册 provider、获取 Component 和初始化 Component。([GitHub][4])

### 2. 构造函数注入默认按类型，字段注入必须显式标注

构造函数参数即使没有 `@Inject`，也会被 `InjectAnnotationDependencyResolver` 作为候选依赖解析；字段则必须标注 `@Inject` 才会处理。([GitHub][3])

### 3. 字段注入只处理 public 非 final 字段

SCX DI 不做 private 字段反射注入，也不会修改 final 字段。字段注入实现中明确跳过非 public 或 final 字段。([GitHub][9])

### 4. 构造函数中不允许拿到半成品对象

只要循环链中存在构造函数注入，就被视为不可解决。源码注释解释了原因：构造函数中拿到的依赖应该永远是完整对象，而不是半成品对象。([GitHub][16])

### 5. `@Value` 的实际来源由用户决定

SCX DI 不内置配置文件读取，也不规定类型转换规则。`@Value` 只声明 key，`ValueResolver` 决定如何解析。([GitHub][19])

## 常见问题

### 为什么没有自动包扫描？

SCX DI 当前定位是轻量 DI 核心，公开 API 是手动注册 Component class 或已有实例，没有提供包扫描能力。([GitHub][4])

### 为什么按接口类型获取时报 `NoUniqueComponentException`？

因为多个 Component 的类型都可以赋值给该接口。按类型获取要求结果唯一；如果有多个实现，请按名称获取，或者在注入点使用 `@Inject("componentName")`。([GitHub][2])

### 为什么我的字段没有被注入？

字段注入需要满足三个条件：

```text
字段是 public
字段不是 final
字段标注了 @Inject 或 @Value
```

`FieldInjectComponentProvider` 只处理 public、非 final 字段。([GitHub][9])

### 为什么多个 public 构造函数会报错？

多个 public 构造函数时，容器无法自动判断应该使用哪一个。请在唯一一个期望使用的构造函数上标注 `@Inject`。如果没有任何 `@Inject` 或多个构造函数都标注了 `@Inject`，都会抛出 `NoUniqueConstructorException`。([GitHub][8])

### 为什么构造函数循环依赖不能解决？

SCX DI 不允许构造函数拿到半成品对象。循环链中只要出现构造函数注入，就会被视为不可解决循环依赖，并抛出 `ComponentCreationException`。([GitHub][16])

### 单例字段循环为什么可以工作？

字段注入发生在对象创建之后。对于单例 Component，容器可以在字段注入过程中提前暴露正在注入的对象，从而打破字段循环。`FieldInjectComponentProvider` 中专门有 `INJECTING` 状态和早期对象返回逻辑。([GitHub][9])

### `@Value` 会自动做类型转换吗？

SCX DI 自身不做值转换。`@Value` 的 key 会交给 `ValueResolver`，解析结果是什么、是否转换类型、缺失值是否抛异常，都由 `ValueResolver` 实现决定。([GitHub][11])

[1]: https://raw.githubusercontent.com/scx-projects/scx-di/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/DefaultComponentContainer.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/dependency_resolver/InjectAnnotationDependencyResolver.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/ComponentContainer.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/test/java/dev/scx/di/test/DITest.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/provider/InstanceComponentProvider.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/test/java/dev/scx/di/test/DITest5.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/provider/ConstructorInjectComponentProvider.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/provider/FieldInjectComponentProvider.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/annotation/Inject.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/annotation/Value.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/test/java/dev/scx/di/test/MapValueResolver.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/dependency_resolver/ComponentDependencyResolver.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/dependency_resolver/DependencyResolutionIntent.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/provider/SingletonComponentProvider.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/resolution_context/ComponentResolutionContext.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/test/java/dev/scx/di/test/DITest6.java "raw.githubusercontent.com"
[18]: https://github.com/scx-projects/scx-di/tree/master/src/main/java/dev/scx/di/exception "scx-di/src/main/java/dev/scx/di/exception at master · scx-projects/scx-di · GitHub"
[19]: https://raw.githubusercontent.com/scx-projects/scx-di/master/src/main/java/dev/scx/di/dependency_resolver/ValueResolver.java "raw.githubusercontent.com"
