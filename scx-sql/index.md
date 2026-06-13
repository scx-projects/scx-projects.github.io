# SCX SQL

SCX SQL 是一个轻量的 SQL / JDBC 工具库。

SCX SQL 本身不是 ORM，也不隐藏 SQL。它的核心定位是：让用户继续写清晰的 SQL，同时减少重复的 JDBC 样板代码。

它提供 `SQLClient`、`SQLRunner`、`SQL`、`BatchSQL`、`ResultSetExtractor`、`TypeSQLResolver`、`TypeSQLHandler`、`Dialect` 和 schema metadata 抽象，用来简化 JDBC 参数绑定、查询结果提取、事务复用、批量更新、生成键读取、类型读写和表结构描述。

当前版本为 `0.4.0`，依赖 `scx-transaction` 和 `scx-reflect`。

[GitHub](https://github.com/scx-projects/scx-sql)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-sql</artifactId>
    <version>0.4.0</version>
</dependency>
```

SCX SQL 不内置具体数据库驱动。实际使用时，需要根据数据库自行添加 JDBC Driver。`SQLClient` 可以接收已有 `DataSource`，也可以通过 `JDBCConnectionInfo` 配合 `Dialect#createDataSource(...)` 创建数据源。

## 基本概念

SCX SQL 中最常用的概念包括：

```text
SQL                  一条 SQL 语句和参数
BatchSQL             一条 SQL 语句和多组批量参数
SQLClient            面向 DataSource 的查询、更新、执行和事务入口
SQLRunner            面向 Connection 的静态执行工具
UpdateResult         更新结果，包含影响行数和生成键
ResultSetExtractor   ResultSet 提取器
TypeSQLResolver      Java 类型与 JDBC 读写处理器选择器
TypeSQLHandler       单个 Java 类型的 bind/read 处理器
Dialect              数据库方言
SchemaDialect        schema / DDL 相关方言接口
DatabaseMetadataReader JDBC metadata 读取工具
```

## 快速开始

假设你已经有一个 `DataSource` 和对应的 `Dialect`：

```java
import dev.scx.sql.SQLClient;
import dev.scx.sql.dialect.Dialect;

import javax.sql.DataSource;

DataSource dataSource = ...;
Dialect dialect = ...;

SQLClient sql = SQLClient.of(dataSource, dialect);
```

查询列表：

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofMapList;

var users = sql.query(
    sql("select id, name, age from user where age > ?", 18),
    ofMapList()
);
```

查询为对象：

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBeanList;

var users = sql.query(
    sql("select id, name, age from user where age > ?", 18),
    ofBeanList(User.class)
);
```

插入并读取生成键：

```java
import static dev.scx.sql.SQL.sql;

var result = sql.update(
    sql("insert into user(name, age) values (?, ?)", "Tom", 18)
);

Long id = result.firstGeneratedKey();
```

`SQL.sql(...)` 创建问号占位的 SQL 和参数；执行时会使用 `PreparedStatement` 填充参数并执行查询或更新。

## 创建 SQLClient

### 使用已有 DataSource

```java
SQLClient sql = SQLClient.of(dataSource, dialect);
```

使用默认 `TypeSQLResolver`：

```java
SQLClient sql = SQLClient.of(dataSource, dialect);
```

使用自定义 `TypeSQLResolver`：

```java
SQLClient sql = SQLClient.of(dataSource, resolver, dialect);
```

### 自动推断 Dialect

```java
SQLClient sql = SQLClient.of(dataSource);
```

自动推断会通过 `DialectSelector.findDialect(dataSource)` 从 `ServiceLoader` 加载到的 `Dialect` 列表中查找可处理当前 `DataSource` 的方言；找不到时会抛出异常。

当前 `scx-sql` 主仓库的 `src/main/resources` 目录只有 `.gitkeep`，因此具体方言通常需要由其他模块或用户项目通过 `ServiceLoader` 提供。

### 通过 JDBCConnectionInfo 创建

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

var connectionInfo = new JDBCConnectionInfo(
    "jdbc:xxx://localhost:3306/test",
    "root",
    "password"
);

SQLClient sql = SQLClient.of(connectionInfo);
```

`JDBCConnectionInfo` 只是保存 `url`、`username`、`password` 和可选参数；真正如何创建 `DataSource` 由 `Dialect#createDataSource(...)` 决定。

从 `JDBCConnectionInfo` 创建时，还可以传入一个或多个 `Function<DataSource, DataSource>` wrapper。`SQLClient` 会先根据 JDBC URL 推断 `Dialect`，再通过方言创建原始 `DataSource`，最后按传入顺序依次包装它。这个入口适合接入连接池、SQL spy 或其他 DataSource 代理。

```java
SQLClient sql = SQLClient.of(
    connectionInfo,
    dataSource -> {
        var h = new HikariDataSource();
        h.setDataSource(dataSource);
        return h;
    },
    dataSource -> ScxJdbcSpy.spy(dataSource, listener)
);
```

如果需要自定义类型处理器，可以使用带 `TypeSQLResolver` 的重载：

```java
SQLClient sql = SQLClient.of(connectionInfo, resolver, dataSource -> wrap(dataSource));
```

## SQL

`SQL` 表示一条 SQL 语句和它的参数。

```java
import static dev.scx.sql.SQL.sql;

var s = sql(
    "select id, name from user where id = ?",
    1L
);
```

接口内容很简单：

```java
String sql();

Object[] params();
```

SQL 使用普通 JDBC 问号占位符。参数会通过 `TypeSQLResolver` 找到对应的 `TypeSQLHandler`，然后绑定到 `PreparedStatement`。

## 查询数据

### 查询为 Map

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofMap;

var user = sql.query(
    sql("select id, name, age from user where id = ?", 1L),
    ofMap()
);
```

没有查询到数据时，`ofMap()` 返回 `null`。

### 查询为 Map 列表

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofMapList;

var users = sql.query(
    sql("select id, name, age from user"),
    ofMapList()
);
```

`ofMapList()` 会遍历整个 `ResultSet` 并返回 `List<Map<String, Object>>`。默认 `MapBuilder` 使用 `LinkedHashMap`，key 来自 `ResultSetMetaData#getColumnLabel(...)`，value 来自 `ResultSet#getObject(...)`。

### 自定义 Map key

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofMapList;

var users = sql.query(
    sql("select id, user_name, age from user"),
    ofMapList(columnLabel -> {
        if (columnLabel.equals("user_name")) {
            return "userName";
        }
        return columnLabel;
    })
);
```

`MapBuilder.of(Function<String, String>)` 支持把 column label 映射为自定义 map key；如果映射函数返回 `null`，会回退到原始 column label。

### 查询为 Bean

```java
public class User {
    public Long id;
    public String name;
    public Integer age;
}
```

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBean;

var user = sql.query(
    sql("select id, name, age from user where id = ?", 1L),
    ofBean(User.class)
);
```

普通 class 需要有 public 无参构造函数，并且只映射 public、非 static、非 final 的字段。没有出现在结果集中的字段会被跳过。

### 查询为 Bean 列表

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBeanList;

var users = sql.query(
    sql("select id, name, age from user"),
    ofBeanList(User.class)
);
```

`ofBeanList(...)` 会先根据 `ResultSetMetaData` 创建一次映射计划，然后逐行构建对象，避免每行重复解析列索引和类型处理器。

### 查询为 Record

```java
public record UserDTO(Long id, String name, Integer age) {
}
```

```java
var users = sql.query(
    sql("select id, name, age from user"),
    ofBeanList(UserDTO.class)
);
```

`BeanBuilder` 把普通 class 和 record 都视为 Bean。对于 record，会使用 record 构造函数参数顺序创建对象；如果某个 record 字段在结果集中不存在，会使用对应 `TypeSQLHandler#missingValue()` 作为参数值。

### 字段名和列标签不一致

```java
var users = sql.query(
    sql("select user_id, user_name from user"),
    ofBeanList(UserDTO.class, field -> {
        return switch (field.name()) {
            case "id" -> "user_id";
            case "name" -> "user_name";
            default -> field.name();
        };
    })
);
```

`ofBean(...)` 和 `ofBeanList(...)` 都支持 `FieldInfo -> columnLabel` 的映射函数。列标签匹配使用 `ResultSetMetaData#getColumnLabel(...)`，所以 SQL 中的别名也可以参与匹配。

也可以直接在 SQL 中使用别名：

```java
var users = sql.query(
    sql("select user_id as id, user_name as name from user"),
    ofBeanList(UserDTO.class)
);
```

### 查询单个值

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofSingleValue;

Long count = sql.query(
    sql("select count(*) from user"),
    ofSingleValue(Long.class)
);
```

指定列名：

```java
String name = sql.query(
    sql("select name from user where id = ?", 1L),
    ofSingleValue("name", String.class)
);
```

指定列索引：

```java
String name = sql.query(
    sql("select name from user where id = ?", 1L),
    ofSingleValue(1, String.class)
);
```

`SingleValueExtractor` 默认读取第 1 列。如果没有任何行，会抛出 `SQLException`。

## 流式消费结果

如果不想一次性收集所有结果，可以使用 consumer extractor。

### 消费 Map

```java
sql.query(
    sql("select id, name from user"),
    ResultSetExtractor.ofMapConsumer(row -> {
        System.out.println(row);
    })
);
```

### 消费 Bean

```java
sql.query(
    sql("select id, name, age from user"),
    ResultSetExtractor.ofBeanConsumer(User.class, user -> {
        System.out.println(user.name);
    })
);
```

`ResultSetExtractor` 的必须在 `extract(...)` 执行期间同步消费 `ResultSet`，不能在返回后继续持有或使用它。

## 更新数据

```java
import static dev.scx.sql.SQL.sql;

var result = sql.update(
    sql("update user set name = ? where id = ?", "Jerry", 1L)
);

System.out.println(result.affectedItemsCount());
```

`SQLRunner.update(...)` 使用 `PreparedStatement#executeLargeUpdate()` 获取影响行数，并读取 `PreparedStatement#getGeneratedKeys()` 得到生成键列表。

### 插入并读取生成键

```java
var result = sql.update(
    sql("insert into user(name, age) values (?, ?)", "Tom", 18)
);

System.out.println(result.affectedItemsCount());
System.out.println(result.generatedKeys());
System.out.println(result.firstGeneratedKey());
```

`UpdateResult` 包含：

```java
long affectedItemsCount();

List<Long> generatedKeys();

Long firstGeneratedKey();
```

当没有生成键时，`firstGeneratedKey()` 返回 `null`。

## 批量更新

`BatchSQL` 表示一条 SQL 和多组参数。

```java
import static dev.scx.sql.BatchSQL.batchSQL;

var result = sql.update(
    batchSQL(
        "insert into user(name, age) values (?, ?)",
        List.of(
            new Object[]{"Tom", 18},
            new Object[]{"Jerry", 20},
            new Object[]{"Alice", 22}
        )
    )
);
```

`SQLRunner.update(BatchSQL)` 会对每组参数调用 `addBatch()`，再执行 `executeLargeBatch()`，并累加每组更新数量。

## execute

`execute(...)` 用于执行不一定是标准查询或更新的 SQL：

```java
long updateCount = sql.execute(
    sql("create table user(id bigint, name varchar(128))")
);
```

`SQLRunner.execute(...)` 会执行 `PreparedStatement#execute()`，然后返回 `getLargeUpdateCount()`。

## 事务

`SQLClient` 继承自 `TransactionManager`，并提供事务相关能力。事务底层使用同一个 `Connection`，`SQLClientImpl` 通过 `ScopedValue` 在事务作用域中复用该连接；非事务场景下，每次操作会创建自动提交连接并在使用后关闭。

```java
SQLTransaction tx = sql.begin();

try {
    sql.with(tx, () -> {
        sql.update(sql("insert into user(name, age) values (?, ?)", "Tom", 18));
        sql.update(sql("insert into user(name, age) values (?, ?)", "Jerry", 20));
        return null;
    });

    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
} finally {
    tx.close();
}
```

`SQLTransaction` 持有当前事务的 `Connection`，提供 `commit()`、`rollback()` 和 `close()`；`SQLClientImpl#begin()` 创建 `autoCommit=false` 的连接。

如果把不属于当前 `SQLClient` 的事务传给 `with(...)`，会抛出 `IllegalArgumentException`。

## 类型读写

SCX SQL 使用 `TypeSQLResolver` 统一处理参数绑定和结果读取。

默认 resolver 注册了：

```text
byte / Byte
short / Short
int / Integer
long / Long
float / Float
double / Double
boolean / Boolean
String
BigInteger
BigDecimal
LocalDate
LocalTime
LocalDateTime
OffsetTime
OffsetDateTime
Instant
byte[]
Enum
```

这些默认处理器在 `TypeSQLResolver#registerDefaultHandlers(...)` 中注册。

### null 参数

当参数值为 `null` 时，`TypeSQLResolver#bind(...)` 会直接调用 `PreparedStatement#setNull(index, Types.NULL)`；非空值才会根据运行时类型选择 `TypeSQLHandler`。

```java
sql.update(
    sql("update user set nickname = ? where id = ?", null, 1L)
);
```

### 基本类型缺失值

`TypeSQLHandler#missingValue()` 用于处理“当前位置没有可用 SQL 值”的情况。默认返回 `null`；基本类型处理器通常会返回对应零值，例如 `int` 返回 `0`。

这个能力主要影响 record 映射：如果 record 构造参数对应的列不存在，就会使用该类型的 `missingValue()`。

### Enum

默认 enum 处理器写入时使用 `Enum#name()`，读取时从字符串恢复 enum；如果结果值无法转换为目标 enum，会抛出 `SQLDataException`。

```java
public enum Status {
    ACTIVE, DISABLED
}
```

```java
sql.update(
    sql("insert into user_status(status) values (?)", Status.ACTIVE)
);
```

### 时间类型

`LocalDateTimeSQLHandler` 和 `InstantSQLHandler` 会优先使用 `setObject` / `getObject`，如果驱动不支持，则降级为 `Timestamp`。

## 自定义类型处理器

如果默认类型处理器不满足需求，可以实现 `TypeSQLHandler`。

```java
import dev.scx.reflect.TypeInfo;
import dev.scx.sql.handler.TypeSQLHandler;

import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.UUID;

import static dev.scx.reflect.ScxReflect.typeOf;

public final class UUIDSQLHandler implements TypeSQLHandler<UUID> {

    @Override
    public TypeInfo valueType() {
        return typeOf(UUID.class);
    }

    @Override
    public void bind(PreparedStatement ps, int i, UUID value) throws SQLException {
        ps.setString(i, value.toString());
    }

    @Override
    public UUID read(ResultSet rs, int i) throws SQLException {
        var value = rs.getString(i);
        return value == null ? null : UUID.fromString(value);
    }

}
```

注册：

```java
var resolver = TypeSQLResolver
    .registerDefaultHandlers(TypeSQLResolver.builder())
    .registerHandler(new UUIDSQLHandler())
    .build();

SQLClient sql = SQLClient.of(dataSource, resolver, dialect);
```

`TypeSQLResolverBuilder` 支持注册固定 `TypeSQLHandler`，也支持注册 `TypeSQLHandlerFactory`。

### 自定义 HandlerFactory

`TypeSQLHandlerFactory` 可以按 `TypeInfo` 动态创建处理器，例如默认 enum 就是通过 `EnumSQLHandlerFactory` 创建的。

```java
public final class MyHandlerFactory implements TypeSQLHandlerFactory {

    @Override
    public TypeSQLHandler<?> createHandler(TypeInfo typeInfo) {
        // 可以根据 typeInfo 动态返回处理器
        return null;
    }

}
```

注册时可以指定顺序：

```java
var resolver = TypeSQLResolver
    .registerDefaultHandlers(TypeSQLResolver.builder())
    .registerHandlerFactory(new MyHandlerFactory(), 0)
    .build();
```

`TypeSQLHandlerSelectorImpl` 会缓存已经找到或动态创建的 handler；找不到时返回 `null`，而 `TypeSQLResolver#findHandler(...)` 会把找不到的情况转换为 `IllegalArgumentException`。

## Dialect

`Dialect` 表示数据库方言。它负责判断是否支持某个 JDBC URL 或 `DataSource`，也负责根据 `JDBCConnectionInfo` 创建 `DataSource`。

```java
public interface Dialect extends SchemaDialect {

    boolean canHandle(String url);

    boolean canHandle(DataSource dataSource);

    DataSource createDataSource(JDBCConnectionInfo connectionInfo);

}
```

### SyntaxDialect

`SyntaxDialect` 描述 SQL 语法层面的差异，例如标识符转义、分页、锁和布尔表达式。默认实现包括：

```java
quoteIdentifier(identifier)
applyLimit(sql, offset, limit)
applySharedLock(sql)
applyExclusiveLock(sql)
trueExpression()
falseExpression()
```

默认分页实现为 `LIMIT`，共享锁为 `FOR SHARE`，排他锁为 `FOR UPDATE`。

示例：

```java
String sqlText = dialect.applyLimit(
    "select * from user",
    0L,
    20L
);
```

### SchemaDialect

`SchemaDialect` 描述 schema / DDL 相关差异，例如数据类型名称映射、创建表、添加列、删除列、添加索引、删除索引等。

```java
List<String> ddls = dialect.getCreateTableDDLs(table);

for (var ddl : ddls) {
    sql.execute(SQL.sql(ddl));
}
```

`SchemaDialect` 的 DDL 方法返回 `List<String>`，因为同一个结构动作在不同数据库中可能需要一条或多条 DDL。例如同一个表结构在 MySQL 和 SQLite 下可能生成不同数量的 DDL。

## Schema 定义

SCX SQL 用 `Table`、`Column`、`DataType`、`Key`、`Index` 描述表结构。

```text
Table      表
Column     列
DataType   数据类型
Key        键，目前只支持单列
Index      索引，目前只支持单列
```

`Table` 包含 catalog、schema、表名、列、键、索引和注释，并提供 `getColumn(...)`、`getKey(...)`、`getIndex(...)` 便捷方法。

### 手动定义表结构

```java
import dev.scx.sql.schema.DataTypeKind;
import dev.scx.sql.schema.definition.*;

var table = new TableDefinition()
    .setName("user")
    .addColumn(new ColumnDefinition()
        .setName("id")
        .setDataType(new DataTypeDefinition()
            .setKind(DataTypeKind.BIGINT))
        .setNotNull(true)
        .setAutoIncrement(true))
    .addColumn(new ColumnDefinition()
        .setName("name")
        .setDataType(new DataTypeDefinition()
            .setKind(DataTypeKind.VARCHAR)
            .setLength(128))
        .setNotNull(true))
    .addKey(new KeyDefinition()
        .setName("pk_user")
        .setColumnName("id")
        .setPrimary(true))
    .addIndex(new IndexDefinition()
        .setName("idx_user_name")
        .setColumnName("name")
        .setUnique(false));
```

`TableDefinition`、`ColumnDefinition`、`DataTypeDefinition`、`KeyDefinition` 和 `IndexDefinition` 都是用于手动编写 schema 对象的可变实现。

### 数据类型种类

`DataTypeKind` 提供了标准化的数据类型枚举，包括整数、浮点数、布尔、精确数字、日期时间、文本、BLOB 和 JSON。它还提供 `ofJavaType(Class<?>)`，可以从常见 Java 类型推断数据类型种类。

```java
var kind = DataTypeKind.ofJavaType(String.class);
// VARCHAR
```

## 读取数据库 metadata

`DatabaseMetadataReader` 基于 JDBC `DatabaseMetaData` 提供 catalog、schema、table、column、primary key 和 index metadata 读取能力。

```java
try (var connection = dataSource.getConnection()) {
    var metaData = connection.getMetaData();

    var tables = DatabaseMetadataReader.readTableMetadataList(
        metaData,
        connection.getCatalog(),
        connection.getSchema(),
        null
    );
}
```

加载完整表结构：

```java
try (var connection = dataSource.getConnection()) {
    var table = DatabaseMetadataReader.loadCurrentTable(
        connection,
        "user",
        dialect
    );
}
```

`loadTable(...)` 会读取表基本信息、列、主键和索引，并组装成 `TableDefinition`；如果找不到表，返回 `null`；如果找到多个同名表，会抛出 `IllegalStateException`。

## 直接使用 SQLRunner

如果你已经有一个 `Connection`，可以不创建 `SQLClient`，直接使用 `SQLRunner`。

```java
try (var connection = dataSource.getConnection()) {
    var users = SQLRunner.query(
        connection,
        SQL.sql("select id, name from user"),
        ResultSetExtractor.ofMapList()
    );
}
```

更新：

```java
try (var connection = dataSource.getConnection()) {
    var result = SQLRunner.update(
        connection,
        SQL.sql("insert into user(name) values (?)", "Tom")
    );
}
```

`SQLRunner` 提供 query、update、batch update、execute 的静态方法，并且都有使用默认 `TypeSQLResolver` 的重载。

## 完整示例

```java
import dev.scx.sql.SQL;
import dev.scx.sql.SQLClient;
import dev.scx.sql.SQLTransaction;
import dev.scx.sql.dialect.Dialect;

import javax.sql.DataSource;
import java.util.List;

import static dev.scx.sql.BatchSQL.batchSQL;
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.*;

public class UserRepository {

    private final SQLClient sql;

    public UserRepository(DataSource dataSource, Dialect dialect) {
        this.sql = SQLClient.of(dataSource, dialect);
    }

    public Long add(String name, int age) throws Exception {
        var result = sql.update(
            sql("insert into user(name, age) values (?, ?)", name, age)
        );
        return result.firstGeneratedKey();
    }

    public User getById(Long id) throws Exception {
        return sql.query(
            sql("select id, name, age from user where id = ?", id),
            ofBean(User.class)
        );
    }

    public List<User> list() throws Exception {
        return sql.query(
            sql("select id, name, age from user order by id"),
            ofBeanList(User.class)
        );
    }

    public long updateName(Long id, String name) throws Exception {
        return sql.update(
            sql("update user set name = ? where id = ?", name, id)
        ).affectedItemsCount();
    }

    public long batchAdd(List<User> users) throws Exception {
        var params = users.stream()
            .map(u -> new Object[]{u.name, u.age})
            .toList();

        return sql.update(
            batchSQL("insert into user(name, age) values (?, ?)", params)
        ).affectedItemsCount();
    }

    public void addTwoUsersInTransaction() throws Exception {
        SQLTransaction tx = sql.begin();

        try {
            sql.with(tx, () -> {
                add("Tom", 18);
                add("Jerry", 20);
                return null;
            });

            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        } finally {
            tx.close();
        }
    }

    public static class User {
        public Long id;
        public String name;
        public Integer age;
    }

}
```

## 设计说明

### 1. SCX SQL 不隐藏 SQL

SCX SQL 的基本输入仍然是 SQL 字符串和参数。它不是 ORM，也不负责自动推导查询语义。`SQL` 只保存问号占位 SQL 和参数数组。

### 2. 类型处理围绕 typed bind / typed read

`TypeSQLHandler` 同时负责 `PreparedStatement` 参数绑定和 `ResultSet` 列读取。需要注意，SQL/JDBC 面对的是 typed bind / typed read、二进制、时间、typed null、列语义等 operation-oriented 问题，而不是一个统一值对象问题。

### 3. TypeSQLHandler 和 Dialect 分层独立

可以这样区分：Dialect 负责“SQL 怎么说”，`TypeSQLHandler` 负责“值怎么过桥”。因此，方言描述 SQL 语言差异，类型处理器描述 Java 值如何绑定和读取，两者不应该互相收编。

### 4. ResultSet 必须同步消费

`ResultSetExtractor` 的 `extract(...)` 必须在执行期间同步消费 `ResultSet`，不要在方法返回后继续持有 `ResultSet`。

### 5. Bean 映射依赖 public 字段

当前 Bean 映射不是完整 JavaBean getter/setter 映射。普通 class 映射 public、非 static、非 final 字段，并要求 public 无参构造函数；record 则通过 record 构造函数创建。

## 常见问题

### SCX SQL 会自动生成业务 SQL 吗？

不会。SCX SQL 主要封装 JDBC 执行、参数绑定、结果提取、事务复用、类型处理和 schema/dialect 工具。业务查询 SQL 仍然由用户编写。`SQL` 接口本身只包含 SQL 字符串和参数。

### `SQLClient.of(dataSource)` 找不到方言怎么办？

`SQLClient.of(dataSource)` 会通过 `DialectSelector` 从 `ServiceLoader` 加载到的 `Dialect` 中查找可处理该 `DataSource` 的实现。找不到时会抛出 `IllegalArgumentException`。可以改用 `SQLClient.of(dataSource, dialect)` 手动传入方言。

### Bean 查询为什么字段没有赋值？

普通 class 只映射 public、非 static、非 final 字段；列匹配使用 `ResultSetMetaData#getColumnLabel(...)`。如果字段名和列标签不一致，可以使用 SQL 别名，或者传入 `FieldInfo -> columnLabel` 映射函数。

### `ofMap()` 和 `ofBean()` 没查到数据会怎样？

`ofMap()` 和 `ofBean()` 在没有行时返回 `null`；`ofMapList()` 和 `ofBeanList()` 返回列表；`ofSingleValue(...)` 在没有行时抛出 `SQLException`。

### 如何支持自定义类型？

实现 `TypeSQLHandler<T>`，然后通过 `TypeSQLResolverBuilder#registerHandler(...)` 注册；如果需要按类型动态创建处理器，可以实现 `TypeSQLHandlerFactory`。
