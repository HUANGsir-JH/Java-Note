
这是 Spring 面试中的“重灾区”，也是实际开发中最容易踩的坑。Spring 事务的本质是 **AOP**，所以大部分失效场景都与 **AOP 代理机制** 或 **数据库底层配置** 有关。

---

### 1. 核心结论 (Core Conclusion)
Spring 事务失效的根本原因通常归结为三类：**代理机制被绕过（非 Public、类内部调用）**、**异常处理不当（捕获未抛出、异常类型不匹配）**、以及**底层基础不支持（非单例 Bean、数据库引擎不支持）**。

---

### 2. 深度解析 (Detailed Breakdown)

#### ① AOP 代理层面（最常见）
* **非 Public 方法：** Spring 要求 `@Transactional` 只能修饰 `public` 方法。如果是 `protected`、`private` 或默认作用域，Spring 事务拦截器不会生效。
* **自调用（Self-invocation）：** 同一个类中，方法 A 调用方法 B，而 B 上有 `@Transactional`。
 * *原因：* A 实际上是调用 `this.B()`，绕过了代理对象。
* **Method 是 final 或 static：** CGLIB 代理无法重写 final 方法，导致无法织入事务逻辑。

#### ② 异常处理层面
* **吞掉异常：** 在 `try-catch` 块中捕获了异常但没有重新抛出。Spring 必须感知到异常才能触发回滚。
* **错误的异常类型：** Spring 默认只回滚 **`RuntimeException`** 和 **`Error`**。如果抛出的是检查异常（Checked Exception，如 `IOException`），事务**不会**回滚。
 * *对策：* 使用 `@Transactional(rollbackFor = Exception.class)`。

#### ③ 配置与 Bean 管理层面
* **Bean 没被 Spring 管理：** 如果你用 `new` 出来的对象，或者类上没加 `@Service/@Component`，Spring 根本没机会给它生成代理。
* **多线程调用：** 事务信息是存在 `ThreadLocal` 里的。如果在方法内部开启新线程操作数据库，新线程拿不到主线程的事务上下文。
* **数据库引擎不支持：** 比如 MySQL 使用了 **MyISAM** 引擎（不支持事务），而非 **InnoDB**，那代码写得再对也没用。

#### ④ 传播属性设置错误
* 使用了 `Propagation.NOT_SUPPORTED`（以非事务方式运行）或 `Propagation.NEVER`。

---

### 3. 面试官视角：追问与陷阱 (Interviewer's Lens)

#### 问：如何解决类内部调用导致的事务失效？
* **方案 1：** 注入自己（Self-injection）。在 A 类中通过 `@Autowired` 注入 A。
* **方案 2：** 使用 `AopContext.currentProxy()` 获取当前代理对象（需配置 `exposeProxy=true`）。
* **方案 3：** 拆分到不同的 Service 类中。

#### 问：事务方法在子线程里执行，能回滚吗？
* **答：** **不能**。Spring 的事务管理器（TransactionManager）通过 `ThreadLocal` 获取当前线程的数据库连接（Connection）。新线程会有新的连接，两者不在一个事务内。

#### 核心坑点 (Pitfalls)：
* **只写了注解忘了配置：** 在 Spring Boot 出现前，需要显式开启 `@EnableTransactionManagement`。
* **错误的 RollbackFor 设置：** 很多候选人只知道异常不回滚，但不记得默认只管 `RuntimeException`。

---

### 4. 代码演示 (Code Snippet)

```java
@Service
public class OrderService {

    // 场景 1：内部调用失效
    public void createOrder() {
        // 直接调用，事务不生效
        this.saveOrderToDb(); 
    }

    @Transactional
    public void saveOrderToDb() {
        // DB 操作
    }

    // 场景 2：异常吞掉失效
    @Transactional
    public void updateOrder() {
        try {
            // DB 操作
            throw new RuntimeException("数据库出错了");
        } catch (Exception e) {
            // 异常被吃了，Spring 不知道，事务会提交！
            log.error("手动捕获了异常", e);
        }
    }

    // 场景 3：正确做法
    @Transactional(rollbackFor = Exception.class) // 指定回滚所有异常
    public void correctWay() throws Exception {
        // 逻辑...
        throw new IOException("检查异常也要回滚");
    }
}
```

---

### 5. 总结 (Summary)

* **一看方法：** 必须是 `public` 且不能是内部调用。
* **二看异常：** 必须抛出且默认应为 `RuntimeException`（或配置 `rollbackFor`）。
* **三看环境：** 必须是单例 Bean 且数据库引擎支持。

---

**面试建议：**
回答这类问题时，可以按照“**代理、异常、配置**”这三个维度来分点陈述，会显得你的思路非常清晰。

**既然聊到了事务失效，你是否想顺便了解一下：Spring 的 7 种“事务传播行为”分别在什么场景下使用？或者我们换个方向，聊聊 Redis 的持久化机制？**
