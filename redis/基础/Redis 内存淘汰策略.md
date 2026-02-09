当 Redis 的内存使用达到上限（通过 `maxmemory` 配置）时，为了能够继续提供服务，它必须决定哪些旧数据该被“请出去”为新数据腾地方。这就是 **内存淘汰策略 (Eviction Policy)**。

---

### 1\. 核心结论 (Core Conclusion)

Redis 提供了 **8 种** 淘汰策略，主要分为四大类：**不淘汰、LRU（最近最少使用）、LFU（最不经常使用）、随机淘汰、以及根据过期时间淘汰**。默认策略是 `noeviction`（不淘汰，写满直接报错）。

---

### 2\. 深度原理剖析 (Detailed Breakdown)

我们将这 8 种策略按作用范围分为两组：

#### ① 全局键空间 (allkeys) —— 针对所有 Key

1.  **allkeys-lru**：在所有 key 中，回收最近最少使用的键。（最常用）
    
2.  **allkeys-lfu**：在所有 key 中，回收最不经常使用的键。（4.0+ 版本引入）
    
3.  **allkeys-random**：在所有 key 中，随机回收某些键。
    

#### ② 设置了过期时间的键 (volatile) —— 只针对带 TTL 的 Key

4.  **volatile-lru**：在设置了过期的键中，回收最近最少使用的。
    
5.  **volatile-lfu**：在设置了过期的键中，回收最不经常使用的。
    
6.  **volatile-random**：在设置了过期的键中，随机回收。
    
7.  **volatile-ttl**：回收那些最快要过期的键（TTL 最小的）。
    

#### ③ 拒绝服务 (Default)

8.  **noeviction**：不删除任何键。当内存满时，写操作（如 `SET`, `LPUSH`）会返回错误，但读操作（如 `GET`）可以继续。
    

![pasted_image_9073396a-5a29-410b-9421-c2830f892df0.png](file://D:\CherryStudio\Data\Files\9073396a-5a29-410b-9421-c2830f892df0.png)

---

### 3\. LRU 与 LFU 的深度对比 (The "Brain" of Redis)

这是面试中最常被追问的细节：

-   **LRU (Least Recently Used)：最近最少使用**
    
-   **核心逻辑：** 关注 **“时间点”**。如果一个数据最近没被访问过，那么将来被访问的概率也低。
    
-   **Redis 实现：** Redis 并没有使用严格的链表实现 LRU（为了省内存），而是采用 **近似 LRU 算法**。它通过随机采样 5 个键，然后淘汰其中最久没被访问的那个。
    
-   **LFU (Least Frequently Used)：最不经常使用**
    
-   **核心逻辑：** 关注 **“频率”**。如果一个数据访问频率极高，即使最近没被访问，也不该被淘汰。
    
-   **解决痛点：** 解决了 LRU 的“冷数据瞬间霸榜”问题（比如一个一年没用的数据刚才被偶然点了一下，LRU 就会认为它是热点）。
    
-   **实现：** 内部使用了一个 8 bit 的计数器，值随访问增加，随时间流逝而减少（衰减机制）。
    

---

### 4\. 面试官视角 (Interviewer's Lens)

#### 问：如何选择合适的策略？

-   **allkeys-lru**：如果你的应用请求符合“幂律分布”（部分数据热门，部分冷门），选它。
    
-   **allkeys-random**：如果请求是循环均匀分布的，选它。
    
-   **volatile-ttl**：如果你想让 Redis 表现得像一个纯粹的“带期限缓存”，选它。
    
-   **LFU**：如果你的系统中有明显的热点数据，且热点会随时间缓慢变化。
    

#### 问：Redis 什么时候执行淘汰？

-   *答案：* 它是 **“被动执行”** 的。每当客户端执行一条新命令时，Redis 都会检查内存。如果超过 `maxmemory`，就会根据策略立即触发一次淘汰，直到内存降到限制以下。
    

#### 潜在陷阱 (Pitfalls)：

-   **内存碎片：** 即使执行了淘汰，操作系统显示的内存占用可能并不会立刻减少。这是因为内存分配器（如 jemalloc）还没把空间还给系统。
    
-   **主从同步：** 主节点执行淘汰后，会向从节点发送一条 `DEL` 命令，保证主从数据一致。
    

![pasted_image_7a31c6fc-2ad1-408c-8f1f-6ebe401f38bc.png](file://D:\CherryStudio\Data\Files\7a31c6fc-2ad1-408c-8f1f-6ebe401f38bc.png)![pasted_image_f7816546-40de-4b06-b27a-e75a7d99879f.png](file://D:\CherryStudio\Data\Files\f7816546-40de-4b06-b27a-e75a7d99879f.png)

---

### 5\. 总结 (Summary)

-   **默认不删：** `noeviction`。
    
-   **主流选择：** `allkeys-lru`。
    
-   **精准打击：** `LFU`（考虑频率）优于 `LRU`（只考虑时间）。
    
-   **内存保护：** 必须设置 `maxmemory`，否则可能导致服务器 OOM 崩溃。
    

**既然提到了 LRU 的近似算法实现，你是否有兴趣了解一下：如果让你手写一个标准的 LRU Cache（比如用 Java 的 LinkedHashMap），你会怎么写？或者是我们聊聊 Redis 的过期键删除策略（定时删除 vs 惰性删除）？**