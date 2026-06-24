# SCX Data JS

SCX Data JS 是 [SCX Data](../scx-data/index.md) 和 [SCX Data X](../scx-data-x/index.md) 中部分能力的 JavaScript 版本。

它让浏览器或其它 JavaScript 环境可以使用与 Java 端一致的模型构建 `Query` 和 `FieldPolicy`，再将它们转换为能够通过 JSON 传输的普通 JavaScript 对象。

查询、字段策略以及 Node 结构的具体含义，以以下文档为准：

- [SCX Data](../scx-data/index.md)：`Query`、`Where`、`OrderBy`、`FieldPolicy` 等数据模型和 DSL。
- [SCX Data X](../scx-data-x/index.md)：`Query`、`FieldPolicy` 与 `Node` 之间的转换结构。

[GitHub](https://github.com/scx-projects/scx-data-js)

## 安装

```shell
npm install scx-data-js
```

SCX Data JS 使用 ES Module，可以直接从包根入口导入：

```javascript
import {
    and,
    include,
    queryToNode,
    fieldPolicyToNode,
} from "scx-data-js";
```

## 与 Java 端配合

SCX Data JS 生成的普通对象与 `scx-data-x` 使用的 Node 结构对应，通常用于把前端构建的查询描述发送给 Java 后端。

```javascript
import {
    SKIP_IF_BLANK_STRING,
    and,
    queryToNode,
} from "scx-data-js";

const query = and()
    .like("name", "scx", SKIP_IF_BLANK_STRING)
    .eq("status", "ACTIVE")
    .desc("createdAt")
    .limit(20);

const body = queryToNode(query);
```

`body` 可以直接交给 `JSON.stringify`，Java 端再通过 `scx-data-x` 将对应 Node 还原为 `Query`。

`FieldPolicy` 的传输方式相同：

```javascript
import {
    fieldPolicyToNode,
    include,
} from "scx-data-js";

const fieldPolicy = include("id", "name", "email");
const body = fieldPolicyToNode(fieldPolicy);
```

完整的 DSL 用法和编码结构，请直接参考 [SCX Data](../scx-data/index.md) 与 [SCX Data X](../scx-data-x/index.md)。

## 当前支持范围

SCX Data JS 当前提供：

```text
Query
Condition / And / Or / Not / WhereClause
OrderBy / offset / limit
BuildControl
FieldPolicy
VirtualField / AssignField
Query -> 普通 JavaScript 对象
FieldPolicy -> 普通 JavaScript 对象
```

主要构建入口包括：

```text
query / query_ / where
and / or / not / whereClause
condition / condition2
 eq / ne / lt / lte / gt / gte
like / notLike / likeRegex / notLikeRegex
in_ / notIn / between / notBetween
orderBy / orderBys / asc / desc
offset / limit

includeAll / excludeAll
include / exclude
ignoreNull / ignoreNull_
virtualField / virtualFields
assignField / assignFields

queryToNode
fieldPolicyToNode
```

## 与 Java 版本的差异

SCX Data JS 并不是完整数据访问实现，它只包含适合 JavaScript 客户端使用的描述模型和编码能力。

当前不包含：

```text
Repository / Finder
Aggregation
LockableRepository
数据库连接和查询执行
Node -> Query
Node -> FieldPolicy
```

因此，SCX Data JS 负责构建和发送查询描述，真正的数据访问仍由 Java 端的 `scx-data` 实现及具体 `Repository` 完成。

另外，JavaScript 端的 `Node` 使用普通对象和数组表示；表达式内容以及 `SKIP_IF_*` 等构建控制的最终解释，也由接收端负责。

## 常见问题

### 为什么是 `in_`，而不是 `in`？

因为 `in` 是 JavaScript 的关键字和运算符，不能直接作为导入后的函数标识符使用。

SCX Data JS 使用 `in_` 避免与 JavaScript 语法冲突，其含义与 Java 端的 `in(...)` 相同。

```javascript
in_("status", ["ACTIVE", "PENDING"])
```

### 为什么有些方法以 `_` 结尾？

Java 支持方法重载，可以使用相同的方法名表示不同的参数形式；JavaScript 没有与 Java 相同的重载机制，因此 SCX Data JS 使用 `_` 区分这些方法。

例如：

```javascript
query()
query_(oldQuery)
```

`query()` 创建一个空查询，`query_(oldQuery)` 根据已有查询创建一个新查询。

```javascript
ignoreNull(true)
ignoreNull_("name", true)
```

`ignoreNull(...)` 设置全局规则，`ignoreNull_(...)` 设置指定字段的规则。

### SCX Data JS 会执行查询吗？

不会。

SCX Data JS 只负责构建查询描述并将其转换为普通 JavaScript 对象。查询最终如何执行，由 Java 端的 `Repository` 实现决定。

### `queryToNode` 返回的是 Java 中的 Node 对象吗？

不是。

JavaScript 端没有创建 Java 的 `Node` 实例，而是使用普通对象、数组和基础类型表示相同的数据结构。

因此，转换结果可以直接交给 `JSON.stringify(...)`：

```javascript
const body = JSON.stringify(queryToNode(query));
```

Java 服务端读取 JSON 后，可以先将其解析为 `Node`，再由 SCX Data X 转换为对应的 Java 模型。

### `SKIP_IF_*` 会立即删除 JavaScript 端的条件吗？

不会。

`SKIP_IF_NULL`、`SKIP_IF_EMPTY_LIST`、`SKIP_IF_EMPTY_STRING` 和 `SKIP_IF_BLANK_STRING` 会作为条件的构建控制信息一起编码。

SCX Data JS 不会在调用 `queryToNode(...)` 时自行删除这些条件。具体跳过规则由接收端按照 SCX Data 的语义解释。

### 可以把转换后的普通对象再还原为 `Query` 或 `FieldPolicy` 吗？

当前不可以。

SCX Data JS 目前只提供：

```text
Query -> 普通 JavaScript 对象
FieldPolicy -> 普通 JavaScript 对象
```

暂未提供反向转换。Java 服务端可以使用 SCX Data X 完成 `Node` 到 Java 模型的转换。

### SCX Data JS 可以在 Node.js 中使用吗？

可以。

SCX Data JS 使用 ES Module，不限于浏览器，也可以在支持 ES Module 的 Node.js 或其它 JavaScript 环境中使用。

### 可以把用户输入直接作为表达式发送吗？

不建议直接发送未经检查的用户输入。

`whereClause`、`USE_EXPRESSION`、`USE_EXPRESSION_VALUE`、`VirtualField` 和 `AssignField` 中的表达式只会被当作字符串传输，最终语义由具体的 Java `Repository` 实现决定。

如果表达式来自外部输入，服务端应进行校验、限制或白名单处理。

