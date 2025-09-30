# SCX Data

`scx-data` 是一套数据访问层抽象接口库，致力于提供灵活、可扩展且易用的数据查询和聚合能力，助力业务层高效操作数据。

---

## 核心接口

三类核心 `Repository` 接口满足不同数据操作需求：

| 接口                       | 职责            |
|--------------------------|---------------|
| `Repository`             | 基础数据操作（CRUD）  |
| `LockableRepository`     | 支持查询锁定，实现并发控制 |
| `AggregatableRepository` | 支持复杂聚合查询      |

实现类可根据功能需求选择实现相应接口。

---

## 主要组件简介

### Query

描述查询条件，支持树形条件构造、分页与排序。

> 推荐使用 `QueryBuilder` 来快速构建 `Query` 对象。

### FieldPolicy

定义字段访问策略，支持字段包含/排除、虚拟字段、表达式赋值等复杂场景。

> 推荐使用 `FieldPolicyBuilder` 来快速创建 `FieldPolicy` 对象。

### Aggregation

定义聚合操作，支持多字段分组、多种聚合函数及表达式分组。

> 推荐使用 `AggregationBuilder` 来快速构建 `Aggregation` 对象。

---

## Query 使用示例

### 简单示例：单条件分页排序查询

```java
Query query = eq("status", 1)
        .asc("createdAt")
        .limit(20);
```

*查询状态为1，按创建时间升序，取前20条数据。*

---

### 复杂示例：多条件逻辑组合 + 多排序 + 分页 + BuildControl

```java
Query query = query()
        .where(or(
                and(
                        eq("status", "active", BuildControl.SKIP_IF_NULL),
                        between("createdAt", "2023-01-01", "2023-12-31")
                ),
                and(
                        ne("status", "deleted"),
                        or(
                                in("type", List.of("premium", "vip")),
                                like("description", "%exclusive%", BuildControl.SKIP_IF_EMPTY_STRING)
                        ),
                        not(eq("region", "restricted"))
                )
        ))
        .orderBys(
                asc("priority"),
                desc("lastUpdated", BuildControl.USE_EXPRESSION)
        )
        .limit(50)
        .offset(100);
```

*说明：*

* 查询条件逻辑：

    * （状态为活跃且创建时间在2023年内）

    * 或

    * （状态不等于“deleted”，且（类型是“premium”或“vip”或描述包含“exclusive”），且区域不为“restricted”）

* 排序：优先级升序，最后更新时间倒序（表达式排序）

* 分页：跳过100条，取50条

* `BuildControl` 用于控制条件和排序的构建细节

---

## FieldPolicy 使用示例

### 简单示例：包含指定字段，忽略空值

```java
FieldPolicy fieldPolicy = include("id", "name", "email")
        .ignoreNull(true);
```

*只包含 `id`、`name`、`email` 字段，插入/更新时忽略空值。*

---

### 复杂示例：排除字段、虚拟字段、表达式赋值及空值控制

```java
FieldPolicy fieldPolicy = exclude("password", "secretToken")
        .virtualField("fullName", "CONCAT(firstName, ' ', lastName)")
        .assignField("updatedAt", "CURRENT_TIMESTAMP")
        .ignoreNull("email", true)
        .ignoreNull("phone", false);
```

*虚拟字段简化客户端逻辑，表达式赋值实现自动字段填充，精细控制空值行为。*

---

## Aggregation 使用示例

### 简单示例：按字段分组计数

```java
Aggregation aggregation = groupBy("category")
        .agg("count", "COUNT(*)");
```

---

### 复杂示例：多字段分组与多聚合函数

```java
Aggregation aggregation = groupBy("region")
        .groupBy("month", "DATE_FORMAT(orderDate, '%Y-%m')")
        .agg("totalSales", "SUM(salesAmount)")
        .agg("avgDiscount", "AVG(discount)")
        .agg("maxOrder", "MAX(orderAmount)")
        .agg("orderCount", "COUNT(*)");
```

---

## Repository 使用示例

### 基础查询

```java
Repository<User, Long> userRepository = ...;

Query query = eq("status", "active").limit(10);

List<User> activeUsers = userRepository.find(query);
```

---

### 查询时包含虚拟字段

```java
FieldPolicy fieldPolicy = include("id", "name", "email")
        .virtualField("fullName", "CONCAT(firstName, ' ', lastName)");

Query query = eq("role", "admin");

List<User> admins = userRepository.find(query, fieldPolicy);
```

---

### 插入时自动表达式赋值

```java
FieldPolicy fieldPolicy = assignField("updatedAt", "CURRENT_TIMESTAMP");

productRepository.add(newProduct, fieldPolicy);
```

---

## 上下文与事务管理支持

`scx-data` 提供完善的上下文和事务管理，支持自动与手动事务控制，保障操作资源管理和事务一致性。

### 核心设计

* **上下文自动管理**：通过 `ContextManager` 自动管理资源打开关闭。
* **灵活事务控制**：`TransactionManager` 支持手动和自动事务提交/回滚。
* **事务上下文抽象**：`TransactionContext` 统一提交、回滚接口，支持多实现替换。

---

### 使用示例

#### 自动上下文执行

```java
String result = contextManager.autoContext(() -> {
    return someRepository.findSomething();
});
```

*自动管理资源，简化异常处理。*

---

#### 手动事务控制

```java
transactionManager.withTransaction(txContext -> {
    userRepository.add(user);
    orderRepository.add(order);

    if (someCondition) {
        txContext.commit();
    } else {
        txContext.rollback();
    }
});
```

*灵活控制事务提交或回滚。*

---

#### 自动事务执行

```java
transactionManager.autoTransaction(() -> {
    userRepository.add(user);
    orderRepository.add(order);
});
```

*异常自动回滚，无异常自动提交，简化事务代码。*

---

## 设计理念

* **灵活性**：丰富接口满足多样查询、聚合和字段策略需求。
* **惰性执行**：查询与聚合延迟执行，便于底层数据源适配。
* **清晰分层**：查询、字段策略、聚合职责分明，方便扩展维护。
* **表达式支持**：字段策略和排序支持表达式，简化复杂业务。
* **细粒度控制**：插入更新时灵活控制空值忽略、表达式赋值、虚拟字段等。

---

## 关于 JOIN 的设计决策

`scx-data` **不支持 JOIN**，基于以下考虑：

* 避免数据重复、性能瓶颈，简化接口设计。
* 业务层自行组合关联数据，提高灵活性与维护性。
* 支持多数据源、NoSQL 场景，提高兼容性。

### 推荐替代方案

* 多步查询结合业务层聚合。
* 缓存和服务层组合关联数据。
* 视图或中间表预先聚合。

---

## 总结

不支持 JOIN 是刻意设计，追求简洁、高效、灵活的接口，让业务系统自由掌控复杂关联逻辑，提升整体健壮性与可维护性。

---
