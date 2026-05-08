# SCX JDBC Spy

SCX JDBC Spy 是一个轻量 JDBC 监听和 SQL 日志工具库。

它通过包装 JDBC 对象，在不改变原始 JDBC 调用方式的前提下，拦截 `DataSource`、`Connection`、`Statement`、`PreparedStatement` 和 `CallableStatement` 的关键操作，并把这些操作通知给对应的 listener。

SCX JDBC Spy 本身不是 JDBC Driver，也不是连接池，也不是 ORM。它不会自己创建数据库连接，不会解析 JDBC URL，也不会执行 SQL。它只是包装已有 JDBC 对象，并在执行 SQL、设置参数、添加 batch、清空 batch 等关键位置插入监听逻辑。

当前版本为 `0.3.0`。

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-jdbc-spy</artifactId>
    <version>0.3.0</version>
</dependency>
```

## 基本概念

SCX JDBC Spy 中最核心的概念包括：

```text
ScxJdbcSpy                  包装入口
SpyDataSource               DataSource 包装器
SpyConnection               Connection 包装器
SpyStatement                Statement 包装器
SpyPreparedStatement        PreparedStatement 包装器
SpyCallableStatement        CallableStatement 包装器
SpyWrapper                  Wrapper / unwrap / isWrapperFor 基础实现

DataSourceListener          DataSource 监听器
ConnectionListener          Connection 监听器
StatementListener           Statement 监听器
PreparedStatementListener   PreparedStatement 监听器
CallableStatementListener   CallableStatement 监听器

LoggingDataSourceListener          默认日志型 DataSource listener
LoggingConnectionListener          默认日志型 Connection listener
LoggingStatementListener           默认日志型 Statement listener
LoggingPreparedStatementListener   默认日志型 PreparedStatement listener
LoggingCallableStatementListener   默认日志型 CallableStatement listener
PreparedStatementLogStyle          PreparedStatement 日志风格
```

它们之间的关系可以简单理解为：

```text
原始 DataSource
    ↓
ScxJdbcSpy.spy(dataSource, listener)
    ↓
SpyDataSource
    ↓ getConnection()
SpyConnection
    ↓ createStatement() / prepareStatement() / prepareCall()
SpyStatement / SpyPreparedStatement / SpyCallableStatement
    ↓ execute / executeQuery / executeUpdate / addBatch / executeBatch
listener 回调
```

也就是说：

```text
包装器负责拦截 JDBC 调用
listener 负责处理拦截事件
logging listener 负责把 SQL 打到 System.Logger
```

## 快速开始

最常见的用法是包装已有 `DataSource`。

```java
import dev.scx.jdbc.spy.ScxJdbcSpy;
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.SQL_AND_PARAMETERS;

var spyDataSource = ScxJdbcSpy.spy(
    dataSource,
    new LoggingDataSourceListener(SQL_AND_PARAMETERS)
);
```

之后像普通 `DataSource` 一样使用：

```java
try (var connection = spyDataSource.getConnection();
     var ps = connection.prepareStatement("select * from user where id = ?")) {

    ps.setLong(1, 100L);

    try (var rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }
    }

}
```

如果当前 logger 的 `DEBUG` 级别开启，会输出类似：

```text
SQL and Parameters:
select * from user where id = ?
Parameters: [1=100]
```

如果希望输出渲染后的 SQL：

```java
import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.RENDERED_SQL;

var spyDataSource = ScxJdbcSpy.spy(
    dataSource,
    new LoggingDataSourceListener(RENDERED_SQL)
);
```

输出类似：

```text
Rendered SQL:
select * from user where id = 100
```

需要注意，`RENDERED_SQL` 只是朴素替换 `?` 占位符，不是真正数据库最终执行 SQL。

## ScxJdbcSpy

`ScxJdbcSpy` 是整个库的包装入口。

它提供五个重载方法：

```java
public static DataSource spy(
    DataSource dataSource,
    DataSourceListener dataSourceListener
)
```

```java
public static Connection spy(
    Connection connection,
    ConnectionListener connectionListener
)
```

```java
public static Statement spy(
    Statement statement,
    StatementListener statementListener
)
```

```java
public static PreparedStatement spy(
    PreparedStatement preparedStatement,
    PreparedStatementListener preparedStatementListener
)
```

```java
public static CallableStatement spy(
    CallableStatement callableStatement,
    CallableStatementListener callableStatementListener
)
```

示例：

```java
var spyConnection = ScxJdbcSpy.spy(
    connection,
    new LoggingConnectionListener(SQL_AND_PARAMETERS)
);
```

或者只包装一个 `PreparedStatement`：

```java
var spyPreparedStatement = ScxJdbcSpy.spy(
    preparedStatement,
    new LoggingPreparedStatementListener(
        "select * from user where id = ?",
        SQL_AND_PARAMETERS
    )
);
```

## 避免重复包装

`ScxJdbcSpy.spy(...)` 会避免 wrapper 嵌套。

例如：

```java
var spy1 = ScxJdbcSpy.spy(dataSource, listener1);

var spy2 = ScxJdbcSpy.spy(spy1, listener2);
```

`spy2` 不会变成：

```text
SpyDataSource(SpyDataSource(original))
```

而是：

```text
SpyDataSource(original, listener2)
```

也就是说，如果传入的对象已经是对应的 spy wrapper，会解开底层原始对象，并替换 listener。

这适合在不同环境下重新配置 listener，例如：

```java
dataSource = ScxJdbcSpy.spy(dataSource, new LoggingDataSourceListener(RENDERED_SQL));
```

不用担心重复包装导致日志输出多次。

## 默认 Logger

`ScxJdbcSpy` 中定义了一个默认 logger：

```java
public static final System.Logger SCX_JDBC_SPY_LOGGER =
    System.getLogger("ScxJdbcSpy");
```

内置 logging listener 如果没有显式传入 logger，就会使用这个 logger。

例如：

```java
new LoggingDataSourceListener(SQL_AND_PARAMETERS)
```

等价于使用：

```java
System.getLogger("ScxJdbcSpy")
```

如果你使用 `SCX Logging`，可以把这个 logger 名称配置到 `DEBUG`。

```java
import dev.scx.logging.ScxLoggerConfig;
import dev.scx.logging.ScxLogging;

ScxLogging.setConfig(
    "ScxJdbcSpy",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.DEBUG)
);
```

## DataSource 包装

包装 `DataSource` 是最推荐的使用方式。

```java
var spyDataSource = ScxJdbcSpy.spy(
    dataSource,
    new LoggingDataSourceListener(SQL_AND_PARAMETERS)
);
```

之后所有从这个 `DataSource` 获取到的连接都会是 `SpyConnection`。

```java
Connection connection = spyDataSource.getConnection();
```

内部流程是：

```text
SpyDataSource#getConnection()
    ↓
原始 DataSource#getConnection()
    ↓
dataSourceListener.createConnectionListener()
    ↓
new SpyConnection(...)
```

带用户名密码的方式也会被包装：

```java
Connection connection = spyDataSource.getConnection(username, password);
```

其它 `DataSource` 方法会直接委托给底层对象，例如：

```text
getLogWriter
setLogWriter
getLoginTimeout
setLoginTimeout
getParentLogger
createConnectionBuilder
createShardingKeyBuilder
```

## Connection 包装

`SpyConnection` 会拦截下面这些创建 statement 的方法：

```text
createStatement()
createStatement(resultSetType, resultSetConcurrency)
createStatement(resultSetType, resultSetConcurrency, resultSetHoldability)

prepareStatement(sql)
prepareStatement(sql, resultSetType, resultSetConcurrency)
prepareStatement(sql, resultSetType, resultSetConcurrency, resultSetHoldability)
prepareStatement(sql, autoGeneratedKeys)
prepareStatement(sql, columnIndexes)
prepareStatement(sql, columnNames)

prepareCall(sql)
prepareCall(sql, resultSetType, resultSetConcurrency)
prepareCall(sql, resultSetType, resultSetConcurrency, resultSetHoldability)
```

示例：

```java
try (var connection = spyDataSource.getConnection()) {
    var statement = connection.createStatement();

    var preparedStatement = connection.prepareStatement(
        "select * from user where id = ?"
    );

    var callableStatement = connection.prepareCall(
        "{call update_user(?)}"
    );
}
```

创建出的对象分别是：

```text
SpyStatement
SpyPreparedStatement
SpyCallableStatement
```

其它 `Connection` 方法大多直接委托给底层连接，例如：

```text
commit
rollback
close
setAutoCommit
getAutoCommit
setReadOnly
getMetaData
setTransactionIsolation
setSavepoint
releaseSavepoint
createBlob
createClob
createArrayOf
setSchema
setNetworkTimeout
beginRequest
endRequest
setShardingKey
```

因此 SCX JDBC Spy 不会改变事务语义。

它只在创建 statement 时继续传播 spy 包装。

## Statement 包装

`SpyStatement` 会拦截普通 `Statement` 的 SQL 执行方法。

包括：

```text
executeQuery(String sql)

executeUpdate(String sql)
executeUpdate(String sql, int autoGeneratedKeys)
executeUpdate(String sql, int[] columnIndexes)
executeUpdate(String sql, String[] columnNames)

execute(String sql)
execute(String sql, int autoGeneratedKeys)
execute(String sql, int[] columnIndexes)
execute(String sql, String[] columnNames)

executeLargeUpdate(String sql)
executeLargeUpdate(String sql, int autoGeneratedKeys)
executeLargeUpdate(String sql, int[] columnIndexes)
executeLargeUpdate(String sql, String[] columnNames)

addBatch(String sql)
clearBatch()
executeBatch()
executeLargeBatch()
```

每次执行会调用对应的 listener。

以 `executeQuery(String sql)` 为例，流程是：

```text
beforeExecuteQuery(statement, sql)
    ↓
底层 statement.executeQuery(sql)
    ↓
afterExecuteQuery(statement, sql, elapsedNanos, e)
```

示意代码：

```java
l.beforeExecuteQuery(d, sql);

SQLException e = null;
var start = System.nanoTime();

try {
    return d.executeQuery(sql);
} catch (SQLException ex) {
    e = ex;
    throw e;
} finally {
    l.afterExecuteQuery(d, sql, System.nanoTime() - start, e);
}
```

其中：

```text
elapsedNanos    执行耗时，单位是纳秒
e               执行期间抛出的 SQLException；成功时为 null
```

## PreparedStatement 包装

`SpyPreparedStatement` 会拦截两类操作：

```text
参数设置
无参执行
```

### 参数设置

例如：

```java
ps.setLong(1, 100L);

ps.setString(2, "Tom");
```

会通知：

```java
preparedStatementListener.setParameter(1, 100L);

preparedStatementListener.setParameter(2, "Tom");
```

支持的参数设置方法包括常见 JDBC setter：

```text
setNull
setBoolean
setByte
setShort
setInt
setLong
setFloat
setDouble
setBigDecimal
setString
setBytes
setDate
setTime
setTimestamp
setAsciiStream
setUnicodeStream
setBinaryStream
setObject
setCharacterStream
setRef
setBlob
setClob
setArray
setURL
setRowId
setNString
setNCharacterStream
setNClob
setSQLXML
```

不同重载最终记录的是对应参数值本身。

例如：

```java
ps.setObject(1, 100, Types.INTEGER);
```

记录的是：

```text
1=100
```

而不是 `Types.INTEGER`。

### clearParameters

```java
ps.clearParameters();
```

会同时：

```text
调用底层 PreparedStatement#clearParameters()
通知 listener.clearParameters()
```

### 无参执行

`PreparedStatement` 的典型执行方法没有 SQL 参数：

```java
executeQuery()
executeUpdate()
execute()
executeLargeUpdate()
```

它们会调用 `PreparedStatementListener` 中对应的回调：

```text
beforeExecuteQuery(PreparedStatement preparedStatement)
afterExecuteQuery(PreparedStatement preparedStatement, elapsedNanos, e)

beforeExecuteUpdate(PreparedStatement preparedStatement)
afterExecuteUpdate(PreparedStatement preparedStatement, elapsedNanos, e)

beforeExecute(PreparedStatement preparedStatement)
afterExecute(PreparedStatement preparedStatement, elapsedNanos, e)
```

示例：

```java
try (var ps = connection.prepareStatement("select * from user where id = ?")) {
    ps.setLong(1, 100L);

    ps.executeQuery();
}
```

日志型 listener 会在 `executeQuery()` 前输出 SQL 和当前参数。

## CallableStatement 包装

`SpyCallableStatement` 继承自 `SpyPreparedStatement`。

因此它支持：

```text
位置参数 setParameter 追踪
execute / executeQuery / executeUpdate / executeLargeUpdate 监听
addBatch / clearBatch / executeBatch 监听
```

例如：

```java
try (var cs = connection.prepareCall("{call update_user(?)}")) {
    cs.setLong(1, 100L);
    cs.execute();
}
```

会按 `PreparedStatement` 的方式记录参数和执行。

需要注意，`CallableStatement` 中按参数名设置的 setter 当前只是直接委托到底层对象，不会调用 `setParameter(...)`。

例如：

```java
cs.setString("name", "Tom");
```

当前不会被内置参数记录逻辑捕获。

`registerOutParameter(...)` 和各种 `getXxx(...)` 输出参数读取方法也只是委托到底层 `CallableStatement`，不会触发额外 listener。

## Listener 层级

SCX JDBC Spy 的 listener 是分层创建的。

```text
DataSourceListener
    ↓ createConnectionListener()
ConnectionListener
    ↓ createStatementListener()
    ↓ createPreparedStatementListener(sql)
    ↓ createCallableStatementListener(sql)
StatementListener
PreparedStatementListener
CallableStatementListener
```

这个设计可以让每条连接、每个 statement、每个 prepared statement 都拥有自己的 listener 实例。

例如 logging listener 中，`PreparedStatement` 的参数就是保存在单个 `LoggingPreparedStatementListener` 实例中的。

这样不同 `PreparedStatement` 之间的参数不会互相污染。

## DataSourceListener

`DataSourceListener` 只有一个方法：

```java
public interface DataSourceListener {

    ConnectionListener createConnectionListener();

}
```

每次 `SpyDataSource#getConnection(...)` 成功后，都会调用它创建一个新的 `ConnectionListener`。

示例：

```java
public final class MyDataSourceListener implements DataSourceListener {

    @Override
    public ConnectionListener createConnectionListener() {
        return new MyConnectionListener();
    }

}
```

## ConnectionListener

`ConnectionListener` 用于为 `Connection` 创建 statement listener。

```java
public interface ConnectionListener {

    StatementListener createStatementListener();

    PreparedStatementListener createPreparedStatementListener(String sql);

    CallableStatementListener createCallableStatementListener(String sql);

}
```

其中：

```text
createStatementListener()               对应 createStatement(...)
createPreparedStatementListener(sql)    对应 prepareStatement(sql, ...)
createCallableStatementListener(sql)    对应 prepareCall(sql, ...)
```

示例：

```java
public final class MyConnectionListener implements ConnectionListener {

    @Override
    public StatementListener createStatementListener() {
        return new MyStatementListener();
    }

    @Override
    public PreparedStatementListener createPreparedStatementListener(String sql) {
        return new MyPreparedStatementListener(sql);
    }

    @Override
    public CallableStatementListener createCallableStatementListener(String sql) {
        return new MyCallableStatementListener(sql);
    }

}
```

## StatementListener

`StatementListener` 用于监听普通 `Statement` 执行。

接口提供的回调包括：

```java
default void beforeExecuteQuery(Statement statement, String sql) {
}

default void afterExecuteQuery(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
) {
}
```

```java
default void beforeExecuteUpdate(Statement statement, String sql) {
}

default void afterExecuteUpdate(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
) {
}
```

```java
default void beforeExecute(Statement statement, String sql) {
}

default void afterExecute(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
) {
}
```

```java
default void beforeAddBatch(Statement statement, String sql) {
}

default void afterAddBatch(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
) {
}
```

```java
default void beforeClearBatch(Statement statement) {
}

default void afterClearBatch(
    Statement statement,
    long elapsedNanos,
    SQLException e
) {
}
```

```java
default void beforeExecuteBatch(Statement statement) {
}

default void afterExecuteBatch(
    Statement statement,
    long elapsedNanos,
    SQLException e
) {
}
```

所有方法都有默认空实现。

因此自定义 listener 时，只需要覆盖自己关心的方法。

## PreparedStatementListener

`PreparedStatementListener` 继承自 `StatementListener`，并增加了 prepared statement 专用回调。

```java
public interface PreparedStatementListener extends StatementListener {

    default void beforeExecuteQuery(PreparedStatement preparedStatement) {
    }

    default void afterExecuteQuery(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
    }

    default void beforeExecuteUpdate(PreparedStatement preparedStatement) {
    }

    default void afterExecuteUpdate(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
    }

    default void beforeExecute(PreparedStatement preparedStatement) {
    }

    default void afterExecute(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
    }

    default void beforeAddBatch(PreparedStatement preparedStatement) {
    }

    default void afterAddBatch(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
    }

    default void setParameter(int parameterIndex, Object value) {
    }

    default void clearParameters() {
    }

}
```

其中：

```text
setParameter(...)     由 PreparedStatement 的 setXxx(...) 方法触发
clearParameters()     由 PreparedStatement#clearParameters() 触发
```

## CallableStatementListener

`CallableStatementListener` 当前只是继承 `PreparedStatementListener`。

```java
public interface CallableStatementListener extends PreparedStatementListener {

}
```

它没有额外定义新的回调方法。

也就是说，当前 `CallableStatement` 的监听语义和 `PreparedStatement` 基本一致。

## 自定义 listener 示例

下面是一个简单的慢 SQL listener。

```java
import dev.scx.jdbc.spy.listener.PreparedStatementListener;

import java.sql.PreparedStatement;
import java.sql.SQLException;

public final class SlowSqlPreparedStatementListener implements PreparedStatementListener {

    private final String sql;

    private final long thresholdNanos;

    public SlowSqlPreparedStatementListener(String sql, long thresholdNanos) {
        this.sql = sql;
        this.thresholdNanos = thresholdNanos;
    }

    @Override
    public void afterExecuteQuery(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
        if (elapsedNanos >= thresholdNanos) {
            System.out.println("Slow SQL: " + sql + ", elapsedNanos=" + elapsedNanos);
        }
    }

    @Override
    public void afterExecuteUpdate(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
        if (elapsedNanos >= thresholdNanos) {
            System.out.println("Slow SQL: " + sql + ", elapsedNanos=" + elapsedNanos);
        }
    }

    @Override
    public void afterExecute(
        PreparedStatement preparedStatement,
        long elapsedNanos,
        SQLException e
    ) {
        if (elapsedNanos >= thresholdNanos) {
            System.out.println("Slow SQL: " + sql + ", elapsedNanos=" + elapsedNanos);
        }
    }

}
```

配套 `ConnectionListener`：

```java
import dev.scx.jdbc.spy.listener.CallableStatementListener;
import dev.scx.jdbc.spy.listener.ConnectionListener;
import dev.scx.jdbc.spy.listener.PreparedStatementListener;
import dev.scx.jdbc.spy.listener.StatementListener;

public final class SlowSqlConnectionListener implements ConnectionListener {

    private final long thresholdNanos;

    public SlowSqlConnectionListener(long thresholdNanos) {
        this.thresholdNanos = thresholdNanos;
    }

    @Override
    public StatementListener createStatementListener() {
        return new SlowSqlStatementListener(thresholdNanos);
    }

    @Override
    public PreparedStatementListener createPreparedStatementListener(String sql) {
        return new SlowSqlPreparedStatementListener(sql, thresholdNanos);
    }

    @Override
    public CallableStatementListener createCallableStatementListener(String sql) {
        return new SlowSqlCallableStatementListener(sql, thresholdNanos);
    }

}
```

配套 `DataSourceListener`：

```java
import dev.scx.jdbc.spy.listener.ConnectionListener;
import dev.scx.jdbc.spy.listener.DataSourceListener;

public final class SlowSqlDataSourceListener implements DataSourceListener {

    private final long thresholdNanos;

    public SlowSqlDataSourceListener(long thresholdNanos) {
        this.thresholdNanos = thresholdNanos;
    }

    @Override
    public ConnectionListener createConnectionListener() {
        return new SlowSqlConnectionListener(thresholdNanos);
    }

}
```

使用：

```java
var spyDataSource = ScxJdbcSpy.spy(
    dataSource,
    new SlowSqlDataSourceListener(100_000_000L)
);
```

这里阈值是：

```text
100ms
```

因为：

```text
100_000_000 ns = 100 ms
```

## LoggingDataSourceListener

`LoggingDataSourceListener` 是内置日志 listener 的最高层入口。

```java
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

var listener = new LoggingDataSourceListener(SQL_AND_PARAMETERS);
```

也可以指定 logger：

```java
var listener = new LoggingDataSourceListener(
    System.getLogger("sql"),
    SQL_AND_PARAMETERS
);
```

它的职责是：

```text
为每个 Connection 创建 LoggingConnectionListener
```

内部逻辑可以理解为：

```java
@Override
public LoggingConnectionListener createConnectionListener() {
    return new LoggingConnectionListener(logger, logStyle);
}
```

## LoggingConnectionListener

`LoggingConnectionListener` 负责创建具体 statement logging listener。

```java
var listener = new LoggingConnectionListener(SQL_AND_PARAMETERS);
```

它会创建：

```text
LoggingStatementListener
LoggingPreparedStatementListener
LoggingCallableStatementListener
```

对应关系是：

```java
@Override
public LoggingStatementListener createStatementListener() {
    return new LoggingStatementListener(logger);
}
```

```java
@Override
public LoggingPreparedStatementListener createPreparedStatementListener(String sql) {
    return new LoggingPreparedStatementListener(logger, sql, logStyle);
}
```

```java
@Override
public LoggingCallableStatementListener createCallableStatementListener(String sql) {
    return new LoggingCallableStatementListener(logger, sql, logStyle);
}
```

## LoggingStatementListener

`LoggingStatementListener` 用于记录普通 `Statement` 的 SQL。

当执行：

```java
statement.executeQuery("select * from user");
```

如果 logger 的 `DEBUG` 级别开启，会输出：

```text
select * from user
```

它覆盖了：

```text
beforeExecuteQuery
beforeExecuteUpdate
beforeExecute
beforeExecuteBatch
afterAddBatch
afterClearBatch
afterExecuteBatch
```

### 普通执行

普通执行会在 before 阶段输出 SQL。

```java
statement.execute("delete from user where id = 1");
```

输出：

```text
delete from user where id = 1
```

### Statement batch

对于普通 `Statement`：

```java
statement.addBatch("insert into user(name) values ('Tom')");
statement.addBatch("insert into user(name) values ('Jerry')");
statement.executeBatch();
```

`LoggingStatementListener` 会在 `addBatch(...)` 成功后保存 SQL。

执行 `executeBatch()` 前，如果 `DEBUG` 级别开启，会逐条输出 batch SQL。

```text
insert into user(name) values ('Tom')
insert into user(name) values ('Jerry')
```

`clearBatch()` 成功后会清空已保存的 batch SQL。

`executeBatch()` 结束后，无论成功还是失败，都会清空已保存的 batch SQL。

## LoggingPreparedStatementListener

`LoggingPreparedStatementListener` 用于记录 `PreparedStatement` 的 SQL 和参数。

它保存三类状态：

```text
sql               原始 SQL 模板
parameters        当前参数 Map
batchParameters   batch 参数快照列表
```

其中 `parameters` 使用：

```java
TreeMap<Integer, Object>
```

所以参数会按 index 从小到大输出。

示例：

```java
try (var ps = connection.prepareStatement(
    "select * from user where id = ? and name = ?"
)) {
    ps.setString(2, "Tom");
    ps.setLong(1, 100L);

    ps.executeQuery();
}
```

虽然设置顺序是 `2` 再 `1`，输出仍然是：

```text
Parameters: [1=100, 2='Tom']
```

## PreparedStatementLogStyle

`PreparedStatementLogStyle` 有两个值：

```java
public enum PreparedStatementLogStyle {

    SQL_AND_PARAMETERS,

    RENDERED_SQL,

}
```

### SQL_AND_PARAMETERS

`SQL_AND_PARAMETERS` 会输出 SQL 模板和参数列表。

```java
var listener = new LoggingDataSourceListener(SQL_AND_PARAMETERS);
```

示例输出：

```text
SQL and Parameters:
select * from user where id = ? and name = ?
Parameters: [1=100, 2='Tom']
```

这种方式更接近 JDBC 调用真实状态。

推荐默认使用这个模式。

### RENDERED_SQL

`RENDERED_SQL` 会把参数朴素替换到 SQL 中。

```java
var listener = new LoggingDataSourceListener(RENDERED_SQL);
```

示例输出：

```text
Rendered SQL:
select * from user where id = 100 and name = 'Tom'
```

需要注意：

```text
RENDERED_SQL 只是用于日志阅读的近似展示
不是数据库实际执行 SQL
```

它不会解析 SQL 字符串字面量、注释、转义规则或数据库方言。

例如：

```sql
select '?' as x where id = ?
```

这里第一个 `?` 在 SQL 字符串字面量中，但朴素替换并不知道这一点。

因此不要把 `RENDERED_SQL` 的输出用于重新执行 SQL。

## 参数格式化规则

`LoggingPreparedStatementListenerHelper#getParameterString(...)` 用于把参数值转换成日志字符串。

规则如下：

```text
null                 -> null
String               -> 'value'
Character            -> 'c'
Number               -> number.toString()
Boolean              -> true / false
TemporalAccessor     -> 'value'
java.util.Date       -> 'value'
java.net.URL         -> 'value'

byte[]               -> 空字符串
InputStream          -> 空字符串
Reader               -> 空字符串
Blob                 -> 空字符串
Clob                 -> 空字符串
SQLXML               -> 空字符串
Array                -> 空字符串
Ref                  -> 空字符串
RowId                -> 空字符串

其它对象             -> String.valueOf(parameter)
```

示例：

```java
ps.setString(1, "Tom");
ps.setInt(2, 18);
ps.setBoolean(3, true);
ps.setNull(4, Types.VARCHAR);
```

`SQL_AND_PARAMETERS` 输出类似：

```text
Parameters: [1='Tom', 2=18, 3=true, 4=null]
```

二进制、大对象和流式参数默认不会把内容写进日志。

例如：

```java
ps.setBytes(1, bytes);
ps.setBinaryStream(2, inputStream);
ps.setBlob(3, blob);
```

对应值会被格式化为空字符串。

这可以避免在日志中输出大量二进制内容或不可重复读取的流。

## PreparedStatement batch

对于 `PreparedStatement`，batch 的心智模型和普通 `Statement` 不一样。

普通 `Statement` 的 batch 是：

```text
多条 SQL
```

而 `PreparedStatement` 的 batch 是：

```text
同一条 SQL 模板 + 多组参数快照
```

示例：

```java
try (var ps = connection.prepareStatement(
    "insert into user(name, age) values (?, ?)"
)) {
    ps.setString(1, "Tom");
    ps.setInt(2, 18);
    ps.addBatch();

    ps.setString(1, "Jerry");
    ps.setInt(2, 20);
    ps.addBatch();

    ps.executeBatch();
}
```

`LoggingPreparedStatementListener` 会在 `addBatch()` 成功后保存当前参数快照。

执行 `executeBatch()` 前会输出 batch 参数。

为了避免日志过大，当前实现只渲染第一条，并提示剩余条数。

`SQL_AND_PARAMETERS` 输出类似：

```text
SQL and Parameters:
insert into user(name, age) values (?, ?)
Parameters: [1='Tom', 2=18]
... (and 1 more batch entries)
```

`RENDERED_SQL` 输出类似：

```text
Rendered SQL:
insert into user(name, age) values ('Tom', 18)
... (and 1 more batch entries)
```

`clearBatch()` 成功后会清空 batch 参数快照。

`executeBatch()` 结束后，无论成功还是失败，都会清空 batch 参数快照。

## LoggingCallableStatementListener

`LoggingCallableStatementListener` 继承自 `LoggingPreparedStatementListener`。

```java
public final class LoggingCallableStatementListener
        extends LoggingPreparedStatementListener
        implements CallableStatementListener {
}
```

因此它的日志行为和 `PreparedStatement` 相同。

示例：

```java
try (var cs = connection.prepareCall("{call update_user(?)}")) {
    cs.setLong(1, 100L);
    cs.execute();
}
```

输出类似：

```text
SQL and Parameters:
{call update_user(?)}
Parameters: [1=100]
```

按名称设置的参数当前不会被记录：

```java
cs.setString("name", "Tom");
```

这类调用当前只是委托到底层 `CallableStatement`。

## unwrap 和 isWrapperFor

所有 spy wrapper 都继承自 `SpyWrapper`，并实现 JDBC `Wrapper` 接口。

### unwrap

```java
T unwrap(Class<T> iface) throws SQLException
```

处理逻辑是：

```text
1. 如果 iface 是当前 spy wrapper 的类型，返回 this
2. 如果 iface 是底层对象 d 的类型，返回 d
3. 否则调用 d.unwrap(iface)
```

示例：

```java
var spyConnection = ScxJdbcSpy.spy(connection, listener);

var original = spyConnection.unwrap(Connection.class);
```

如果 `Connection.class` 匹配底层对象，会返回底层 connection。

也可以取回 spy wrapper：

```java
var spy = spyConnection.unwrap(SpyConnection.class);
```

### isWrapperFor

```java
boolean isWrapperFor(Class<?> iface) throws SQLException
```

处理逻辑是：

```text
1. 如果 iface 是当前 spy wrapper 的类型，返回 true
2. 如果 iface 是底层对象 d 的类型，返回 true
3. 否则调用 d.isWrapperFor(iface)
```

这样可以兼容某些 JDBC wrapper 实现中 `unwrap(...)` / `isWrapperFor(...)` 行为不够直接的情况。

## 只包装关键路径

SCX JDBC Spy 不会包装所有 JDBC 返回对象。

例如：

```java
ResultSet rs = statement.executeQuery(sql);
```

返回的 `ResultSet` 是底层 JDBC driver 返回的原始 `ResultSet`。

SCX JDBC Spy 当前不会包装 `ResultSet`，也不会监听 `ResultSet#next()`、`getString(...)`、`close()` 等操作。

同样，下面这些对象也不会被 spy 包装：

```text
DatabaseMetaData
ResultSetMetaData
ParameterMetaData
Blob
Clob
Array
SQLXML
Savepoint
```

它们通常直接来自底层 JDBC 对象。

## 事务不会被监听

`SpyConnection` 中的事务方法直接委托到底层连接。

例如：

```java
connection.setAutoCommit(false);

connection.commit();

connection.rollback();
```

当前不会触发专门的 listener 回调。

如果你需要监听事务操作，可以在当前库基础上扩展 `ConnectionListener` 和 `SpyConnection`，或者在上层事务管理器中记录。

## 日志级别

内置 logging listener 使用：

```java
System.Logger.Level.DEBUG
```

也就是说，只有当 logger 的 `DEBUG` 级别开启时，才会输出 SQL。

内部判断类似：

```java
if (logger.isLoggable(DEBUG)) {
    logger.log(DEBUG, logMessage);
}
```

如果你没有看到 SQL 日志，首先应该确认：

```text
ScxJdbcSpy logger 是否开启 DEBUG
```

例如使用 SCX Logging：

```java
ScxLogging.setConfig(
    "ScxJdbcSpy",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.DEBUG)
);
```

如果使用其它 `System.Logger` 实现，也需要用对应方式开启 `DEBUG`。

## 完整示例：DataSource SQL 日志

```java
import dev.scx.jdbc.spy.ScxJdbcSpy;
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

import javax.sql.DataSource;

import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.SQL_AND_PARAMETERS;

public class JdbcSpyDemo {

    public static void main(String[] args) throws Exception {
        DataSource rawDataSource = createDataSource();

        DataSource dataSource = ScxJdbcSpy.spy(
            rawDataSource,
            new LoggingDataSourceListener(SQL_AND_PARAMETERS)
        );

        try (var connection = dataSource.getConnection();
             var ps = connection.prepareStatement(
                 "select * from user where id = ? and name = ?"
             )) {

            ps.setLong(1, 100L);
            ps.setString(2, "Tom");

            try (var rs = ps.executeQuery()) {
                while (rs.next()) {
                    System.out.println(rs.getString("name"));
                }
            }
        }
    }

    private static DataSource createDataSource() {
        // 返回你的真实 DataSource
        throw new UnsupportedOperationException();
    }

}
```

## 完整示例：Rendered SQL 日志

```java
import dev.scx.jdbc.spy.ScxJdbcSpy;
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.RENDERED_SQL;

var dataSource = ScxJdbcSpy.spy(
    rawDataSource,
    new LoggingDataSourceListener(RENDERED_SQL)
);

try (var connection = dataSource.getConnection();
     var ps = connection.prepareStatement(
         "insert into user(name, age) values (?, ?)"
     )) {

    ps.setString(1, "Tom");
    ps.setInt(2, 18);

    ps.executeUpdate();
}
```

输出类似：

```text
Rendered SQL:
insert into user(name, age) values ('Tom', 18)
```

## 完整示例：Statement batch

```java
import dev.scx.jdbc.spy.ScxJdbcSpy;
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.SQL_AND_PARAMETERS;

var dataSource = ScxJdbcSpy.spy(
    rawDataSource,
    new LoggingDataSourceListener(SQL_AND_PARAMETERS)
);

try (var connection = dataSource.getConnection();
     var statement = connection.createStatement()) {

    statement.addBatch("insert into user(name) values ('Tom')");
    statement.addBatch("insert into user(name) values ('Jerry')");

    statement.executeBatch();
}
```

输出类似：

```text
insert into user(name) values ('Tom')
insert into user(name) values ('Jerry')
```

## 完整示例：PreparedStatement batch

```java
import dev.scx.jdbc.spy.ScxJdbcSpy;
import dev.scx.jdbc.spy.listener.logging.LoggingDataSourceListener;

import static dev.scx.jdbc.spy.listener.logging.PreparedStatementLogStyle.SQL_AND_PARAMETERS;

var dataSource = ScxJdbcSpy.spy(
    rawDataSource,
    new LoggingDataSourceListener(SQL_AND_PARAMETERS)
);

try (var connection = dataSource.getConnection();
     var ps = connection.prepareStatement(
         "insert into user(name, age) values (?, ?)"
     )) {

    ps.setString(1, "Tom");
    ps.setInt(2, 18);
    ps.addBatch();

    ps.setString(1, "Jerry");
    ps.setInt(2, 20);
    ps.addBatch();

    ps.executeBatch();
}
```

输出类似：

```text
SQL and Parameters:
insert into user(name, age) values (?, ?)
Parameters: [1='Tom', 2=18]
... (and 1 more batch entries)
```

## 完整示例：自定义统计 listener

下面示例统计 SQL 执行次数。

```java
import dev.scx.jdbc.spy.listener.StatementListener;

import java.sql.SQLException;
import java.sql.Statement;
import java.util.concurrent.atomic.AtomicLong;

public final class CountingStatementListener implements StatementListener {

    private final AtomicLong count;

    public CountingStatementListener(AtomicLong count) {
        this.count = count;
    }

    @Override
    public void afterExecuteQuery(
        Statement statement,
        String sql,
        long elapsedNanos,
        SQLException e
    ) {
        count.incrementAndGet();
    }

    @Override
    public void afterExecuteUpdate(
        Statement statement,
        String sql,
        long elapsedNanos,
        SQLException e
    ) {
        count.incrementAndGet();
    }

    @Override
    public void afterExecute(
        Statement statement,
        String sql,
        long elapsedNanos,
        SQLException e
    ) {
        count.incrementAndGet();
    }

}
```

配套连接 listener：

```java
import dev.scx.jdbc.spy.listener.CallableStatementListener;
import dev.scx.jdbc.spy.listener.ConnectionListener;
import dev.scx.jdbc.spy.listener.PreparedStatementListener;
import dev.scx.jdbc.spy.listener.StatementListener;

import java.util.concurrent.atomic.AtomicLong;

public final class CountingConnectionListener implements ConnectionListener {

    private final AtomicLong count;

    public CountingConnectionListener(AtomicLong count) {
        this.count = count;
    }

    @Override
    public StatementListener createStatementListener() {
        return new CountingStatementListener(count);
    }

    @Override
    public PreparedStatementListener createPreparedStatementListener(String sql) {
        return new CountingPreparedStatementListener(count);
    }

    @Override
    public CallableStatementListener createCallableStatementListener(String sql) {
        return new CountingCallableStatementListener(count);
    }

}
```

使用：

```java
var count = new AtomicLong();

var dataSource = ScxJdbcSpy.spy(
    rawDataSource,
    () -> new CountingConnectionListener(count)
);
```

## 方法总览

### ScxJdbcSpy

```java
public static DataSource spy(
    DataSource dataSource,
    DataSourceListener dataSourceListener
)
```

```java
public static Connection spy(
    Connection connection,
    ConnectionListener connectionListener
)
```

```java
public static Statement spy(
    Statement statement,
    StatementListener statementListener
)
```

```java
public static PreparedStatement spy(
    PreparedStatement preparedStatement,
    PreparedStatementListener preparedStatementListener
)
```

```java
public static CallableStatement spy(
    CallableStatement callableStatement,
    CallableStatementListener callableStatementListener
)
```

### DataSourceListener

```java
ConnectionListener createConnectionListener()
```

### ConnectionListener

```java
StatementListener createStatementListener()

PreparedStatementListener createPreparedStatementListener(String sql)

CallableStatementListener createCallableStatementListener(String sql)
```

### StatementListener

```java
void beforeExecuteQuery(Statement statement, String sql)

void afterExecuteQuery(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeExecuteUpdate(Statement statement, String sql)

void afterExecuteUpdate(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeExecute(Statement statement, String sql)

void afterExecute(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeAddBatch(Statement statement, String sql)

void afterAddBatch(
    Statement statement,
    String sql,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeClearBatch(Statement statement)

void afterClearBatch(
    Statement statement,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeExecuteBatch(Statement statement)

void afterExecuteBatch(
    Statement statement,
    long elapsedNanos,
    SQLException e
)
```

### PreparedStatementListener

```java
void beforeExecuteQuery(PreparedStatement preparedStatement)

void afterExecuteQuery(
    PreparedStatement preparedStatement,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeExecuteUpdate(PreparedStatement preparedStatement)

void afterExecuteUpdate(
    PreparedStatement preparedStatement,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeExecute(PreparedStatement preparedStatement)

void afterExecute(
    PreparedStatement preparedStatement,
    long elapsedNanos,
    SQLException e
)
```

```java
void beforeAddBatch(PreparedStatement preparedStatement)

void afterAddBatch(
    PreparedStatement preparedStatement,
    long elapsedNanos,
    SQLException e
)
```

```java
void setParameter(int parameterIndex, Object value)

void clearParameters()
```

### CallableStatementListener

```java
public interface CallableStatementListener extends PreparedStatementListener {

}
```

### LoggingDataSourceListener

```java
public LoggingDataSourceListener(
    PreparedStatementLogStyle logStyle
)

public LoggingDataSourceListener(
    System.Logger logger,
    PreparedStatementLogStyle logStyle
)

public LoggingConnectionListener createConnectionListener()
```

### LoggingConnectionListener

```java
public LoggingConnectionListener(
    PreparedStatementLogStyle logStyle
)

public LoggingConnectionListener(
    System.Logger logger,
    PreparedStatementLogStyle logStyle
)

public LoggingStatementListener createStatementListener()

public LoggingPreparedStatementListener createPreparedStatementListener(String sql)

public LoggingCallableStatementListener createCallableStatementListener(String sql)
```

### PreparedStatementLogStyle

```java
SQL_AND_PARAMETERS

RENDERED_SQL
```

## 设计说明

### 1. SCX JDBC Spy 是 wrapper，不是 driver

SCX JDBC Spy 不实现 JDBC Driver。

它不会自己连接数据库。

你需要先有一个真实的：

```text
DataSource
Connection
Statement
PreparedStatement
CallableStatement
```

然后再用 `ScxJdbcSpy.spy(...)` 包装它。

### 2. 推荐包装 DataSource

虽然可以单独包装 `Connection`、`Statement` 或 `PreparedStatement`，但最推荐的是包装 `DataSource`。

因为这样从连接到 statement 的整个创建链路都能被自动包装。

```java
var dataSource = ScxJdbcSpy.spy(rawDataSource, listener);
```

### 3. listener 按层级创建

`DataSourceListener` 创建 `ConnectionListener`。

`ConnectionListener` 创建 `StatementListener` / `PreparedStatementListener` / `CallableStatementListener`。

这样每个 JDBC 对象都可以拥有自己的 listener 状态。

例如 `PreparedStatement` 参数 Map 就保存在对应的 `PreparedStatementListener` 中。

### 4. SQL 执行前后都能监听

普通 statement 和 prepared statement 的执行方法都会在执行前后调用 listener。

执行后回调会包含：

```text
elapsedNanos
SQLException
```

这样可以实现：

```text
SQL 日志
慢 SQL 统计
异常 SQL 收集
执行耗时监控
批处理记录
```

### 5. 内置日志只在 DEBUG 输出

`LoggingStatementListener` 和 `LoggingPreparedStatementListener` 都会先判断：

```java
logger.isLoggable(DEBUG)
```

只有 `DEBUG` 开启时才会输出。

这避免了关闭 debug 时还生成大量日志字符串。

### 6. PreparedStatement 参数是监听 setXxx 得到的

SCX JDBC Spy 不从 JDBC driver 里反向读取参数。

它是在你调用：

```java
setInt
setString
setObject
setTimestamp
...
```

时记录参数。

因此，如果某些参数设置方式没有被包装，就不会出现在日志里。

当前 `CallableStatement` 的按名称参数 setter 就是这种情况。

### 7. RENDERED_SQL 只是日志展示

`RENDERED_SQL` 不是真 SQL 解析器。

它只是从左到右把 SQL 字符串中的 `?` 替换成已记录参数。

它不会处理：

```text
字符串字面量中的 ?
SQL 注释中的 ?
数据库方言转义规则
驱动内部类型转换
服务端预编译行为
```

所以它只能用于阅读日志，不能作为真实执行 SQL。

### 8. batch 日志做了降噪

`PreparedStatement` batch 可能包含大量参数组。

当前 logging listener 不会把所有 batch 参数都完整输出。

它只输出第一条，并显示剩余数量。

```text
... (and 99 more batch entries)
```

这样可以避免 batch 太大时日志爆炸。

### 9. ResultSet 不会被包装

当前库只监听 statement 执行和参数设置。

`ResultSet` 不会被 spy 包装。

因此不会监听：

```text
ResultSet#next()
ResultSet#getString(...)
ResultSet#close()
```

如果需要统计结果集读取行为，需要扩展 wrapper 范围。

### 10. 不改变 JDBC 语义

SCX JDBC Spy 的包装器大多数方法都直接委托到底层对象。

它不应该改变：

```text
事务提交
事务回滚
连接关闭
statement 关闭
查询结果
更新结果
异常类型
数据库行为
```

listener 只是旁路观察。

但是如果自定义 listener 自己抛出运行时异常，仍然会影响调用流程。

## 常见问题

### SCX JDBC Spy 是 JDBC Driver 吗？

不是。

它不提供 JDBC URL，也不注册 Driver。

它只是包装已有 JDBC 对象。

### 最推荐包装哪个对象？

推荐包装 `DataSource`。

```java
var dataSource = ScxJdbcSpy.spy(rawDataSource, listener);
```

这样后续 `getConnection()`、`prepareStatement()`、`execute()` 都可以自动走 spy 链路。

### 可以只包装 Connection 吗？

可以。

```java
var connection = ScxJdbcSpy.spy(rawConnection, listener);
```

之后通过这个 connection 创建的 statement 会被包装。

### 可以只包装 PreparedStatement 吗？

可以。

但需要自己传入对应的 `PreparedStatementListener`。

```java
var ps = ScxJdbcSpy.spy(rawPs, preparedStatementListener);
```

### 重复 spy 会导致重复日志吗？

不会。

`ScxJdbcSpy.spy(...)` 会避免 wrapper 嵌套。

如果对象已经是对应 spy wrapper，会解开底层对象并替换 listener。

### 默认会输出 SQL 吗？

取决于 logger 是否开启 `DEBUG`。

内置 logging listener 只在 `DEBUG` 级别输出 SQL。

### 默认 logger 名称是什么？

```text
ScxJdbcSpy
```

### 如何让 SQL 日志输出？

需要让 `ScxJdbcSpy` logger 开启 `DEBUG`。

如果使用 SCX Logging，可以这样：

```java
ScxLogging.setConfig(
    "ScxJdbcSpy",
    new ScxLoggerConfig()
        .setLevel(System.Logger.Level.DEBUG)
);
```

### SQL_AND_PARAMETERS 和 RENDERED_SQL 选哪个？

推荐默认使用：

```java
SQL_AND_PARAMETERS
```

因为它更忠实地表达 JDBC 调用。

`RENDERED_SQL` 更适合临时调试和人工阅读。

### RENDERED_SQL 是真实执行 SQL 吗？

不是。

它只是日志展示。

真实执行仍然是 JDBC driver 处理的 prepared statement。

### PreparedStatement 参数从哪里来？

来自 `setXxx(...)` 方法调用。

例如：

```java
ps.setString(1, "Tom");
```

会记录：

```text
1='Tom'
```

### clearParameters 会清掉日志参数吗？

会。

`SpyPreparedStatement#clearParameters()` 会调用 listener 的：

```java
clearParameters()
```

### addBatch 会保存参数吗？

对于 `PreparedStatement`，`addBatch()` 成功后会保存当前参数快照。

对于普通 `Statement`，`addBatch(sql)` 成功后会保存 SQL 字符串。

### executeBatch 后 batch 会清空吗？

会。

无论执行成功还是失败，内置 logging listener 都会清空已保存 batch。

### clearBatch 后 batch 会清空吗？

如果底层 `clearBatch()` 成功，会清空内置保存的 batch 状态。

### CallableStatement 支持吗？

支持基本包装。

`CallableStatement` 会按 `PreparedStatement` 的方式记录位置参数和执行。

### CallableStatement 的命名参数会记录吗？

当前不会。

例如：

```java
cs.setString("name", "Tom");
```

当前不会写入 logging listener 的参数 Map。

### registerOutParameter 会记录吗？

当前不会。

它只是委托到底层 `CallableStatement`。

### 会记录执行耗时吗？

listener 的 after 方法会收到 `elapsedNanos`。

但内置 logging listener 当前没有把耗时输出到日志中。

如果需要慢 SQL 或耗时日志，可以自定义 listener。

### 会记录 SQLException 吗？

listener 的 after 方法会收到执行期间抛出的 `SQLException`。

内置 logging listener 当前没有专门输出异常信息。

如果需要异常 SQL 收集，可以自定义 listener。

### 会包装 ResultSet 吗？

不会。

`executeQuery(...)` 返回的是底层 JDBC driver 返回的 `ResultSet`。

### 会监听 commit / rollback 吗？

不会。

`SpyConnection#commit()` 和 `rollback()` 直接委托到底层连接。

### unwrap 能拿到底层对象吗？

可以。

`SpyWrapper#unwrap(...)` 会先判断当前 spy wrapper，再判断底层对象，最后委托到底层 `unwrap(...)`。

### isWrapperFor 能识别 spy wrapper 吗？

可以。

如果传入的是当前 spy wrapper 类型，会返回 `true`。

### Logging listener 会输出二进制参数吗？

不会输出具体内容。

`byte[]`、`InputStream`、`Reader`、`Blob`、`Clob` 等类型会被格式化为空字符串。

### 参数顺序按什么排列？

`LoggingPreparedStatementListener` 使用 `TreeMap` 保存参数。

所以输出时按参数 index 从小到大排列。

### null 参数怎么显示？

显示为：

```text
null
```

### String 参数怎么显示？

显示为带单引号的形式：

```text
'Tom'
```

### 什么时候用 SCX JDBC Spy？

适合下面这些场景：

1. 想快速查看 JDBC 执行的 SQL。
2. 想记录 PreparedStatement 参数。
3. 想调试 batch SQL。
4. 想统计 SQL 执行次数。
5. 想实现慢 SQL 监听。
6. 想收集执行失败的 SQL。
7. 想在不更换 JDBC Driver 的情况下插入监听逻辑。
8. 想包装已有 DataSource，而不是修改业务代码中的每个 SQL 调用。