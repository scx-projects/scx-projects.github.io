# SCX Collection

SCX Collection 是一个轻量集合工具库。

它提供了两个常用集合结构：

```text
MultiMap    一个 key 对应多个 value 的映射
CountMap    一个 key 对应一个计数值的映射
```

同时提供 `ScxCollection` 工具类，用于快速把普通数组或 `Iterable` 转换成 `MultiMap` 或 `CountMap`。

SCX Collection 本身不是 JDK Collection Framework 的替代品，也不是 Stream API 的替代品。它更像是对 Java 集合的一组补充，用来处理“按 key 分组”和“按 key 计数”这两类常见场景。

[GitHub](https://github.com/scx-projects/scx-collection)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-collection</artifactId>
    <version>0.1.0</version>
</dependency>
```

## 基本概念

SCX Collection 中最核心的概念包括：

```text
ScxCollection      集合工具类
MultiMap           多值映射接口
DefaultMultiMap    MultiMap 的默认实现
MultiMapEntry      MultiMap 的遍历条目

CountMap           计数映射接口
DefaultCountMap    CountMap 的默认实现
CountMapEntry      CountMap 的遍历条目
```

它们之间的关系可以简单理解为：

```text
ScxCollection.groupingBy(...)  -> DefaultMultiMap
ScxCollection.countingBy(...)  -> DefaultCountMap

DefaultMultiMap                -> 内部使用 Map<K, List<V>>
DefaultCountMap                -> 内部使用 Map<K, Long>
```

也就是说：

```text
MultiMap 解决一个 key 对应多个 value 的问题
CountMap 解决一个 key 对应一个累计数量的问题
```

## 快速开始

### 分组

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var users = List.of(
    new User("Tom", "dev"),
    new User("Jerry", "dev"),
    new User("Alice", "ops")
);

var map = ScxCollection.groupingBy(users, User::department);

System.out.println(map.getAll("dev"));
System.out.println(map.getAll("ops"));
```

结果类似：

```text
[User[name=Tom, department=dev], User[name=Jerry, department=dev]]
[User[name=Alice, department=ops]]
```

### 计数

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var words = List.of("a", "b", "a", "c", "b", "a");

var countMap = ScxCollection.countingBy(words);

System.out.println(countMap.get("a"));
System.out.println(countMap.get("b"));
System.out.println(countMap.get("c"));
```

结果：

```text
3
2
1
```

### 手动使用 MultiMap

```java
import dev.scx.collection.multi_map.DefaultMultiMap;

var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");
map.add("ops", "Alice");

System.out.println(map.get("dev"));
System.out.println(map.getAll("dev"));
System.out.println(map.size());
```

结果：

```text
Tom
[Tom, Jerry]
3
```

需要注意，`MultiMap#size()` 返回的是所有 value 的总数，不是 key 的数量。

### 手动使用 CountMap

```java
import dev.scx.collection.count_map.DefaultCountMap;

var countMap = new DefaultCountMap<String>();

countMap.add("apple", 1);
countMap.add("apple", 2);
countMap.add("orange", 1);

System.out.println(countMap.get("apple"));
System.out.println(countMap.get("orange"));
System.out.println(countMap.size());
```

结果：

```text
3
1
2
```

这里 `CountMap#size()` 返回的是 key 的数量。

## ScxCollection

`ScxCollection` 是工具类。

它目前提供两类方法：

```text
groupingBy    按 key 分组，返回 MultiMap
countingBy    按 key 计数，返回 CountMap
```

支持的输入包括：

```text
Iterable<T>
T[]
```

也就是说，可以处理：

```java
List<T>
Set<T>
Collection<T>
T[]
```

## groupingBy

`groupingBy(...)` 用于把一组数据按 key 分组。

最简单的形式是：

```java
var multiMap = ScxCollection.groupingBy(list, keyFn);
```

其中：

```text
list     原始数据
keyFn    从每个元素中提取 key 的函数
```

示例：

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var users = List.of(
    new User("Tom", "dev"),
    new User("Jerry", "dev"),
    new User("Alice", "ops")
);

var map = ScxCollection.groupingBy(users, User::department);
```

等价逻辑大致是：

```java
var map = new DefaultMultiMap<String, User>();

for (var user : users) {
    map.add(user.department(), user);
}
```

### 指定 valueFn

默认情况下，`groupingBy(...)` 会把原始元素本身作为 value。

如果只想保存元素中的某个字段，可以传入 `valueFn`。

```java
var map = ScxCollection.groupingBy(
    users,
    User::department,
    User::name
);
```

结果类似：

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

也就是说：

```text
keyFn      决定分组 key
valueFn    决定放入 MultiMap 的 value
```

完整示例：

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

public class GroupingDemo {

    public static void main(String[] args) {
        var users = List.of(
            new User("Tom", "dev"),
            new User("Jerry", "dev"),
            new User("Alice", "ops")
        );

        var map = ScxCollection.groupingBy(
            users,
            User::department,
            User::name
        );

        System.out.println(map.getAll("dev"));
        System.out.println(map.getAll("ops"));
    }

    public record User(String name, String department) {

    }

}
```

输出：

```text
[Tom, Jerry]
[Alice]
```

### 数组分组

`groupingBy(...)` 也支持数组。

```java
var users = new User[]{
    new User("Tom", "dev"),
    new User("Jerry", "dev"),
    new User("Alice", "ops")
};

var map = ScxCollection.groupingBy(users, User::department);
```

指定 valueFn：

```java
var map = ScxCollection.groupingBy(
    users,
    User::department,
    User::name
);
```

### groupingBy 返回值

`groupingBy(...)` 返回的是：

```java
MultiMap<K, V>
```

默认实现是：

```java
DefaultMultiMap<K, V>
```

内部使用：

```text
HashMap<K, List<V>>
ArrayList<V>
```

## countingBy

`countingBy(...)` 用于把一组数据按 key 计数。

最简单的形式是：

```java
var countMap = ScxCollection.countingBy(list);
```

示例：

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var words = List.of("a", "b", "a", "c", "b", "a");

var countMap = ScxCollection.countingBy(words);

System.out.println(countMap);
```

结果类似：

```text
{a=3, b=2, c=1}
```

默认情况下：

```text
keyFn      使用元素本身作为 key
countFn    每个元素贡献 1
```

等价逻辑大致是：

```java
var countMap = new DefaultCountMap<String>();

for (var word : words) {
    countMap.add(word, 1L);
}
```

### 指定 keyFn

如果希望按元素中的某个字段计数，可以传入 `keyFn`。

```java
var countMap = ScxCollection.countingBy(users, User::department);
```

示例：

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var users = List.of(
    new User("Tom", "dev"),
    new User("Jerry", "dev"),
    new User("Alice", "ops")
);

var countMap = ScxCollection.countingBy(users, User::department);

System.out.println(countMap.get("dev"));
System.out.println(countMap.get("ops"));
```

结果：

```text
2
1
```

### 指定 countFn

如果每个元素贡献的数量不是固定的 `1`，可以传入 `countFn`。

```java
var countMap = ScxCollection.countingBy(
    orders,
    Order::productId,
    Order::quantity
);
```

示例：

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

var orders = List.of(
    new Order("apple", 2L),
    new Order("orange", 3L),
    new Order("apple", 5L)
);

var countMap = ScxCollection.countingBy(
    orders,
    Order::productId,
    Order::quantity
);

System.out.println(countMap.get("apple"));
System.out.println(countMap.get("orange"));
```

结果：

```text
7
3
```

### countFn 返回 null

如果 `countFn` 返回 `null`，该元素会被跳过。

```java
var countMap = ScxCollection.countingBy(
    orders,
    Order::productId,
    order -> order.enabled() ? order.quantity() : null
);
```

等价逻辑是：

```java
for (var order : orders) {
    var key = keyFn.apply(order);
    var count = countFn.apply(order);

    if (count != null) {
        countMap.add(key, count);
    }
}
```

这适合把过滤和计数合在一起的简单场景。

### 数组计数

`countingBy(...)` 也支持数组。

```java
var words = new String[]{"a", "b", "a", "c"};

var countMap = ScxCollection.countingBy(words);
```

指定 keyFn：

```java
var countMap = ScxCollection.countingBy(users, User::department);
```

指定 keyFn 和 countFn：

```java
var countMap = ScxCollection.countingBy(
    orders,
    Order::productId,
    Order::quantity
);
```

## MultiMap

`MultiMap<K, V>` 表示一个 key 可以对应多个 value 的映射。

它和普通 `Map<K, V>` 的区别是：

```text
Map<K, V>          一个 key 对应一个 value
MultiMap<K, V>     一个 key 对应多个 value
```

例如：

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

这种结构适合下面这些场景：

1. 按部门分组用户。
2. 按类型分组配置项。
3. 按标签分组文章。
4. 按请求头名称保存多个请求头值。
5. 按字段名保存多个错误信息。
6. 按 key 聚合多个匹配结果。

## 创建 MultiMap

默认实现是 `DefaultMultiMap`。

```java
import dev.scx.collection.multi_map.DefaultMultiMap;
import dev.scx.collection.multi_map.MultiMap;

MultiMap<String, String> map = new DefaultMultiMap<>();
```

默认情况下，内部使用：

```text
HashMap
ArrayList
```

也可以指定内部 `Map` 和 `List` 的实现。

```java
import dev.scx.collection.multi_map.DefaultMultiMap;

import java.util.LinkedHashMap;
import java.util.LinkedList;

var map = new DefaultMultiMap<String, String>(
    LinkedHashMap::new,
    LinkedList::new
);
```

这适合下面这些场景：

1. 希望 key 按插入顺序保存。
2. 希望 value 使用指定 List 实现。
3. 希望控制底层集合类型。
4. 希望在测试中得到更稳定的遍历顺序。

## add

`add(...)` 用于给某个 key 追加 value。

### 添加单个 value

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");
```

结果：

```text
dev -> [Tom, Jerry]
```

### 添加多个 value

```java
map.add("dev", "Tom", "Jerry", "Alice");
```

结果：

```text
dev -> [Tom, Jerry, Alice]
```

### 添加 Collection

```java
map.add("dev", List.of("Tom", "Jerry"));
```

结果：

```text
dev -> [Tom, Jerry]
```

### 添加 Map

如果有普通 `Map<K, V>`，可以把每个 entry 添加到 `MultiMap`。

```java
var source = Map.of(
    "dev", "Tom",
    "ops", "Alice"
);

map.add(source);
```

等价于：

```java
source.forEach(map::add);
```

### 添加另一个 MultiMap

```java
var a = new DefaultMultiMap<String, String>();
a.add("dev", "Tom");

var b = new DefaultMultiMap<String, String>();
b.add("dev", "Jerry");
b.add("ops", "Alice");

a.add(b);
```

结果：

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

`add(MultiMap)` 会把另一个 `MultiMap` 中的每个 key 对应的所有 values 追加到当前对象中。

## set

`set(...)` 用于覆盖某个 key 对应的 values。

### 覆盖为单个 value

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");

var oldValues = map.set("dev", "Alice");
```

执行后：

```text
dev -> [Alice]
```

`oldValues` 是覆盖前的值：

```text
[Tom, Jerry]
```

如果 key 原来不存在，返回 `null`。

### 覆盖为多个 value

```java
map.set("dev", "Tom", "Jerry");
```

结果：

```text
dev -> [Tom, Jerry]
```

### 覆盖为 Collection

```java
map.set("dev", List.of("Tom", "Jerry"));
```

结果：

```text
dev -> [Tom, Jerry]
```

### 使用 Map 覆盖

```java
var source = Map.of(
    "dev", "Tom",
    "ops", "Alice"
);

map.set(source);
```

等价于：

```java
source.forEach(map::set);
```

### 使用 MultiMap 覆盖

```java
map.set(otherMultiMap);
```

这会遍历另一个 `MultiMap` 的每个 entry，然后用对应 values 覆盖当前 key。

## get

`get(key)` 用于获取某个 key 对应的第一个 value。

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");

var value = map.get("dev");
```

结果：

```text
Tom
```

如果 key 不存在，或者 key 对应的 values 为空，返回：

```text
null
```

需要注意，`get(key)` 只返回第一个 value。

如果需要所有 value，应该使用：

```java
getAll(key)
```

## getAll

`getAll(key)` 用于获取某个 key 对应的所有 values。

```java
var values = map.getAll("dev");
```

结果：

```text
[Tom, Jerry]
```

如果 key 不存在，返回一个新的空 List。

```java
var values = map.getAll("unknown");
```

结果：

```text
[]
```

需要注意：

1. 如果 key 存在，返回的是内部 List。
2. 如果 key 不存在，返回的是一个新的空 List，但不会自动放入 map。
3. 修改已存在 key 的返回 List，会影响 MultiMap 内部数据。
4. 修改不存在 key 的返回空 List，不会影响 MultiMap。

示例：

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");

var values = map.getAll("dev");

values.add("Jerry");

System.out.println(map.getAll("dev"));
```

结果：

```text
[Tom, Jerry]
```

因为 `values` 是内部 List。

## containsKey

`containsKey(key)` 用于判断是否存在某个 key。

```java
map.add("dev", "Tom");

boolean b1 = map.containsKey("dev");
boolean b2 = map.containsKey("ops");
```

结果：

```text
true
false
```

## containsValue

`containsValue(value)` 用于判断所有 values 中是否存在某个 value。

```java
map.add("dev", "Tom");
map.add("ops", "Alice");

boolean b1 = map.containsValue("Tom");
boolean b2 = map.containsValue("Jerry");
```

结果：

```text
true
false
```

它会遍历所有 key 对应的所有 values。

因此它不是只检查某个 key，而是检查整个 `MultiMap`。

## remove

`remove(...)` 用于删除某个 key 下的指定 value。

### 删除单个 value

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");

boolean removed = map.remove("dev", "Tom");
```

执行后：

```text
dev -> [Jerry]
```

返回值：

```text
true
```

如果 value 不存在，返回：

```text
false
```

### 删除多个 value

```java
map.remove("dev", "Tom", "Jerry");
```

或者：

```java
map.remove("dev", List.of("Tom", "Jerry"));
```

这些方法会从指定 key 对应的 List 中删除对应 values。

### 删除后自动移除空 key

如果删除后某个 key 对应的 List 为空，`DefaultMultiMap` 会把这个 key 从内部 map 中移除。

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");

map.remove("dev", "Tom");

System.out.println(map.containsKey("dev"));
```

结果：

```text
false
```

## removeAll

`removeAll(key)` 用于删除某个 key 对应的所有 values。

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");

var oldValues = map.removeAll("dev");
```

执行后：

```text
dev 不再存在
```

`oldValues` 是删除前的 values：

```text
[Tom, Jerry]
```

如果 key 不存在，返回：

```text
null
```

## keys

`keys()` 返回所有 key。

```java
var keys = map.keys();
```

结果类型是：

```java
Set<K>
```

需要注意，`keys()` 返回的是内部 map 的 `keySet()` 视图。

这意味着：

1. 它不是副本。
2. 当前 MultiMap 改变后，keys 视图也会变化。
3. 修改 keys 视图可能影响 MultiMap 内部数据。

如果需要安全副本，可以自己复制：

```java
var keysCopy = new HashSet<>(map.keys());
```

## values

`values()` 返回所有 value 的扁平列表。

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");
map.add("ops", "Alice");

var values = map.values();
```

结果：

```text
[Tom, Jerry, Alice]
```

`values()` 会创建一个新的 List，然后把所有 key 对应的 values 合并进去。

因此修改返回的 List，不会直接影响 MultiMap 内部结构。

```java
var values = map.values();

values.clear();

System.out.println(map.size());
```

`map` 本身不会因此被清空。

## size

`size()` 返回所有 values 的总数量。

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");
map.add("ops", "Alice");

System.out.println(map.size());
```

结果：

```text
3
```

需要注意，这不是 key 的数量。

如果要获取 key 的数量，可以使用：

```java
map.keys().size()
```

示例：

```java
System.out.println(map.keys().size());
```

结果：

```text
2
```

## isEmpty

`isEmpty()` 判断 `MultiMap` 中是否没有任何 value。

```java
var map = new DefaultMultiMap<String, String>();

System.out.println(map.isEmpty());

map.add("dev", "Tom");

System.out.println(map.isEmpty());
```

结果：

```text
true
false
```

内部语义等价于：

```java
size() == 0L
```

## clear

`clear()` 用于清空 `MultiMap`。

```java
map.clear();
```

它会：

1. 清空每个 key 对应的 List。
2. 清空内部 map。

调用后：

```java
map.size()
```

结果为：

```text
0
```

## toMultiValueMap

`toMultiValueMap()` 返回内部的多值 map。

```java
Map<String, List<String>> rawMap = map.toMultiValueMap();
```

需要特别注意：

```text
toMultiValueMap() 返回的是内部 map，不是副本。
```

因此修改返回的 map 会影响原始 `MultiMap`。

```java
var rawMap = map.toMultiValueMap();

rawMap.clear();

System.out.println(map.isEmpty());
```

结果：

```text
true
```

如果需要副本，可以自己复制：

```java
var copy = new HashMap<String, List<String>>();

map.forEachEntry((key, values) -> {
    copy.put(key, new ArrayList<>(values));
});
```

## toSingleValueMap

`toSingleValueMap()` 会把 `MultiMap` 转成普通 `Map<K, V>`。

转换规则是：

```text
每个 key 只保留第一个 value
```

示例：

```java
var map = new DefaultMultiMap<String, String>();

map.add("dev", "Tom");
map.add("dev", "Jerry");
map.add("ops", "Alice");

var singleMap = map.toSingleValueMap();
```

结果：

```text
dev -> Tom
ops -> Alice
```

也可以指定目标 map 类型：

```java
var singleMap = map.toSingleValueMap(LinkedHashMap::new);
```

这适合希望保留指定 map 实现的场景。

## MultiMap 遍历

### forEach

`forEach(...)` 会遍历每一个 key-value 组合。

```java
map.forEach((key, value) -> {
    System.out.println(key + " -> " + value);
});
```

如果 map 内容是：

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

输出：

```text
dev -> Tom
dev -> Jerry
ops -> Alice
```

也就是说，`forEach(...)` 是扁平遍历。

### forEachEntry

`forEachEntry(...)` 会按 entry 遍历。

```java
map.forEachEntry((key, values) -> {
    System.out.println(key + " -> " + values);
});
```

输出：

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

也就是说，`forEachEntry(...)` 是按 key 遍历，每次拿到这个 key 对应的完整 values。

### Iterator

`MultiMap` 实现了 `Iterable<MultiMapEntry<K, V>>`。

因此可以使用增强 for 循环：

```java
for (var entry : map) {
    System.out.println(entry.key());
    System.out.println(entry.value());
    System.out.println(entry.values());
}
```

`MultiMapEntry` 提供：

```java
K key();

V value();

List<V> values();
```

其中：

```text
key()       当前 key
value()     当前 key 对应的第一个 value
values()    当前 key 对应的所有 values
```

如果 values 为空，`value()` 返回 `null`。

## CountMap

`CountMap<K>` 表示一个 key 对应一个计数值的映射。

它和普通 `Map<K, Long>` 很像，但提供了更适合计数场景的 `add(...)` 方法。

例如：

```text
apple  -> 3
orange -> 1
banana -> 6
```

这种结构适合下面这些场景：

1. 统计词频。
2. 统计标签出现次数。
3. 统计分组数量。
4. 统计商品数量。
5. 统计事件次数。
6. 对某个 key 累加权重。

## 创建 CountMap

默认实现是 `DefaultCountMap`。

```java
import dev.scx.collection.count_map.CountMap;
import dev.scx.collection.count_map.DefaultCountMap;

CountMap<String> countMap = new DefaultCountMap<>();
```

默认情况下，内部使用：

```text
HashMap<K, Long>
```

也可以指定内部 map 实现。

```java
import dev.scx.collection.count_map.DefaultCountMap;

import java.util.LinkedHashMap;

var countMap = new DefaultCountMap<String>(LinkedHashMap::new);
```

这适合下面这些场景：

1. 希望 key 按插入顺序保存。
2. 希望使用指定 Map 实现。
3. 希望测试输出顺序更稳定。

## add

`add(key, count)` 用于给某个 key 累加数量。

```java
var countMap = new DefaultCountMap<String>();

long count = countMap.add("apple", 5);
```

执行后：

```text
apple -> 5
```

返回值是添加后的数量：

```text
5
```

继续添加：

```java
count = countMap.add("apple", 3);
```

执行后：

```text
apple -> 8
```

返回值：

```text
8
```

也就是说：

```java
add("apple", 3)
```

不是覆盖，而是累加。

### 可以添加负数

`add(...)` 使用加法累计。

因此可以添加负数。

```java
countMap.add("apple", 10);
countMap.add("apple", -3);
```

结果：

```text
apple -> 7
```

需要注意，`CountMap` 不会阻止计数变成负数。

如果业务上不允许负数，应由调用方自己限制。

## set

`set(key, count)` 用于直接设置某个 key 的计数。

```java
var countMap = new DefaultCountMap<String>();

Long oldValue = countMap.set("apple", 10);
```

如果 key 原来不存在，返回：

```text
null
```

继续设置：

```java
oldValue = countMap.set("apple", 15);
```

执行后：

```text
apple -> 15
```

返回值是覆盖前的数量：

```text
10
```

`set(...)` 和 `add(...)` 的区别是：

```text
add    累加数量
set    覆盖数量
```

## get

`get(key)` 用于获取某个 key 的数量。

```java
countMap.add("apple", 5);

Long count = countMap.get("apple");
```

结果：

```text
5
```

如果 key 不存在，返回：

```text
null
```

因此如果你想把不存在的 key 当作 `0` 处理，可以这样写：

```java
long count = countMap.get("apple") == null ? 0L : countMap.get("apple");
```

或者：

```java
var count = countMap.get("apple");

long safeCount = count == null ? 0L : count;
```

## containsKey

`containsKey(key)` 用于判断是否存在某个 key。

```java
countMap.add("apple", 5);

boolean b1 = countMap.containsKey("apple");
boolean b2 = countMap.containsKey("orange");
```

结果：

```text
true
false
```

需要注意，`containsKey(...)` 判断的是 key 是否存在，而不是数量是否大于 `0`。

例如：

```java
countMap.set("apple", 0);
```

此时：

```java
countMap.containsKey("apple")
```

结果仍然是：

```text
true
```

## remove

`remove(key)` 用于删除某个 key，并返回删除前的数量。

```java
countMap.add("apple", 5);

Long oldValue = countMap.remove("apple");
```

结果：

```text
5
```

删除后：

```java
countMap.get("apple")
```

结果：

```text
null
```

如果 key 不存在，返回：

```text
null
```

## keys

`keys()` 返回所有 key。

```java
var keys = countMap.keys();
```

结果类型是：

```java
Set<K>
```

需要注意，`keys()` 返回的是内部 map 的 `keySet()` 视图。

这意味着：

1. 它不是副本。
2. 当前 CountMap 改变后，keys 视图也会变化。
3. 修改 keys 视图可能影响 CountMap 内部数据。

如果需要安全副本，可以自己复制：

```java
var keysCopy = new HashSet<>(countMap.keys());
```

## size

`size()` 返回 key 的数量。

```java
var countMap = new DefaultCountMap<String>();

countMap.add("apple", 5);
countMap.add("orange", 3);

System.out.println(countMap.size());
```

结果：

```text
2
```

需要注意，这和所有 count 的总和不是一回事。

例如：

```text
apple  -> 5
orange -> 3
```

`size()` 是：

```text
2
```

总计数是：

```text
8
```

如果需要总计数，可以自己遍历：

```java
long total = 0;

for (var entry : countMap) {
    total = total + entry.count();
}
```

## isEmpty

`isEmpty()` 用于判断是否没有任何 key。

```java
var countMap = new DefaultCountMap<String>();

System.out.println(countMap.isEmpty());

countMap.add("apple", 1);

System.out.println(countMap.isEmpty());
```

结果：

```text
true
false
```

## clear

`clear()` 用于清空所有计数。

```java
countMap.clear();
```

调用后：

```java
countMap.isEmpty()
```

结果为：

```text
true
```

## toMap

`toMap()` 用于把 `CountMap` 转成普通 `Map<K, Long>`。

```java
var map = countMap.toMap();
```

默认返回一个新的 `HashMap`。

也可以指定目标 map 类型：

```java
var map = countMap.toMap(LinkedHashMap::new);
```

需要注意，`toMap(...)` 返回的是副本。

修改返回的 map，不会影响原始 `CountMap`。

```java
var map = countMap.toMap();

map.clear();

System.out.println(countMap.isEmpty());
```

`countMap` 不会因此被清空。

## CountMap 遍历

### forEach

`forEach(...)` 会遍历每个 key-count 组合。

```java
countMap.forEach((key, count) -> {
    System.out.println(key + " -> " + count);
});
```

示例输出：

```text
apple -> 5
orange -> 3
```

### Iterator

`CountMap` 实现了 `Iterable<CountMapEntry<K>>`。

因此可以使用增强 for 循环：

```java
for (var entry : countMap) {
    System.out.println(entry.key());
    System.out.println(entry.count());
}
```

`CountMapEntry` 提供：

```java
K key();

long count();
```

## 方法总览

### ScxCollection

```java
static <T, K> MultiMap<K, T> groupingBy(
    Iterable<T> list,
    Function<T, K> keyFn
)

static <T, K, V> MultiMap<K, V> groupingBy(
    Iterable<T> list,
    Function<T, K> keyFn,
    Function<T, V> valueFn
)

static <T, K> MultiMap<K, T> groupingBy(
    T[] list,
    Function<T, K> keyFn
)

static <T, K, V> MultiMap<K, V> groupingBy(
    T[] list,
    Function<T, K> keyFn,
    Function<T, V> valueFn
)
```

```java
static <K> CountMap<K> countingBy(
    Iterable<K> list
)

static <T, K> CountMap<K> countingBy(
    Iterable<T> list,
    Function<T, K> keyFn
)

static <T, K> CountMap<K> countingBy(
    Iterable<T> list,
    Function<T, K> keyFn,
    Function<T, Long> countFn
)

static <K> CountMap<K> countingBy(
    K[] list
)

static <T, K> CountMap<K> countingBy(
    T[] list,
    Function<T, K> keyFn
)

static <T, K> CountMap<K> countingBy(
    T[] list,
    Function<T, K> keyFn,
    Function<T, Long> countFn
)
```

### MultiMap

```java
boolean add(K key, V value)

boolean add(K key, V... values)

boolean add(K key, Collection<V> values)

void add(Map<K, V> map)

void add(MultiMap<K, V> map)
```

```java
List<V> set(K key, V value)

List<V> set(K key, V... values)

List<V> set(K key, Collection<V> values)

void set(Map<K, V> map)

void set(MultiMap<K, V> map)
```

```java
V get(K key)

List<V> getAll(K key)

boolean containsKey(K key)

boolean containsValue(V value)
```

```java
boolean remove(K key, V value)

boolean remove(K key, V... values)

boolean remove(K key, Collection<V> values)

List<V> removeAll(K key)
```

```java
Set<K> keys()

List<V> values()

long size()

boolean isEmpty()

void clear()
```

```java
Map<K, List<V>> toMultiValueMap()

Map<K, V> toSingleValueMap()

Map<K, V> toSingleValueMap(Supplier<Map<K, V>> mapSupplier)
```

```java
<X extends Throwable> void forEach(
    Function2Void<K, V, X> action
) throws X

<X extends Throwable> void forEachEntry(
    Function2Void<K, List<V>, X> action
) throws X
```

### CountMap

```java
long add(K key, long count)

Long set(K key, long count)

Long get(K key)

boolean containsKey(K key)

Long remove(K key)
```

```java
Set<K> keys()

long size()

boolean isEmpty()

void clear()
```

```java
Map<K, Long> toMap()

Map<K, Long> toMap(Supplier<Map<K, Long>> mapSupplier)
```

```java
<X extends Throwable> void forEach(
    Function2Void<K, Long, X> action
) throws X
```

## 完整示例：分组用户

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

public class GroupingUsersDemo {

    public static void main(String[] args) {
        var users = List.of(
            new User("Tom", "dev"),
            new User("Jerry", "dev"),
            new User("Alice", "ops"),
            new User("Bob", "ops"),
            new User("Lucy", "qa")
        );

        var usersByDepartment = ScxCollection.groupingBy(
            users,
            User::department
        );

        usersByDepartment.forEachEntry((department, departmentUsers) -> {
            System.out.println(department + " -> " + departmentUsers);
        });
    }

    public record User(String name, String department) {

    }

}
```

输出类似：

```text
dev -> [User[name=Tom, department=dev], User[name=Jerry, department=dev]]
ops -> [User[name=Alice, department=ops], User[name=Bob, department=ops]]
qa -> [User[name=Lucy, department=qa]]
```

如果只想保留用户名：

```java
var namesByDepartment = ScxCollection.groupingBy(
    users,
    User::department,
    User::name
);
```

结果类似：

```text
dev -> [Tom, Jerry]
ops -> [Alice, Bob]
qa -> [Lucy]
```

## 完整示例：统计词频

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

public class WordCountDemo {

    public static void main(String[] args) {
        var words = List.of(
            "java",
            "scx",
            "java",
            "collection",
            "scx",
            "java"
        );

        var countMap = ScxCollection.countingBy(words);

        countMap.forEach((word, count) -> {
            System.out.println(word + " -> " + count);
        });
    }

}
```

输出类似：

```text
java -> 3
scx -> 2
collection -> 1
```

## 完整示例：按商品累计数量

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

public class OrderCountDemo {

    public static void main(String[] args) {
        var orders = List.of(
            new Order("apple", 2L),
            new Order("orange", 3L),
            new Order("apple", 5L),
            new Order("banana", 1L)
        );

        var countMap = ScxCollection.countingBy(
            orders,
            Order::productId,
            Order::quantity
        );

        System.out.println(countMap.get("apple"));
        System.out.println(countMap.get("orange"));
        System.out.println(countMap.get("banana"));
    }

    public record Order(String productId, Long quantity) {

    }

}
```

输出：

```text
7
3
1
```

## 完整示例：MultiMap 和 CountMap 配合

```java
import dev.scx.collection.ScxCollection;

import java.util.List;

public class CollectionDemo {

    public static void main(String[] args) {
        var users = List.of(
            new User("Tom", "dev", List.of("java", "sql")),
            new User("Jerry", "dev", List.of("java", "web")),
            new User("Alice", "ops", List.of("linux", "sql"))
        );

        var usersByDepartment = ScxCollection.groupingBy(
            users,
            User::department,
            User::name
        );

        System.out.println(usersByDepartment);

        var allTags = users.stream()
            .flatMap(user -> user.tags().stream())
            .toList();

        var tagCount = ScxCollection.countingBy(allTags);

        System.out.println(tagCount);
    }

    public record User(String name, String department, List<String> tags) {

    }

}
```

输出类似：

```text
{dev=[Tom, Jerry], ops=[Alice]}
{java=2, sql=2, web=1, linux=1}
```

## 设计说明

### 1. MultiMap 是 Map<K, List<V>> 的封装

`DefaultMultiMap` 内部使用：

```java
Map<K, List<V>>
```

它的目的不是发明一套复杂集合模型，而是把下面这种重复代码封装起来：

```java
map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
```

使用 `MultiMap` 后，可以直接写：

```java
multiMap.add(key, value);
```

### 2. CountMap 是 Map<K, Long> 的封装

`DefaultCountMap` 内部使用：

```java
Map<K, Long>
```

它主要封装了累计逻辑。

```java
countMap.add(key, count);
```

等价于：

```java
map.merge(key, count, Long::sum);
```

### 3. groupingBy 返回 MultiMap

`ScxCollection.groupingBy(...)` 的结果是 `MultiMap`。

它适合处理“一对多”的分组结果。

```text
department -> users
tag        -> articles
type       -> configs
```

### 4. countingBy 返回 CountMap

`ScxCollection.countingBy(...)` 的结果是 `CountMap`。

它适合处理“一个 key 对应一个累计数量”的结果。

```text
word      -> count
productId -> quantity
status    -> event count
```

### 5. 默认实现不是线程安全的

`DefaultMultiMap` 默认使用 `HashMap` 和 `ArrayList`。

`DefaultCountMap` 默认使用 `HashMap`。

因此它们默认都不是线程安全集合。

如果需要在多线程中共享修改，应由调用方负责同步，或者提供线程安全的底层集合实现。

### 6. 默认遍历顺序不稳定

默认内部 map 是 `HashMap`。

因此 key 的遍历顺序不应被依赖。

如果需要稳定顺序，可以使用 `LinkedHashMap`。

```java
var multiMap = new DefaultMultiMap<String, String>(
    LinkedHashMap::new,
    ArrayList::new
);

var countMap = new DefaultCountMap<String>(
    LinkedHashMap::new
);
```

### 7. 有些方法返回内部视图

需要注意下面这些方法返回的不是纯副本：

```text
MultiMap.keys()
MultiMap.getAll(key)    当 key 存在时
MultiMap.toMultiValueMap()

CountMap.keys()
```

修改这些返回值，可能影响原始集合。

下面这些方法返回的是新集合：

```text
MultiMap.values()
MultiMap.toSingleValueMap()
CountMap.toMap()
```

### 8. MultiMap 的 size 是 value 总数

`MultiMap#size()` 返回的是所有 values 的扁平总数。

```text
dev -> [Tom, Jerry]
ops -> [Alice]
```

`size()` 是：

```text
3
```

而不是：

```text
2
```

如果需要 key 的数量，请使用：

```java
multiMap.keys().size()
```

### 9. CountMap 的 size 是 key 数量

`CountMap#size()` 返回的是 key 的数量。

```text
apple  -> 5
orange -> 3
```

`size()` 是：

```text
2
```

而不是：

```text
8
```

如果需要所有 count 的总和，需要自己遍历累加。

### 10. null 的处理遵循底层集合

默认实现基于 `HashMap` 和 `ArrayList`。

因此默认情况下，key 和 value 都可以是 `null`。

例如：

```java
var map = new DefaultMultiMap<String, String>();

map.add(null, "value");
map.add("key", null);
```

对于 `CountMap`：

```java
var countMap = new DefaultCountMap<String>();

countMap.add(null, 1);
```

是否允许 `null`，最终取决于底层 map/list 实现。

如果你换成不支持 `null` 的集合实现，那么对应行为也会变化。

### 11. equals 只比较相同默认实现

`DefaultMultiMap#equals(...)` 会和另一个 `DefaultMultiMap` 比较内部 map。

`DefaultCountMap#equals(...)` 会和另一个 `DefaultCountMap` 比较内部 map。

也就是说，它们主要用于默认实现之间的相等判断。

不要假定不同 `MultiMap` 实现之间一定可以通过 `equals(...)` 比较为相等。

## 常见问题

### SCX Collection 是不是替代 Java Stream？

不是。

Java Stream 已经提供了很多强大的集合处理能力。

SCX Collection 只是提供更直接的 `MultiMap` 和 `CountMap` 结构，以及对应的简单构造工具。

### groupingBy 和 Stream Collectors.groupingBy 有什么区别？

`Collectors.groupingBy(...)` 返回的是普通 `Map<K, List<T>>`。

`ScxCollection.groupingBy(...)` 返回的是 `MultiMap<K, V>`。

`MultiMap` 提供了更直接的 `add(...)`、`get(...)`、`getAll(...)`、`values()`、`toSingleValueMap()` 等方法。

### countingBy 和 Stream Collectors.counting 有什么区别？

`Collectors.counting()` 通常配合 `groupingBy(...)` 使用，返回普通 `Map<K, Long>`。

`ScxCollection.countingBy(...)` 返回的是 `CountMap<K>`。

`CountMap` 提供了更直接的 `add(...)`、`set(...)`、`get(...)`、`toMap(...)` 等方法。

### MultiMap 的 get 返回什么？

返回某个 key 对应的第一个 value。

如果 key 不存在，返回 `null`。

### MultiMap 的 getAll 返回什么？

返回某个 key 对应的所有 values。

如果 key 不存在，返回空 List。

### getAll 返回的是副本吗？

如果 key 存在，返回的是内部 List。

如果 key 不存在，返回的是新的空 List。

### MultiMap 的 size 是 key 数量吗？

不是。

`MultiMap#size()` 是所有 values 的总数。

如果需要 key 数量，使用：

```java
multiMap.keys().size()
```

### CountMap 的 size 是所有 count 的总和吗？

不是。

`CountMap#size()` 是 key 的数量。

如果需要所有 count 的总和，需要自己遍历累加。

### CountMap 的 add 是覆盖吗？

不是。

`add(...)` 是累加。

如果需要覆盖，使用：

```java
set(...)
```

### CountMap 可以添加负数吗？

可以。

`add(...)` 只是做加法累计，不限制正负。

### CountMap 中不存在的 key 返回 0 吗？

不会。

`get(...)` 返回的是 `Long`。

如果 key 不存在，返回 `null`。

### toMultiValueMap 返回副本吗？

不是。

`toMultiValueMap()` 返回内部 map。

修改返回的 map 会影响原始 `MultiMap`。

### toSingleValueMap 返回副本吗？

是。

它会创建一个新的 map，并把每个 key 的第一个 value 放进去。

### CountMap 的 toMap 返回副本吗？

是。

`toMap()` 会创建一个新的 map。

### 默认实现是否线程安全？

不是。

默认使用 `HashMap` 和 `ArrayList`。

多线程共享修改时，需要调用方自己同步。

### 默认遍历顺序固定吗？

不固定。

默认使用 `HashMap`。

如果需要固定顺序，可以使用 `LinkedHashMap` 作为底层 map。

### 可以自定义底层集合吗？

可以。

`DefaultMultiMap` 可以指定 map 和 list 的 supplier。

```java
new DefaultMultiMap<>(LinkedHashMap::new, LinkedList::new)
```

`DefaultCountMap` 可以指定 map 的 supplier。

```java
new DefaultCountMap<>(LinkedHashMap::new)
```

### MultiMap 支持 null key 或 null value 吗？

默认实现支持，因为默认底层是 `HashMap` 和 `ArrayList`。

但如果你换成其它不支持 null 的集合实现，行为取决于对应集合。

### CountMap 支持 null key 吗？

默认实现支持，因为默认底层是 `HashMap`。

### remove 后如果 values 为空，key 还会保留吗？

不会。

`DefaultMultiMap` 在删除 value 后，如果该 key 对应的 List 为空，会把 key 从 map 中移除。

### groupingBy 会跳过 null 吗？

不会主动跳过。

`groupingBy(...)` 会直接使用 `keyFn` 和 `valueFn` 的结果调用 `multiMap.add(...)`。

如果 key 或 value 是 `null`，是否允许取决于底层 `MultiMap` 实现。

### countingBy 会跳过 null 吗？

`countingBy(...)` 只会在 `countFn` 返回 `null` 时跳过该元素。

如果 keyFn 返回 `null`，默认 `DefaultCountMap` 可以接受这个 null key。

### 什么时候用 MultiMap？

适合一个 key 对应多个 value 的场景。

例如：

```text
部门 -> 用户列表
标签 -> 文章列表
请求头名 -> 请求头值列表
字段名 -> 错误信息列表
```

### 什么时候用 CountMap？

适合一个 key 对应一个累计数量的场景。

例如：

```text
单词 -> 出现次数
商品 -> 购买数量
状态 -> 事件次数
标签 -> 使用次数
```
