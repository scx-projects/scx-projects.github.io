# SCX Data

SCX Data 是一个底层无关的数据访问抽象库。

它提供 `Repository`、`Finder`、`Query`、`FieldPolicy`、`Aggregation` 等接口和 DSL，用来描述数据的添加、查询、更新、删除、字段选择、排序分页、聚合和锁查询。SCX Data 本身不绑定任何具体存储系统，也不定义底层查询语言；一个 `Repository` 实现可以把这些描述转换为任意数据源能够理解的访问方式。项目当前版本为 `0.2.0`。([GitHub][1])

## 安装

### Maven

```xml id="8dn7kn"
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-data</artifactId>
    <version>0.2.0</version>
</dependency>
```

## 基本概念

SCX Data 中最常用的概念包括：

```text id="28j2mr"
Repository              数据仓库接口
Finder                  查询结果读取器
Query                   查询描述，包含条件、排序、offset、limit
Where                   查询条件
FieldPolicy             字段策略
AggregatableRepository  支持聚合的数据仓库
LockableRepository      支持锁查询的数据仓库
DataAccessException     数据访问异常
```

一般业务代码只需要面向 `Repository<Entity, ID>` 编程：

```java id="6a6mtk"
Repository<User, Long> userRepo = ...;
```

然后通过它完成添加、查询、更新、删除等操作。

## 快速开始

```java id="fgtm4v"
import dev.scx.data.Repository;

import java.util.List;

import static dev.scx.data.field_policy.FieldPolicyBuilder.*;
import static dev.scx.data.query.QueryBuilder.*;

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
                .desc("createdAt")
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

测试代码中也展示了这种简单用法，例如 `repo.find(eq("id", 1))` 和 `repo.delete(eq("id", 1))`。([GitHub][2])

## Repository

`Repository<Entity, ID>` 是 SCX Data 的核心接口。它定义了添加、批量添加、创建 `Finder`、更新、删除和清空数据等基础能力。([GitHub][3])

```java id="wvuq6g"
ID add(Entity entity, FieldPolicy fieldPolicy);

List<ID> add(Collection<Entity> entityList, FieldPolicy fieldPolicy);

Finder<Entity> finder(Query query, FieldPolicy fieldPolicy);

long update(Entity entity, FieldPolicy fieldPolicy, Query query);

long delete(Query query);

void clear();
```

它也提供了一组便捷方法：

```java id="ffl90t"
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

```java id="t1maku"
var id = userRepo.add(new User(null, "Tom", 18));
```

指定参与添加的字段：

```java id="no9c0c"
var id = userRepo.add(
    user,
    include("name", "age", "email")
);
```

忽略空值：

```java id="45i77n"
var id = userRepo.add(
    user,
    includeAll().ignoreNull(true)
);
```

使用字段表达式：

```java id="eex3cr"
var id = userRepo.add(
    user,
    include("name", "age")
        .assignField("createdAt", "<current-time-expression>")
);
```

当 `entity` 为 `null` 时，也可以只通过字段策略添加数据：

```java id="pifxsz"
var id = userRepo.add(
    assignField("createdAt", "<current-time-expression>")
);
```

这里的表达式字符串不会被 SCX Data 自身解析。它只是被保存在 `FieldPolicy` 中，具体含义由对应的 `Repository` 实现决定。`Repository#add` 的源码注释也说明了：当 `entity` 为 `null` 时，会使用 `fieldPolicy` 进行纯表达式添加。([GitHub][3])

## 查询数据

### 查询全部

```java id="pnvf9l"
var users = userRepo.find();
```

### 条件查询

```java id="g31s7p"
var user = userRepo.findFirst(
    eq("id", 1L)
);
```

```java id="fjanmu"
var users = userRepo.find(
    eq("status", "ACTIVE")
);
```

条件可以继续追加排序和分页：

```java id="x6hft3"
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("createdAt")
        .limit(20)
);
```

多个条件：

```java id="f9v19u"
var users = userRepo.find(
    and(
        eq("status", "ACTIVE"),
        gte("age", 18)
    )
        .desc("createdAt")
        .offset(0)
        .limit(20)
);
```

动态条件：

```java id="25flxr"
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

var users = userRepo.find(
    and()
        .like("name", name, SKIP_IF_BLANK_STRING)
        .eq("status", status, SKIP_IF_NULL)
        .gte("age", minAge, SKIP_IF_NULL)
        .desc("createdAt")
        .limit(20)
);
```

`Query` 支持条件、排序、`offset` 和 `limit`；`QueryLike` 让条件对象和条件组合对象也可以继续调用 `asc`、`desc`、`offset`、`limit` 等方法。([GitHub][4])

### 查询指定字段

```java id="wf3tni"
var users = userRepo.find(
    eq("status", "ACTIVE"),
    include("id", "name", "email")
);
```

排除字段：

```java id="y8bm2q"
var users = userRepo.find(
    eq("status", "ACTIVE"),
    exclude("password", "secret")
);
```

### 查询单条

```java id="odtwj0"
var user = userRepo.findFirst(eq("id", 1L));
```

如果没有查询到数据，`Finder#first()` 返回 `null`。([GitHub][5])

### 查询为 Map

```java id="mfwz91"
var rows = userRepo
    .finder(eq("status", "ACTIVE"))
    .listMap();
```

查询第一条 Map：

```java id="43mldy"
var row = userRepo
    .finder(eq("id", 1L))
    .firstMap();
```

### 查询为指定类型

```java id="zs0zf8"
var rows = userRepo
    .finder(eq("status", "ACTIVE"))
    .list(UserDTO.class);
```

```java id="eg590a"
var dto = userRepo
    .finder(eq("id", 1L))
    .first(UserDTO.class);
```

`Finder` 支持读取为实体、指定类型或 `Map`，也支持 `forEach` / `forEachMap` 遍历结果。([GitHub][5])

## 查询条件

常用条件方法来自 `QueryBuilder`：

```java id="z1u7sk"
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

```java id="6xvsj5"
and(
    eq("status", "ACTIVE"),
    gte("age", 18)
)
```

```java id="2t4zf3"
or(
    eq("role", "ADMIN"),
    eq("role", "OWNER")
)
```

```java id="z1qa8q"
not(eq("deleted", true))
```

`ConditionType` 中定义了等于、不等于、大小比较、模糊匹配、正则匹配、集合匹配和范围匹配等条件类型；`EQ` / `NE` 支持 `null` 比较，`IN` / `NOT_IN` 允许集合中包含 `null`。([GitHub][6])

## 动态查询

动态查询时，可以用 `BuildControl` 控制某个值为空时是否跳过条件：

```java id="dage2o"
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

var users = userRepo.find(
    and()
        .eq("status", status, SKIP_IF_NULL)
        .like("name", name, SKIP_IF_BLANK_STRING)
        .in("role", roles, SKIP_IF_EMPTY_LIST)
        .between("createdAt", startTime, endTime, SKIP_IF_NULL)
        .desc("createdAt")
        .limit(20)
);
```

可用控制项：

```java id="g7yiqs"
SKIP_IF_NULL
SKIP_IF_EMPTY_LIST
SKIP_IF_EMPTY_STRING
SKIP_IF_BLANK_STRING
USE_EXPRESSION
USE_EXPRESSION_VALUE
```

`SKIP_IF_NULL` 用于值为 `null` 时跳过条件；`SKIP_IF_EMPTY_LIST` 用于空集合或空数组；`SKIP_IF_EMPTY_STRING` 用于空字符串；`SKIP_IF_BLANK_STRING` 用于空白字符串。`USE_EXPRESSION` 和 `USE_EXPRESSION_VALUE` 表示对应内容作为实现相关表达式处理，不由 SCX Data 进行普通字段或普通值转换。([GitHub][7])

示例：

```java id="4lpk1g"
var users = userRepo.find(
    eq("<normalized-name-selector>", "<normalized-name-value>", USE_EXPRESSION, USE_EXPRESSION_VALUE)
);
```

表达式的语法和含义由具体 `Repository` 实现决定。SCX Data 只保存这类描述，不规定它应该如何被解释。

## 排序和分页

升序：

```java id="ibgc7t"
var users = userRepo.find(
    eq("status", "ACTIVE")
        .asc("name")
);
```

降序：

```java id="j8bhm0"
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("createdAt")
);
```

分页：

```java id="7vje2t"
var users = userRepo.find(
    eq("status", "ACTIVE")
        .desc("createdAt")
        .offset(20)
        .limit(20)
);
```

`offset` 和 `limit` 必须大于等于 `0`，否则会抛出 `IllegalArgumentException`。([GitHub][8])

## 更新数据

更新实体：

```java id="9vwebs"
var patch = new User();
patch.setName("Jerry");

long updated = userRepo.update(
    patch,
    include("name"),
    eq("id", 1L)
);
```

忽略空值更新：

```java id="q6ayzu"
long updated = userRepo.update(
    patch,
    includeAll().ignoreNull(true),
    eq("id", 1L)
);
```

排除不允许更新的字段：

```java id="mgrx30"
long updated = userRepo.update(
    patch,
    exclude("id", "createdAt"),
    eq("id", 1L)
);
```

使用字段表达式：

```java id="cl97b8"
long updated = userRepo.update(
    assignField("updatedAt", "<current-time-expression>"),
    eq("id", 1L)
);
```

修改某个字段为实现相关表达式：

```java id="mfwngp"
long updated = userRepo.update(
    assignField("viewCount", "<increment-expression>"),
    eq("id", 1L)
);
```

`Repository#update` 返回更新的数据条数。当 `entity` 为 `null` 时，可以使用 `FieldPolicy` 进行纯表达式更新；此时要求 `fieldPolicy` 至少包含一个字段表达式。([GitHub][3])

## 删除数据

```java id="o6xtp5"
long deleted = userRepo.delete(eq("id", 1L));
```

```java id="j1s6cc"
long deleted = userRepo.delete(
    eq("status", "DELETED")
);
```

删除方法返回删除的数据条数。([GitHub][3])

## 统计数量

```java id="ax0oka"
long count = userRepo.count();
```

```java id="tjad9s"
long activeCount = userRepo.count(
    eq("status", "ACTIVE")
);
```

`Finder#count()` 会忽略 `offset` 和 `limit`。([GitHub][5])

## FieldPolicy 字段策略

`FieldPolicy` 用于描述一次操作中哪些字段参与、哪些字段排除、是否忽略空值，以及是否附加虚拟字段或字段表达式。它可以用于查询、添加和更新；其中虚拟字段用于查询，字段表达式用于添加和更新。([GitHub][9])

### 包含字段

```java id="wc10ta"
include("id", "name", "email")
```

### 排除字段

```java id="kgl2e5"
exclude("password", "secret")
```

### 包含全部字段

```java id="5oz92l"
includeAll()
```

### 排除全部字段

```java id="yrt694"
excludeAll()
```

`includeAll()` 表示包含所有字段；`excludeAll()` 表示排除所有字段。`include("name")` 表示只包含指定字段，`exclude("password")` 表示排除指定字段。([GitHub][10])

### 忽略 null

默认字段策略会忽略 `null` 值。可以通过 `ignoreNull(false)` 改为不忽略：

```java id="18beq2"
includeAll().ignoreNull(false)
```

也可以针对单个字段设置：

```java id="dsazgt"
includeAll()
    .ignoreNull("nickname", false)
```

`FieldPolicyImpl` 中全局 `ignoreNull` 的默认值为 `true`。([GitHub][11])

### 虚拟字段

虚拟字段主要用于查询：

```java id="qrqkks"
var rows = userRepo.find(
    eq("status", "ACTIVE"),
    include("id", "name")
        .virtualField("displayName", "<display-name-expression>")
);
```

也可以直接使用构造方法：

```java id="7pdl7h"
virtualField("displayName", "<display-name-expression>")
```

虚拟字段表达式的含义由具体实现决定。SCX Data 只把它作为一个字段描述保存下来。([GitHub][9])

### 字段表达式

字段表达式主要用于添加和更新：

```java id="wedm8p"
assignField("updatedAt", "<current-time-expression>")
```

```java id="vxwxlk"
include("name")
    .assignField("updatedAt", "<current-time-expression>")
```

字段表达式同样是实现相关字符串。SCX Data 不解析、不校验、不规定表达式语法。([GitHub][9])

## Finder

`Finder<Entity>` 是查询结果读取器。

```java id="hmcbsc"
Finder<User> finder = userRepo.finder(
    eq("status", "ACTIVE")
        .desc("createdAt")
        .limit(20)
);
```

读取列表：

```java id="ouey1o"
List<User> users = finder.list();
```

读取指定类型：

```java id="7zv2aw"
List<UserDTO> users = finder.list(UserDTO.class);
```

读取 Map：

```java id="s8rbya"
List<Map<String, Object>> rows = finder.listMap();
```

读取第一条：

```java id="0h5w35"
User user = finder.first();
```

遍历：

```java id="p6rmdd"
finder.forEach(user -> {
    System.out.println(user);
});
```

遍历 Map：

```java id="6gds8k"
finder.forEachMap(row -> {
    System.out.println(row);
});
```

`first()`、`first(Class<T>)` 和 `firstMap()` 找不到数据时返回 `null`；`forEach` 和 `forEachMap` 中用户回调抛出的异常会被包装为 `ScxWrappedException`。([GitHub][5])

## 聚合查询

如果一个仓库实现了 `AggregatableRepository<Entity, ID>`，就可以使用聚合查询。

```java id="p5q9jv"
import static dev.scx.data.aggregation.AggregationBuilder.*;
import static dev.scx.data.query.QueryBuilder.*;

AggregatableRepository<Order, Long> orderRepo = ...;

var rows = orderRepo.aggregate(
    eq("status", "PAID"),
    aggregation()
        .groupBy("userId")
        .agg("totalAmount", "<total-amount-expression>")
        .agg("orderCount", "<order-count-expression>"),
    desc("totalAmount").limit(10)
);
```

聚合对象：

```java id="0ng2hm"
var agg = aggregation()
    .groupBy("userId")
    .agg("totalAmount", "<total-amount-expression>")
    .agg("orderCount", "<order-count-expression>");
```

按表达式分组：

```java id="2z1bjr"
var agg = aggregation()
    .groupBy("period", "<period-expression>")
    .agg("totalAmount", "<total-amount-expression>");
```

读取第一条聚合结果：

```java id="du4oiv"
var row = orderRepo.aggregateFirst(
    eq("status", "PAID"),
    aggregation().agg("totalAmount", "<total-amount-expression>")
);
```

`Aggregation` 负责描述分组项和聚合项，`AggregationBuilder` 提供 `aggregation()`、`groupBy(...)` 和 `agg(...)` 等构造方法；其中表达式字符串由具体实现解释。([GitHub][12])

`AggregatableRepository` 的核心方法是 `aggregator(beforeAggregateQuery, aggregation, afterAggregateQuery)`，同时提供 `aggregate(...)` 和 `aggregateFirst(...)` 等便捷方法。聚合结果默认可以读取为 `Map<String, Object>`，也可以通过 `Aggregator` 读取为指定类型。([GitHub][13])

## 锁查询

如果一个仓库实现了 `LockableRepository<Entity, ID>`，就可以使用锁模式查询。

```java id="r4cjnr"
import static dev.scx.data.LockMode.EXCLUSIVE;
import static dev.scx.data.query.QueryBuilder.*;

LockableRepository<User, Long> userRepo = ...;

var user = userRepo.findFirst(
    eq("id", 1L),
    EXCLUSIVE
);
```

共享锁：

```java id="2o9ye8"
import static dev.scx.data.LockMode.SHARED;

var users = userRepo.find(
    eq("status", "ACTIVE"),
    SHARED
);
```

锁模式包括：

```java id="s48bag"
SHARED
EXCLUSIVE
```

`SHARED` 表示共享锁，允许多个读取、阻止写入；`EXCLUSIVE` 表示排他锁，阻止读取和写入。`LockableRepository` 在普通 `finder(query, fieldPolicy)` 之外增加了带 `LockMode` 的 `finder(query, fieldPolicy, lockMode)`，并提供带锁的 `find(...)`、`findFirst(...)` 等便捷方法。([GitHub][14])

## 异常处理

SCX Data 使用 `DataAccessException` 表示数据访问异常。它继承自 `RuntimeException`，提供 message、cause、message + cause 三种构造方式。([GitHub][15])

```java id="ig9e14"
try {
    var users = userRepo.find(eq("status", "ACTIVE"));
} catch (DataAccessException e) {
    // 记录日志或转换为业务异常
}
```

## 实现 Repository

SCX Data 是抽象层，真正的数据访问由具体实现提供。一个最小的 `Repository` 实现大致如下：

```java id="n9zqos"
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
        // 根据 entity 和 fieldPolicy 执行添加
        return null;
    }

    @Override
    public List<Long> add(Collection<User> entityList, FieldPolicy fieldPolicy) throws DataAccessException {
        // 批量添加
        return List.of();
    }

    @Override
    public Finder<User> finder(Query query, FieldPolicy fieldPolicy) {
        // 根据 query 和 fieldPolicy 创建 Finder
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

测试代码中的 `TestRepository` 是一个最小实现示例，它实现了 `AggregatableRepository` 和 `LockableRepository`，并返回自定义的 `TestFinder`。([GitHub][16])

## 完整示例

```java id="nevd83"
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
                .assignField("createdAt", "<current-time-expression>")
        );
    }

    public List<User> searchUsers(String name, String status, Integer minAge, int page, int size) {
        return userRepo.find(
            and()
                .like("name", name, SKIP_IF_BLANK_STRING)
                .eq("status", status, SKIP_IF_NULL)
                .gte("age", minAge, SKIP_IF_NULL)
                .desc("createdAt")
                .offset((long) (page - 1) * size)
                .limit(size),
            exclude("password", "secret")
        );
    }

    public User getUser(Long id) {
        return userRepo.findFirst(eq("id", id));
    }

    public long updateUser(Long id, User patch) {
        return userRepo.update(
            patch,
            exclude("id", "createdAt").ignoreNull(true),
            eq("id", id)
        );
    }

    public long increaseViewCount(Long id) {
        return userRepo.update(
            assignField("viewCount", "<increment-expression>"),
            eq("id", id)
        );
    }

    public long deleteUser(Long id) {
        return userRepo.delete(eq("id", id));
    }

}
```

## 设计说明

### 1. SCX Data 不绑定底层实现

SCX Data 只定义接口、查询描述、字段策略和聚合描述。它不关心数据来自什么系统，也不规定表达式语法。具体如何执行，由 `Repository` 实现决定。

### 2. Query 是结构化查询描述

`Query` 描述的是条件、排序、offset 和 limit。它不是某种具体查询语言的字符串。条件、排序和分页如何转换为底层操作，由具体实现负责。([GitHub][4])

### 3. FieldPolicy 是字段策略描述

`FieldPolicy` 描述字段包含、字段排除、空值处理、虚拟字段和字段表达式。它不要求底层一定有“表字段”或“列”的概念，只要求实现者能理解这些字段选择和字段表达式。([GitHub][9])

### 4. 表达式是实现相关字符串

`assignField(...)`、`virtualField(...)`、`groupBy(alias, expression)`、`agg(alias, expression)`、`whereClause(...)` 中的表达式都只是字符串。SCX Data 不解析它们，也不规定语法。一个实现可以选择支持表达式，也可以限制表达式能力。

### 5. Repository 是边界

业务代码面向 `Repository` 编程；具体数据源适配、查询翻译、结果映射、锁语义和异常转换，都应该放在 `Repository` / `Finder` / `Aggregator` 的实现中。

## 常见问题

### SCX Data 可以直接访问某个存储系统吗？

SCX Data 本身不提供具体存储系统实现。它提供的是抽象接口和描述对象。你需要使用已有的 `Repository` 实现，或者自己实现 `Repository` / `Finder`。

### 默认会忽略 null 吗？

会。`FieldPolicyImpl` 的全局 `ignoreNull` 默认是 `true`。可以使用 `ignoreNull(false)` 改成不忽略，也可以使用 `ignoreNull("fieldName", false)` 针对单个字段设置。([GitHub][11])

### `count()` 会受分页影响吗？

不会。`Finder#count()` 会忽略 `offset` 和 `limit`。([GitHub][5])

### 什么时候使用 `whereClause`？

当普通结构化条件无法描述某个实现特有能力时，可以使用 `whereClause(expression, params...)`。这里的 `expression` 和 `params` 只是一段实现相关描述，具体如何解释由底层 `Repository` 实现决定。([GitHub][17])

### `includeAll()` 和 `excludeAll()` 分别是什么意思？

`includeAll()` 表示包含所有字段，`excludeAll()` 表示排除所有字段。`include("name")` 表示只包含指定字段，`exclude("password")` 表示排除指定字段。([GitHub][10])

[1]: https://raw.githubusercontent.com/scx-projects/scx-data/master/pom.xml "raw.githubusercontent.com"
[2]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/test/java/dev/scx/data/test/DataTest.java "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/Repository.java "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/Query.java "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/Finder.java "raw.githubusercontent.com"
[6]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/ConditionType.java "raw.githubusercontent.com"
[7]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/BuildControl.java "raw.githubusercontent.com"
[8]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/QueryImpl.java "raw.githubusercontent.com"
[9]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicy.java "raw.githubusercontent.com"
[10]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicyBuilder.java "raw.githubusercontent.com"
[11]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/field_policy/FieldPolicyImpl.java "raw.githubusercontent.com"
[12]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/aggregation/AggregationBuilder.java "raw.githubusercontent.com"
[13]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/AggregatableRepository.java "raw.githubusercontent.com"
[14]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/LockMode.java "raw.githubusercontent.com"
[15]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/exception/DataAccessException.java "raw.githubusercontent.com"
[16]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/test/java/dev/scx/data/test/TestRepository.java "raw.githubusercontent.com"
[17]: https://raw.githubusercontent.com/scx-projects/scx-data/master/src/main/java/dev/scx/data/query/QueryBuilder.java "raw.githubusercontent.com"
