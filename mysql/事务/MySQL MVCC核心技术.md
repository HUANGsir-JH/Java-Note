MVCC（Multi-Version Concurrency Control，多版本并发控制）是 MySQL InnoDB 引擎实现隔离级别的核心技术。

### 1\. 核心结论 (Core Conclusion)

MVCC 的本质是：**“数据快照” + “版本链” + “可见性算法”**。 它通过维护一条记录的多个历史版本，让读操作（快照读）不需要加锁就能查到一致性的数据，从而实现了**读-写并行**，极大提升了数据库的并发性能。

---

### 2\. 深度剖析 (Deep Dive) —— MVCC 的三大支柱

要理解 MVCC，必须看懂这三个底层设计：

#### ① 隐藏字段 (Hidden Columns) —— 每一行的“身份证”

InnoDB 会在每一行数据后面自动加上两个关键的隐藏列：

-   `DB_TRX_ID` **(事务 ID)：** 记录最后一次修改该行数据的事务 ID。
    
-   `DB_ROLL_PTR` **(回滚指针)：** 指向这条记录上一个版本的 **Undo Log** 地址。
    
-   *(注：如果没有主键，还会有一个隐藏的* `DB_ROW_ID`*)*。
    

#### ② Undo Log 版本链 —— 数据的“时光机”

当一个事务修改一行数据时：

1.  先把原始数据拷贝到 **Undo Log** 中。
    
2.  修改原始数据，并将 `DB_TRX_ID` 改为当前事务 ID。
    
3.  将 `DB_ROLL_PTR` 指向刚才拷贝出来的那个 Undo Log。
    

-   **结果：** 这样就形成了一条由回滚指针连接的**历史版本链**（最新的在表里，旧的在日志里）。
    

#### ③ ReadView (一致性视图) —— 读数据的“滤镜”

ReadView 是事务在进行“快照读”时产生的对象。它像一个滤镜，决定了当前事务能看到版本链中的哪一个版本。 ReadView 包含四个核心字段：

1.  `m_ids`：当前系统中所有**活跃且未提交**的事务 ID 列表。
    
2.  `min_trx_id`：`m_ids` 中的最小值。
    
3.  `max_trx_id`：系统即将分配给下一个事务的 ID 值（即目前最大 ID + 1）。
    
4.  `creator_trx_id`：创建这个 ReadView 的事务 ID。
    

---

### 3\. 可见性算法 (Visibility Algorithm) —— 核心判断逻辑

当你读一行数据时，MySQL 会拿着该行当前的 `trx_id` 去跟 ReadView 里的字段比对：

1.  **等于** `creator_trx_id`**？** 说明这行是我自己改的，**可见**。
    
2.  **小于** `min_trx_id`**？** 说明改这行的事务在我出生前就提交了，**可见**。
    
3.  **大于等于** `max_trx_id`**？** 说明改这行的事务是在我创建 ReadView 之后才开启的，**不可见**。
    
4.  **在** `min_trx_id` **和** `max_trx_id` **之间？**
    

-   如果 `trx_id` 在 `m_ids` 列表中：说明改这行的事务还没提交，**不可见**。
    
-   如果不在：说明已经提交了，**可见**。
    

**如果不可见怎么办？** 沿着回滚指针 `DB_ROLL_PTR` 找上一个版本，重复上述判断，直到找到可见的版本。

![pasted_image_682655e6-cf38-41f8-9538-558a8a2f8e7a.png](file://D:\CherryStudio\Data\Files\682655e6-cf38-41f8-9538-558a8a2f8e7a.png)

---

### 4\. 面试官视角：RC vs. RR (Critical Difference)

这是考察你是否深入理解的“杀手锏”：

-   **RC (Read Committed) 级别：**
    
-   **生成时机：** 事务中**每一条 SELECT 语句**执行前，都会重新生成一个新的 ReadView。
    
-   **结果：** 所以你能读到别人刚提交的数据（导致不可重复读）。
    
-   **RR (Repeatable Read) 级别：**
    
-   **生成时机：** 只在事务中**第一条 SELECT 语句**执行前生成一个 ReadView，后续整个事务都复用它。
    
-   **结果：** 整个事务期间看到的“快照”是一模一样的（解决了不可重复读）。
    

---

### 5\. 代码与流程演示 (Logic Flow)

```sql
-- 事务 A (id=10)        -- 事务 B (id=20)
BEGIN;                  BEGIN;
SELECT name FROM user;  -- 产生 ReadView, 此时 m_ids=[10, 20]
                        UPDATE user SET name='张三' WHERE id=1; -- 产生 Undo Log
                        COMMIT;
SELECT name FROM user;  
-- 如果是 RC: 重新生成 ReadView, 此时 B 不在活跃列表, 看到 '张三'
-- 如果是 RR: 复用旧 ReadView, 此时 B 仍在活跃列表(旧快照), 看到原名
```

---

### 6\. 总结 (Summary)

1.  **MVCC = 隐藏列 + Undo Log + ReadView**。
    
2.  **实现读写不冲突：** 读走版本链（快照读），写走加锁（当前读）。
    
3.  **RC 和 RR 的区别：** 本质是 **ReadView 的生成频率**（RC 每句一换，RR 全程不换）。
    

---

**下一阶段建议：** 聊完了 MVCC，你已经掌握了 InnoDB 如何处理“读”的并发。那么对于“写”的并发，**锁机制（记录锁、间隙锁、临键锁）** 是如何配合 MVCC 来防止幻读的呢？要不要深入探讨一下 **“Next-Key Lock”**？