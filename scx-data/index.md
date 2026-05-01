# SCX Data

SCX Data 是一个轻量的数据访问抽象库。它提供 `Repository`、`Finder`、`Query`、`FieldPolicy`、`Aggregation` 等接口和 DSL，用来描述数据的增删改查、字段选择、排序分页、聚合查询和锁查询。SCX Data 本身不绑定具体数据库，实际的数据读写由具体的 `Repository` 实现完成。项目当前版本为 `0.2.0`。([GitHub][1])

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-data</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX Data 中最常用的几个概念是：

```text
Repository              数据仓库接口
Finder                  查询结果读取器
Query                   查询描述，包含条件、排序、offset、limit
Where                   查询条件
FieldPolicy             字段策略
AggregatableRepository  支持聚合的数据仓库
LockableRepository      支持锁查询的数据仓库
DataAccessException     数据访问异常
```

通常业务代码只需要面向 `Repository<Entity, ID>` 编程：

```java
Repository<User, Long> userRepo = ...;
```

然后通过它完成添加、查询、更新和删除。

## 快速开始

```java
import dev.scx.data.Repository;

import static dev.scx.data.query.QueryBuilder.*;
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;

public class UserService {

    private final Repository<User, Long> userRepo;

    public UserService(Repository<User, Long> userRepo) {
        this.userRepo = userRepo;
    }

    public Long create(User user) {
        return userRepo.add(user);
    }

    public User getById(Long id) {
        return userRepo.findFirst(eq("id", id));
    }

    public List<User> listActiveUsers() {
        return userRepo.find(
            eq("status", "ACTIVE")
                .desc("created_at")
                .limit(20)
        );
    }

    public long updateName(Long id, String name) {
        var patch = new User();
        patch.setName(name);

        return userRepo.update(
            patch,
            include("name"),
            eq("id", id)
        );
    }

    public long delete(Long id) {
        return userRepo.delete(eq("id", id));
    }

}
```

测试代码中也使用了这种简洁风格，例如 `repo.find(eq("id", 1))` 和 `repo.delete(eq("id", 1))`。([GitHub][2])

## Repository

`Repository<Entity, ID>` 是 SCX Data 的核心接口。

```java
ID add(Entity entity, FieldPolicy fieldPolicy);

List<ID> add(Collection<Entity> entityList, FieldPolicy fieldPolicy);

Finder<Entity> finder(Query query, FieldPolicy fieldPolicy);

long update(Entity entity, FieldPolicy fieldPolicy, Query query);

long delete(Query query);

void clear();
```

它也提供了一组默认便捷方法：

```java
repo.add(entity);
repo.add(entityList);

repo.find();
repo.find(query);
repo.find(fieldPolicy);
repo.find(query, fieldPolicy);

repo.findFirst(query);
repo.findFirst(query, fieldPolicy);

repo.update(entity, query);
repo.update(entity, fieldPolicy, query);

repo.delete(query);

repo.count();
repo.count(query);
```

`add(entity)` 默认使用 `includeAll()` 字段策略；`find(...)` 内部通过 `finder(...).list()` 获取结果；`count(query)` 内部通过 `finder(query).count()` 获取数量。([GitHub][3])

## 添加数据

最简单的添加方式：

```java
var id = userRepo.add(new User(null, "Tom", 18));
```

指定参与插入的字段：

```java
var id = userRepo.add(
    user,
    include("name", "age", "email")
);
```

忽略空值：

```java
var id = userRepo.add(
    user,
    includeAll().ignoreNull(true)
);
```

使用字段表达式：

```java
var id = userRepo.add(
    user,
    include("name", "age")
        .assignField("created_at", "CURRENT_TIMESTAMP")
);
```

当 `entity` 为 `null` 时，也可以只通过字段表达式添加数据：

```java
var id = userRepo.add(
    assignField("created_at", "CURRENT_TIMESTAMP")
);
```

这个能力由 `Repository#add(Entity, FieldPolicy)` 定义，具体表达式如何执行由底层实现决定。([GitHub][3])

## 查询数据

### 查询全部

```java
var users = userRepo.find();
```

### 条件查询

```java
var user = userRepo.findFirst(
    eq("id", 1L)
);
```

```java
var users = userRepo.find(
    eq("status", "ACTIVE")
);
```

条件可以继续追加排序和分页：

```java
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("created_at")
        .limit(20)
);
```

多个条件：

```java
var users = userRepo.find(
    and(
        eq("status", "ACTIVE"),
        gte("age", 18)
    )
        .desc("created_at")
        .offset(0)
        .limit(20)
);
```

动态条件：

```java
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

var users = userRepo.find(
    and()
        .like("name", name, SKIP_IF_BLANK_STRING)
        .eq("status", status, SKIP_IF_NULL)
        .gte("age", minAge, SKIP_IF_NULL)
        .desc("created_at")
        .limit(20)
);
```

`eq`、`and`、`or`、`not`、`whereClause` 等查询对象都基于 `QueryLike`，可以使用 `asc`、`desc`、`offset`、`limit` 等查询方法。([GitHub][4])

### 查询指定字段

```java
var users = userRepo.find(
    eq("status", "ACTIVE"),
    include("id", "name", "email")
);
```

排除字段：

```java
var users = userRepo.find(
    eq("status", "ACTIVE"),
    exclude("password", "salt")
);
```

### 查询单条

```java
var user = userRepo.findFirst(eq("id", 1L));
```

如果没有查询到数据，`Finder#first()` 返回 `null`。([GitHub][5])

### 查询为 Map

```java
var rows = userRepo
    .finder(eq("status", "ACTIVE"))
    .listMap();
```

查询第一条 Map：

```java
var row = userRepo
    .finder(eq("id", 1L))
    .firstMap();
```

### 查询为指定类型

```java
var rows = userRepo
    .finder(eq("status", "ACTIVE"))
    .list(UserDTO.class);
```

```java
var dto = userRepo
    .finder(eq("id", 1L))
    .first(UserDTO.class);
```

`Finder` 支持读取为实体、指定类型或 `Map`，也支持 `forEach` / `forEachMap` 流式处理。([GitHub][5])

## 查询条件

常用条件方法来自 `QueryBuilder`：

```java
eq("id", 1)
ne("status", "DELETED")

lt("age", 18)
lte("age", 18)
gt("age", 18)
gte("age", 18)

like("name", "tom")
notLike("name", "test")

likeRegex("email", ".*@example.com")
notLikeRegex("email", ".*@example.com")

in("status", List.of("ACTIVE", "LOCKED"))
notIn("status", List.of("DELETED"))

between("age", 18, 60)
notBetween("age", 18, 60)
```

组合条件：

```java
and(
    eq("status", "ACTIVE"),
    gte("age", 18)
)
```

```java
or(
    eq("role", "ADMIN"),
    eq("role", "OWNER")
)
```

```java
not(eq("deleted", true))
```

原始条件表达式：

```java
whereClause("created_at >= ? AND created_at < ?", startTime, endTime)
```

条件类型包括等于、不等于、大小比较、模糊匹配、正则匹配、集合匹配和范围匹配；`EQ` / `NE` 支持 `null` 比较，`IN` / `NOT_IN` 允许集合中包含 `null`。([GitHub][6])

## 动态查询

动态查询时，可以使用 `BuildControl` 控制空值是否跳过：

```java
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

var users = userRepo.find(
    and()
        .eq("status", status, SKIP_IF_NULL)
        .like("name", name, SKIP_IF_BLANK_STRING)
        .in("role", roles, SKIP_IF_EMPTY_LIST)
        .between("created_at", startTime, endTime, SKIP_IF_NULL)
        .desc("created_at")
        .limit(20)
);
```

可用控制项：

```java
SKIP_IF_NULL
SKIP_IF_EMPTY_LIST
SKIP_IF_EMPTY_STRING
SKIP_IF_BLANK_STRING
USE_EXPRESSION
USE_EXPRESSION_VALUE
```

`SKIP_IF_NULL` 用于值为 `null` 时跳过条件；`SKIP_IF_EMPTY_LIST` 用于空集合或空数组；`SKIP_IF_EMPTY_STRING` 用于空字符串；`SKIP_IF_BLANK_STRING` 用于空白字符串。`USE_EXPRESSION` 表示字段选择器按表达式处理，`USE_EXPRESSION_VALUE` 表示值按表达式处理。([GitHub][7])

示例：

```java
var users = userRepo.find(
    eq("LOWER(name)", "LOWER(?)", USE_EXPRESSION, USE_EXPRESSION_VALUE)
);
```

具体表达式如何转换，由底层数据源实现决定。

## 排序和分页

升序：

```java
var users = userRepo.find(
    eq("status", "ACTIVE")
        .asc("name")
);
```

降序：

```java
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("created_at")
);
```

分页：

```java
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("created_at")
        .offset(20)
        .limit(20)
);
```

`offset` 和 `limit` 必须大于等于 `0`，否则会抛出 `IllegalArgumentException`。([GitHub][8])

## 更新数据

更新实体：

```java
var patch = new User();
patch.setName("Jerry");

long updated = userRepo.update(
    patch,
    include("name"),
    eq("id", 1L)
);
```

忽略空值更新：

```java
long updated = userRepo.update(
    patch,
    includeAll().ignoreNull(true),
    eq("id", 1L)
);
```

排除不允许更新的字段：

```java
long updated = userRepo.update(
    patch,
    exclude("id", "created_at"),
    eq("id", 1L)
);
```

使用字段表达式：

```java
long updated = userRepo.update(
    assignField("updated_at", "CURRENT_TIMESTAMP"),
    eq("id", 1L)
);
```

自增字段：

```java
long updated = userRepo.update(
    assignField("view_count", "view_count + 1"),
    eq("id", 1L)
);
```

`Repository#update` 返回更新的数据条数。当 `entity` 为 `null` 时，可以使用 `FieldPolicy` 进行纯表达式更新。([GitHub][3])

## 删除数据

```java
long deleted = userRepo.delete(eq("id", 1L));
```

```java
long deleted = userRepo.delete(
    eq("status", "DELETED")
);
```

删除方法返回删除的数据条数。([GitHub][3])

## 统计数量

```java
long count = userRepo.count();
```

```java
long activeCount = userRepo.count(
    eq("status", "ACTIVE")
);
```

`Finder#count()` 会忽略 `offset` 和 `limit`。([GitHub][5])

## FieldPolicy 字段策略

`FieldPolicy` 用于控制哪些字段参与查询、插入或更新，也可以设置虚拟字段、字段表达式和空值处理策略。([GitHub][9])

### 包含字段

```java
include("id", "name", "email")
```

### 排除字段

```java
exclude("password", "salt")
```

### 包含全部字段

```java
includeAll()
```

### 排除全部字段

```java
excludeAll()
```

`includeAll()` 表示包含所有字段；`excludeAll()` 表示排除所有字段。内部通过 `FilterMode.INCLUDED` 和 `FilterMode.EXCLUDED` 表示包含模式和排除模式。([GitHub][10])

### 忽略 null

默认字段策略会忽略 `null` 值。可以通过 `ignoreNull(false)` 改为不忽略：

```java
includeAll().ignoreNull(false)
```

也可以针对单个字段设置：

```java
includeAll()
    .ignoreNull("nickname", false)
```

`FieldPolicyImpl` 中全局 `ignoreNull` 的默认值为 `true`。([GitHub][11])

### 虚拟字段

虚拟字段主要用于查询：

```java
var rows = userRepo.find(
    eq("status", "ACTIVE"),
    include("id", "name")
        .virtualField("age_group", "CASE WHEN age >= 18 THEN 'adult' ELSE 'minor' END")
);
```

也可以直接使用构造方法：

```java
virtualField("age_group", "CASE WHEN age >= 18 THEN 'adult' ELSE 'minor' END")
```

`VirtualField` 要求虚拟字段名和表达式都不能为 `null`。([GitHub][12])

### 字段表达式

字段表达式主要用于插入和更新：

```java
assignField("updated_at", "CURRENT_TIMESTAMP")
```

```java
include("name")
    .assignField("updated_at", "CURRENT_TIMESTAMP")
```

`AssignField` 要求字段名和表达式都不能为 `null`。([GitHub][13])

## Finder

`Finder<Entity>` 是查询结果读取器。

```java
Finder<User> finder = userRepo.finder(
    eq("status", "ACTIVE")
        .desc("created_at")
        .limit(20)
);
```

读取列表：

```java
List<User> users = finder.list();
```

读取指定类型：

```java
List<UserDTO> users = finder.list(UserDTO.class);
```

读取 Map：

```java
List<Map<String, Object>> rows = finder.listMap();
```

读取第一条：

```java
User user = finder.first();
```

遍历：

```java
finder.forEach(user -> {
    System.out.println(user);
});
```

遍历 Map：

```java
finder.forEachMap(row -> {
    System.out.println(row);
});
```

`first()`、`first(Class<T>)` 和 `firstMap()` 找不到数据时返回 `null`；`forEach` 和 `forEachMap` 中用户回调抛出的异常会被包装为 `ScxWrappedException`。([GitHub][5])

## 聚合查询

如果一个仓库实现了 `AggregatableRepository<Entity, ID>`，就可以使用聚合查询。

```java
import static dev.scx.data.aggregation.AggregationBuilder.*;
import static dev.scx.data.query.QueryBuilder.*;

AggregatableRepository<Order, Long> orderRepo = ...;

var rows = orderRepo.aggregate(
    eq("status", "PAID"),
    aggregation()
        .groupBy("user_id")
        .agg("total_amount", "SUM(amount)")
        .agg("order_count", "COUNT(*)"),
    desc("total_amount").limit(10)
);
```

聚合对象：

```java
var agg = aggregation()
    .groupBy("user_id")
    .agg("total_amount", "SUM(amount)")
    .agg("order_count", "COUNT(*)");
```

按表达式分组：

```java
var agg = aggregation()
    .groupBy("day", "DATE(created_at)")
    .agg("total_amount", "SUM(amount)");
```

读取第一条聚合结果：

```java
var row = orderRepo.aggregateFirst(
    eq("status", "PAID"),
    aggregation().agg("total_amount", "SUM(amount)")
);
```

`AggregatableRepository` 的核心方法是 `aggregator(beforeAggregateQuery, aggregation, afterAggregateQuery)`，聚合查询结果默认读取为 `Map<String, Object>`，也可以通过 `Aggregator` 读取为指定类型。([GitHub][14])

## 锁查询

如果一个仓库实现了 `LockableRepository<Entity, ID>`，就可以使用锁模式查询。

```java
import static dev.scx.data.LockMode.EXCLUSIVE;
import static dev.scx.data.query.QueryBuilder.*;

LockableRepository<User, Long> userRepo = ...;

var user = userRepo.findFirst(
    eq("id", 1L),
    EXCLUSIVE
);
```

共享锁：

```java
var users = userRepo.find(
    eq("status", "ACTIVE"),
    SHARED
);
```

锁模式包括：

```java
SHARED
EXCLUSIVE
```

`SHARED` 表示共享锁，允许多个读取、阻止写入；`EXCLUSIVE` 表示排他锁，阻止读取和写入。([GitHub][15])

## 异常处理

SCX Data 使用 `DataAccessException` 表示数据访问异常。它是运行时异常，提供 message、cause、message + cause 三种构造方式。([GitHub][16])

```java
try {
    var users = userRepo.find(eq("status", "ACTIVE"));
} catch (DataAccessException e) {
    // 记录日志或转换为业务异常
}
```

## 实现 Repository

SCX Data 是抽象层，真正的数据访问由具体实现提供。一个最小的 `Repository` 实现大致如下：

```java
import dev.scx.data.Finder;
import dev.scx.data.Repository;
import dev.scx.data.exception.DataAccessException;
import dev.scx.data.field_policy.FieldPolicy;
import dev.scx.data.query.Query;

import java.util.Collection;
import java.util.List;

public class UserRepository implements Repository<User, Long> {

    @Override
    public Long add(User entity, FieldPolicy fieldPolicy) throws DataAccessException {
        // 根据 entity 和 fieldPolicy 执行插入
        return null;
    }

    @Override
    public List<Long> add(Collection<User> entityList, FieldPolicy fieldPolicy) throws DataAccessException {
        // 批量插入
        return List.of();
    }

    @Override
    public Finder<User> finder(Query query, FieldPolicy fieldPolicy) {
        // 创建 Finder
        return new UserFinder(query, fieldPolicy);
    }

    @Override
    public long update(User entity, FieldPolicy fieldPolicy, Query query) throws DataAccessException {
        // 根据 entity、fieldPolicy 和 query 执行更新
        return 0;
    }

    @Override
    public long delete(Query query) throws DataAccessException {
        // 根据 query 执行删除
        return 0;
    }

    @Override
    public void clear() throws DataAccessException {
        // 清空数据
    }

}
```

测试代码中的 `TestRepository` 就是一个最小实现示例，它实现了 `AggregatableRepository` 和 `LockableRepository`，并返回自定义的 `TestFinder`。([GitHub][17])

## 完整示例

```java
import dev.scx.data.Repository;
import dev.scx.data.exception.DataAccessException;

import java.util.List;

import static dev.scx.data.field_policy.FieldPolicyBuilder.*;
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

public class UserService {

    private final Repository<User, Long> userRepo;

    public UserService(Repository<User, Long> userRepo) {
        this.userRepo = userRepo;
    }

    public Long createUser(User user) throws DataAccessException {
        return userRepo.add(
            user,
            include("name", "email", "age", "status")
                .ignoreNull(true)
                .assignField("created_at", "CURRENT_TIMESTAMP")
        );
    }

    public List<User> searchUsers(String name, String status, Integer minAge, int page, int size) {
        return userRepo.find(
            and()
                .like("name", name, SKIP_IF_BLANK_STRING)
                .eq("status", status, SKIP_IF_NULL)
                .gte("age", minAge, SKIP_IF_NULL)
                .desc("created_at")
                .offset((long) (page - 1) * size)
                .limit(size),
            exclude("password", "salt")
        );
    }

    public User getUser(Long id) {
        return userRepo.findFirst(eq("id", id));
    }

    public long updateUser(Long id, User patch) {
        return userRepo.update(
            patch,
            exclude("id", "created_at").ignoreNull(true),
            eq("id", id)
        );
    }

    public long increaseViewCount(Long id) {
        return userRepo.update(
            assignField("view_count", "view_count + 1"),
            eq("id", id)
        );
    }

    public long deleteUser(Long id) {
        return userRepo.delete(eq("id", id));
    }

}
```

## 常见问题

### SCX Data 可以直接连接数据库吗？

SCX Data 本身提供的是数据访问抽象、查询 DSL 和字段策略，不包含具体数据库连接、SQL 执行或连接池实现。你需要使用已有适配器，或者自己实现 `Repository` / `Finder`。([GitHub][18])

### 默认会忽略 null 吗？

会。`FieldPolicyImpl` 的全局 `ignoreNull` 默认是 `true`。可以使用 `ignoreNull(false)` 改成不忽略，也可以使用 `ignoreNull("fieldName", false)` 针对单个字段设置。([GitHub][11])

### `count()` 会受分页影响吗？

不会。`Finder#count()` 会忽略 `offset` 和 `limit`。([GitHub][5])

### 什么时候使用 `whereClause`？

当结构化条件无法表达底层查询能力时，可以使用 `whereClause(expression, params...)`。例如 SQL 实现中可能把它转换为原始 SQL 条件片段。具体语义由底层实现决定。([GitHub][19])

### `includeAll()` 和 `excludeAll()` 分别是什么意思？

`includeAll()` 表示包含所有字段，`excludeAll()` 表示排除所有字段。`include("name")` 表示只包含指定字段，`exclude("password")` 表示排除指定字段。

[1]: https://raw.githubusercontent.com/scx-projects/scx-data/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/test/java/dev/scx/data/test/DataTest.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/Repository.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/QueryLike.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/Finder.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/QueryBuilder.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/BuildControl.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/QueryImpl.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicy.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicyBuilder.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicyImpl.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/VirtualField.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/AssignField.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/AggregatableRepository.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/LockMode.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/exception/DataAccessException.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/test/java/dev/scx/data/test/TestRepository.java "raw.githubusercontent.com"
[18]: https://github.com/scx-projects/scx-data/tree/master/src/main/java/dev/scx/data "scx-data/src/main/java/dev/scx/data at master · scx-projects/scx-data · GitHub"
[19]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/WhereClause.java "raw.githubusercontent.com"
