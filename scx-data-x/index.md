# SCX Data X

SCX Data X 是 `scx-data` 的扩展库，用来承载围绕 SCX Data 的增强能力。

主要提供 `scx-data` 与 `scx-node` 之间的 `Node` 转换支持，可以将 `Query`、`FieldPolicy`、`Aggregation` 等结构化描述转换为 `Node`，也可以从 `Node` 还原对应对象。

```text
Query        <-> Node
FieldPolicy  <-> Node
Aggregation  <-> Node
```

[GitHub](https://github.com/scx-projects/scx-data-x)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-data-x</artifactId>
    <version>0.0.1</version>
</dependency>
```

`scx-data-x` 会引入 `scx-data` 和 `scx-object-x`。实际使用中，如果你要把 `Node` 写成 JSON、XML 或其它外部格式，通常还会搭配 `scx-format-json`、`scx-format-xml` 或其它 `Node` 格式库。

## 基本概念

SCX Data X 最核心的能力由三个转换器提供：

```text
QueryNodeConverter        Query <-> Node
FieldPolicyNodeConverter  FieldPolicy <-> Node
AggregationNodeConverter  Aggregation <-> Node
```

每个转换器都遵循同一套策略：

```text
对象 -> Node：严格编码，入口对象不允许为 null。
Node -> 对象：宽松解析，null / NullNode 会被解释为对应的空对象或默认值。
```

这样设计的原因是：编码时应该尽量生成完整、明确、可校验的结构；解析时则方便处理外部传入的简化结构或缺省字段。

## 适用场景

`scx-data-x` 适合这些场景：

```text
把前端传入的查询条件解析成 scx-data Query
把 Query / FieldPolicy / Aggregation 存储到配置文件
通过 HTTP、消息队列或 RPC 传输结构化查询描述
为不同 Repository 实现提供统一的查询描述交换格式
把 scx-data 的 DSL 暴露成更通用的 Node / JSON 结构
```

它不适合直接写 SQL，也不负责把表达式转换成数据库语法。表达式字符串仍然由具体的 `Repository` 或数据源适配层解释。

## 快速开始

### Query 转 Node

```java
import dev.scx.data.query.Query;
import dev.scx.node.Node;

import static dev.scx.data.query.BuildControl.*;
import static dev.scx.data.query.QueryBuilder.*;
import static dev.scx.data.x.QueryNodeConverter.*;

public class QueryNodeExample {

    public static void main(String[] args) {
        Query query = and()
            .like("name", "scx", SKIP_IF_BLANK_STRING)
            .eq("status", "ACTIVE", SKIP_IF_NULL)
            .gte("age", 18, SKIP_IF_NULL)
            .desc("createdAt")
            .offset(0)
            .limit(20);

        Node node = queryToNode(query);

        Query parsed = nodeToQuery(node);
    }

}
```

### FieldPolicy 转 Node

```java
import dev.scx.data.field_policy.FieldPolicy;
import dev.scx.node.Node;

import static dev.scx.data.field_policy.FieldPolicyBuilder.*;
import static dev.scx.data.x.FieldPolicyNodeConverter.*;

public class FieldPolicyNodeExample {

    public static void main(String[] args) {
        FieldPolicy fieldPolicy = include("id", "name", "email")
            .virtualField("displayName", "concat(first_name, ' ', last_name)")
            .assignField("updatedAt", "now()")
            .ignoreNull(true)
            .ignoreNull("avatar", false);

        Node node = fieldPolicyToNode(fieldPolicy);

        FieldPolicy parsed = nodeToFieldPolicy(node);
    }

}
```

### Aggregation 转 Node

```java
import dev.scx.data.aggregation.Aggregation;
import dev.scx.node.Node;

import static dev.scx.data.aggregation.AggregationBuilder.*;
import static dev.scx.data.x.AggregationNodeConverter.*;

public class AggregationNodeExample {

    public static void main(String[] args) {
        Aggregation aggregation = groupBy("department")
            .groupBy("month", "date_trunc('month', created_at)")
            .agg("total", "count(*)")
            .agg("amount", "sum(amount)");

        Node node = aggregationToNode(aggregation);

        Aggregation parsed = nodeToAggregation(node);
    }

}
```

## QueryNodeConverter

`QueryNodeConverter` 负责转换 `Query`、`Where`、`OrderBy`、分页参数和条件构建控制信息。

### API

```java
Node queryToNode(Query query);

Query nodeToQuery(Node node);
```

`queryToNode(null)` 会抛出 `QueryToNodeException`。

`nodeToQuery(null)` 和 `nodeToQuery(NullNode.NULL)` 会返回空 `QueryImpl`。

### Query 的 Node 结构

一个完整的 Query 会被编码成：

```json
{
  "@type": "Query",
  "where": null,
  "orderBys": [],
  "offset": null,
  "limit": null
}
```

复杂一些的例子：

```json
{
  "@type": "Query",
  "where": {
    "@type": "And",
    "clauses": [
      {
        "@type": "Condition",
        "selector": "name",
        "conditionType": "EQ",
        "value1": "scx",
        "value2": null,
        "useExpression": true,
        "useExpressionValue": true,
        "skipIfInfo": {
          "@type": "SkipIfInfo",
          "skipIfNull": true,
          "skipIfEmptyList": false,
          "skipIfEmptyString": false,
          "skipIfBlankString": false
        }
      },
      {
        "@type": "Condition",
        "selector": "age",
        "conditionType": "BETWEEN",
        "value1": 18,
        "value2": 30,
        "useExpression": false,
        "useExpressionValue": false,
        "skipIfInfo": {
          "@type": "SkipIfInfo",
          "skipIfNull": false,
          "skipIfEmptyList": true,
          "skipIfEmptyString": false,
          "skipIfBlankString": false
        }
      }
    ]
  },
  "orderBys": [
    {
      "@type": "OrderBy",
      "selector": "createdAt",
      "orderByType": "DESC",
      "useExpression": false
    }
  ],
  "offset": 0,
  "limit": 20
}
```

### 支持的 Where 类型

`where` 字段通过 `@type` 区分具体类型：

```text
Condition    普通结构化条件
And          条件与
Or           条件或
Not          条件取反
WhereClause  原始表达式条件
```

对应结构为：

```json
{
  "@type": "Condition",
  "selector": "name",
  "conditionType": "EQ",
  "value1": "scx",
  "value2": null,
  "useExpression": false,
  "useExpressionValue": false,
  "skipIfInfo": {
    "@type": "SkipIfInfo",
    "skipIfNull": false,
    "skipIfEmptyList": false,
    "skipIfEmptyString": false,
    "skipIfBlankString": false
  }
}
```

```json
{
  "@type": "And",
  "clauses": []
}
```

```json
{
  "@type": "Or",
  "clauses": []
}
```

```json
{
  "@type": "Not",
  "clause": null
}
```

```json
{
  "@type": "WhereClause",
  "expression": "status = ?",
  "params": ["ACTIVE"]
}
```

`Condition#value1`、`Condition#value2` 和 `WhereClause#params` 会通过 `scx-object-x` 的默认对象映射在 Java 对象和 Node 之间转换，因此它们支持的值类型取决于 `DefaultObjectNodeConverter` 的默认规则。

### Query 解析规则

Node 解析成 Query 时使用宽松默认值：

```text
根节点 null / NullNode          -> 空 QueryImpl
where 缺失 / null / NullNode     -> null
orderBys 缺失 / null / NullNode  -> 空数组
offset 缺失 / null / NullNode    -> null
limit 缺失 / null / NullNode     -> null
Condition.useExpression 缺失     -> false
Condition.useExpressionValue 缺失 -> false
Condition.skipIfInfo 缺失        -> 全部 false
OrderBy.useExpression 缺失       -> false
And.clauses 缺失                 -> 空数组
Or.clauses 缺失                  -> 空数组
WhereClause.params 缺失          -> null
```

这些字段解析时会严格校验：

```text
根节点必须是 ObjectNode，并且 @type 必须是 Query。
where 如果存在，必须是 ObjectNode，并且 @type 必须是已知 Where 类型。
orderBys 如果存在，必须是数组，元素不能是 null。
offset / limit 必须能转成 long，且必须 >= 0。
selector、conditionType、orderByType 等必填字段不能为 null。
```

## FieldPolicyNodeConverter

`FieldPolicyNodeConverter` 负责转换字段包含/排除策略、虚拟字段、字段表达式和空值处理规则。

### API

```java
Node fieldPolicyToNode(FieldPolicy fieldPolicy);

FieldPolicy nodeToFieldPolicy(Node node);
```

`fieldPolicyToNode(null)` 会抛出 `FieldPolicyToNodeException`。

`nodeToFieldPolicy(null)` 和 `nodeToFieldPolicy(NullNode.NULL)` 会返回 `includeAll()` 等价的策略，也就是 `FilterMode.EXCLUDED` 且 `fieldNames` 为空。

### FieldPolicy 的 Node 结构

```json
{
  "@type": "FieldPolicy",
  "filterMode": "EXCLUDED",
  "fieldNames": [],
  "virtualFields": [],
  "assignFields": [],
  "ignoreNull": true,
  "ignoreNulls": {}
}
```

包含指定字段并追加虚拟字段、表达式字段的例子：

```json
{
  "@type": "FieldPolicy",
  "filterMode": "INCLUDED",
  "fieldNames": ["id", "name"],
  "virtualFields": [
    {
      "@type": "VirtualField",
      "virtualFieldName": "displayName",
      "expression": "concat(first_name, ' ', last_name)"
    }
  ],
  "assignFields": [
    {
      "@type": "AssignField",
      "fieldName": "updatedAt",
      "expression": "now()"
    }
  ],
  "ignoreNull": false,
  "ignoreNulls": {
    "nickname": true,
    "avatar": false
  }
}
```

### FieldPolicy 解析规则

Node 解析成 FieldPolicy 时使用这些默认值：

```text
根节点 null / NullNode              -> includeAll()
fieldNames 缺失 / null / NullNode    -> 空数组
virtualFields 缺失 / null / NullNode -> 空数组
assignFields 缺失 / null / NullNode  -> 空数组
ignoreNull 缺失 / null / NullNode    -> true
ignoreNulls 缺失 / null / NullNode   -> 空 Map
```

这些字段解析时会严格校验：

```text
根节点必须是 ObjectNode，并且 @type 必须是 FieldPolicy。
filterMode 必须存在，必须是 INCLUDED 或 EXCLUDED。
virtualFields 如果存在，必须是数组，元素不能是 null。
assignFields 如果存在，必须是数组，元素不能是 null。
VirtualField.virtualFieldName 和 expression 不能为 null。
AssignField.fieldName 和 expression 不能为 null。
ignoreNulls 的 key 和 value 都不能为 null。
```

## AggregationNodeConverter

`AggregationNodeConverter` 负责转换分组和聚合描述。

### API

```java
Node aggregationToNode(Aggregation aggregation);

Aggregation nodeToAggregation(Node node);
```

`aggregationToNode(null)` 会抛出 `AggregationToNodeException`。

`nodeToAggregation(null)` 和 `nodeToAggregation(NullNode.NULL)` 会返回空 `AggregationImpl`。

### Aggregation 的 Node 结构

空聚合会被编码成：

```json
{
  "@type": "Aggregation",
  "groupBys": [],
  "aggs": []
}
```

带字段分组、表达式分组和聚合表达式的例子：

```json
{
  "@type": "Aggregation",
  "groupBys": [
    {
      "@type": "FieldGroupBy",
      "fieldName": "department"
    },
    {
      "@type": "ExpressionGroupBy",
      "alias": "month",
      "expression": "date_trunc('month', created_at)"
    }
  ],
  "aggs": [
    {
      "@type": "Agg",
      "alias": "total",
      "expression": "count(*)"
    },
    {
      "@type": "Agg",
      "alias": "amount",
      "expression": "sum(amount)"
    }
  ]
}
```

### Aggregation 解析规则

Node 解析成 Aggregation 时使用这些默认值：

```text
根节点 null / NullNode          -> 空 AggregationImpl
groupBys 缺失 / null / NullNode  -> 空数组
aggs 缺失 / null / NullNode      -> 空数组
```

这些字段解析时会严格校验：

```text
根节点必须是 ObjectNode，并且 @type 必须是 Aggregation。
groupBys 如果存在，必须是数组，元素不能是 null。
aggs 如果存在，必须是数组，元素不能是 null。
GroupBy 的 @type 只能是 FieldGroupBy 或 ExpressionGroupBy。
Agg 的 @type 必须是 Agg。
fieldName、alias、expression 等必填字符串不能为 null。
```

## 与格式库配合

`scx-data-x` 转换能力的边界是 `Node`，不是 JSON 字符串。如果你需要对外传输，可以把它和格式库组合使用：

```text
Query / FieldPolicy / Aggregation
        ↓ scx-data-x
Node
        ↓ scx-format-json / scx-format-xml / 其它格式实现
JSON / XML / 其它外部格式
```

反向流程也是一样：先把外部格式读成 `Node`，再通过 `scx-data-x` 转成 `Query`、`FieldPolicy` 或 `Aggregation`。

## 设计说明

### 1. SCX Data X 是扩展库

`scx-data-x` 的定位是承载 `scx-data` 的扩展能力，而不是替代 `scx-data`，也不是某个具体存储实现。主要提供的是与 `scx-node` 的 `Node` 互转能力。

### 2. 通过 @type 保留结构类型

`Query` 的 `where`、`Aggregation` 的 `groupBys` 都是多态结构。SCX Data X 在每个多态节点上写入 `@type`，解析时再根据 `@type` 分发到具体类型。

### 3. 编码结果尽量完整

对象转 Node 时会写出完整字段，例如空数组、`null`、布尔值和 `skipIfInfo`。这样生成的结构更适合调试、存储和跨版本校验。

### 4. 解析允许缺省字段

Node 转对象时，对于没有歧义的字段会提供默认值，例如缺失的 `orderBys` 解释为空数组，缺失的 `ignoreNull` 解释为 `true`，缺失的 `skipIfInfo` 解释为全 false。

### 5. 表达式只是字符串

`WhereClause.expression`、`VirtualField.expression`、`AssignField.expression`、`ExpressionGroupBy.expression` 和 `Agg.expression` 都只是字符串。SCX Data X 不解析表达式，也不保证表达式可被某个底层系统执行。

### 6. 值转换复用 scx-object-x

`Condition` 的值、`WhereClause` 的参数、`fieldNames` 和 `ignoreNulls` 等结构复用 `scx-object-x` 的默认对象映射能力。这样可以避免在 SCX Data X 中重复维护通用对象到 Node 的映射逻辑。

## 常见问题

### SCX Data X 会执行查询吗？

不会。它提供的是结构转换能力。查询执行仍然由 `Repository`、`Finder`、`AggregatableRepository` 或具体的数据源适配库负责。

### 可以把 Node 当 JSON 用吗？

Node 是内存中的树形数据模型，不等同于 JSON 字符串。它很适合再交给 `scx-format-json` 转成 JSON，也可以交给其它格式实现转成其它外部表示。

### Node 解析为什么有些字段可以缺省，有些字段必须存在？

没有歧义的字段可以缺省，例如空数组、`null`、布尔默认值。有歧义或影响语义的字段必须存在，例如 `@type`、`filterMode`、`conditionType`、`orderByType` 和必填字符串。

### `offset` 和 `limit` 可以是负数吗？

不可以。解析时 `offset` 和 `limit` 必须能转换成 `long`，并且必须大于等于 `0`。

### 什么时候需要引入 SCX Data X？

当你的边界只在 Java 进程内部，并且不需要持久化或传输查询描述时，直接使用 `scx-data` 就够了。只有当你需要把查询、字段策略或聚合描述转成通用结构时，才需要引入 `scx-data-x`。
