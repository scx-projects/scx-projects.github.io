# SCX Reflect

SCX Reflect 是一个轻量的 Java 反射信息抽象库。

它在 Java 原生反射 `Class`、`Type`、`Field`、`Method`、`Constructor`、`Parameter`、`RecordComponent` 之上，提供了一套更统一、更适合泛型解析的模型：`TypeInfo`、`ClassInfo`、`FieldInfo`、`MethodInfo`、`ConstructorInfo`、`ParameterInfo`、`RecordComponentInfo` 和 `TypeBindings`。当前版本为 `0.2.0`。([GitHub][1])

SCX Reflect 的重点不是替代 Java 反射，而是让上层库更方便地处理：

```text
泛型类型解析
字段 / 方法 / 构造函数信息读取
record component 信息读取
继承层级分析
方法重写关系分析
注解读取
类型变量绑定
TypeReference 泛型捕获
```

`scx-di`、`scx-sql`、`scx-serialize` 等库都可以基于这类反射模型做依赖注入、Bean 映射、序列化、类型处理等工作。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-reflect</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX Reflect 中最常用的概念包括：

```text
ScxReflect              入口工具类
TypeReference           泛型类型捕获工具
TypeInfo                类型信息总接口
ClassInfo               普通类 / 接口 / 枚举 / 注解 / record 类型
PrimitiveTypeInfo       基本类型
ArrayTypeInfo           数组类型
TypeBindings            泛型变量绑定信息
FieldInfo               字段信息
MethodInfo              方法信息
ConstructorInfo         构造函数信息
ParameterInfo           参数信息
RecordComponentInfo     record component 信息
AccessModifier          访问修饰符
ClassKind               类类型
```

`TypeInfo` 是核心类型模型。它只保留三类最终类型：`ClassInfo`、`PrimitiveTypeInfo` 和 `ArrayTypeInfo`；`TypeVariable` 和 `WildcardType` 不会作为长期存在的类型模型，而是会被解析成具体类型，或退化为上界。源码注释中明确说明了这套设计。([GitHub][2])

## 快速开始

### 获取普通类型信息

```java
import dev.scx.reflect.ClassInfo;
import dev.scx.reflect.ScxReflect;

ClassInfo type = (ClassInfo) ScxReflect.typeOf(User.class);

System.out.println(type.name());
System.out.println(type.classKind());
System.out.println(type.accessModifier());
```

### 获取字段信息

```java
for (var field : type.fields()) {
    System.out.println(field.name() + " : " + field.fieldType());
}
```

### 获取方法信息

```java
for (var method : type.methods()) {
    System.out.println(method.name() + " -> " + method.returnType());
}
```

### 获取泛型类型信息

```java
import dev.scx.reflect.ClassInfo;
import dev.scx.reflect.ScxReflect;
import dev.scx.reflect.TypeReference;

import java.util.List;

ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);

System.out.println(type.rawClass());       // interface java.util.List
System.out.println(type.bindings().get(0)); // String
```

`ScxReflect` 提供 `typeOf(Class)`、`typeOf(Type)` 和 `typeOf(TypeReference)` 三个入口；`TypeReference` 通过匿名子类捕获真实泛型参数。([GitHub][3])

## ScxReflect

`ScxReflect` 是最常用的入口类。

```java
TypeInfo type1 = ScxReflect.typeOf(String.class);

TypeInfo type2 = ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);

TypeInfo type3 = ScxReflect.typeOf(someJavaReflectType);
```

它提供三个静态方法：

```java
typeOf(Class clazz)

typeOf(Type type)

typeOf(TypeReference typeReference)
```

内部会把 Java 反射中的 `Class`、`ParameterizedType`、`GenericArrayType`、`TypeVariable` 和 `WildcardType` 转换成 SCX Reflect 自己的 `TypeInfo` 模型。([GitHub][3])

## TypeReference

`TypeReference` 用于捕获泛型类型。

```java
TypeInfo type = ScxReflect.typeOf(
    new TypeReference<Map<String, List<Integer>>>() {}
);
```

不要这样写：

```java
new TypeReference() {};
```

`TypeReference` 必须带实际类型参数。如果没有实际类型参数，构造函数会抛出 `IllegalArgumentException`。([GitHub][4])

常见用法：

```java
TypeInfo listString = ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);

TypeInfo mapType = ScxReflect.typeOf(
    new TypeReference<Map<String, Integer>>() {}
);

TypeInfo arrayType = ScxReflect.typeOf(
    new TypeReference<List<String>[]>() {}
);
```

测试代码中也验证了 `Class`、`Type` 和 `TypeReference` 三种入口在相同语义下会得到一致的 `TypeInfo`。([GitHub][5])

## TypeInfo

`TypeInfo` 是所有类型信息的公共接口。

```java
Class rawClass();

boolean isRaw();
```

`rawClass()` 返回当前类型对应的运行时原始 `Class`；`isRaw()` 表示当前类型是否不包含任何实际泛型绑定信息。`TypeInfo` 只允许三种实现：`ClassInfo`、`PrimitiveTypeInfo` 和 `ArrayTypeInfo`。([GitHub][2])

示例：

```java
TypeInfo type = ScxReflect.typeOf(String.class);

System.out.println(type.rawClass()); // class java.lang.String
System.out.println(type.isRaw());    // true
```

对于 `List<String>`：

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);

System.out.println(type.rawClass()); // interface java.util.List
System.out.println(type.isRaw());    // false
```

## ClassInfo

`ClassInfo` 表示普通 class、interface、enum、annotation 或 record。

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(User.class);
```

它提供：

```java
TypeBindings bindings();

ClassInfo declaringClass();

String name();

ClassKind classKind();

boolean isStatic();

boolean isFinal();

boolean isAbstract();

boolean isAnonymousClass();

boolean isMemberClass();

boolean isLocalClass();

ClassInfo superClass();

ClassInfo[] interfaces();

ConstructorInfo[] constructors();

FieldInfo[] fields();

MethodInfo[] methods();

RecordComponentInfo[] recordComponents();
```

`ClassInfo` 还提供一组继承层级辅助方法，例如 `allBindings()`、`allSuperClasses()`、`allInterfaces()`、`allFields()`、`allMethods()`、`defaultConstructor()`、`recordConstructor()`、`enumClass()` 和 `findSuperType(...)`。([GitHub][6])

### 判断类类型

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(User.class);

if (type.classKind() == ClassKind.RECORD) {
    System.out.println("record type");
}
```

`ClassKind` 包括：

```text
CLASS
INTERFACE
ENUM
ANNOTATION
RECORD
```

测试代码中验证了普通类、接口、注解、枚举和 record 都会被识别成对应的 `ClassKind`。([GitHub][7])

### 访问修饰符

```java
System.out.println(type.accessModifier());
```

`AccessModifier` 包括：

```text
PUBLIC
PRIVATE
PROTECTED
PACKAGE_PRIVATE
```

每个访问修饰符也有对应的文本值，例如 `PACKAGE_PRIVATE` 的文本是 `package-private`。([GitHub][8])

## 字段信息

### 获取当前类声明的字段

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(User.class);

for (FieldInfo field : type.fields()) {
    System.out.println(field.name());
    System.out.println(field.fieldType());
}
```

`fields()` 只返回当前类自己声明的字段，不包含父类字段。SCX Reflect 会过滤掉 synthetic 字段。([GitHub][9])

### 获取继承体系中的所有字段

```java
for (FieldInfo field : type.allFields()) {
    System.out.println(field.declaringClass() + "." + field.name());
}
```

`allFields()` 会包含当前类字段、父类字段和接口字段。测试代码中也验证了接口字段、父接口字段、父类字段和子类字段都会出现在 `allFields()` 中。([GitHub][9])

### 读取和设置字段值

```java
FieldInfo field = type.fields()[0];

field.setAccessible(true);

Object value = field.get(user);

field.set(user, "new value");
```

`FieldInfo` 提供 `get(...)`、`set(...)` 和 `setAccessible(...)` 这些便捷方法，底层仍然调用 Java 原生 `Field`。([GitHub][10])

### 字段属性

```java
System.out.println(field.name());
System.out.println(field.accessModifier());
System.out.println(field.isStatic());
System.out.println(field.isFinal());
System.out.println(field.fieldType());
```

`FieldInfoImpl` 会根据 `Field#getGenericType()` 和当前类的泛型绑定解析出最终字段类型。([GitHub][11])

例如：

```java
class Box<T> {
    T value;
}

ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<Box<String>>() {}
);

FieldInfo valueField = type.fields()[0];

System.out.println(valueField.fieldType()); // String
```

## 方法信息

### 获取当前类声明的方法

```java
for (MethodInfo method : type.methods()) {
    System.out.println(method.name() + " -> " + method.returnType());
}
```

`methods()` 只返回当前类声明的方法，不包含父类方法。SCX Reflect 会过滤 bridge method 和 synthetic method。([GitHub][9])

### 获取继承体系中的所有方法

```java
for (MethodInfo method : type.allMethods()) {
    System.out.println(method.signature());
}
```

`allMethods()` 会合并当前类、父类和接口中的方法，并移除被重写的方法；静态方法会完整保留。源码中对此有专门处理逻辑。([GitHub][9])

### 方法属性

```java
System.out.println(method.name());
System.out.println(method.accessModifier());
System.out.println(method.isStatic());
System.out.println(method.isFinal());
System.out.println(method.isAbstract());
System.out.println(method.isDefault());
System.out.println(method.isNative());
System.out.println(method.returnType());
System.out.println(method.signature());
```

`MethodInfoImpl` 会解析方法参数、返回类型、方法签名、访问修饰符以及 `static`、`final`、`abstract`、`default`、`native` 等标记。([GitHub][12])

### 调用方法

```java
method.setAccessible(true);

Object result = method.invoke(user, arg1, arg2);
```

`MethodInfo#invoke(...)` 底层调用的是 Java 原生 `Method#invoke(...)`。([GitHub][13])

## 方法签名和重写关系

`MethodSignature` 由方法名和参数类型组成。

```java
MethodSignature signature = method.signature();

System.out.println(signature.name());
System.out.println(Arrays.toString(signature.parameterTypes()));
```

`MethodSignature` 不包含返回值，因为 Java 方法重写判断主要依赖方法名和参数类型，返回值由编译器保证。源码中 `MethodSignature` 只保存 `name` 和 `parameterTypes`。([GitHub][14])

### 查找直接父方法

```java
MethodInfo[] superMethods = method.superMethods();
```

### 查找所有父方法

```java
MethodInfo[] allSuperMethods = method.allSuperMethods();
```

`superMethods()` 返回当前方法在继承关系中直接对应的父方法集合；`allSuperMethods()` 返回全部父方法，按广度遍历顺序展开。([GitHub][13])

测试代码中覆盖了普通类重写、接口 default method、private method、static method、package-private 跨包方法等情况。([GitHub][15])

SCX Reflect 的重写判断规则包括：

```text
static 方法不参与重写
final 方法不能被重写
private 方法不能被重写
package-private 方法只有同包才可重写
方法签名必须相同
```

这些规则由 `ReflectSupport#isOverride(...)` 实现。([GitHub][9])

## 构造函数信息

### 获取构造函数

```java
for (ConstructorInfo constructor : type.constructors()) {
    System.out.println(constructor);
}
```

`constructors()` 返回当前类声明的所有构造函数，包括非 public 构造函数。源码注释中明确说明：和字段、方法不同，构造函数会完整保留。([GitHub][9])

### 查找无参构造函数

```java
ConstructorInfo constructor = type.defaultConstructor();

if (constructor != null) {
    Object instance = constructor.newInstance();
}
```

`defaultConstructor()` 返回无参构造函数；如果不存在，返回 `null`。对于非静态成员类，当前定义下不会把带外部类隐式参数的构造函数当作无参构造函数。测试代码覆盖了这些情况。([GitHub][6])

### 创建实例

```java
constructor.setAccessible(true);

Object obj = constructor.newInstance(arg1, arg2);
```

`ConstructorInfo#newInstance(...)` 底层调用 Java 原生 `Constructor#newInstance(...)`。([GitHub][16])

## 参数信息

`ParameterInfo` 表示方法参数或构造函数参数。

```java
for (ParameterInfo parameter : method.parameters()) {
    System.out.println(parameter.name());
    System.out.println(parameter.parameterType());
}
```

`ParameterInfo` 提供：

```java
Parameter rawParameter();

ExecutableInfo declaringExecutable();

String name();

TypeInfo parameterType();
```

参数类型会根据当前声明类的泛型绑定进行解析。([GitHub][17])

## RecordComponentInfo

对于 record，可以读取 record component 信息。

```java
record UserDTO(String name, Integer age) {
}

ClassInfo type = (ClassInfo) ScxReflect.typeOf(UserDTO.class);

for (RecordComponentInfo component : type.recordComponents()) {
    System.out.println(component.name());
    System.out.println(component.recordComponentType());
}
```

`RecordComponentInfo` 提供 `name()`、`recordComponentType()`、`declaringClass()` 和 `get(obj)` 等方法。`get(obj)` 会调用 record component 的 accessor。([GitHub][18])

### 查找 record 规范构造函数

```java
ConstructorInfo constructor = type.recordConstructor();
```

`recordConstructor()` 会按 record component 类型查找规范构造函数；如果当前类型不是 record，返回 `null`。测试代码中验证了泛型 record 的规范构造函数也会正确解析参数类型。([GitHub][9])

## 泛型绑定 TypeBindings

`TypeBindings` 表示类型变量到实际类型的绑定关系。

```java
class Pair<A, B> {
}

ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<Pair<String, Integer>>() {}
);

TypeBindings bindings = type.bindings();

System.out.println(bindings.size());     // 2
System.out.println(bindings.get(0));     // String
System.out.println(bindings.get(1));     // Integer
System.out.println(bindings.get("A"));   // String
System.out.println(bindings.get("B"));   // Integer
```

`TypeBindings` 支持按 `TypeVariable`、变量名和索引获取绑定值，也可以遍历全部绑定。([GitHub][19])

### allBindings

`bindings()` 只表示当前类自己的类型参数绑定；`allBindings()` 会把非静态成员类的外部类泛型绑定也合并进来。

```java
class Outer<T> {
    class Inner<U> {
    }
}

ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<Outer<String>.Inner<Integer>>() {}
);

TypeBindings bindings = type.allBindings();

System.out.println(bindings.get(0));   // String
System.out.println(bindings.get(1));   // Integer
```

合并顺序是外部类在前，当前类在后；静态嵌套类不会继承外部类的泛型绑定。源码和测试代码都验证了这两个规则。([GitHub][9])

如果外部类和内部类有同名类型变量，按名称查找时当前作用域优先。`TypeBindingsImpl#get(String)` 使用倒序遍历实现这个规则。([GitHub][20])

## 数组类型 ArrayTypeInfo

数组类型会被表示为 `ArrayTypeInfo`。

```java
ArrayTypeInfo type = (ArrayTypeInfo) ScxReflect.typeOf(String[].class);

System.out.println(type.componentType()); // String
System.out.println(type.rawClass());      // class [Ljava.lang.String;
```

`ArrayTypeInfo` 提供：

```java
TypeInfo componentType();

Object newArray(int length);
```

`newArray(length)` 会基于 component type 创建新数组。测试代码中验证了普通引用数组、基本类型数组、多维数组和泛型数组。([GitHub][21])

示例：

```java
ArrayTypeInfo stringArray = (ArrayTypeInfo) ScxReflect.typeOf(String[].class);

Object array = stringArray.newArray(3);

System.out.println(array.getClass()); // class [Ljava.lang.String;
```

## 基本类型 PrimitiveTypeInfo

基本类型会被表示为 `PrimitiveTypeInfo`。

```java
PrimitiveTypeInfo intType = (PrimitiveTypeInfo) ScxReflect.typeOf(int.class);

System.out.println(intType.rawClass()); // int
System.out.println(intType.isRaw());    // true
```

`int` 和 `Integer` 是不同类型，测试代码中也验证了 primitive 和 wrapper 不相等。([GitHub][22])

```java
TypeInfo p = ScxReflect.typeOf(int.class);
TypeInfo w = ScxReflect.typeOf(Integer.class);

System.out.println(p.equals(w)); // false
```

## 注解读取

`ClassInfo`、`FieldInfo`、`MethodInfo`、`ConstructorInfo`、`ParameterInfo` 和 `RecordComponentInfo` 都可以读取注解。

```java
MyAnno anno = field.findAnnotation(MyAnno.class);

MyAnno[] annotations = field.findAnnotations(MyAnno.class);

Annotation[] all = field.annotations();
```

`AnnotatedElementInfo` 提供：

```java
Annotation[] annotations();

<T extends Annotation> T findAnnotation(Class<T> annotationClass);

<T extends Annotation> T[] findAnnotations(Class<T> annotationClass);
```

这些方法只读取当前元素上直接声明的注解，底层使用 Java 原生 `AnnotatedElement#getDeclaredAnnotation(...)` 和相关 API。([GitHub][23])

## 查找父类型

可以通过 `findSuperType(...)` 在当前类型继承体系中查找某个 raw type 对应的 `ClassInfo`。

```java
interface Base<T> {
}

class Impl implements Base<String> {
}

ClassInfo impl = (ClassInfo) ScxReflect.typeOf(Impl.class);

ClassInfo base = impl.findSuperType(Base.class);

System.out.println(base.bindings().get(0)); // String
```

`findSuperType(...)` 会根据目标类型是接口还是类，在 `allInterfaces()` 或 `allSuperClasses()` 中查找对应 raw class。测试代码中验证了参数化接口也能被找到。([GitHub][6])

## 类型缓存和实例一致性

SCX Reflect 的 `TypeFactory` 是线程安全的，并且语义上等价的类型会映射到同一个 `TypeInfo` 实例。源码注释中明确写了这两个设计目标。([GitHub][24])

```java
TypeInfo a = ScxReflect.typeOf(String.class);
TypeInfo b = ScxReflect.typeOf(new TypeReference<String>() {});

System.out.println(a == b); // true
```

测试代码中验证了多种入口、普通数组、泛型数组、成员内部类、递归泛型等情况下的 `same instance`、`equals` 和 `hashCode` 一致性。([GitHub][5])

## TypeVariable 和 WildcardType 的处理

SCX Reflect 不把 `TypeVariable` 和 `WildcardType` 作为最终模型保留下来。

处理规则是：

```text
TypeVariable:
优先从当前 TypeBindings 中查找实际绑定；
如果找不到，退化为第一个上界。

WildcardType:
忽略下界，退化为第一个上界。
```

源码中 `typeOfTypeVariable(...)` 和 `typeOfWildcardType(...)` 实现了这个规则。([GitHub][24])

示例：

```java
class Box<T extends Number> {
    T value;
}

ClassInfo rawBox = (ClassInfo) ScxReflect.typeOf(Box.class);

FieldInfo value = rawBox.fields()[0];

System.out.println(value.fieldType()); // Number
```

如果使用参数化类型：

```java
ClassInfo stringBox = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<Box<Integer>>() {}
);

FieldInfo value = stringBox.fields()[0];

System.out.println(value.fieldType()); // Integer
```

## 递归泛型处理

对于递归泛型引用，SCX Reflect 会退化为原始类型，避免 `toString()`、`hashCode()`、`equals()` 或类型解析时无限递归。

```java
class Node<T extends Node<T>> {
}
```

`TypeResolutionContext` 的源码注释说明：当前策略是在检测到递归泛型引用时返回对应 raw class 的 `TypeInfo`，从而彻底避免 `ClassInfo` 中形成递归引用结构。([GitHub][25])

测试代码中专门覆盖了递归成员泛型和更深层递归泛型，确保不会栈溢出。([GitHub][5])

## 完整示例

```java
import dev.scx.reflect.*;

import java.util.List;

public class ReflectExample {

    public static void main(String[] args) throws Exception {
        ClassInfo type = (ClassInfo) ScxReflect.typeOf(
            new TypeReference<UserService<String>>() {}
        );

        System.out.println("type = " + type);
        System.out.println("rawClass = " + type.rawClass());
        System.out.println("classKind = " + type.classKind());
        System.out.println("bindings = " + type.bindings());

        System.out.println("fields:");
        for (FieldInfo field : type.allFields()) {
            System.out.println("  " + field.name() + " : " + field.fieldType());
        }

        System.out.println("methods:");
        for (MethodInfo method : type.allMethods()) {
            System.out.println("  " + method.signature() + " -> " + method.returnType());
        }

        ConstructorInfo constructor = type.defaultConstructor();

        if (constructor != null) {
            constructor.setAccessible(true);

            Object service = constructor.newInstance();

            FieldInfo nameField = findField(type, "name");
            nameField.setAccessible(true);
            nameField.set(service, "demo");

            System.out.println(nameField.get(service));
        }
    }

    private static FieldInfo findField(ClassInfo type, String name) {
        for (var field : type.allFields()) {
            if (field.name().equals(name)) {
                return field;
            }
        }
        throw new IllegalArgumentException("No such field: " + name);
    }

    public static class Base<T> {

        public T value;

        public T getValue() {
            return value;
        }

    }

    public static class UserService<T> extends Base<T> {

        public String name;

        public List<T> list;

        public UserService() {
        }

        @Override
        public T getValue() {
            return value;
        }

    }

}
```

## 设计说明

### 1. TypeInfo 是更纯粹的类型模型

SCX Reflect 把 Java 原生反射中的 `Class`、`ParameterizedType`、`GenericArrayType`、`TypeVariable` 和 `WildcardType` 统一转换为 `TypeInfo`。最终模型只保留 `ClassInfo`、`PrimitiveTypeInfo` 和 `ArrayTypeInfo`。([GitHub][2])

### 2. 泛型会尽量被上下文解析

字段类型、方法返回类型、方法参数类型、构造函数参数类型、record component 类型都会结合当前声明类的 `allBindings()` 解析。例如 `Box<String>` 中字段 `T value` 会被解析为 `String`。([GitHub][11])

### 3. TypeInfo 会缓存并保持语义一致

`TypeFactory` 通过缓存保证语义等价类型对应同一个 `TypeInfo` 实例，并且对 `Class`、`ParameterizedType`、`GenericArrayType`、上下文泛型解析和数组类型都有专门处理。([GitHub][24])

### 4. fields() / methods() 是 declared 视图

`fields()` 和 `methods()` 只返回当前类型自己声明的字段或方法；`allFields()` 和 `allMethods()` 才会进入继承体系。字段会过滤 synthetic，方法会过滤 bridge 和 synthetic。([GitHub][9])

### 5. allMethods() 尝试提供“当前类型视图”

`allMethods()` 不是简单拼接父类和接口方法，而是会处理重写关系、接口 default method、private method、package-private 跨包可见性、静态方法等 Java 方法继承语义。([GitHub][9])

### 6. 注解读取只看直接声明

`findAnnotation(...)` 和 `findAnnotations(...)` 使用的是 `getDeclaredAnnotation(...)` 和 `getDeclaredAnnotationsByType(...)`，因此只读取当前元素直接声明的注解，不做继承搜索。([GitHub][23])

## 常见问题

### SCX Reflect 和 Java 原生反射是什么关系？

SCX Reflect 是 Java 原生反射之上的抽象层。它不会替代 `Class`、`Field`、`Method`、`Constructor`，而是把这些对象包装成 `ClassInfo`、`FieldInfo`、`MethodInfo`、`ConstructorInfo` 等更统一的模型，并额外处理泛型绑定和继承视图。

### 为什么 `TypeInfo` 没有 `TypeVariableInfo` 或 `WildcardTypeInfo`？

这是刻意设计。源码注释中说明，`TypeVariable` 和 `WildcardType` 更像 Java 泛型实现中的占位或约束结构，不应长期存在于最终类型模型中；SCX Reflect 会尝试通过上下文解析它们，无法解析时退化为上界。([GitHub][2])

### 如何表示 `List<String>`？

使用 `TypeReference`：

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);
```

然后通过 `type.bindings().get(0)` 读取 `String`。

### `fields()` 和 `allFields()` 有什么区别？

`fields()` 只返回当前类自己声明的字段；`allFields()` 返回当前类、父类和接口中的字段。([GitHub][26])

### `methods()` 和 `allMethods()` 有什么区别？

`methods()` 只返回当前类自己声明的方法；`allMethods()` 返回当前类型视图下的全部方法，并移除被重写的方法，同时保留静态方法。([GitHub][27])

### 为什么 bridge method 不见了？

SCX Reflect 会过滤 bridge method 和 synthetic method，让上层库看到更接近源码语义的方法列表。测试代码中也验证了泛型桥接方法会被过滤。([GitHub][9])

### record 的构造函数怎么找？

使用 `recordConstructor()`：

```java
ConstructorInfo constructor = type.recordConstructor();
```

它会根据 record component 类型查找规范构造函数；非 record 类型返回 `null`。([GitHub][9])

### `TypeInfo` 可以用 `==` 比较吗？

可以, 这是设计目的之一

[1]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeInfo.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ScxReflect.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeReference.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/test/java/dev/scx/reflect/test/TypeModelResolutionTest.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ClassInfo.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ClassKind.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/AccessModifier.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ReflectSupport.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/FieldInfo.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/FieldInfoImpl.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/MethodInfoImpl.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/MethodInfo.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/MethodSignature.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/test/java/dev/scx/reflect/test/MethodHierarchyTest.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ConstructorInfo.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ParameterInfo.java "raw.githubusercontent.com"
[18]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/RecordComponentInfo.java "raw.githubusercontent.com"
[19]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeBindings.java "raw.githubusercontent.com"
[20]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeBindingsImpl.java "raw.githubusercontent.com"
[21]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/ArrayTypeInfo.java "raw.githubusercontent.com"
[22]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/PrimitiveTypeInfo.java "raw.githubusercontent.com"
[23]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/AnnotatedElementInfo.java "raw.githubusercontent.com"
[24]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeFactory.java "raw.githubusercontent.com"
[25]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/main/java/dev/scx/reflect/TypeResolutionContext.java "raw.githubusercontent.com"
[26]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/test/java/dev/scx/reflect/test/DeclaredFieldsTest.java "raw.githubusercontent.com"
[27]: https://raw.githubusercontent.com/scx-projects/scx-reflect/master/src/test/java/dev/scx/reflect/test/DeclaredMethodsTest.java "raw.githubusercontent.com"
