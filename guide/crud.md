# 增删改查

我们提供了一个统一的数据访问层抽象接口 `scx-data`，理论上可以兼容各种关系型或非关系型数据库（未来会进行拓展），但目前框架只提供了一套常用的 `scx-data-jdbc` 实现。以下演示基于 MySQL 来完成。

## 1. 引入依赖

首先，你需要在项目中引入相关的依赖：

```xml
<!-- scx-data 的 JDBC 实现 -->
<dependency>
    <groupId>cool.scx</groupId>
    <artifactId>scx-data-jdbc</artifactId>
    <version>{version}</version>
</dependency>
<!-- MySQL 的驱动 -->
<dependency>
    <groupId>cool.scx</groupId>
    <artifactId>scx-jdbc-mysql</artifactId>
    <version>{version}</version>
</dependency>
```

## 2. 创建实体类用于映射

接下来，我们创建一个 `User` 类作为实体，映射到数据库中的数据表：

```java
public class User {
    public Long id;            // ID
    public Integer age;        // 年龄
    public String name;        // 姓名
    public String[] favorites; // 喜欢的事物
}
```

如你所见，我们没有使用 `get` 和 `set` 方法，也无需这些方法。这是框架对数据访问层的一种设计取舍，直接使用字段，拒绝冗余的 getter/setter 方法，简化代码。

## 3. 配置 JDBC 上下文

接下来，配置与数据库的连接。你可以通过 `JDBCContext` 来帮助处理与 MySQL 的连接：

```java
var mysqlDataSource = new MysqlDataSource();
mysqlDataSource.setUrl("jdbc:mysql://localhost:3306/test");
mysqlDataSource.setUser("root");
mysqlDataSource.setPassword("password");
var jdbcContext = new JDBCContext(mysqlDataSource);
```

## 4. 创建与实体相关的 Repository

然后，我们创建与 `User` 相关的 `Repository`，用于数据的增删改查：

```java
var userRepository = new JDBCRepository<>(User.class, jdbcContext);
```

## 5. 创建数据库表

如果在数据库中尚未创建相关数据表，我们也可以使用这个帮助函数来完成自动创建：

```java
var userTable = userRepository.table();
// 根据 userTable 的表定义在数据库中创建对应的表结构
SchemaHelper.fixTable(userTable, jdbcContext);
```

至此，我们已经有了一个可以使用的 `userRepository`，并且没有任何魔法，也没有任何隐式行为。

## 6. 简单的插入操作

接下来，我们进行一个简单的插入操作。创建一个新的 `User` 对象，并使用 `add` 方法将其插入到数据库：

```java
var user = new User();
user.id = 1L;
user.age = 25;
user.name = "John Doe";
user.favorites = new String[]{"Reading", "Cycling"};

var userId = userRepository.add(user); // 将用户数据插入到数据库
System.out.println("Inserted user with ID: " + userId);
```

`add` 方法会返回插入数据的主键（在此例中是 `user.id`），如果数据库没有主键，则返回 `null`。

## 7. 查询数据

我们可以通过 `finder` 方法来创建查询器，也可以使用简单的 `find` 方法查询符合条件的数据。以下是一个简单的查询操作：

```java
var userList = userRepository.find(eq("age", 25)); // 查询年龄为 25 的用户
var userFinder = userRepository.finder(eq("age", 25)); // 查询器
//流式查询
userFinder.forEach(user -> {
    System.out.println("user name" + user.name);
});
```

## 8. 更新数据

使用 `update` 方法来更新已有的数据。假设我们想更新 ID 为 1 的用户的名字：

```java
var user= new User();
user.name = "Jane Doe";
var updatedCount = userRepository.update(user, eq("id", 1));
System.out.println("Updated " + updatedCount + " user(s).");
```

`update` 方法会返回更新成功的记录数。

## 9. 删除数据

使用 `delete` 方法来删除符合条件的数据。例如，删除年龄为 25 的用户：

```java
var deletedCount = userRepository.delete(eq("age", 25));
System.out.println("Deleted " + deletedCount + " user(s).");
```

`delete` 方法会返回删除成功的记录数。

## 10. 查询数据总数

使用 `count` 方法来查询符合条件的数据条数。例如，查询年龄为 25 的用户数量：

```java
var userCount = userRepository.count(eq("age", 25));
System.out.println("Found " + userCount + " user(s) with age 25.");
```

---

通过上述操作，你可以轻松实现数据的增删改查操作，并能通过简单的查询条件构建来灵活获取所需的数据。`scx-data` 提供了一个高效、简洁的数据访问层，使得操作数据库变得更加方便和可维护。
