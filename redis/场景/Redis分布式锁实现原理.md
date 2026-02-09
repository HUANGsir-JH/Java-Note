
实现 Redis 分布式锁，本质上是利用 Redis 的**原子性**在分布式环境下维护一个“占位符”。

---

### 1. 核心结论 (Core Conclusion)

一个严谨的 Redis 分布式锁必须具备三个核心要素：**互斥性**（只有一个能拿到锁）、**防死锁**（超时自动释放）、**解铃还须系铃人**（谁加的锁谁才能解）。
最标准的实现方式是使用：`SET key value NX PX milliseconds` 命令结合 **Lua 脚本**进行释放。

---

### 2. 演进与原理 (Detailed Breakdown)

#### ① 加锁：互斥与原子性
在 Redis 2.6.12 版本之后，加锁只需一条原子命令：
```bash
SET lock_key unique_request_id NX PX 30000
```
* **lock_key**：锁的名称。
* **unique_request_id**：由客户端生成的唯一标识（如 UUID）。**非常重要**，用于后续识别锁的所有者。
* **NX**：只有 Key 不存在时才设置成功（实现互斥）。
* **PX 30000**：设置 30 秒过期时间（防止持有锁的服务器宕机导致死锁）。

#### ② 解锁：身份验证与原子性
释放锁时，不能简单地 `DEL`。如果 A 的业务逻辑执行太慢，锁过期了，B 拿到了锁，此时 A 执行完再去 `DEL`，就会把 B 的锁给删掉。
**正确做法：** 先判断锁的值是不是自己当初设定的 `unique_request_id`，如果是再删除。
为了保证“判断+删除”是原子的，必须使用 **Lua 脚本**：
```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

#### ③ 续期：看门狗机制 (Watchdog)
**问题：** 如果业务还没跑完，锁就过期了怎么办？
**方案：** 像 **Redisson** 这样的成熟框架会启动一个后台线程（看门狗），每隔一段时间（默认 10 秒）检查一下，如果业务还没完，就自动延长锁的过期时间。

---

### 3. 面试官视角 (Interviewer's Lens)

#### 问：如果 Redis 是主从架构，Master 拿到锁后挂了，还没同步到 Slave，怎么办？ (The Failover Trap)
* **答案：** 这是 Redis 分布式锁的经典弱点。由于 Redis 异步复制，Slave 可能还没收到锁信息就变成了 Master，导致另一个客户端也能拿到锁。
* **解决方案：**
 1. 如果业务能容忍极小概率的并发，可以不处理。
 2. 如果追求高可用性，使用 **Redlock（红锁）** 算法：向 5 个独立的 Redis 节点发起加锁请求，超过半数（3个）成功才算成功。但红锁争议较大，运维成本也高。

#### 问：如何实现可重入锁？ (Reentrancy)
* **答案：** 参考 Java 的 `ReentrantLock`。Redis 需改用 **Hash** 结构：
 * `key`: 锁名称。
 * `field`: 线程标识 (UUID+ThreadID)。
 * `value`: 重入计数。
 * 加锁时计数+1，解锁时计数-1，直到为 0 时删除。

#### 问：如果 Redis 内存满了，会导致锁失效吗？
* **答案：** 可能会。如果 Redis 的淘汰策略设为 `allkeys-lru`，可能会把还没过期的锁给删了。
* **建议：** 生产环境分布式锁建议使用独立的 Redis 实例，或将淘汰策略设为 `noeviction`。

---

### 4. 代码演示：Java (Redisson 推荐做法)

虽然可以手写 Lua，但生产环境下 99% 的 Java 项目建议使用 **Redisson**，它封装了所有细节。

```java
// 1. 获取锁实例
RLock lock = redisson.getLock("myLock");

try {
    // 2. 尝试加锁，支持自动续期（Watchdog）
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        try {
            // 执行业务逻辑
            doBusiness();
        } finally {
            // 3. 释放锁
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

### 5. 总结 (Summary)

1. **加锁原子化**：`SET NX PX` 合并执行。
2. **解锁标识化**：必须匹配唯一 Value，通过 Lua 脚本原子删除。
3. **生存自动延期**：业务长时，需引入看门狗机制。
4. **选型建议**：简单场景手写 Lua，复杂场景直接上 Redisson。

**关于 Redis 的一致性，分布式锁其实是一个很好的切入点。你是否想更深一步了解：Redlock 算法为什么被一些分布式专家（如 Martin Kleppmann）批评？或者我们聊聊 Redis 的持久化（RDB/AOF）是如何在故障后保护你的锁数据的？**
