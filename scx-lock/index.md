# SCX Lock

SCX Lock 是一个极简的按 key 加锁工具库。

它提供 `KeyedLock` 和默认实现 `DefaultKeyedLock`，用于解决“同一个 key 的任务需要串行执行，不同 key 的任务可以并行执行”这类问题。

SCX Lock 本身不是分布式锁，也不是事务锁，也不是完整的锁框架。它只在当前 JVM 进程内工作，用来把锁的粒度从“一个全局锁”缩小到“一个 key 一个锁”。

它适合用于：

```text
同一个用户的操作串行执行
同一个订单的修改串行执行
同一个资源 ID 的任务串行执行
同一个文件路径的处理串行执行
同一个缓存 key 的刷新串行执行
```

[GitHub](https://github.com/scx-projects/scx-lock)

## 安装

### Maven

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-lock</artifactId>
    <version>0.1.0</version>
</dependency>
```

## 基本概念

SCX Lock 当前只有两个核心类型：

```text
KeyedLock           按 key 加锁的接口
DefaultKeyedLock    KeyedLock 的默认实现
```

它们之间的关系可以简单理解为：

```text
KeyedLock
    ↓
lock(key)
    ↓
获取指定 key 对应的锁
    ↓
执行业务逻辑
    ↓
unlock(key)
    ↓
释放指定 key 对应的锁
```

默认实现内部使用：

```text
ConcurrentHashMap<T, SemaphoreWrapper>
Semaphore
AtomicInteger
```

也就是说：

```text
ConcurrentHashMap 负责保存 key 和锁对象的映射
Semaphore 负责阻塞和释放等待线程
AtomicInteger 负责记录当前 key 上的使用数量
```

默认情况下，每个 key 对应一个公平的单许可 `Semaphore`：

```java
new Semaphore(1, true)
```

因此默认语义是：

```text
同一个 key 同一时间只能有一个线程进入临界区
不同 key 之间互不影响，可以同时进入临界区
```

## 快速开始

最常见的用法是创建一个 `DefaultKeyedLock`，然后在业务代码中使用 `lock` 和 `unlock` 包住临界区。

```java
import dev.scx.lock.DefaultKeyedLock;
import dev.scx.lock.KeyedLock;

public class OrderService {

    private final KeyedLock<String> lock = new DefaultKeyedLock<>();

    public void updateOrder(String orderID) {
        lock.lock(orderID);
        try {
            // 同一个 orderID 的更新会串行执行
            doUpdateOrder(orderID);
        } finally {
            lock.unlock(orderID);
        }
    }

    private void doUpdateOrder(String orderID) {
        // 更新订单
    }

}
```

上面的代码中：

```text
updateOrder("A") 和 updateOrder("A") 会串行执行
updateOrder("A") 和 updateOrder("B") 可以并行执行
```

因此它比直接使用一个全局锁更细粒度。

```java
synchronized (this) {
    // 所有订单都会互相阻塞
}
```

而使用 `KeyedLock` 后，只有相同 key 才会互相阻塞。

## KeyedLock

`KeyedLock<T>` 是按 key 加锁的接口。

接口定义非常简单：

```java
public interface KeyedLock<T> {

    void lock(T key);

    void unlock(T key);

}
```

其中：

```text
T      key 的类型
key    锁的标识
```

例如：

```java
KeyedLock<String> stringLock = new DefaultKeyedLock<>();

KeyedLock<Long> longLock = new DefaultKeyedLock<>();

KeyedLock<OrderKey> orderKeyLock = new DefaultKeyedLock<>();
```

只要 key 能作为 `ConcurrentHashMap` 的 key 使用，就可以作为 `KeyedLock` 的 key。

通常 key 应该满足：

```text
equals/hashCode 稳定
不会在加锁期间改变参与 equals/hashCode 的字段
不是 null
```

## lock

`lock(key)` 用于获取指定 key 对应的锁。

```java
lock.lock(key);
```

当当前 key 没有其它线程持有锁时，当前线程会立即获得锁并继续执行。

当当前 key 已经被其它线程持有时，当前线程会等待，直到该 key 的锁被释放。

默认实现中，等待使用的是：

```java
Semaphore#acquireUninterruptibly()
```

因此 `lock(key)` 不会抛出 `InterruptedException`。

如果等待过程中线程被中断，它仍然会继续等待，直到最终获取到锁。

示例：

```java
lock.lock("user:1");
try {
    // 只有拿到 user:1 这把锁后才会执行到这里
} finally {
    lock.unlock("user:1");
}
```

## unlock

`unlock(key)` 用于释放指定 key 对应的锁。

```java
lock.unlock(key);
```

它通常应该和 `lock(key)` 成对出现。

推荐写法是：

```java
lock.lock(key);
try {
    // 临界区
} finally {
    lock.unlock(key);
}
```

这样即使临界区中抛出异常，也可以确保锁被释放。

不推荐写成：

```java
lock.lock(key);

// 如果这里抛出异常，unlock 就不会执行
businessLogic();

lock.unlock(key);
```

### unlock 可以在不同线程调用

`DefaultKeyedLock` 底层使用 `Semaphore`，并不检查释放锁的线程是否就是获取锁的线程。

因此下面这种方式在语义上是允许的：

```java
lock.lock(key);

Thread.ofVirtual().start(() -> {
    lock.unlock(key);
});
```

这和 `ReentrantLock` 不同。

`ReentrantLock` 要求由持有锁的线程释放锁，而 `Semaphore` 更像是一个许可计数器，只要释放同一个 key 对应的许可即可。

不过在普通业务代码中，仍然建议使用 `try/finally` 在同一个逻辑流程中释放锁，这样更容易保证 `lock` 和 `unlock` 成对出现。

## DefaultKeyedLock

`DefaultKeyedLock<T>` 是 `KeyedLock<T>` 的默认实现。

它提供两个构造方法：

```java
public DefaultKeyedLock()

public DefaultKeyedLock(Function<T, Semaphore> semaphoreBuilder)
```

默认构造方法等价于：

```java
new DefaultKeyedLock<>(key -> new Semaphore(1, true))
```

也就是说：

```text
每个 key 最多允许 1 个线程同时进入
等待队列使用公平模式
```

示例：

```java
KeyedLock<String> lock = new DefaultKeyedLock<>();
```

### 内部结构

默认实现内部维护一个 `ConcurrentHashMap`。

可以简化理解为：

```text
key -> SemaphoreWrapper
```

其中 `SemaphoreWrapper` 内部包含：

```text
Semaphore      当前 key 对应的信号量
AtomicInteger  当前 key 上正在使用或等待的数量
```

当第一次对某个 key 调用 `lock(key)` 时，会创建对应的 `SemaphoreWrapper`。

当该 key 上已经没有持有者，也没有等待者时，对应的映射会从 `ConcurrentHashMap` 中移除。

这意味着：

```text
锁对象是按需创建的
锁对象会在不再使用时自动清理
不会为每个历史 key 永久保留一个锁对象
```

## 同 key 串行执行

假设有多个线程同时处理同一个用户。

```java
import dev.scx.lock.DefaultKeyedLock;
import dev.scx.lock.KeyedLock;

public class UserTaskService {

    private final KeyedLock<Long> lock = new DefaultKeyedLock<>();

    public void runUserTask(long userID) {
        lock.lock(userID);
        try {
            System.out.println("start user " + userID);
            doTask(userID);
            System.out.println("end user " + userID);
        } finally {
            lock.unlock(userID);
        }
    }

    private void doTask(long userID) {
        // 执行任务
    }

}
```

如果同时执行：

```java
service.runUserTask(1L);
service.runUserTask(1L);
service.runUserTask(1L);
```

那么这三个任务会按顺序进入临界区。

也就是说，同一个 `userID` 的任务不会同时执行。

## 不同 key 并行执行

如果同时处理不同用户：

```java
service.runUserTask(1L);
service.runUserTask(2L);
service.runUserTask(3L);
```

这些任务使用的是不同 key。

默认情况下，它们不会互相阻塞，可以并行执行。

这种方式适合替代过粗的全局锁。

例如不推荐：

```java
synchronized void runUserTask(long userID) {
    // userID 不同也会互相阻塞
}
```

推荐：

```java
void runUserTask(long userID) {
    lock.lock(userID);
    try {
        // 只有相同 userID 才会互相阻塞
    } finally {
        lock.unlock(userID);
    }
}
```

## 使用自定义 Semaphore

`DefaultKeyedLock` 支持传入自定义 `Semaphore` 构造函数。

```java
var lock = new DefaultKeyedLock<String>(key -> {
    return new Semaphore(1, true);
});
```

参数是当前 key：

```text
key -> Semaphore
```

因此你可以根据 key 创建不同的 `Semaphore`。

例如，默认所有 key 都是公平锁：

```java
var lock = new DefaultKeyedLock<String>(key -> new Semaphore(1, true));
```

也可以使用非公平模式：

```java
var lock = new DefaultKeyedLock<String>(key -> new Semaphore(1, false));
```

公平模式和非公平模式的区别主要在等待线程的唤醒顺序。

```text
fair = true     更倾向于按等待顺序获取许可
fair = false    可能有更高吞吐，但不保证等待顺序
```

### 每个 key 允许多个并发

虽然库名叫 Lock，但默认实现底层是 `Semaphore`，所以也可以为每个 key 设置多个许可。

例如：

```java
var lock = new DefaultKeyedLock<String>(key -> new Semaphore(2, true));
```

这表示：

```text
同一个 key 最多允许 2 个线程同时进入临界区
第 3 个线程开始等待
```

这种用法更像“按 key 限流”或“按 key 并发控制”，而不是严格互斥锁。

如果需要严格互斥，应使用：

```java
new Semaphore(1, true)
```

这也是默认行为。

## 常见使用场景

### 按用户串行

```java
private final KeyedLock<Long> userLock = new DefaultKeyedLock<>();

public void updateUserBalance(long userID) {
    userLock.lock(userID);
    try {
        // 查询余额
        // 修改余额
        // 保存余额
    } finally {
        userLock.unlock(userID);
    }
}
```

这样可以避免同一个用户的余额更新逻辑在同一个 JVM 内并发执行。

### 按订单串行

```java
private final KeyedLock<String> orderLock = new DefaultKeyedLock<>();

public void changeOrderStatus(String orderID) {
    orderLock.lock(orderID);
    try {
        // 检查订单状态
        // 修改订单状态
        // 写入记录
    } finally {
        orderLock.unlock(orderID);
    }
}
```

这样可以避免同一个订单被多个线程同时修改。

### 按缓存 key 防止重复刷新

```java
private final KeyedLock<String> cacheLock = new DefaultKeyedLock<>();

public Object refreshCache(String cacheKey) {
    cacheLock.lock(cacheKey);
    try {
        // 重新检查缓存
        // 加载数据
        // 写入缓存
        return loadAndPut(cacheKey);
    } finally {
        cacheLock.unlock(cacheKey);
    }
}
```

这样可以避免同一个缓存 key 在同一时间被重复刷新。

### 按文件路径串行

```java
private final KeyedLock<String> fileLock = new DefaultKeyedLock<>();

public void writeFile(String path, byte[] data) {
    fileLock.lock(path);
    try {
        // 写入指定 path
    } finally {
        fileLock.unlock(path);
    }
}
```

这样可以让同一个路径的写入串行执行，而不同路径之间仍然可以并行。

## 与 synchronized 的区别

`synchronized` 通常有两种用法。

一种是锁住当前对象：

```java
synchronized (this) {
    // 临界区
}
```

另一种是锁住某个共享对象：

```java
synchronized (lockObject) {
    // 临界区
}
```

如果直接锁 `this`，那么所有请求都会竞争同一把锁。

```text
user:1 会阻塞 user:2
user:2 会阻塞 user:3
```

而 `KeyedLock` 是按 key 加锁。

```text
user:1 只会阻塞 user:1
user:2 只会阻塞 user:2
user:1 不会阻塞 user:2
```

因此当业务天然有 key 时，`KeyedLock` 可以减少不必要的互相阻塞。

## 与 ReentrantLock 的区别

`ReentrantLock` 是一把明确的锁对象。

```java
private final ReentrantLock lock = new ReentrantLock();
```

如果想实现按 key 加锁，通常需要自己维护：

```text
Map<key, ReentrantLock>
引用计数
锁对象清理
并发创建控制
```

`DefaultKeyedLock` 把这些逻辑封装起来。

你只需要使用：

```java
lock.lock(key);
try {
    // 临界区
} finally {
    lock.unlock(key);
}
```

需要注意，`DefaultKeyedLock` 默认不是可重入锁。

在同一个线程中连续对同一个 key 调用两次 `lock(key)`：

```java
lock.lock("A");
lock.lock("A");
```

默认情况下第二次调用会等待第一次调用释放许可。

如果没有其它代码先执行 `unlock("A")`，就会造成自等待。

因此不要把它当作 `ReentrantLock` 使用。

## 线程中断

`DefaultKeyedLock#lock` 使用的是：

```java
Semaphore#acquireUninterruptibly()
```

这意味着：

```text
等待锁时不会抛出 InterruptedException
即使线程被中断，也会继续等待直到拿到锁
```

如果你的业务需要“等待锁时可以响应中断”，当前接口并没有提供 `lockInterruptibly` 这类方法。

这种情况下可以考虑：

```text
在业务层控制超时或取消
扩展新的 KeyedLock 实现
直接使用更适合的并发工具
```

## key 的选择

key 的选择很重要。

好的 key 应该能准确表达你希望串行化的资源范围。

例如按用户串行：

```java
lock.lock(userID);
```

按订单串行：

```java
lock.lock(orderID);
```

按租户和资源组合串行：

```java
record ResourceKey(long tenantID, String resourceID) {}

lock.lock(new ResourceKey(tenantID, resourceID));
```

不要使用过粗的 key。

例如：

```java
lock.lock("global");
```

这样所有任务都会竞争同一个 key，效果接近全局锁。

也不要使用过细且不稳定的 key。

例如每次都创建一个不相等的新对象作为 key，即使它们代表同一个业务资源，也无法达到串行效果。

## null key

`DefaultKeyedLock` 内部使用 `ConcurrentHashMap`。

`ConcurrentHashMap` 不支持 `null` key。

因此不要传入：

```java
lock.lock(null);
```

也不要传入：

```java
lock.unlock(null);
```

key 应该始终是非空值。

## 异常安全

使用锁时最重要的规则是：

```text
lock 成功后，必须确保 unlock 最终执行
```

推荐固定写法：

```java
lock.lock(key);
try {
    // 临界区
} finally {
    lock.unlock(key);
}
```

如果临界区中有返回值，也一样：

```java
public Result update(String key) {
    lock.lock(key);
    try {
        return doUpdate(key);
    } finally {
        lock.unlock(key);
    }
}
```

如果临界区中抛出异常，`finally` 仍然会执行。

```java
lock.lock(key);
try {
    throw new RuntimeException("error");
} finally {
    lock.unlock(key);
}
```

这样可以避免锁永久不释放。

## 完整示例：按订单更新

```java
import dev.scx.lock.DefaultKeyedLock;
import dev.scx.lock.KeyedLock;

public class OrderService {

    private final KeyedLock<String> orderLock = new DefaultKeyedLock<>();

    public void pay(String orderID) {
        orderLock.lock(orderID);
        try {
            var order = findOrder(orderID);
            if (order.isPaid()) {
                return;
            }
            order.markPaid();
            saveOrder(order);
        } finally {
            orderLock.unlock(orderID);
        }
    }

    private Order findOrder(String orderID) {
        return new Order(orderID);
    }

    private void saveOrder(Order order) {
        // 保存订单
    }

    private static final class Order {

        private final String id;
        private boolean paid;

        private Order(String id) {
            this.id = id;
        }

        private boolean isPaid() {
            return paid;
        }

        private void markPaid() {
            paid = true;
        }

    }

}
```

这个示例中，同一个 `orderID` 的支付逻辑会串行执行。

不同 `orderID` 的支付逻辑可以并行执行。

## 完整示例：按 key 控制并发数

如果希望同一个 key 最多允许 2 个任务同时执行，可以使用自定义 `Semaphore`。

```java
import dev.scx.lock.DefaultKeyedLock;

import java.util.concurrent.Semaphore;

public class ResourceService {

    private final DefaultKeyedLock<String> lock = new DefaultKeyedLock<>(key -> {
        return new Semaphore(2, true);
    });

    public void useResource(String resourceID) {
        lock.lock(resourceID);
        try {
            // 同一个 resourceID 最多 2 个线程同时执行到这里
        } finally {
            lock.unlock(resourceID);
        }
    }

}
```

这时它更像是“按 key 的信号量”。

对于严格互斥，不需要传入自定义构造函数，直接使用默认构造即可。

## 方法总览

### KeyedLock

```java
void lock(T key)
```

获取指定 key 的锁。

如果锁已被占用，则等待。

```java
void unlock(T key)
```

释放指定 key 的锁。

通常必须和 `lock(key)` 成对调用。

### DefaultKeyedLock

```java
DefaultKeyedLock()
```

创建默认按 key 互斥锁。

每个 key 使用一个公平的单许可 `Semaphore`。

```java
DefaultKeyedLock(Function<T, Semaphore> semaphoreBuilder)
```

创建自定义 `Semaphore` 构建逻辑的按 key 锁。

可以控制公平模式，也可以控制每个 key 的许可数量。

## 设计说明

### 1. 只做 JVM 内按 key 加锁

SCX Lock 只在当前 JVM 进程内生效。

如果应用部署了多个进程或多个节点，不同进程之间不会共享锁状态。

因此它不是分布式锁。

如果需要跨进程互斥，需要使用数据库、Redis、ZooKeeper、etcd 或其它外部分布式协调机制。

### 2. 默认使用 Semaphore 而不是 synchronized

`DefaultKeyedLock` 使用 `Semaphore`，不是 `synchronized`。

这样有几个特点：

```text
可以使用公平队列
可以通过自定义 Semaphore 控制许可数量
可以由不同线程释放同一个 key 的锁
```

默认情况下，每个 key 是一个公平的单许可 `Semaphore`。

```java
new Semaphore(1, true)
```

### 3. 锁对象按需创建并自动清理

`DefaultKeyedLock` 不会提前为所有 key 创建锁对象。

只有调用 `lock(key)` 时，才会为该 key 创建对应的内部对象。

当该 key 没有持有者也没有等待者时，内部对象会被移除。

这样可以避免 key 数量不断增长时，锁对象永久堆积。

### 4. lock 和 unlock 不记录线程所有权

`DefaultKeyedLock` 不记录“哪个线程持有了锁”。

它只根据 key 找到对应的 `Semaphore`，然后获取或释放许可。

因此调用者需要自己保证：

```text
lock 和 unlock 成对出现
unlock 的 key 和 lock 的 key 一致
不要多次 unlock
不要漏掉 unlock
```

最简单可靠的方式就是始终使用：

```java
lock.lock(key);
try {
    // 临界区
} finally {
    lock.unlock(key);
}
```

### 5. 默认不是可重入锁

`DefaultKeyedLock` 默认每个 key 只有一个许可。

同一个线程在没有释放前，再次获取同一个 key，会和其它线程一样等待许可。

因此它不是 `ReentrantLock` 的按 key 版本。

如果业务逻辑存在嵌套调用，需要避免重复获取同一个 key，或者在业务层重新设计锁的作用域。
