# SCX Data SQL

SCX Data SQL 是 `scx-data` 和 `scx-sql` 之间的桥接库。

`scx-data` 提供底层无关的 `Repository`、`Query`、`FieldPolicy`、`Aggregation` 抽象；

`scx-sql` 提供 SQL / JDBC 执行能力；而 `scx-data-sql` 负责把 `scx-data` 的查询描述、字段策略和聚合描述转换成 SQL，并通过 `SQLClient` 执行。

当前版本为 `0.5.0`。

它适合这种场景：

```text
你希望业务层使用 scx-data 的 Repository API，
但底层实际数据源是 SQL 数据库。
```

[GitHub](https://github.com/scx-projects/scx-data-sql)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-data-sql</artifactId>
    <version>0.5.0</version>
</dependency>
```

`scx-data-sql` 本身依赖 `scx-data` 和 `scx-sql`；如果你要连接具体数据库，还需要引入对应的 `scx-sql-*` 方言模块和 JDBC 驱动。MySQL 场景通常会搭配 `scx-sql-mysql`；如果需要观察 SQL，可以再搭配 `scx-jdbc-spy` 和 `scx-logging`。

例如 MySQL 场景通常还需要：

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-sql-mysql</artifactId>
    <version>0.4.0</version>
</dependency>
```

## 基本概念

SCX Data SQL 中最常用的概念包括：

```text
SQLRepository          SQL 版 Repository 实现
SQLFinder              SQL 查询结果读取器
SQLAggregator          SQL 聚合查询读取器
EntityTable            实体表映射
EntityColumn           实体字段和 SQL 列的映射
@Table                 声明实体对应的表名
@Column                声明字段对应的列信息
@NoColumn              排除字段，不映射为列
SQLClient              来自 scx-sql 的 SQL 执行入口
```

其中最核心的是 `SQLRepository`。它实现了 `AggregatableRepository` 和 `LockableRepository`，所以可以直接使用 `scx-data` 的普通查询、聚合查询和锁查询 API。

## 快速开始

### 1. 定义实体

```java
import dev.scx.data.sql.annotation.Column;
import dev.scx.data.sql.annotation.Table;

@Table("user")
public class User {

    @Column(primary = true, autoIncrement = true)
    public Long id;

    public String name;

    public Integer age;

    public String status;

}
```

`@Table` 用来声明表名；`@Column` 可以声明列名、主键、自增、默认值、索引等信息；普通 public 字段会自动映射为列。`AnnotationConfigTable` 要求实体类必须标注 `@Table`。

### 2. 创建 SQLClient

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

SQLClient sqlClient = SQLClient.of(
    new JDBCConnectionInfo(
        "jdbc:mysql://127.0.0.1:3306/demo",
        "root",
        "root",
        "createDatabaseIfNotExist=true"
    )
);
```

典型写法是通过 `JDBCConnectionInfo` 创建 `SQLClient`，然后把它交给 `SQLRepository` 使用。

### 3. 创建 SQLRepository

```java
import dev.scx.data.sql.SQLRepository;

SQLRepository<User> userRepository = new SQLRepository<>(User.class, sqlClient);
```

`SQLRepository(Class, SQLClient)` 会根据实体类上的注解创建 `AnnotationConfigTable`，并初始化查询结果映射器、SQL 构造器和字段 / 列映射关系。

### 4. 使用 Repository API

```java
import static dev.scx.data.query.QueryBuilder.*;
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;

var user = new User();
user.name = "Tom";
user.age = 18;
user.status = "ACTIVE";

Long id = userRepository.add(user);

User saved = userRepository.findFirst(eq("id", id));

var users = userRepository.find(
    eq("status", "ACTIVE")
        .desc("id")
        .limit(20)
);

var patch = new User();
patch.name = "Jerry";

long updated = userRepository.update(
    patch,
    include("name"),
    eq("id", id)
);

long deleted = userRepository.delete(eq("id", id));
```

常见用法包括 `add`、`find`、`finder().listMap()`、`update`、`aggregate` 等用法。

## 创建表结构

`scx-data-sql` 可以把实体映射为 `Table` metadata，然后交给 `scx-sql` 的方言生成建表 DDL。

```java
import dev.scx.sql.SQL;

SQLRepository<User> userRepository = new SQLRepository<>(User.class, sqlClient);

var ddls = sqlClient.dialect().getCreateTableDDLs(userRepository.table());

for (var ddl : ddls) {
    sqlClient.execute(SQL.sql(ddl));
}
```

一个常见流程是先 `drop table if exists car`，再通过 `sqlClient.dialect().getCreateTableDDLs(carRepository.table())` 生成建表语句并执行。

## 实体映射

### `@Table`

```java
@Table("car")
public class Car {
}
```

`@Table` 标注在类上，用来指定表名。没有 `@Table` 时，`AnnotationConfigTable` 会抛出 `IllegalArgumentException`。

### 普通字段映射

```java
@Table("user")
public class User {

    public Long id;

    public String name;

    public Integer age;

}
```

普通 class 只映射 public 字段；record 会取所有字段。带有 `@NoColumn` 的字段会被排除，不参与列映射。

### `@Column`

```java
@Table("user")
public class User {

    @Column(primary = true, autoIncrement = true)
    public Long id;

    @Column(columnName = "user_name", notNull = true)
    public String name;

    @Column(defaultValue = "0")
    public Long money;

}
```

`@Column` 支持：

```java
String[] columnName() default {};
DataType[] dataType() default {};
String[] defaultValue() default {};
String[] onUpdate() default {};
boolean notNull() default false;
boolean autoIncrement() default false;
boolean primary() default false;
boolean unique() default false;
boolean index() default false;
```

如果没有显式指定 `columnName`，默认使用字段名作为列名。`dataType`、`defaultValue`、`onUpdate`、`notNull`、`autoIncrement`、`primary`、`unique` 和 `index` 主要用于创建或修复表结构。

### `@NoColumn`

```java
@Table("user")
public class User {

    public Long id;

    public String name;

    @NoColumn
    public String temporaryValue;

}
```

标注 `@NoColumn` 的字段不会映射为数据库列。

### 指定数据类型

```java
import dev.scx.data.sql.annotation.DataType;

import static dev.scx.sql.schema.DataTypeKind.VARCHAR;

@Table("user")
public class User {

    @Column(dataType = @DataType(value = VARCHAR, length = 256))
    public String name;

}
```

`@DataType` 包含 `DataTypeKind value()` 和 `int length() default -1`。如果没有显式指定数据类型，SCX Data SQL 会根据 Java 字段类型推断；枚举默认映射为 `VARCHAR`，无法识别的类型默认映射为 `JSON`。

`VARCHAR` 的默认长度是 `128`。

## 字段名和列名映射

实体字段名和 SQL 列名可以不同：

```java
@Table("car")
public class Car {

    @Column(primary = true, autoIncrement = true)
    public Long id;

    public String name;

    @Column(columnName = "sIzE")
    public Integer size;

    @Column(columnName = "CITY")
    public String city;

}
```

示例里的 `Car` 就展示了字段 `size` 映射到列 `sIzE`，字段 `city` 映射到列 `CITY`。

查询时，业务代码仍然使用实体字段名：

```java
var cars = carRepository.find(
    eq("city", "Tokyo")
);
```

SCX Data SQL 会通过 `EntityTable#getColumn(...)` 找到对应列，并使用方言的 `quoteIdentifier(...)` 包裹列名。`SQLColumnNameParser` 在普通字段模式下会查表字段映射，找不到对应列时抛出异常。

Map 查询结果也会从列名映射回实体字段名。`MapKeyMapping` 的语义是“列名 -> 字段名”，`finder().listMap()` 返回的 key 是实体字段名，而不是数据库列名。

## 添加数据

### 添加单条数据

```java
var car = new Car();
car.name = "BMW";
car.size = 3;
car.city = "Tokyo";

Long id = carRepository.add(car);
```

`SQLRepository#add(entity, fieldPolicy)` 会构造 `INSERT` SQL，通过 `SQLClient#update(...)` 执行，并返回第一条生成键。

### 指定字段添加

```java
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;

Long id = carRepository.add(
    car,
    include("name", "size", "city")
);
```

### 排除字段添加

```java
Long id = carRepository.add(
    car,
    exclude("id")
);
```

批量插入自增主键实体时，常见做法是使用 `exclude("id")`，让自增主键由数据库生成。

### 使用字段表达式

```java
Long id = carRepository.add(
    car,
    exclude("city")
        .assignField("color", "<color-expression>")
);
```

字段表达式会直接写入 SQL 的 `VALUES` 部分，不会作为参数绑定。`InsertSQLBuilder` 会把普通字段生成 `?`，把 `assignField` 的表达式作为插入值拼接进去。

也可以使用 `assignField("color", "...")` 来插入表达式生成的颜色值。

### 批量添加

```java
List<Car> cars = List.of(car1, car2, car3);

List<Long> ids = carRepository.add(
    cars,
    exclude("id")
);
```

`SQLRepository#add(Collection, FieldPolicy)` 会构造 `BatchSQL`，执行批量插入，并返回生成键列表。

## 查询数据

### 查询全部

```java
List<Car> cars = carRepository.find();
```

### 条件查询

```java
import static dev.scx.data.query.QueryBuilder.*;

Car car = carRepository.findFirst(eq("id", 1L));
```

```java
List<Car> cars = carRepository.find(
    eq("name", "BMW")
        .desc("id")
        .limit(20)
);
```

`SQLFinder` 会调用 `repository.buildSelectSQL(...)` 构造查询 SQL，然后通过 `SQLClient` 和 `ResultSetExtractor` 读取实体、Map 或指定类型。

### 查询指定字段

```java
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;

List<Car> cars = carRepository.find(
    eq("name", "BMW"),
    include("id", "name", "size")
);
```

排除字段：

```java
List<Car> cars = carRepository.find(
    eq("name", "BMW"),
    exclude("city")
);
```

`SelectSQLBuilder` 会根据 `FieldPolicy` 过滤查询列，如果最终查询列为空，会抛出异常。

### 查询为 Map

```java
List<Map<String, Object>> rows = carRepository
    .finder(eq("name", "BMW"))
    .listMap();
```

返回的 Map key 会映射为实体字段名，而不是数据库列名。这个行为由 `MapKeyMapping` 提供，这个行为在查询 Map 结果时同样适用。

### 查询为其他类型

```java
List<CarDTO> cars = carRepository
    .finder(eq("name", "BMW"))
    .list(CarDTO.class);
```

```java
CarDTO car = carRepository
    .finder(eq("id", 1L))
    .first(CarDTO.class);
```

`SQLFinder#list(Class<T>)` 和 `first(Class<T>)` 会使用 `ofBeanList(resultType, fieldColumnLabelMapping)` 或 `ofBean(resultType, fieldColumnLabelMapping)` 来读取结果。

### 遍历查询结果

```java
carRepository.finder().forEach(car -> {
    System.out.println(car.name);
});
```

```java
carRepository.finder().forEachMap(row -> {
    System.out.println(row);
});
```

`SQLFinder` 支持 `forEach`、`forEach(resultType)` 和 `forEachMap`，用户回调抛出的异常会被包装为 `ScxWrappedException`。

## 条件转换规则

SCX Data SQL 会把 `scx-data` 的条件对象转换成 SQL WHERE 子句。

常见转换包括：

```text
eq      -> =
ne      -> <>
lt      -> <
lte     -> <=
gt      -> >
gte     -> >=
like    -> LIKE
notLike -> NOT LIKE
in      -> IN
notIn   -> NOT IN
between -> BETWEEN
```

这些映射由 `SQLWhereParser#getWhereKeyWord(...)` 定义。

### null 比较

```java
eq("city", null)
ne("city", null)
```

会被转换成：

```sql
city IS NULL
city IS NOT NULL
```

`SQLWhereParser#parseEQ(...)` 对 `EQ` / `NE` 的 `null` 值做了专门处理。

### LIKE

```java
like("name", "BM")
```

会被转换为包含匹配：

```sql
name LIKE CONCAT('%', ?, '%')
```

`LIKE` / `NOT_LIKE` 会使用 `CONCAT('%', ? ,'%')` 包裹参数。

### IN / NOT IN

```java
in("status", List.of("ACTIVE", "LOCKED"))
```

`IN` 支持数组、集合和基本类型数组。`SQLParserHelper#toObjectArray(...)` 会把集合、对象数组和基本类型数组转换成 `Object[]`。

空数组会被特殊处理：

```text
IN 空数组      -> falseExpression()
NOT_IN 空数组  -> trueExpression()
```

这个行为由 `SQLWhereParser#parseIN(...)` 定义。

如果 `IN` / `NOT_IN` 的值中包含 `null`，会额外拼接 `IS NULL` 或 `IS NOT NULL` 条件。

### SQL 子查询

条件值可以是 `dev.scx.sql.SQL`：

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.data.query.QueryBuilder.*;

var cars = carRepository.find(
    in("id", sql("select car_id from car_tag where tag = ?", "hot"))
);
```

`SQLWhereParser` 对条件值为 `SQL` 的情况做了专门处理，会把子 SQL 放到括号中，并合并其参数。

### 自定义 where 片段

```java
import static dev.scx.data.query.QueryBuilder.*;

var cars = carRepository.find(
    whereClause("name is not null")
);
```

`WhereClause` 和 `SQL` 片段会被括号包裹，避免和其他条件组合时产生歧义。

### 表达式字段和值

```java
import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;

var cars = carRepository.find(
    eq("LOWER(name)", "LOWER(?)", USE_EXPRESSION, USE_EXPRESSION_VALUE)
);
```

当使用 `USE_EXPRESSION` 时，字段选择器会被当作表达式处理，并包裹为 `(expression)`；当使用 `USE_EXPRESSION_VALUE` 时，值也会作为表达式拼接，而不是作为参数绑定。

## 排序和分页

```java
var cars = carRepository.find(
    eq("name", "BMW")
        .asc("size")
        .desc("id")
        .offset(20)
        .limit(20)
);
```

排序字段会通过字段 / 列映射转换成 SQL 列名，再追加 `ASC` 或 `DESC`。分页由 `Dialect#applyLimit(...)` 处理，因此不同数据库的分页语法由方言决定。

## 更新数据

### 更新指定字段

```java
var patch = new Car();
patch.name = "Audi";

long updated = carRepository.update(
    patch,
    include("name"),
    eq("id", 1L)
);
```

`SQLRepository#update(...)` 会构造 `UPDATE` SQL 并返回影响行数。

### 忽略 null 更新

```java
var patch = new Car();
patch.name = "Audi";
patch.city = null;

long updated = carRepository.update(
    patch,
    includeAll().ignoreNull(true),
    eq("id", 1L)
);
```

字段过滤由 `SQLBuilderHelper#filterByUpdateFieldPolicy(...)` 完成，它会根据字段策略、`assignField` 和 `ignoreNull` 过滤参与插入 / 更新的列。

### 使用字段表达式更新

```java
long updated = carRepository.update(
    assignField("size", "size + 1"),
    eq("id", 1L)
);
```

`UpdateSQLBuilder` 会把普通字段构造成 `column = ?`，把 `assignField` 构造成 `column = expression`。

### 更新必须带条件

```java
carRepository.update(
    patch,
    include("name"),
    eq("id", 1L)
);
```

`UpdateSQLBuilder` 要求更新必须指定 where 条件；如果 `query.getWhere()` 为 `null`，会抛出异常，避免误更新整张表。

## 删除数据

```java
long deleted = carRepository.delete(eq("id", 1L));
```

```java
long deleted = carRepository.delete(
    eq("name", "BMW")
        .limit(10)
);
```

`DeleteSQLBuilder` 会构造 `DELETE FROM ... WHERE ...`，并且要求必须存在 where 条件；没有 where 条件时会抛出异常，避免误删整张表。删除时的 `limit` 会交给方言处理，并且不会使用 offset。

## 清空表

```java
carRepository.clear();
```

`SQLRepository#clear()` 当前执行的是：

```sql
truncate <tableName>
```

表名来自实体映射。

## 统计数量

```java
long count = carRepository.count(eq("name", "BMW"));
```

`CountSQLBuilder` 生成的 SQL 形如：

```sql
SELECT COUNT(*) AS count FROM <table> WHERE ...
```

并通过列名 `count` 读取 `Long` 结果。

## 虚拟字段

虚拟字段用于查询时追加表达式列：

```java
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;

var rows = carRepository.finder(
    virtualField("displayName", "CONCAT(name, '-', color)")
).listMap();
```

`SelectSQLBuilder` 会把虚拟字段转换成：

```sql
expression AS quotedFieldName
```

虚拟字段名不会再做表字段映射，因为它可能并不是表中的真实列。

例如：

```java
virtualField("name", "REVERSE(name)")
```

来验证虚拟字段查询。

## 聚合查询

`SQLRepository` 实现了 `AggregatableRepository`，因此可以直接使用 `scx-data` 的聚合 API。

```java
import static dev.scx.data.aggregation.AggregationBuilder.*;
import static dev.scx.data.query.QueryBuilder.*;

var rows = carRepository.aggregate(
    lt("id", 100),
    groupBy("name")
        .agg("totalSize", "SUM(size)"),
    eq("name", "BMW")
);
```

参数含义：

```text
第一个 Query       聚合前过滤，转换成 WHERE
Aggregation       分组和聚合列，转换成 SELECT / GROUP BY
第三个 Query       聚合后过滤、排序、分页，转换成 HAVING / ORDER BY / LIMIT
```

`AggregateSQLBuilder` 中，`beforeAggregateQuery` 转换为 `WHERE`，`afterAggregateQuery.getWhere()` 转换为 `HAVING`，`afterAggregateQuery.getOrderBys()` 转换为 `ORDER BY`。

常见用法包括：

```java
carRepository.aggregate(
    lt("id", 3),
    groupBy("name").agg("totalSize", "SUM(size)"),
    eq("name", "奔驰")
);
```

以及：

```java
carRepository.aggregateFirst(
    agg("totalSize", "SUM(size)")
);
```

### 聚合返回 Map

```java
var rows = carRepository.aggregate(
    lt("id", 100),
    groupBy("name").agg("totalSize", "SUM(size)"),
    desc("totalSize").limit(10)
);
```

默认聚合结果是 `List<Map<String, Object>>`。`SQLAggregator#list()` 使用 `ofMapList(repository.mapBuilder)` 读取结果。

### 聚合返回指定类型

```java
List<CarStat> stats = carRepository
    .aggregator(
        lt("id", 100),
        groupBy("name").agg("totalSize", "SUM(size)"),
        desc("totalSize").limit(10)
    )
    .list(CarStat.class);
```

`SQLAggregator#list(Class<T>)` 会使用 `ofBeanList(resultType, repository.fieldColumnLabelMapping)` 读取指定类型。

## 锁查询

`SQLRepository` 实现了 `LockableRepository`，所以可以使用 `LockMode`。

```java
import dev.scx.data.LockMode;

var user = userRepository.findFirst(
    eq("id", 1L),
    LockMode.EXCLUSIVE
);
```

`SelectSQLBuilder` 会根据锁模式调用方言方法：

```text
SHARED     -> dialect.applySharedLock(sql)
EXCLUSIVE  -> dialect.applyExclusiveLock(sql)
```

转账这类并发敏感场景可以在事务中使用 `LockMode.EXCLUSIVE` 查询转出和转入用户，避免并发修改余额。

```java
sqlClient.autoTransaction(() -> {
    var fromUser = userRepository.findFirst(eq("id", fromUserId), LockMode.EXCLUSIVE);
    var toUser = userRepository.findFirst(eq("id", toUserId), LockMode.EXCLUSIVE);

    fromUser.money -= amount;
    toUser.money += amount;

    userRepository.update(fromUser, include("money"), eq("id", fromUserId));
    userRepository.update(toUser, include("money"), eq("id", toUserId));
});
```

## 事务

事务由 `scx-sql` 的 `SQLClient` 提供。`scx-data-sql` 的 `SQLRepository` 内部所有读写都走同一个 `SQLClient`，因此可以自然参与 `SQLClient` 的事务上下文。

```java
sqlClient.autoTransaction(() -> {
    var user = userRepository.findFirst(eq("id", 1L), LockMode.EXCLUSIVE);

    user.money += 100L;

    userRepository.update(
        user,
        include("money"),
        eq("id", user.id)
    );
});
```

并发转账这类流程可以通过 `sqlClient.autoTransaction(...)` 包裹两次锁查询和两次余额更新。

## 查看生成的 SQL

`SQLRepository` 暴露了一组 `build*SQL` 方法，可以直接查看最终生成的 `SQL` 或 `BatchSQL`。

```java
var sql = carRepository.buildSelectSQL(
    eq("name", "BMW").limit(10),
    include("id", "name")
);

System.out.println(sql.sql());
System.out.println(Arrays.toString(sql.params()));
```

可用方法包括：

```java
buildInsertSQL(entity, fieldPolicy)
buildInsertBatchSQL(entityList, fieldPolicy)
buildSelectSQL(query, fieldPolicy)
buildSelectFirstSQL(query, fieldPolicy)
buildUpdateSQL(entity, fieldPolicy, query)
buildDeleteSQL(query)
buildCountSQL(query)
buildAggregateSQL(beforeAggregateQuery, aggregation, afterAggregateQuery)
buildAggregateFirstSQL(beforeAggregateQuery, aggregation, afterAggregateQuery)
```

这些方法都由 `SQLRepository` 公开。

## 使用自定义 EntityTable

除了直接使用注解映射，也可以自己实现 `EntityTable`，然后传给 `SQLRepository`：

```java
EntityTable table = ...;

SQLRepository<User> repo = new SQLRepository<>(table, sqlClient);
```

`SQLRepository` 提供两个构造方法：一个接收实体类并创建 `AnnotationConfigTable`，另一个直接接收 `EntityTable`。

`EntityTable` 继承自 `scx-sql` 的 `Table`，并额外提供 `entityClass()`、`columns()` 和 `getColumn(...)`；`EntityColumn` 继承自 `Column`，并额外提供 `javaField()`。

## 完整示例

```java
import dev.scx.data.LockMode;
import dev.scx.data.sql.SQLRepository;
import dev.scx.data.sql.annotation.Column;
import dev.scx.data.sql.annotation.Table;
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQL;
import dev.scx.sql.SQLClient;

import java.sql.SQLException;
import java.util.List;

import static dev.scx.data.aggregation.AggregationBuilder.*;
import static dev.scx.data.field_policy.FieldPolicyBuilder.*;
import static dev.scx.data.query.QueryBuilder.*;

public class DataSqlExample {

    public static void main(String[] args) throws SQLException {
        SQLClient sqlClient = SQLClient.of(
            new JDBCConnectionInfo(
                "jdbc:mysql://127.0.0.1:3306/demo",
                "root",
                "root",
                "createDatabaseIfNotExist=true"
            )
        );

        SQLRepository<User> userRepository = new SQLRepository<>(User.class, sqlClient);

        sqlClient.execute(SQL.sql("drop table if exists user"));

        for (var ddl : sqlClient.dialect().getCreateTableDDLs(userRepository.table())) {
            sqlClient.execute(SQL.sql(ddl));
        }

        var user = new User();
        user.name = "Alice";
        user.money = 1000L;
        user.status = "ACTIVE";

        Long id = userRepository.add(user);

        User saved = userRepository.findFirst(eq("id", id));

        List<User> activeUsers = userRepository.find(
            eq("status", "ACTIVE")
                .desc("id")
                .limit(20)
        );

        var patch = new User();
        patch.money = 1200L;

        userRepository.update(
            patch,
            include("money"),
            eq("id", id)
        );

        var rows = userRepository.aggregate(
            eq("status", "ACTIVE"),
            groupBy("status").agg("totalMoney", "SUM(money)"),
            desc("totalMoney")
        );

        sqlClient.autoTransaction(() -> {
            var locked = userRepository.findFirst(eq("id", id), LockMode.EXCLUSIVE);
            locked.money += 100L;

            userRepository.update(
                locked,
                include("money"),
                eq("id", id)
            );
        });

        userRepository.delete(eq("id", id));
    }

    @Table("user")
    public static class User {

        @Column(primary = true, autoIncrement = true)
        public Long id;

        public String name;

        @Column(defaultValue = "0")
        public Long money;

        public String status;

    }

}
```

## 设计说明

### 1. SCX Data SQL 是 SQL 版 Repository

`scx-data` 本身是底层无关的；`scx-data-sql` 则是明确面向 SQL 数据库的实现。它把 `Query`、`FieldPolicy`、`Aggregation` 转换为 SQL，并通过 `SQLClient` 执行。`SQLRepository` 内部持有 `InsertSQLBuilder`、`SelectSQLBuilder`、`UpdateSQLBuilder`、`DeleteSQLBuilder`、`CountSQLBuilder` 和 `AggregateSQLBuilder`。

### 2. 业务层仍然使用 scx-data API

用户大多数时候不需要直接写 SQL，可以使用：

```java
repo.find(eq("name", "Tom").limit(20));
repo.update(patch, include("name"), eq("id", 1L));
repo.delete(eq("id", 1L));
```

这些都是 `scx-data` 的 API，只是底层由 SCX Data SQL 翻译成 SQL 执行。

### 3. 字段策略会影响 SQL 列

`include` / `exclude` 决定普通列是否参与查询、插入和更新；`assignField` 会变成插入或更新表达式；`virtualField` 会变成查询表达式列。相关过滤逻辑集中在 `SQLBuilderHelper`。

### 4. 危险操作有保护

更新要求必须指定 where 条件；删除也要求必须指定 where 条件。否则会抛出异常，避免误更新或误删除整张表。

### 5. 方言负责 SQL 差异

表名 / 列名转义、分页、共享锁、排他锁、DDL 生成等 SQL 差异由 `scx-sql` 的 `Dialect` 处理。SCX Data SQL 在生成查询时会调用 `dialect.quoteIdentifier(...)`、`dialect.applyLimit(...)`、`dialect.applySharedLock(...)` 和 `dialect.applyExclusiveLock(...)`。

## 常见问题

### SCX Data SQL 和 SCX Data 是什么关系？

`scx-data` 定义底层无关的 Repository 和查询 DSL；`scx-data-sql` 是它的 SQL 实现。`SQLRepository` 实现了 `AggregatableRepository` 和 `LockableRepository`，所以可以直接使用 `scx-data` 的查询、聚合和锁查询 API。

### SCX Data SQL 和 SCX SQL 是什么关系？

`scx-sql` 负责 JDBC 执行、参数绑定、结果提取、事务和方言；`scx-data-sql` 负责把 `scx-data` 的描述对象翻译成 `scx-sql` 的 `SQL` / `BatchSQL`，然后交给 `SQLClient` 执行。`SQLRepository` 内部所有操作都通过 `sqlClient.query(...)`、`sqlClient.update(...)` 或 `sqlClient.execute(...)` 完成。

### 普通 class 和 record 都支持吗？

支持，但字段选择规则不同。普通 class 只取 public 字段；record 取所有字段；标注 `@NoColumn` 的字段会被排除。

### 字段名和列名不一致时，查询里写哪个？

写实体字段名。`SQLColumnNameParser` 会通过 `EntityTable#getColumn(...)` 找到真实列名，再由方言转义。

### Map 查询返回的 key 是字段名还是列名？

是实体字段名。`MapKeyMapping` 把列名映射回字段名，这个行为适用于查询 Map 结果。

### 可以直接看生成的 SQL 吗？

可以。`SQLRepository` 暴露了 `buildSelectSQL`、`buildInsertSQL`、`buildUpdateSQL`、`buildDeleteSQL`、`buildAggregateSQL` 等方法。

### 为什么删除必须带条件？

`DeleteSQLBuilder` 明确要求 where 条件不能为空，否则抛出异常，避免误删整张表。

### 为什么更新必须带条件？

`UpdateSQLBuilder` 在构造更新 SQL 时检查 `query.getWhere()`，没有 where 条件会抛出异常，避免误更新整张表。
