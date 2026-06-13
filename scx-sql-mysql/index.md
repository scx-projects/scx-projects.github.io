# SCX SQL MySQL

SCX SQL MySQL 是 `scx-sql` 的 MySQL 方言模块。

它提供 `MySQLDialect`，用于让 `scx-sql` 能够识别 MySQL JDBC URL、识别 MySQL DataSource、创建 MySQL DataSource、处理 MySQL 标识符转义、完成 MySQL 类型名映射，并生成 MySQL 风格的表结构 DDL。

SCX SQL MySQL 本身不是 ORM，也不是新的 SQL 执行框架。真正的 SQL 执行、参数绑定、结果集提取、事务复用等能力来自 `scx-sql`；本模块只负责 MySQL 相关的方言差异。

当前版本为 `0.5.0`。

[GitHub](https://github.com/scx-projects/scx-sql-mysql)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-sql-mysql</artifactId>
    <version>0.5.0</version>
</dependency>
```

`scx-sql-mysql` 已经依赖：

```text
scx-sql
mysql-connector-j
```

因此在普通使用场景中，只需要引入 `scx-sql-mysql`，就可以同时获得 `scx-sql` 的核心能力和 MySQL JDBC Driver。

## 基本概念

SCX SQL MySQL 中最核心的概念包括：

```text
MySQLDialect          MySQL 方言实现
MySQLDialectHelper    MySQL 类型名和 SCX 标准数据类型之间的映射工具
ServiceLoader         自动注册 MySQLDialect
MysqlDataSource       MySQL Connector/J 提供的数据源实现
```

它们之间的关系可以简单理解为：

```text
scx-sql             提供 SQLClient / SQLRunner / SQL / BatchSQL / Schema 抽象
scx-sql-mysql       提供 MySQLDialect
mysql-connector-j   提供 MysqlDataSource 和 MySQL JDBC Driver
```

也就是说：

```text
SCX SQL 负责执行 SQL
SCX SQL MySQL 负责告诉 SCX SQL：MySQL 应该怎么处理
```

## 快速开始

最简单的方式是通过 `JDBCConnectionInfo` 创建 `SQLClient`。

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

var connectionInfo = new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root"
);

var sqlClient = SQLClient.of(connectionInfo);
```

然后就可以正常使用 `scx-sql` 的查询、更新和事务能力：

```java
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofMapList;

var users = sqlClient.query(
    sql("select id, name, age from user where age > ?", 18),
    ofMapList()
);
```

插入数据：

```java
import static dev.scx.sql.SQL.sql;

var result = sqlClient.update(
    sql("insert into user(name, age) values (?, ?)", "Tom", 18)
);

System.out.println(result.affectedItemsCount());
System.out.println(result.firstGeneratedKey());
```

执行 DDL：

```java
import static dev.scx.sql.SQL.sql;

sqlClient.execute(sql("""
    create table if not exists user
    (
        id bigint primary key auto_increment,
        name varchar(128) not null,
        age int
    )
    """));
```

## 使用已有 MysqlDataSource

也可以手动创建 `MysqlDataSource`，然后交给 `SQLClient`。

```java
import com.mysql.cj.jdbc.MysqlDataSource;
import dev.scx.sql.SQLClient;

var dataSource = new MysqlDataSource();

dataSource.setServerName("127.0.0.1");
dataSource.setPort(3306);
dataSource.setDatabaseName("test");
dataSource.setUser("root");
dataSource.setPassword("root");

var sqlClient = SQLClient.of(dataSource);
```

这里的 `SQLClient.of(dataSource)` 会通过 `ServiceLoader` 加载 `Dialect`，然后找到能够处理当前 `DataSource` 的方言。

因为 `scx-sql-mysql` 已经注册了 `MySQLDialect`，所以只要依赖中包含本模块，`MysqlDataSource` 就可以被自动识别。

## 手动指定 MySQLDialect

如果不想依赖自动推断，也可以手动指定方言。

```java
import com.mysql.cj.jdbc.MysqlDataSource;
import dev.scx.sql.SQLClient;
import dev.scx.sql.mysql.MySQLDialect;

var dataSource = new MysqlDataSource();

dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/test");
dataSource.setUser("root");
dataSource.setPassword("root");

var sqlClient = SQLClient.of(dataSource, new MySQLDialect());
```

这种方式不会通过 `DialectSelector` 查找方言，适合下面这些场景：

1. 当前 `DataSource` 被外层代理包装后，无法被自动识别。
2. 项目中同时存在多个方言模块，希望明确指定。
3. 测试时希望直接控制使用哪个方言。
4. 自动推断失败，但你明确知道这是 MySQL。

## JDBCConnectionInfo 参数

`JDBCConnectionInfo` 支持在 `url`、`username`、`password` 后面追加参数。

```java
var connectionInfo = new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "allowMultiQueries=true",
    "rewriteBatchedStatements=true",
    "createDatabaseIfNotExist=true"
);

var sqlClient = SQLClient.of(connectionInfo);
```

`MySQLDialect#createDataSource(...)` 会创建一个 `MysqlDataSource`，然后：

1. 调用 `setUrl(...)` 设置 JDBC URL。
2. 调用 `setUser(...)` 设置用户名。
3. 调用 `setPassword(...)` 设置密码。
4. 遍历 `parameters`。
5. 将每个参数按第一个 `=` 拆成 key 和 value。
6. 通过 MySQL Connector/J 的属性定义解析 value。
7. 设置到 `MysqlDataSource` 对应属性上。

例如：

```text
allowMultiQueries=true
rewriteBatchedStatements=true
createDatabaseIfNotExist=true
```

这些参数名应使用 MySQL Connector/J 支持的属性名。

如果某个参数不是 `key=value` 格式，会被忽略。

## MySQLDialect

`MySQLDialect` 是本模块的核心类。

它实现了 `Dialect` 接口，因此同时具备三类能力：

```text
Dialect         判断 URL / DataSource，创建 DataSource
SyntaxDialect   处理 SQL 语法差异
SchemaDialect   处理数据类型和 DDL 差异
```

接口层面可以理解为：

```java
public final class MySQLDialect implements Dialect {

    @Override
    public boolean canHandle(String url) {
        ...
    }

    @Override
    public boolean canHandle(DataSource dataSource) {
        ...
    }

    @Override
    public DataSource createDataSource(JDBCConnectionInfo connectionInfo) {
        ...
    }

    @Override
    public String quoteIdentifier(String identifier) {
        ...
    }

    @Override
    public DataTypeKind dialectTypeNameToDataTypeKind(String dialectTypeName) {
        ...
    }

    @Override
    public String dataTypeKindToDialectTypeName(DataTypeKind dataTypeKind) {
        ...
    }

    @Override
    public List<String> getCreateTableDDLs(Table table) {
        ...
    }

    @Override
    public List<String> getAddColumnDDLs(Table table, Column column) {
        ...
    }

    @Override
    public List<String> getDropColumnDDLs(Table table, Column column) {
        ...
    }

    @Override
    public List<String> getAddIndexDDLs(Table table, Index index) {
        ...
    }

    @Override
    public List<String> getDropIndexDDLs(Table table, Index index) {
        ...
    }

}
```

## 自动识别 JDBC URL

`MySQLDialect#canHandle(String url)` 用于判断某个 JDBC URL 是否属于 MySQL。

```java
var dialect = new MySQLDialect();

boolean b1 = dialect.canHandle("jdbc:mysql://127.0.0.1:3306/test");
boolean b2 = dialect.canHandle("jdbc:postgresql://127.0.0.1:5432/test");
```

内部判断由 MySQL Connector/J 的 `NonRegisteringDriver#acceptsURL(...)` 完成。

因此是否能识别，取决于 MySQL Connector/J 对 URL 的判断规则。

## 自动识别 DataSource

`MySQLDialect#canHandle(DataSource dataSource)` 用于判断某个 `DataSource` 是否属于 MySQL。

当前判断逻辑是：

```text
dataSource instanceof MysqlDataSource
或者
dataSource.isWrapperFor(MysqlDataSource.class)
```

也就是说，下面这种可以被识别：

```java
var dataSource = new MysqlDataSource();

var sqlClient = SQLClient.of(dataSource);
```

如果 `DataSource` 被连接池、监控组件或代理组件包装，只要它正确实现了 `isWrapperFor(MysqlDataSource.class)`，也可以被识别。

如果无法识别，可以手动指定方言：

```java
var sqlClient = SQLClient.of(dataSource, new MySQLDialect());
```

## ServiceLoader 自动注册

本模块提供了 ServiceLoader 配置：

```text
META-INF/services/dev.scx.sql.dialect.Dialect
```

内容为：

```text
dev.scx.sql.mysql.MySQLDialect
```

因此当项目中引入 `scx-sql-mysql` 后，`scx-sql` 的 `DialectSelector` 就可以自动加载到 `MySQLDialect`。

常见入口包括：

```java
SQLClient.of(dataSource);
```

以及：

```java
SQLClient.of(connectionInfo);
```

前者通过 `DataSource` 查找方言。

后者先通过 JDBC URL 查找方言，再由方言创建对应的 `DataSource`。

## 标识符转义

MySQL 使用反引号转义表名、字段名、索引名等标识符。

```java
var dialect = new MySQLDialect();

String name = dialect.quoteIdentifier("user");
```

结果为：

```sql
`user`
```

在生成 DDL 时，表名、列名、索引名都会使用这种方式转义。

例如：

```sql
CREATE TABLE `user`
(
    `id` BIGINT NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(128) NOT NULL,
    PRIMARY KEY (`id`)
);
```

如果表结构中带有 schema，则会生成：

```sql
`schema_name`.`table_name`
```

需要注意，当前实现使用的是 `Table#schema()` 作为 MySQL 中的库名前缀，不会使用 `catalog` 参与表名拼接。

## 数据类型映射

SCX SQL 使用 `DataTypeKind` 表示标准化的数据类型。

SCX SQL MySQL 负责在 `DataTypeKind` 和 MySQL 类型名之间转换。

### 标准类型转 MySQL 类型

当前映射关系如下：

```text
DataTypeKind.TINYINT    -> TINYINT
DataTypeKind.SMALLINT   -> SMALLINT
DataTypeKind.INT        -> INT
DataTypeKind.BIGINT     -> BIGINT
DataTypeKind.FLOAT      -> FLOAT
DataTypeKind.DOUBLE     -> DOUBLE
DataTypeKind.BOOLEAN    -> BOOLEAN
DataTypeKind.DECIMAL    -> DECIMAL
DataTypeKind.DATE       -> DATE
DataTypeKind.TIME       -> TIME
DataTypeKind.DATETIME   -> DATETIME
DataTypeKind.VARCHAR    -> VARCHAR
DataTypeKind.TEXT       -> TEXT
DataTypeKind.LONGTEXT   -> LONGTEXT
DataTypeKind.BLOB       -> BLOB
DataTypeKind.LONGBLOB   -> LONGBLOB
DataTypeKind.JSON       -> JSON
```

示例：

```java
var dialect = new MySQLDialect();

String typeName = dialect.dataTypeKindToDialectTypeName(DataTypeKind.VARCHAR);
```

结果为：

```text
VARCHAR
```

### MySQL 类型转标准类型

从数据库 metadata 读取到的 MySQL 类型名，会通过 `dialectTypeNameToDataTypeKind(...)` 归一化为 `DataTypeKind`。

当前支持的映射包括：

```text
TINYINT             -> TINYINT
TINYINT UNSIGNED    -> TINYINT

SMALLINT            -> SMALLINT
SMALLINT UNSIGNED   -> SMALLINT

MEDIUMINT UNSIGNED  -> INT
INT                 -> INT
INT UNSIGNED        -> INT

BIGINT              -> BIGINT
BIGINT UNSIGNED     -> BIGINT

FLOAT               -> FLOAT
DOUBLE              -> DOUBLE

DECIMAL             -> DECIMAL
DECIMAL UNSIGNED    -> DECIMAL

BIT                 -> BOOLEAN

TIME                -> TIME
DATE                -> DATE
DATETIME            -> DATETIME
TIMESTAMP           -> DATETIME

VARCHAR             -> VARCHAR
CHAR                -> VARCHAR

TEXT                -> TEXT
MEDIUMTEXT          -> LONGTEXT
LONGTEXT            -> LONGTEXT

BLOB                -> BLOB
MEDIUMBLOB          -> LONGBLOB
LONGBLOB            -> LONGBLOB
BINARY              -> BLOB
VARBINARY           -> BLOB

JSON                -> JSON

ENUM                -> VARCHAR
SET                 -> VARCHAR
```

类型名匹配是大小写不敏感的。

示例：

```java
var dialect = new MySQLDialect();

var kind1 = dialect.dialectTypeNameToDataTypeKind("varchar");
var kind2 = dialect.dialectTypeNameToDataTypeKind("TIMESTAMP");
var kind3 = dialect.dialectTypeNameToDataTypeKind("json");
```

结果分别为：

```text
VARCHAR
DATETIME
JSON
```

如果传入的类型名不在当前映射表中，会抛出 `IllegalArgumentException`。

```java
dialect.dialectTypeNameToDataTypeKind("GEOMETRY");
```

这类 MySQL 特有类型如果需要支持，应在方言映射中补充对应关系，或者在上层自行处理。

## 生成建表 DDL

`MySQLDialect#getCreateTableDDLs(table)` 用于根据 `Table` 结构生成 MySQL 建表语句。

它返回 `List<String>`，但 MySQL 当前实现通常只返回一条 `CREATE TABLE`。

示例：

```java
import dev.scx.sql.mysql.MySQLDialect;
import dev.scx.sql.schema.DataTypeKind;
import dev.scx.sql.schema.definition.ColumnDefinition;
import dev.scx.sql.schema.definition.DataTypeDefinition;
import dev.scx.sql.schema.definition.IndexDefinition;
import dev.scx.sql.schema.definition.KeyDefinition;
import dev.scx.sql.schema.definition.TableDefinition;

var table = new TableDefinition()
    .setSchema("test")
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
    .addColumn(new ColumnDefinition()
        .setName("age")
        .setDataType(new DataTypeDefinition()
            .setKind(DataTypeKind.INT)))
    .addKey(new KeyDefinition()
        .setName("pk_user")
        .setColumnName("id")
        .setPrimary(true))
    .addIndex(new IndexDefinition()
        .setName("idx_user_name")
        .setColumnName("name")
        .setUnique(false));

var dialect = new MySQLDialect();

var ddls = dialect.getCreateTableDDLs(table);
```

生成结果类似：

```sql
CREATE TABLE `test`.`user`
(
 `id` BIGINT NOT NULL AUTO_INCREMENT,
 `name` VARCHAR(128) NOT NULL,
 `age` INT,
 PRIMARY KEY (`id`),
 KEY `idx_user_name` (`name`)
);
```

需要注意：

1. 表名和列名会使用反引号转义。
2. 如果设置了 `schema`，表名会生成为 `` `schema`.`table` ``。
3. `VARCHAR` 会保留 `length`。
4. 除 `VARCHAR` 之外，当前实现会忽略 `length`。
5. 主键会生成为表级约束。
6. 普通索引和唯一索引会生成为表级约束。
7. 如果某个索引列已经是主键列，建表时会跳过该索引，避免重复表达。

## 列定义

列定义由三部分组成：

```text
列名
数据类型
列约束
```

例如：

```sql
`name` VARCHAR(128) NOT NULL
```

当前支持的列约束包括：

```text
NOT NULL
AUTO_INCREMENT
DEFAULT xxx
ON UPDATE xxx
```

示例：

```java
var column = new ColumnDefinition()
    .setName("created_at")
    .setDataType(new DataTypeDefinition()
        .setKind(DataTypeKind.DATETIME))
    .setNotNull(true)
    .setDefaultValue("CURRENT_TIMESTAMP")
    .setOnUpdate("CURRENT_TIMESTAMP");
```

生成列定义类似：

```sql
`created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

`defaultValue` 和 `onUpdate` 是直接拼接到 DDL 中的 SQL 片段。

因此调用者应传入数据库可接受的表达式，例如：

```text
0
'unknown'
CURRENT_TIMESTAMP
```

而不是 Java 对象本身。

## 添加列

`getAddColumnDDLs(table, column)` 用于生成添加列的 DDL。

```java
var ddls = dialect.getAddColumnDDLs(table, column);
```

生成结果类似：

```sql
ALTER TABLE `user` ADD COLUMN `age` INT;
```

如果表结构带有 schema：

```sql
ALTER TABLE `test`.`user` ADD COLUMN `age` INT;
```

## 删除列

`getDropColumnDDLs(table, column)` 用于生成删除列的 DDL。

```java
var ddls = dialect.getDropColumnDDLs(table, column);
```

生成结果类似：

```sql
ALTER TABLE `user` DROP COLUMN `age`;
```

## 添加索引

`getAddIndexDDLs(table, index)` 用于生成添加索引的 DDL。

普通索引：

```sql
CREATE INDEX `idx_user_name` ON `user` (`name`);
```

唯一索引：

```sql
CREATE UNIQUE INDEX `uk_user_name` ON `user` (`name`);
```

示例：

```java
var index = new IndexDefinition()
    .setName("idx_user_name")
    .setColumnName("name")
    .setUnique(false);

var ddls = dialect.getAddIndexDDLs(table, index);
```

## 删除索引

`getDropIndexDDLs(table, index)` 用于生成删除索引的 DDL。

```sql
DROP INDEX `idx_user_name` ON `user`;
```

MySQL 删除索引时需要指定表名，因此本模块会生成 `DROP INDEX ... ON ...`。

## 创建表时的索引处理

建表时，MySQL 方言会把索引直接写入 `CREATE TABLE` 语句内部。

普通索引：

```sql
KEY `idx_user_name` (`name`)
```

唯一索引：

```sql
UNIQUE KEY `uk_user_name` (`name`)
```

主键：

```sql
PRIMARY KEY (`id`)
```

如果某个索引的列已经作为主键列出现，建表时会跳过这个索引。

例如：

```java
.addKey(new KeyDefinition()
    .setName("pk_user")
    .setColumnName("id")
    .setPrimary(true))
.addIndex(new IndexDefinition()
    .setName("idx_user_id")
    .setColumnName("id"))
```

最终只会生成：

```sql
PRIMARY KEY (`id`)
```

不会再额外生成：

```sql
KEY `idx_user_id` (`id`)
```

## 与 SQLClient 配合使用

`scx-sql-mysql` 最常见的使用方式不是直接调用 `MySQLDialect`，而是配合 `SQLClient` 使用。

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBeanList;

var sqlClient = SQLClient.of(new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "createDatabaseIfNotExist=true"
));

sqlClient.execute(sql("""
    create table if not exists user
    (
        id bigint primary key auto_increment,
        name varchar(128) not null,
        age int
    )
    """));

sqlClient.update(
    sql("insert into user(name, age) values (?, ?)", "Tom", 18)
);

var users = sqlClient.query(
    sql("select id, name, age from user"),
    ofBeanList(User.class)
);
```

Bean 示例：

```java
public class User {

    public Long id;

    public String name;

    public Integer age;

}
```

## 批量插入

MySQL Connector/J 支持通过 `rewriteBatchedStatements=true` 优化批量写入。

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

import java.util.List;

import static dev.scx.sql.BatchSQL.batchSQL;

var sqlClient = SQLClient.of(new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "rewriteBatchedStatements=true"
));

var result = sqlClient.update(
    batchSQL(
        "insert into user(name, age) values (?, ?)",
        List.of(
            new Object[]{"Tom", 18},
            new Object[]{"Jerry", 20},
            new Object[]{"Alice", 22}
        )
    )
);

System.out.println(result.affectedItemsCount());
```

`rewriteBatchedStatements=true` 是 MySQL Connector/J 的参数，本模块只是把它设置到 `MysqlDataSource` 上。

## 多语句执行

如果需要一次执行多个 SQL 语句，可以设置 `allowMultiQueries=true`。

```java
var sqlClient = SQLClient.of(new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "allowMultiQueries=true"
));

sqlClient.execute(sql("""
    drop table if exists user;
    create table user
    (
        id bigint primary key auto_increment,
        name varchar(128) not null
    )
    """));
```

没有开启 `allowMultiQueries=true` 时，MySQL Connector/J 通常不会允许一条 JDBC 语句中包含多个 SQL。

## 事务

事务能力来自 `scx-sql` 的 `SQLClient`，不是 `scx-sql-mysql` 单独实现的能力。

但当 `SQLClient` 使用 MySQL DataSource 和 MySQL 方言时，事务底层使用的就是 MySQL JDBC Connection。

自动事务：

```java
import static dev.scx.sql.SQL.sql;

sqlClient.autoTransaction(() -> {
    sqlClient.update(
        sql("insert into user(name, age) values (?, ?)", "Tom", 18)
    );

    sqlClient.update(
        sql("insert into user(name, age) values (?, ?)", "Jerry", 20)
    );
});
```

如果第二条语句抛出异常，`autoTransaction(...)` 会回滚整个事务。

手动事务：

```java
import static dev.scx.sql.SQL.sql;

var tx = sqlClient.begin();

try {
    sqlClient.with(tx, () -> {
        sqlClient.update(
            sql("insert into user(name, age) values (?, ?)", "Tom", 18)
        );

        sqlClient.update(
            sql("insert into user(name, age) values (?, ?)", "Jerry", 20)
        );
    });

    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
} finally {
    tx.close();
}
```

这里需要注意：

1. `begin()` 开始事务。
2. `with(tx, handler)` 让 handler 内的 SQL 复用事务连接。
3. `commit()` 显式提交。
4. `rollback()` 显式回滚。
5. `close()` 负责最终收尾。

## 读取数据库 Metadata

`scx-sql` 提供 `DatabaseMetadataReader`，可以读取 JDBC metadata 并组装成 `Table`。

`scx-sql-mysql` 提供的 `MySQLDialect` 可以配合它把 MySQL metadata 中的类型名转换为标准 `DataTypeKind`，再重新生成 MySQL DDL。

示例：

```java
import dev.scx.sql.metadata.DatabaseMetadataReader;
import dev.scx.sql.mysql.MySQLDialect;

var dialect = new MySQLDialect();

try (var connection = dataSource.getConnection()) {
    var metaData = connection.getMetaData();

    var table = DatabaseMetadataReader.loadCurrentTable(
        connection,
        "user",
        dialect
    );

    var ddls = dialect.getCreateTableDDLs(table);

    for (var ddl : ddls) {
        System.out.println(ddl);
    }
}
```

这类能力适合用于：

1. 查看数据库当前表结构。
2. 把 JDBC metadata 转为 SCX SQL 的 schema 对象。
3. 根据 schema 对象重新生成 DDL。
4. 做简单的结构对比或迁移辅助。

需要注意，metadata 读取本身来自 JDBC，具体返回哪些 catalog、schema、table、column、index 信息，仍然取决于 MySQL Connector/J 和数据库配置。

## 完整示例

下面是一个完整的 MySQL 使用示例。

```java
import dev.scx.sql.JDBCConnectionInfo;
import dev.scx.sql.SQLClient;

import java.util.List;

import static dev.scx.sql.BatchSQL.batchSQL;
import static dev.scx.sql.SQL.sql;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBean;
import static dev.scx.sql.extractor.ResultSetExtractor.ofBeanList;

public class UserRepository {

    private final SQLClient sqlClient;

    public UserRepository() {
        this.sqlClient = SQLClient.of(new JDBCConnectionInfo(
            "jdbc:mysql://127.0.0.1:3306/test",
            "root",
            "root",
            "createDatabaseIfNotExist=true",
            "rewriteBatchedStatements=true"
        ));
    }

    public void init() throws Exception {
        sqlClient.execute(sql("""
            create table if not exists user
            (
                id bigint primary key auto_increment,
                name varchar(128) not null,
                age int
            )
            """));
    }

    public Long add(String name, Integer age) throws Exception {
        var result = sqlClient.update(
            sql("insert into user(name, age) values (?, ?)", name, age)
        );

        return result.firstGeneratedKey();
    }

    public User getById(Long id) throws Exception {
        return sqlClient.query(
            sql("select id, name, age from user where id = ?", id),
            ofBean(User.class)
        );
    }

    public List<User> list() throws Exception {
        return sqlClient.query(
            sql("select id, name, age from user order by id"),
            ofBeanList(User.class)
        );
    }

    public long batchAdd(List<User> users) throws Exception {
        var params = users.stream()
            .map(user -> new Object[]{user.name, user.age})
            .toList();

        return sqlClient.update(
            batchSQL("insert into user(name, age) values (?, ?)", params)
        ).affectedItemsCount();
    }

    public void addTwoUsersInTransaction() throws Exception {
        sqlClient.autoTransaction(() -> {
            add("Tom", 18);
            add("Jerry", 20);
        });
    }

    public static class User {

        public Long id;

        public String name;

        public Integer age;

    }

}
```

## 设计说明

### 1. SCX SQL MySQL 只负责 MySQL 方言

`scx-sql-mysql` 不负责重新实现 `SQLClient`。

它只是告诉 `scx-sql`：

1. 哪些 JDBC URL 是 MySQL。
2. 哪些 DataSource 是 MySQL。
3. 如何创建 `MysqlDataSource`。
4. MySQL 如何转义标识符。
5. MySQL 类型名如何映射到标准数据类型。
6. 标准数据类型如何映射到 MySQL 类型名。
7. MySQL 的建表、改表、索引 DDL 应该怎么生成。

### 2. 依赖 ServiceLoader 自动接入

本模块通过 `META-INF/services/dev.scx.sql.dialect.Dialect` 注册 `MySQLDialect`。

所以使用者通常不需要手动注册方言。

只要依赖中存在 `scx-sql-mysql`，下面这种写法就可以自动找到 MySQL 方言：

```java
var sqlClient = SQLClient.of(dataSource);
```

或者：

```java
var sqlClient = SQLClient.of(connectionInfo);
```

### 3. URL 判断交给 MySQL Driver

`canHandle(String url)` 没有自己手写 URL 规则，而是交给 MySQL Connector/J 判断。

这样可以减少本模块和 MySQL JDBC URL 规则之间的不一致。

### 4. DataSource 判断优先识别 MysqlDataSource

`canHandle(DataSource dataSource)` 当前只识别：

```text
MysqlDataSource
或者可以 unwrap 为 MysqlDataSource 的 DataSource
```

这意味着某些第三方连接池可能无法被自动识别。

遇到这种情况时，应手动传入 `new MySQLDialect()`。

```java
SQLClient.of(dataSource, new MySQLDialect());
```

### 5. DDL 生成是结构描述，不是完整迁移系统

`getCreateTableDDLs(...)`、`getAddColumnDDLs(...)`、`getDropColumnDDLs(...)`、`getAddIndexDDLs(...)` 和 `getDropIndexDDLs(...)` 只是根据 `Table`、`Column`、`Index` 等结构对象生成对应 DDL。

它不负责：

1. 自动比较两个数据库结构。
2. 自动判断应该添加还是删除。
3. 自动处理数据迁移。
4. 自动处理复杂索引。
5. 自动处理外键。
6. 自动处理分区表。
7. 自动处理存储引擎、字符集、排序规则等 MySQL 专有表选项。

这些能力如果需要，应在更上层实现。

### 6. 当前 schema 模型偏向单列主键和单列索引

`scx-sql` 当前的 `Key` 和 `Index` 模型以 `columnName` 表达列名，因此更适合单列主键和单列索引。

如果需要组合主键、组合唯一索引或组合普通索引，需要扩展 schema 抽象和方言实现。

### 7. VARCHAR 会处理 length，其它类型暂不处理 length

当前 MySQL DDL 生成中，只有 `VARCHAR` 会读取并生成 length。

例如：

```java
new DataTypeDefinition()
    .setKind(DataTypeKind.VARCHAR)
    .setLength(128)
```

会生成：

```sql
VARCHAR(128)
```

其它类型目前直接使用类型名，不拼接 length。

例如：

```text
INT
BIGINT
DECIMAL
DATETIME
TEXT
BLOB
JSON
```

如果需要 `DECIMAL(10,2)`、`INT(11)` 这类更细粒度类型定义，需要扩展数据类型模型或方言生成逻辑。

## 常见问题

### SCX SQL MySQL 是 ORM 吗？

不是。

它只是 `scx-sql` 的 MySQL 方言模块。业务 SQL 仍然由用户编写。

### 引入 scx-sql-mysql 后，还需要单独引入 mysql-connector-j 吗？

通常不需要。

`scx-sql-mysql` 已经依赖 `mysql-connector-j`。

如果你需要指定不同版本的 MySQL Connector/J，可以在项目中自行覆盖依赖版本。

### `SQLClient.of(dataSource)` 找不到 MySQL 方言怎么办？

可以改成手动指定方言：

```java
var sqlClient = SQLClient.of(dataSource, new MySQLDialect());
```

常见原因是当前 `DataSource` 不是 `MysqlDataSource`，并且也不能通过 `isWrapperFor(MysqlDataSource.class)` 识别为 MySQL 数据源。

### `SQLClient.of(connectionInfo)` 是怎么创建 DataSource 的？

它会先根据 JDBC URL 查找 `Dialect`。

找到 `MySQLDialect` 后，会调用：

```java
dialect.createDataSource(connectionInfo)
```

然后创建 `MysqlDataSource`，并设置 URL、用户名、密码和可选参数。

### 参数应该写在 JDBC URL 里，还是写在 JDBCConnectionInfo 的 parameters 里？

两种方式都可以。

写在 URL 中：

```java
new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test?rewriteBatchedStatements=true",
    "root",
    "root"
)
```

写在 parameters 中：

```java
new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "rewriteBatchedStatements=true"
)
```

后者会通过 `MysqlDataSource` 的属性系统设置参数。

### 为什么执行多条 SQL 失败？

如果一条 SQL 字符串中包含多个语句，需要开启 MySQL Connector/J 的 `allowMultiQueries=true`。

```java
new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "allowMultiQueries=true"
)
```

### 为什么批量插入没有明显变快？

MySQL Connector/J 通常需要开启 `rewriteBatchedStatements=true` 才能更好地优化批量写入。

```java
new JDBCConnectionInfo(
    "jdbc:mysql://127.0.0.1:3306/test",
    "root",
    "root",
    "rewriteBatchedStatements=true"
)
```

### MySQL 的 `TIMESTAMP` 会映射成什么？

当前会映射为 `DataTypeKind.DATETIME`。

### MySQL 的 `ENUM` 和 `SET` 会映射成什么？

当前会映射为 `DataTypeKind.VARCHAR`。

### MySQL 的 `MEDIUMTEXT` 会映射成什么？

当前会映射为 `DataTypeKind.LONGTEXT`。

### MySQL 的 `MEDIUMBLOB` 会映射成什么？

当前会映射为 `DataTypeKind.LONGBLOB`。

### 遇到未知 MySQL 类型怎么办？

`dialectTypeNameToDataTypeKind(...)` 遇到未知类型会抛出 `IllegalArgumentException`。

如果你的数据库使用了当前映射表没有覆盖的类型，需要扩展 `MySQLDialectHelper` 的映射关系，或者在 metadata 读取后自行处理。

### 建表 DDL 会生成外键吗？

当前不会。

当前实现主要处理列、主键、普通索引和唯一索引。

### 建表 DDL 会生成字符集、排序规则或存储引擎吗？

当前不会。

例如下面这些 MySQL 表选项当前不会自动生成：

```sql
ENGINE=InnoDB
DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_0900_ai_ci
```

如果需要这些能力，应扩展 MySQL 方言的 DDL 生成逻辑。

### 删除索引为什么要带表名？

这是 MySQL 的语法要求。

因此当前生成的是：

```sql
DROP INDEX `idx_name` ON `table_name`;
```

而不是：

```sql
DROP INDEX `idx_name`;
```
