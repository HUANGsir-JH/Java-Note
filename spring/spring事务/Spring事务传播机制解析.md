这个问题可以说是 Spring 事务模块的\*\*“分水岭”\*\*。普通开发者只会无脑用默认配置，而高级开发能根据业务场景准确选择不同的传播机制，甚至能排查出复杂的“部分回滚、部分提交”的诡异 Bug。

---

### 1\. 核心结论 (Core Conclusion)

**事务传播机制 (Propagation)** 解决的是：**当一个带有事务的方法，调用另一个带有事务的方法时，这两个事务该如何协作？**（是合并成一个，还是各自独立？）

Spring 定义了 7 种传播行为，但日常开发和面试中，你只需要牢牢掌握这 3 种核心机制：

1.  `REQUIRED`**（默认）**：同生共死（加入当前事务）。
    
2.  `REQUIRES_NEW`：各自安好（挂起当前，新建独立事务）。
    
3.  `NESTED`：父慈子孝（利用 Savepoint 实现嵌套事务）。
    

**一句话总结：** 传播机制本质上是在控制这几个方法是共用同一个数据库 Connection，还是新开一个 Connection，或者使用 Savepoint（保存点）。

---

### 2\. 深度解析 (Detailed Breakdown)

#### 2.1 生活化类比：去餐厅吃饭打发票

假设你（方法 A）带朋友去吃饭，结账时需要打发票（调用方法 B）。

1.  `REQUIRED`**（必须有，没有就建）：**
    

-   *规则：* 如果你已经有一张账单（事务A），朋友的花销就记在你账单上；如果你没有，就新开一张账单。
    
-   *结果：* 只要其中任何一道菜出问题（抛异常），**整桌菜全退（全部回滚）**。这是最常用的“一损俱损，一荣俱荣”模式。
    

2.  `REQUIRES_NEW`**（必须建新的）：**
    

-   *规则：* 无论你有没有账单，朋友坚持要**单开一张账单**结账。
    
-   *结果：* 如果朋友的账单出问题，只退朋友的菜；如果你的账单出问题，不会影响朋友那张已经结清的账单。两者的底层逻辑是：**挂起当前事务，向连接池申请一个全新的 Connection。**
    

3.  `NESTED`**（嵌套子事务）：**
    

-   *规则：* 还是同一张账单，但是在朋友点菜前，领班用铅笔在账单上画了一条线（**Savepoint 保存点**）。
    
-   *结果：* 如果朋友点的菜没货了，只擦掉铅笔线以下的部分（回滚到保存点），你之前点的菜还在；但如果你最开始点的菜出问题导致整单作废，朋友的菜也会跟着作废。
    

#### 2.2 其余 4 种边缘机制（了解即可，面试官很少深问）

-   `SUPPORTS`：支持。有事务就加入，没有就算了（非事务运行）。
    
-   `NOT_SUPPORTED`：不支持。不管有没有，都挂起当前事务，以非事务运行。
    
-   `MANDATORY`：强制。必须在事务中运行，如果没有就抛异常。
    
-   `NEVER`：绝不。绝不能在事务中运行，如果有就抛异常。
    

---

### 3\. 面试官视角 (The Interviewer's Lens)

**高频追问与致命陷阱（含金量极高）：**

1.  **陷阱 1：方法 A（REQUIRED）调用方法 B（REQUIRED），B 抛出异常，但我 A 里面用** `try-catch` **把 B 的异常吃掉了，最后 A 会正常提交吗？**
    

-   *致命回答：* 会提交，因为异常被吃掉了。 (❌)
    
-   *满分回答：* **会抛出** `UnexpectedRollbackException`**！**
    
-   *原理：* 因为它们用的是同一个底层事务。B 发生异常时，虽然你 catch 了，但 B 在退出前已经偷偷把底层的 Connection 标记为 `rollback-only`（只能回滚）了。A 执行完准备 commit 时，一查状态，发现被标记了，于是强制回滚并报错。
    

2.  **追问 2：怎么解决上面的问题？我想让 B 失败不影响 A！**
    

-   *满分回答：* 有两种方法。第一种：把 B 的传播机制改为 `REQUIRES_NEW`，这样 B 用的是新连接，它回滚不会污染 A 的连接状态（前提是 A 要 catch 住 B 抛出的异常）。第二种：把 B 改为 `NESTED`，利用 Savepoint，B 失败只回滚到保存点，A 继续提交。
    

3.  **陷阱 3：滥用** `REQUIRES_NEW` **会导致什么严重后果？**
    

-   *满分回答：* **连接池耗尽死锁！** 因为 `REQUIRES_NEW` 会额外占用一个数据库连接，且外层事务的连接会被一直“挂起”等待。如果并发量大，所有线程都在等待子方法获取新连接，而连接池已经空了，系统直接假死。
    

---

### 4\. 代码演示 (Code Snippet)

**经典实战场景：无论下单成功与否，都必须记录核心操作日志。**

```java
@Service
public class OrderService {
    @Autowired
    private LogService logService;

    // 外层事务：默认 REQUIRED
    @Transactional
    public void createOrder(Order order) {
        try {
            // 1. 核心业务（可能失败抛异常）
            insertOrder(order);
            // 模拟发生异常
            int i = 1 / 0; 
        } catch (Exception e) {
            // 注意：外层发生异常，内层的 REQUIRES_NEW 已经早就提交了，不会受影响！
            throw e; 
        } finally {
            // 2. 记录日志（必须独立事务！）
            // 就算外层报错回滚了，这里的日志依然要保存进数据库
            logService.saveLog("尝试创建订单: " + order.getId());
        }
    }
}

@Service
public class LogService {
    // 内层事务：新建独立事务，脱离外层管控
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(String msg) {
        jdbcTemplate.update("INSERT INTO sys_log (msg) VALUES (?)", msg);
    }
}
```

*(注：为保证AOP生效，*`LogService` *必须是另外一个注入的 Bean，不能在* `OrderService` *内部直接写* `saveLog` *方法调用，原因参考上一题。)*

---

### 5\. 总结 (Summary)

-   **核心痛点：** 解决方法嵌套调用时的事务边界问题（同一个 Connection 还是新 Connection）。
    
-   **三大主力：** `REQUIRED`（默认，同生共死）、`REQUIRES_NEW`（新建连接，各自独立）、`NESTED`（同一个连接，通过 Savepoint 局部回滚）。
    
-   **避坑指南：** 警惕 `REQUIRED` 嵌套下的 `catch` 导致 `UnexpectedRollbackException`；警惕高并发下 `REQUIRES_NEW` 导致连接池枯竭。
    

---

**面试官点评：** “这道题你能把 `REQUIRES_NEW` 和 `NESTED` 的底层物理区别（新建 Connection vs 原 Connection 加 Savepoint）讲透，并且指出被吃掉的异常会导致 `rollback-only` 报错，说明你拥有极强的线上排错能力，完全达到了高级开发的标准。”

**到这里，Spring 事务最核心的底层、传播、失效场景我们都过了一遍。接下来，你想挑战一下 MySQL 的底层原理吗？比如：MVCC（多版本并发控制）是怎么实现“读已提交”和“可重复读”的？**