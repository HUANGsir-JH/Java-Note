这个问题问得非常精准。Redis 的哈希表（Dict）扩容不仅涉及到数据的搬迁，更体现了其对**高性能**和**主线程不阻塞**的极致追求。

---

### 1\. 核心结论 (Core Conclusion)

Redis 的哈希表扩容采用的是 **渐进式 Rehash (Progressive Rehashing)**。它不是一次性把所有数据搬完，而是将搬迁工作平摊到后续的每一次增删改查操作中，从而避免了因海量数据迁移导致的主线程长时间卡顿（O(n) 阻塞）。

---

### 2\. 深度原理剖析 (Detailed Breakdown)

#### ① 双表结构 (The Two-Table Setup)

Redis 的字典结构（`dict`）内部维护了**两个**哈希表：

-   **ht[0]**：日常工作的哈希表。
    
-   **ht[1]**：只在扩容或收缩（Rehash）时才会被创建。
    

#### ② 什么时候触发扩容？ (The Triggers)

Redis 通过 **负载因子 (Load Factor)** 来决定是否扩容：$负载因子 = 哈希表已保存节点数量 / 哈希表大小$

1.  **自动扩容：**
    

-   **情况 A：** 负载因子 $\ge 1$，且 Redis 当前没有执行后台子进程（如 BGSAVE 或 BGREWRITEAOF）。
    
-   **情况 B：** 负载因子 $> 5$，说明哈希冲突极其严重，此时**强制**扩容。
    

2.  **自动收缩：**
    

-   当负载因子 $< 0.1$时，自动触发收缩以节省内存。
    

#### ③ 核心过程：渐进式 Rehash

1.  **分配空间：** 为 `ht[1]` 分配空间。扩容时，`ht[1]` 的大小通常是第一个大于等于 `ht[^0].used * 2` 的 $2^n$幂。
    
2.  **分而治之：** Redis 内部维持一个索引计数器 `rehashidx`，设置为 0，表示 Rehash 开始。
    
3.  **步进式搬迁：**
    

-   **被动搬迁：** 每次对字典执行添加、删除、查找或更新操作时，Redis 除了执行该操作外，还会顺带将 `ht[^0]` 在 `rehashidx` 索引上的所有键值对迁移到 `ht[^1]`，然后 `rehashidx++`。
    
-   **主动搬迁：** Redis 还有定时任务（serverCron），在系统空闲时会分批搬迁。
    

4.  **切换与释放：** 当 `ht[^0]` 的所有数据都迁移到 `ht[^1]` 后，`rehashidx` 设为 -1。将 `ht[^1]` 切换为 `ht[^0]`，释放旧表内存。
    

![pasted_image_6e91ca9c-dfee-4bdd-8cb6-566f872398ca.png](file://D:\CherryStudio\Data\Files\6e91ca9c-dfee-4bdd-8cb6-566f872398ca.png)

---

### 3\. 面试官视角 (Interviewer's Lens)

#### 常见追问：

1.  **Rehash 期间，查询操作怎么查？**
    

-   *答案：* 会**同时查询两个表**。先查 `ht[^0]`，如果没找到，再去 `ht[^1]` 查找。
    

2.  **Rehash 期间，新增操作怎么办？**
    

-   *答案：* 所有新增的键值对会 **直接存入 ht[^1]**，保证 `ht[^0]` 的数据只减不增。
    

3.  **为什么 BGSAVE 时尽量不扩容（负载因子调高到 5）？**
    

-   *核心知识点：* **Copy-On-Write (写时复制)**。
    
-   *答案：* BGSAVE 产生的子进程与父进程共享内存。如果此时 Rehash，会导致大量内存页被修改，从而触发操作系统的写时复制，消耗大量额外内存并降低性能。
    

#### 潜在陷阱 (Pitfalls)：

-   **BigKey 影响：** 如果某个哈希槽位（Bucket）挂载了极长的链表，单步迁移该槽位仍可能导致轻微卡顿。
    
-   **内存翻倍：** 在 Rehash 过程中，由于 ht[^0] 和 ht[^1] 同时存在，内存占用会瞬时增大，需要预留足够的内存空间。
    

---

### 4\. 代码模拟 (Pseudocode)

```c
// Redis 字典内部结构伪代码
typedef struct dict {
    dictht ht[2];       // 两个哈希表
    long rehashidx;     // rehash 进度索引，-1 表示不在 rehash
} dict;

// Rehash 时的单步迁移逻辑
int dictRehash(dict *d) {
    if (d->rehashidx == -1) return 0;
    
    // 找到 ht[0] 中第一个非空的索引
    while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;
    
    // 将该索引下的所有 entry 迁移到 ht[1]
    move_entry(d->ht[0].table[d->rehashidx], d->ht[1]);
    
    // 检查是否全部搬完
    if (d->ht[0].used == 0) {
        free(d->ht[0]);
        d->ht[0] = d->ht[1];
        d->rehashidx = -1;
    }
    return 1;
}
```

---

### 5\. 总结 (Summary)

-   **策略：** 渐进式搬迁，平摊 O(n) 压力。
    
-   **触发：** 负载因子控制，并考虑 Copy-On-Write 优化。
    
-   **影响：** 期间查询双表，新增直接入新表，整体对用户无感。
    

**聊完了 Redis 的内存数据结构，你是否想换个方向，了解一下 Redis 的“持久化”机制（AOF/RDB）？或者是直接进行一场关于“Redis 高可用架构”的模拟面试？**