# SCX Transaction

SCX Transaction 是一个底层无关的事务抽象库。

它提供 `Transaction`、`TransactionManager` 和 `TransactionException`，用来描述一笔事务的开始、作用域绑定、提交、回滚和最终收尾。

SCX Transaction 本身不绑定 JDBC、数据库连接池、ORM、Repository 或任何具体存储系统。它只定义事务语义边界，具体事务如何开始、如何绑定到当前执行作用域、如何提交和回滚，由对应实现决定。

[GitHub](https://github.com/scx-projects/scx-transaction)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-transaction</artifactId>
    <version>0.10.0</version>
</dependency>
```

## 基本概念

SCX Transaction 中最核心的概念包括：

```text
Transaction             一笔已经开始但尚未完成的事务
TransactionManager      事务管理器，负责开始事务和建立事务作用域绑定
TransactionException    事务异常
```

它们之间的关系可以简单理解为：

```text
TransactionManager 负责 begin()
TransactionManager 负责 with(tx, handler)
Transaction 负责 commit()
Transaction 负责 rollback()
Transaction 负责 close()
```

其中：

1. `begin()` 只表示开始一笔事务。
2. `with(tx, handler)` 只表示在 `handler` 执行期间绑定当前事务。
3. `with(...)` 不负责提交事务。
4. `with(...)` 不负责回滚事务。
5. `with(...)` 不负责关闭事务。
6. 事务最终如何完成，由 `commit()`、`rollback()` 和 `close()` 决定。

## 快速开始

最直接的用法是手动开始事务，然后在事务作用域内执行逻辑，最后显式提交。

```java
try (var tx = transactionManager.begin()) {
    transactionManager.with(tx, () -> {
        repo1.update(...);
        repo2.delete(...);
    });
    tx.commit();
}
```

上面的代码中：

1. `begin()` 开始一笔新事务。
2. `with(tx, handler)` 在 `handler` 执行期间将 `tx` 绑定为当前事务。
3. `handler` 返回后，`with(...)` 只会解除绑定。
4. `commit()` 由调用者显式执行。
5. `try-with-resources` 会保证 `close()` 最终被调用。

更完整的手动错误处理通常是：

```java
try (var tx = transactionManager.begin()) {
    try {
        transactionManager.with(tx, () -> {
            repo1.update(...);
            repo2.delete(...);
        });
        tx.commit();
    } catch (Throwable e) {
        try {
            tx.rollback();
        } catch (TransactionException rb) {
            e.addSuppressed(rb);
        }
        throw e;
    }
}
```

如果希望使用默认策略，也可以使用 `autoTransaction(...)`：

```java
transactionManager.autoTransaction(() -> {
    repo1.update(...);
    repo2.delete(...);
});
```

`autoTransaction(...)` 的默认策略是：

1. handler 正常返回时自动提交。
2. handler 抛出异常时自动回滚。
3. 回滚时如果再次抛出 `TransactionException`，该异常会被添加为原异常的 suppressed exception。
4. 最终仍然通过 `try-with-resources` 调用 `close()`。

## Transaction

`Transaction` 表示一笔已经开始，但尚未完成的活动事务。

它同时承担两类职责：

```text
commit()     提交事务，使事务内修改生效
rollback()   回滚事务，放弃事务内修改
close()      结束事务句柄生命周期，并释放底层资源
```

接口定义如下：

```java
public interface Transaction extends AutoCloseable {

    void commit() throws TransactionException;

    void rollback() throws TransactionException;

    @Override
    void close() throws TransactionException;

}
```

### commit

`commit()` 用于提交事务。

调用成功后，该事务应进入“已完成”状态，事务内的修改应按照底层实现的语义正式生效。

语义要求：

1. `commit()` 成功后，事务应进入已完成状态。
2. 提交成功后，应结束该事务对底层事务资源的占用。
3. 事务一旦提交成功，就不应再继续参与新的事务作用域。
4. 对已经完成的事务再次调用 `commit()`，通常应视为非法使用；具体实现可以抛出异常或拒绝该操作。

### rollback

`rollback()` 用于回滚事务。

调用成功后，该事务应进入“已完成”状态，事务内尚未生效的修改应按照底层实现的语义被放弃。

语义要求：

1. `rollback()` 成功后，事务应进入已完成状态。
2. 回滚成功后，应结束该事务对底层事务资源的占用。
3. 事务一旦回滚成功，就不应再继续参与新的事务作用域。
4. 对已经完成的事务再次调用 `rollback()`，通常应视为非法使用；具体实现可以抛出异常或拒绝该操作。

### close

`close()` 用于结束事务句柄的生命周期，并释放底层资源。

`close()` 的职责不是“提交事务”，而是保证事务句柄离开使用范围时得到最终收尾，不留下悬而未决的活动事务。

语义要求：

1. 如果事务尚未完成，并且此前没有发生过 `commit()` 或 `rollback()` 失败，则 `close()` 应先放弃该事务，再释放底层资源。
2. 在大多数实现中，这通常等价于先执行 `rollback()`，再释放资源。
3. 如果事务已经完成，则 `close()` 应仅执行资源收尾；对调用者而言，该操作通常应表现为幂等，并可以视为 no-op。
4. 如果事务此前在 `commit()` 或 `rollback()` 时已经抛出异常，则 `close()` 不要求再次尝试提交或回滚；此时它的主要职责是完成最终收尾与资源释放。
5. 调用 `close()` 后，该事务句柄的生命周期即告结束，不应再继续参与新的事务作用域，也不应再被提交或回滚。

因此，一个典型用法通常是：

```java
try (var tx = transactionManager.begin()) {
    transactionManager.with(tx, () -> {
        // 执行业务逻辑
    });
    tx.commit();
}
```

如果业务逻辑在 `commit()` 前抛出异常，则 `close()` 应按照失败路径进行收尾。具体是否自动回滚，取决于该 `Transaction` 实现是否满足上述语义约定。

## TransactionManager

`TransactionManager<T extends Transaction>` 是事务管理器。

它负责两件事：

1. 通过 `begin()` 开始一笔新的活动事务。
2. 通过 `with(tx, handler)` 在给定事务绑定的执行作用域内执行一段同步逻辑。

接口中的核心方法包括：

```java
T begin() throws TransactionException;

<R, X extends Throwable> R with(T tx, Function0<R, X> handler) throws X;

<X extends Throwable> void with(T tx, Function0Void<X> handler) throws X;

default <R, X extends Throwable> R withTransaction(Function1<T, R, X> handler)
        throws TransactionException, X;

default <X extends Throwable> void withTransaction(Function1Void<T, X> handler)
        throws TransactionException, X;

default <R, X extends Throwable> R autoTransaction(Function0<R, X> handler)
        throws TransactionException, X;

default <X extends Throwable> void autoTransaction(Function0Void<X> handler)
        throws TransactionException, X;
```

### begin

`begin()` 用于开始一笔新的活动事务。

调用该方法后，返回的事务在语义上已经开始，但它尚未自动参与任何具体执行。若希望某段代码在该事务中运行，应通过 `with(tx, handler)` 显式建立作用域绑定。

语义要求：

1. `begin()` 返回的事务应处于“活动中且未完成”的状态。
2. `begin()` 返回的事务应归属于当前事务管理器所管理的资源域。
3. 调用者有责任在适当时机对返回的事务执行 `commit()`、`rollback()` 或 `close()`。

### with

`with(tx, handler)` 用于在给定事务绑定的执行作用域内执行 `handler`。

该方法只负责“作用域绑定”，不负责“事务定案”。

具体来说：

1. 在 `handler` 执行期间，将给定事务 `tx` 绑定为当前事务。
2. `handler` 执行结束后，无论正常返回还是抛出异常，都解除该绑定。
3. 该方法不会自动提交事务。
4. 该方法不会自动回滚事务。
5. 该方法不会自动关闭事务。
6. `handler` 抛出的异常会原样向外传播。

需要注意的是，进入该作用域并不意味着 `handler` 内的所有操作都必然参与 `tx`。对某个具体操作而言，是否参与该事务，仍取决于该操作与 `tx` 的资源域兼容性。

下面两段逻辑在事务传播模型上是等价的。

显式传参模型：

```java
var tx = transactionManager.begin();

repo1.update(tx, ...);
repo2.delete(tx, ...);

tx.commit();
```

作用域绑定模型：

```java
var tx = transactionManager.begin();

transactionManager.with(tx, () -> {
    repo1.update(...);
    repo2.delete(...);
});

tx.commit();
```

区别只在于事务传播方式：

```text
显式传参模型    通过参数传播
作用域绑定模型  通过当前执行作用域传播
```

SCX Transaction 采用的是作用域绑定模型，但它不规定底层如何保存“当前事务”。实现可以使用 `ThreadLocal`、调用上下文、请求上下文，或其它机制。

### withTransaction

`withTransaction(handler)` 会创建一笔新事务，并将该事务交给 `handler`。

它的语义是：

1. 创建一笔新的活动事务。
2. 在该事务绑定的执行作用域内执行 `handler`。
3. 在 `handler` 结束后按事务自身语义进行最终收尾。

需要特别注意：`withTransaction(...)` 不会自动提交事务，也不会自动回滚事务。事务是否完成，由 `handler` 显式决定。

常见用法：

```java
transactionManager.withTransaction(tx -> {
    repo1.update(...);
    repo2.delete(...);

    tx.commit();
});
```

如果 `handler` 返回前没有显式 `commit()` 或 `rollback()`，则事务会在 `close()` 时按照事务自身语义收尾。对一个规范实现来说，这通常意味着回滚并释放资源。

### autoTransaction

`autoTransaction(handler)` 是默认事务控制策略的便利封装。

它会：

1. 创建一笔新事务。
2. 在该事务绑定的执行作用域内执行 `handler`。
3. 如果 `handler` 正常返回，则提交事务。
4. 如果 `handler` 抛出异常，则回滚事务。
5. 最后关闭事务。

返回值版本：

```java
var result = transactionManager.autoTransaction(() -> {
    repo1.update(...);
    repo2.delete(...);
    return repo3.find(...);
});
```

无返回值版本：

```java
transactionManager.autoTransaction(() -> {
    repo1.update(...);
    repo2.delete(...);
});
```

如果回滚也失败，`autoTransaction(...)` 会把回滚异常作为 suppressed exception 附加到原始异常上，然后继续抛出原始异常。

## 手动事务和自动事务

SCX Transaction 同时支持手动事务和自动事务。

### 手动事务

手动事务适合需要自己决定提交或回滚时机的场景：

```java
try (var tx = transactionManager.begin()) {
    transactionManager.with(tx, () -> {
        orderRepo.createOrder(...);
        stockRepo.lockAndDecrease(...);
    });

    if (needConfirm) {
        tx.commit();
    } else {
        tx.rollback();
    }
}
```

手动模式下：

1. `begin()` 负责开始事务。
2. `with(...)` 负责绑定事务作用域。
3. `commit()` 或 `rollback()` 由调用者显式决定。
4. `close()` 负责最终收尾。

### 自动事务

自动事务适合最常见的“正常提交，异常回滚”场景：

```java
transactionManager.autoTransaction(() -> {
    orderRepo.createOrder(...);
    stockRepo.lockAndDecrease(...);
});
```

自动模式下：

1. 正常返回时提交。
2. 抛出异常时回滚。
3. 最终关闭事务。

## 作用域绑定

SCX Transaction 的一个重要设计点是：事务对象和事务作用域是分开的。

拿到 `Transaction` 对象，只表示你持有了一笔已经开始的事务；它并不自动影响当前代码中的所有操作。

只有进入 `with(tx, handler)` 后，当前同步执行链中的兼容操作才有机会感知到这笔事务。

示例：

```java
var tx = transactionManager.begin();

// 这里虽然已经有 tx，但还没有建立当前事务绑定
repo1.update(...);

transactionManager.with(tx, () -> {
    // 这里才处于 tx 的事务绑定作用域内
    repo1.update(...);
    repo2.delete(...);
});

tx.commit();
```

这可以避免“只要创建了事务，所有后续操作都被隐式污染”的问题，也让事务边界更加明确。

## 资源域兼容性

SCX Transaction 不承诺跨不兼容资源域的统一事务语义。

一笔事务只应对与其兼容、并且位于同一资源管理域中的操作生效。

例如：

1. 同一个数据库连接上的多个 Repository 操作，可以共享同一事务。
2. 同一个会话或同一个存储运行时中的多个操作，可以共享同一事务。
3. 不同数据库连接、不同数据源、不同存储系统之间，除非具体实现明确支持，否则不应被假定为同一事务。

对于不兼容操作，推荐行为是：

1. 忽略当前事务，使其继续按照原有方式执行。
2. 不要因为当前事务而静默重定向到底层另一个资源域。
3. 不要伪装成跨资源事务已经生效。

换句话说，事务的作用是：

```text
让本来就处于同一资源域中的操作共享同一个事务上下文
```

而不是：

```text
把某个执行方强行改绑到另一个资源管理域或另一种底层实现上去
```

## 嵌套作用域

`with(tx, handler)` 应支持嵌套作用域。

内层作用域结束后，应恢复外层原有的事务绑定。

示例：

```java
transactionManager.with(tx1, () -> {
    repo1.update(...);

    transactionManager.with(tx2, () -> {
        repo2.update(...);
    });

    // 这里应恢复到 tx1
    repo1.delete(...);
});
```

实现时通常可以使用栈来保存当前事务：

```text
tx1 入栈
    tx2 入栈
    tx2 出栈
tx1 出栈
```

需要注意，是否允许 `tx1` 和 `tx2` 属于同一个资源域、是否允许真正的底层嵌套事务、是否映射为 savepoint，都不是 SCX Transaction 强制规定的内容，应由具体实现决定。

## 同步执行范围

`with(tx, handler)` 默认只保证当前同步执行链中的事务绑定语义。

SCX Transaction 不默认承诺下面这些场景会自动继承当前事务：

1. 新线程。
2. 线程池任务。
3. 异步回调。
4. 延迟执行逻辑。
5. Reactive 流中的后续阶段。

如果某个实现希望支持跨线程或异步传播，应由该实现明确说明传播机制和边界。

## 异常处理

SCX Transaction 使用 `TransactionException` 表示事务异常。

它继承自 `RuntimeException`，并提供三种构造方式：

```java
public TransactionException(String message)

public TransactionException(Throwable cause)

public TransactionException(String message, Throwable cause)
```

通常可以在下面这些场景中抛出：

1. 开始事务失败。
2. 提交事务失败。
3. 回滚事务失败。
4. 最终收尾或资源释放失败。
5. 事务状态非法。
6. 传入了不属于当前管理器或不兼容资源域的事务。

示例：

```java
try {
    transactionManager.autoTransaction(() -> {
        repo1.update(...);
        repo2.delete(...);
    });
} catch (TransactionException e) {
    // 记录日志或转换为业务异常
}
```

## 实现 Transaction

SCX Transaction 是抽象层，真正的事务能力由具体实现提供。

一个 `Transaction` 实现通常需要维护至少几类状态：

```text
completed   事务是否已经提交或回滚成功
failed      commit 或 rollback 是否曾经失败
closed      事务句柄是否已经关闭
```

一个简化实现大致如下：

```java
public final class SimpleTransaction implements Transaction {

    private boolean completed;
    private boolean failed;
    private boolean closed;

    @Override
    public void commit() throws TransactionException {
        if (closed) {
            throw new TransactionException("Transaction is closed");
        }
        if (completed) {
            throw new TransactionException("Transaction is completed");
        }
        try {
            doCommit();
            completed = true;
        } catch (Throwable e) {
            failed = true;
            throw new TransactionException(e);
        }
    }

    @Override
    public void rollback() throws TransactionException {
        if (closed) {
            throw new TransactionException("Transaction is closed");
        }
        if (completed) {
            throw new TransactionException("Transaction is completed");
        }
        try {
            doRollback();
            completed = true;
        } catch (Throwable e) {
            failed = true;
            throw new TransactionException(e);
        }
    }

    @Override
    public void close() throws TransactionException {
        if (closed) {
            return;
        }
        closed = true;

        if (!completed && !failed) {
            doRollback();
            completed = true;
        }

        doClose();
    }

    private void doCommit() {
        // 提交底层事务
    }

    private void doRollback() {
        // 回滚底层事务
    }

    private void doClose() {
        // 释放底层资源
    }

}
```

这只是一个示例。实际实现中，`doCommit()`、`doRollback()` 和 `doClose()` 可能对应 JDBC Connection、数据库会话、事务上下文、文件系统事务或其它底层资源。

## 实现 TransactionManager

一个 `TransactionManager` 实现通常需要解决两个问题：

1. 如何开始一笔底层事务。
2. 如何在 `with(tx, handler)` 期间让兼容操作能获取到当前事务。

一个基于 `ThreadLocal` 的简化实现大致如下：

```java
import dev.scx.function.Function0;
import dev.scx.function.Function0Void;
import dev.scx.transaction.TransactionManager;
import dev.scx.transaction.exception.TransactionException;

import java.util.ArrayDeque;
import java.util.Deque;

public final class SimpleTransactionManager implements TransactionManager<SimpleTransaction> {

    private final ThreadLocal<Deque<SimpleTransaction>> current =
            ThreadLocal.withInitial(ArrayDeque::new);

    @Override
    public SimpleTransaction begin() throws TransactionException {
        return new SimpleTransaction();
    }

    public SimpleTransaction currentTransaction() {
        var stack = current.get();
        return stack.peek();
    }

    @Override
    public <R, X extends Throwable> R with(SimpleTransaction tx, Function0<R, X> handler) throws X {
        var stack = current.get();
        stack.push(tx);
        try {
            return handler.apply();
        } finally {
            stack.pop();
            if (stack.isEmpty()) {
                current.remove();
            }
        }
    }

    @Override
    public <X extends Throwable> void with(SimpleTransaction tx, Function0Void<X> handler) throws X {
        with(tx, () -> {
            handler.apply();
            return null;
        });
    }

}
```

Repository 或其它执行组件可以通过 `currentTransaction()` 获取当前事务，并在兼容时加入该事务。

需要注意：

1. 上面的实现只展示作用域绑定思路，不代表完整生产实现。
2. 生产实现还需要校验 `tx` 是否来自当前管理器。
3. 生产实现还需要校验事务是否仍然活动。
4. 生产实现还需要处理资源域兼容性。
5. 如果使用线程池或异步执行，单纯 `ThreadLocal` 不会自动传播事务上下文。

## 与 Repository 的关系

SCX Transaction 不要求只能用于 Repository，但 Repository 是最容易理解的例子。

例如，一个 Repository 实现可以在执行更新时检查当前事务：

```java
public long update(User user) {
    var tx = transactionManager.currentTransaction();

    if (tx != null && isCompatible(tx)) {
        // 使用 tx 对应的底层连接或会话执行更新
    } else {
        // 按普通方式执行
    }
}
```

这体现了 SCX Transaction 的设计边界：

1. 事务管理器只负责开始事务和绑定作用域。
2. Repository 决定自己是否兼容当前事务。
3. 具体执行、连接复用、SQL 提交、异常转换都属于实现层。

## 完整示例

下面是一个业务层使用示例：

```java
public class OrderService {

    private final TransactionManager<?> transactionManager;
    private final OrderRepository orderRepo;
    private final StockRepository stockRepo;

    public OrderService(TransactionManager<?> transactionManager,
                        OrderRepository orderRepo,
                        StockRepository stockRepo) {
        this.transactionManager = transactionManager;
        this.orderRepo = orderRepo;
        this.stockRepo = stockRepo;
    }

    public void createOrder(CreateOrderRequest request) {
        transactionManager.autoTransaction(() -> {
            var order = orderRepo.create(request);
            stockRepo.decrease(order.productId(), order.quantity());
        });
    }

}
```

如果需要更细粒度地控制提交时机，可以使用手动事务：

```java
public void createOrderManually(CreateOrderRequest request) {
    try (var tx = transactionManager.begin()) {
        try {
            transactionManager.with(tx, () -> {
                var order = orderRepo.create(request);
                stockRepo.decrease(order.productId(), order.quantity());
            });

            tx.commit();
        } catch (Throwable e) {
            try {
                tx.rollback();
            } catch (TransactionException rb) {
                e.addSuppressed(rb);
            }
            throw e;
        }
    }
}
```

## 设计说明

### 1. Transaction 是事务句柄

`Transaction` 表示一笔具体的活动事务。它不是注解，不是切面，也不是自动事务模板。

事务是否最终生效，由 `commit()` 和 `rollback()` 决定。事务句柄是否最终收尾，由 `close()` 决定。

### 2. TransactionManager 不等于自动事务

`TransactionManager` 中的“Manager”不表示自动提交、自动回滚或自动关闭。

它的底层职责只有两个：

1. 开始事务。
2. 建立当前执行作用域和某笔事务之间的绑定关系。

自动提交和异常回滚是 `autoTransaction(...)` 这个便利方法提供的默认策略，不是 `with(...)` 的职责。

### 3. begin 和 with 是分开的

`begin()` 只开始事务。

`with(...)` 只建立作用域绑定。

这种分离让事务生命周期和事务传播方式保持清晰，也让调用者可以自由选择：

1. 手动提交。
2. 手动回滚。
3. 自动提交/回滚。
4. 只创建事务并传给其它逻辑控制。

### 4. close 是最终收尾，不是提交

`close()` 不应被理解为“成功提交”。

在规范实现中，如果事务尚未完成，`close()` 应按照失败路径收尾，通常意味着回滚并释放资源。

因此不要写出依赖 `close()` 提交事务的代码。

### 5. 不承诺分布式事务

SCX Transaction 不提供下面这些能力：

1. 跨不兼容资源域的统一事务。
2. 分布式事务。
3. 两阶段提交。
4. 多数据源一致性协调。
5. 自动跨线程事务传播。
6. 自动把任意执行方改绑到另一个资源域。

这些能力如果需要，应由更上层或具体实现单独提供。

## 常见问题

### begin 之后，后面的所有操作都会自动进入事务吗？

不会。

`begin()` 只创建并开始一笔事务。要让某段代码在该事务绑定作用域内运行，需要调用 `with(tx, handler)`。

### with 会自动提交或回滚吗？

不会。

`with(...)` 只负责绑定和解绑当前事务。它不会自动提交、不会自动回滚、也不会自动关闭事务。

### withTransaction 会自动提交吗？

不会。

`withTransaction(...)` 只是创建新事务、进入事务作用域，并在最后调用 `close()` 收尾。事务是否提交或回滚，需要在 handler 中显式决定。

如果需要“正常提交，异常回滚”的默认策略，应使用 `autoTransaction(...)`。

### autoTransaction 的默认策略是什么？

handler 正常返回时提交事务。

handler 抛出异常时回滚事务。

最后关闭事务。

### 如果 handler 抛异常，同时 rollback 也失败，会发生什么？

`autoTransaction(...)` 会把回滚异常添加为原异常的 suppressed exception，然后继续抛出原异常。

### close 会提交事务吗？

不应该。

`close()` 是最终收尾方法。对于未完成事务，规范语义是按失败路径终止该事务，通常是回滚并释放资源。

### 可以跨多个数据库或多个资源做事务吗？

SCX Transaction 本身不承诺这一点。

一笔事务只应对兼容且处于同一资源管理域中的操作生效。跨数据库、跨连接、跨存储系统的一致性需要具体实现或更高层机制支持。

### 可以用于异步或跨线程吗？

接口本身不默认承诺异步传播。

如果实现基于 `ThreadLocal`，那么新线程或线程池任务通常不会自动继承当前事务。需要异步传播时，应由具体实现明确提供上下文传递机制。

### 为什么要把 begin 和 with 分开？

因为“事务生命周期”和“事务传播方式”是两件事。

`begin()` 负责创建事务。

`with(...)` 负责让某段同步代码在这笔事务的作用域内执行。

这种拆分可以让事务边界更明确，也避免事务对象一创建就隐式污染后续所有操作。
