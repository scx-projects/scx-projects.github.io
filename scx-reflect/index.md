# SCX Reflect

SCX Reflect 是一个轻量的 Java 反射信息抽象库。

它在 Java 原生反射 `Class`、`Type`、`Field`、`Method`、`Constructor`、`Parameter`、`RecordComponent` 之上，提供了一套更统一、更适合泛型解析和继承视图分析的模型。

SCX Reflect 的重点不是替代 Java 原生反射，而是把原生反射中比较分散、容易丢失上下文的信息整理成稳定的对象模型：

```text
类型解析
泛型绑定
泛型上下界区间
字段 / 方法 / 构造函数 / 参数信息读取
record component 信息读取
继承层级分析
接口层级分析
方法重写关系分析
注解读取
TypeReference 泛型捕获
```

`scx-di`、`scx-sql`、`scx-serialize` 等库可以基于这套反射模型继续实现依赖注入、Bean 映射、序列化、类型转换、SQL 映射等能力。

[GitHub](https://github.com/scx-projects/scx-reflect)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-reflect</artifactId>
    <version>0.10.0</version>
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
TypeBindings            类型变量绑定集合
TypeRange               类型上下界区间
FieldInfo               字段信息
MethodInfo              方法信息
MethodSignature         方法签名
ConstructorInfo         构造函数信息
MemberInfo              字段、方法、构造函数等成员信息的公共父接口
AccessModifierOwner     可读取访问修饰符的类型
ParameterInfo           参数信息
RecordComponentInfo     record component 信息
AccessModifier          访问修饰符
ClassKind               类类型
```

`TypeInfo` 是核心类型模型。它只保留三类最终类型：`ClassInfo`、`PrimitiveTypeInfo` 和 `ArrayTypeInfo`。Java 反射里的 `TypeVariable` 和 `WildcardType` 不会长期作为最终类型存在；它们会结合上下文被解析成具体类型，或者退化为上界。

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

System.out.println(type.rawClass());        // interface java.util.List
System.out.println(type.bindings().get(0)); // String
```

`get(...)` 是 `TypeBindings` 上的快捷方法。它返回绑定区间的投影类型，默认等价于 `range(...).type()`。如果你需要区分 `? extends Number` 和 `? super Integer` 这类上下界信息，请使用 `range(...)`。

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
TypeInfo typeOf(Class clazz)

TypeInfo typeOf(Type type)

TypeInfo typeOf(TypeReference typeReference)
```

内部会把 Java 反射中的 `Class`、`ParameterizedType`、`GenericArrayType`、`TypeVariable` 和 `WildcardType` 转换成 SCX Reflect 自己的 `TypeInfo` 模型。

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

`TypeReference` 必须带实际类型参数。如果没有实际类型参数，构造函数会抛出 `IllegalArgumentException`。

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

## TypeInfo

`TypeInfo` 是所有类型信息的公共接口。

```java
Class rawClass();

boolean isRaw();

boolean isAssignableFrom(TypeInfo typeInfo);
```

`rawClass()` 返回当前类型对应的运行时原始 `Class`。

`isRaw()` 表示当前类型是否不包含任何实际泛型绑定信息。

`isAssignableFrom(...)` 用于判断指定类型的值是否可以安全赋值给当前类型，判断时会考虑泛型约束。

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

`TypeInfo` 只允许三种实现：

```text
ClassInfo
PrimitiveTypeInfo
ArrayTypeInfo
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

`ClassInfo` 还提供一组继承层级辅助方法：

```java
TypeBindings allBindings();

ClassInfo[] allSuperClasses();

ClassInfo[] allInterfaces();

FieldInfo[] allFields();

MethodInfo[] allMethods();

ConstructorInfo defaultConstructor();

ConstructorInfo recordConstructor();

ClassInfo enumClass();

ClassInfo findSuperType(Class rawTarget);
```

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

### 访问修饰符

```java
System.out.println(type.accessModifier());
System.out.println(type.accessModifier().text());
```

`AccessModifier` 包括：

```text
PUBLIC
PRIVATE
PROTECTED
PACKAGE_PRIVATE
```

每个访问修饰符也有对应文本，例如 `PACKAGE_PRIVATE.text()` 返回 `package-private`。

## 字段信息

### 获取当前类声明的字段

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(User.class);

for (FieldInfo field : type.fields()) {
    System.out.println(field.name());
    System.out.println(field.fieldType());
}
```

`fields()` 只返回当前类自己声明的字段，不包含父类字段。SCX Reflect 会过滤 synthetic 字段。

### 获取继承体系中的所有字段

```java
for (FieldInfo field : type.allFields()) {
    System.out.println(field.declaringClass() + "." + field.name());
}
```

`allFields()` 会包含当前类字段、父类字段和接口字段。字段不会像方法一样发生“重写”，因此同名字段也会按声明来源保留。

### 读取和设置字段值

```java
FieldInfo field = type.fields()[0];

field.setAccessible(true);

Object value = field.get(user);

field.set(user, "new value");
```

`FieldInfo` 提供 `get(...)`、`set(...)` 和 `setAccessible(...)` 这些便捷方法，底层仍然调用 Java 原生 `Field`。

### 字段属性

```java
System.out.println(field.name());
System.out.println(field.accessModifier());
System.out.println(field.isStatic());
System.out.println(field.isFinal());
System.out.println(field.fieldType());
```

字段类型会根据当前类的泛型绑定解析。

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

`methods()` 只返回当前类声明的方法，不包含父类方法。SCX Reflect 会过滤 bridge method 和 synthetic method，让上层库看到更接近 Java 声明语义的方法列表。

### 获取继承体系中的所有方法

```java
for (MethodInfo method : type.allMethods()) {
    System.out.println(method.signature());
}
```

`allMethods()` 会合并当前类、父类和接口中的方法，并尽量呈现“当前类型视图”：

```text
静态方法完整保留
实例方法按重写关系去除已被覆盖的方法
private 方法不参与重写
final 父方法不能被重写
package-private 方法只有同包才可重写 / 继承
接口方法和类方法按 Java 继承规则处理
```

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

`MethodInfo` 会解析方法参数、返回类型、方法签名、访问修饰符以及 `static`、`final`、`abstract`、`default`、`native` 等标记。

### 调用方法

```java
method.setAccessible(true);

Object result = method.invoke(user, arg1, arg2);
```

`MethodInfo#invoke(...)` 底层调用的是 Java 原生 `Method#invoke(...)`。

## 方法签名和重写关系

`MethodSignature` 由方法名和参数原始类型组成。

```java
MethodSignature signature = method.signature();

System.out.println(signature.name());
System.out.println(Arrays.toString(signature.parameterTypes()));
```

`MethodSignature` 不包含返回值，因为 Java 方法重写判断主要依赖方法名和参数类型，返回值由编译器保证。

### 查找直接父方法

```java
MethodInfo[] superMethods = method.superMethods();
```

`superMethods()` 返回当前方法在继承关系中直接对应的父方法集合。

### 查找所有父方法

```java
MethodInfo[] allSuperMethods = method.allSuperMethods();
```

`allSuperMethods()` 返回全部父方法，按广度遍历顺序展开。

SCX Reflect 的重写判断规则包括：

```text
static 方法不参与重写
final 父方法不能被重写
private 方法不能被重写
package-private 方法只有同包才可重写
方法签名必须相同
返回值不参与判断
```

## 构造函数信息

### 获取构造函数

```java
for (ConstructorInfo constructor : type.constructors()) {
    System.out.println(constructor);
}
```

`constructors()` 返回当前类声明的所有构造函数，包括非 public 构造函数。和字段、方法不同，构造函数会完整保留。

### 查找无参构造函数

```java
ConstructorInfo constructor = type.defaultConstructor();

if (constructor != null) {
    Object instance = constructor.newInstance();
}
```

`defaultConstructor()` 返回无参构造函数；如果不存在，返回 `null`。对于非静态成员类，当前定义下不会把带外部类隐式参数的构造函数当作无参构造函数。

### 创建实例

```java
constructor.setAccessible(true);

Object obj = constructor.newInstance(arg1, arg2);
```

`ConstructorInfo#newInstance(...)` 底层调用 Java 原生 `Constructor#newInstance(...)`。

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

参数类型会根据当前声明类的泛型绑定进行解析。

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

`RecordComponentInfo` 提供 `name()`、`recordComponentType()`、`declaringClass()` 和 `get(obj)` 等方法。`get(obj)` 会调用 record component 的 accessor。

### 查找 record 规范构造函数

```java
ConstructorInfo constructor = type.recordConstructor();
```

`recordConstructor()` 会按 record component 类型查找规范构造函数；如果当前类型不是 record，返回 `null`。

## 泛型绑定 TypeBindings

`TypeBindings` 表示类型变量到实际类型区间的绑定关系。

在 `0.3.0` 中，`TypeBindings` 的核心返回值不只是 `TypeInfo`，而是 `TypeRange`：

```java
TypeRange range(TypeVariable<?> typeVariable);

TypeRange range(String name);

TypeRange range(int index);

TypeVariable<?>[] typeVariables();

TypeRange[] typeRanges();

int size();
```

同时，为了常见场景方便，它也保留了 `get(...)` 快捷方法：

```java
TypeInfo get(TypeVariable<?> typeVariable);

TypeInfo get(String name);

TypeInfo get(int index);
```

`get(...)` 会先取得 `TypeRange`，再返回 `range.type()`。`TypeRange#type()` 默认使用上界作为投影类型。

### 读取普通泛型绑定

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

### 读取上下界区间

当泛型参数中存在通配符时，使用 `range(...)` 可以保留下界和上界信息。

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<? extends Number>>() {}
);

TypeRange range = type.bindings().range(0);

System.out.println(range.lowerBound()); // null
System.out.println(range.upperBound()); // Number
System.out.println(range.type());       // Number
```

对于 `? super Integer`：

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<? super Integer>>() {}
);

TypeRange range = type.bindings().range(0);

System.out.println(range.lowerBound()); // Integer
System.out.println(range.upperBound()); // Object
System.out.println(range.type());       // Object
```

如果你只关心“最终可作为类型处理的投影结果”，使用 `get(...)` 更简单；如果你要做严格的泛型可赋值判断，应优先使用 `range(...)`。

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

合并顺序是外部类在前，当前类在后；静态嵌套类不会继承外部类的泛型绑定。

如果外部类和内部类有同名类型变量，按名称查找时当前作用域优先。`TypeBindingsImpl` 在按名称或 `TypeVariable` 查找时会倒序遍历，使内部作用域优先匹配，外部作用域兜底。

## TypeRange

`TypeRange` 表示一个简单的类型上下界区间：

```java
public record TypeRange(TypeInfo lowerBound, TypeInfo upperBound) {

    public TypeInfo type();

    public boolean contains(TypeRange typeRange);

    public boolean lowerBoundContains(TypeRange typeRange);

    public boolean upperBoundContains(TypeRange typeRange);

}
```

它的语义可以理解为：

```text
lowerBound <= actual type <= upperBound
```

其中：

```text
lowerBound 可以为 null
upperBound 永远不为 null
type() 默认返回 upperBound
```

常见映射关系：

```text
String                  -> lowerBound = String,  upperBound = String
? extends Number        -> lowerBound = null,    upperBound = Number
? super Integer         -> lowerBound = Integer, upperBound = Object
```

`TypeRange` 只建模简单的上下界关系，不引入完整的类型约束方程求解。因此它不会精确求解复杂递归泛型边界，例如 `T extends Comparable<T>` 或 `E extends Enum<E>` 这类 F-bounded polymorphism。

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

`newArray(length)` 会基于 component type 创建新数组。

示例：

```java
ArrayTypeInfo stringArray = (ArrayTypeInfo) ScxReflect.typeOf(String[].class);

Object array = stringArray.newArray(3);

System.out.println(array.getClass()); // class [Ljava.lang.String;
```

普通引用数组、基本类型数组、多维数组和泛型数组都会进入 `ArrayTypeInfo` 模型。

## 基本类型 PrimitiveTypeInfo

基本类型会被表示为 `PrimitiveTypeInfo`。

```java
PrimitiveTypeInfo intType = (PrimitiveTypeInfo) ScxReflect.typeOf(int.class);

System.out.println(intType.rawClass()); // int
System.out.println(intType.isRaw());    // true
```

`int` 和 `Integer` 是不同类型。

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

这些方法只读取当前元素上直接声明的注解，底层使用 Java 原生 `AnnotatedElement#getDeclaredAnnotations()`、`getDeclaredAnnotation(...)` 和 `getDeclaredAnnotationsByType(...)`。

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

`findSuperType(...)` 会根据目标类型是接口还是类，在 `allInterfaces()` 或 `allSuperClasses()` 中查找对应 raw class。

## 类型缓存和实例一致性

`TypeFactory` 是线程安全的，并且语义上等价的类型会映射到同一个 `TypeInfo` 实例。

```java
TypeInfo a = ScxReflect.typeOf(String.class);
TypeInfo b = ScxReflect.typeOf(new TypeReference<String>() {});

System.out.println(a == b); // true
```

实现中会针对不同类型采用不同缓存策略：

```text
Class                     直接以 Class 作为缓存 key
ParameterizedType          无上下文时可直接缓存；有上下文时使用构造出的 ClassInfo 作为语义 key
GenericArrayType           会尝试与 raw array Class 复用
TypeVariable               根据上下文绑定解析；无法解析则退化为上界
WildcardType               转换成 TypeRange 后再投影为上界类型
递归泛型引用               退化为 raw class，避免递归结构进入 ClassInfo
```

这意味着相同语义的类型，即使来自不同入口，也尽量复用同一个 `TypeInfo` 实例。

## TypeVariable 和 WildcardType 的处理

SCX Reflect 不把 `TypeVariable` 和 `WildcardType` 作为最终模型保留下来。

作为 `TypeInfo` 使用时，它们会被投影为具体类型：

```text
TypeVariable:
优先从当前 TypeBindings 中查找实际绑定；
如果找不到，退化为第一个上界。

WildcardType:
保留下界 / 上界形成 TypeRange；
作为 TypeInfo 使用时，投影为上界。
```

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
ClassInfo intBox = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<Box<Integer>>() {}
);

FieldInfo value = intBox.fields()[0];

System.out.println(value.fieldType()); // Integer
```

如果你需要知道通配符的下界，例如 `List<? super Integer>` 中的 `Integer`，不要只看 `get(0)`，要看 `range(0).lowerBound()`。

## 递归泛型处理

对于递归泛型引用，SCX Reflect 会退化为原始类型，避免 `toString()`、`hashCode()`、`equals()` 或类型解析时无限递归。

```java
class Node<T extends Node<T>> {
}
```

处理策略是：构建复杂泛型结构时，`TypeResolutionContext` 会记录正在解析的半成品 `ClassInfo`。如果后续解析中再次遇到同一个递归类型，则不直接返回半成品对象，而是返回对应 `rawClass` 的无泛型 `TypeInfo`，从而彻底打断递归引用。

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

SCX Reflect 把 Java 原生反射中的 `Class`、`ParameterizedType`、`GenericArrayType`、`TypeVariable` 和 `WildcardType` 统一转换为 `TypeInfo`。最终模型只保留 `ClassInfo`、`PrimitiveTypeInfo` 和 `ArrayTypeInfo`。

### 2. 泛型会尽量被上下文解析

字段类型、方法返回类型、方法参数类型、构造函数参数类型、record component 类型都会结合当前声明类的 `allBindings()` 解析。例如 `Box<String>` 中字段 `T value` 会被解析为 `String`。

### 3. TypeBindings 现在同时支持 TypeRange 和 TypeInfo 投影

`0.3.0` 中，`TypeBindings` 的底层绑定值是 `TypeRange`。这是为了保留 `? extends` 和 `? super` 这样的上下界信息。

`get(...)` 仍然存在，但它只是 `range(...).type()` 的快捷投影。

### 4. TypeInfo 会缓存并保持语义一致

`TypeFactory` 通过缓存保证语义等价类型对应同一个 `TypeInfo` 实例，并且对 `Class`、`ParameterizedType`、`GenericArrayType`、上下文泛型解析和数组类型都有专门处理。

### 5. fields() / methods() 是 declared 视图

`fields()` 和 `methods()` 只返回当前类型自己声明的字段或方法；`allFields()` 和 `allMethods()` 才会进入继承体系。字段会过滤 synthetic，方法会过滤 bridge 和 synthetic。

### 6. allMethods() 尝试提供“当前类型视图”

`allMethods()` 不是简单拼接父类和接口方法，而是会处理重写关系、接口 default method、private method、package-private 跨包可见性、静态方法等 Java 方法继承语义。

### 7. 注解读取只看直接声明

`findAnnotation(...)` 和 `findAnnotations(...)` 使用的是 `getDeclaredAnnotation(...)` 和 `getDeclaredAnnotationsByType(...)`，因此只读取当前元素直接声明的注解，不做继承搜索。

## 常见问题

### SCX Reflect 和 Java 原生反射是什么关系？

SCX Reflect 是 Java 原生反射之上的抽象层。它不会替代 `Class`、`Field`、`Method`、`Constructor`，而是把这些对象包装成 `ClassInfo`、`FieldInfo`、`MethodInfo`、`ConstructorInfo` 等更统一的模型，并额外处理泛型绑定和继承视图。

### 为什么 TypeInfo 没有 TypeVariableInfo 或 WildcardTypeInfo？

这是刻意设计。`TypeVariable` 和 `WildcardType` 更像 Java 泛型实现中的占位或约束结构，不应长期存在于最终类型模型中；SCX Reflect 会尝试通过上下文解析它们，无法解析时退化为上界。

### 如何表示 List<String>？

使用 `TypeReference`：

```java
ClassInfo type = (ClassInfo) ScxReflect.typeOf(
    new TypeReference<List<String>>() {}
);
```

然后通过 `type.bindings().get(0)` 读取 `String`。

### get(...) 和 range(...) 有什么区别？

`range(...)` 返回完整的 `TypeRange`，包含下界和上界。

`get(...)` 返回 `range(...).type()`，也就是投影类型；当前默认使用上界。

```java
TypeRange range = type.bindings().range(0);
TypeInfo projected = type.bindings().get(0);
```

对于 `List<String>`，二者差异不大；对于 `List<? super Integer>`，`range(0).lowerBound()` 是 `Integer`，但 `get(0)` 返回的是上界投影 `Object`。

### fields() 和 allFields() 有什么区别？

`fields()` 只返回当前类自己声明的字段；`allFields()` 返回当前类、父类和接口中的字段。

### methods() 和 allMethods() 有什么区别？

`methods()` 只返回当前类自己声明的方法；`allMethods()` 返回当前类型视图下的全部方法，并移除被重写的方法，同时保留静态方法。

### 为什么 bridge method 不见了？

SCX Reflect 会过滤 bridge method 和 synthetic method，让上层库看到更接近 Java 声明语义的方法列表。

### record 的构造函数怎么找？

使用 `recordConstructor()`：

```java
ConstructorInfo constructor = type.recordConstructor();
```

它会根据 record component 类型查找规范构造函数；非 record 类型返回 `null`。

### TypeInfo 可以用 == 比较吗？

语义等价类型会尽量复用同一个 `TypeInfo` 实例，因此常见情况下可以用 `==` 做快速比较。不过如果你在写公共逻辑，仍建议使用 `equals(...)` 表达语义相等，除非你明确依赖 SCX Reflect 的缓存一致性设计。
