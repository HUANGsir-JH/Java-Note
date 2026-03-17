
这个问题问得非常深入！很多候选人止步于“AOP代理”，但优秀的后端开发必须知道 AOP 背后，Spring 是如何把底层的数据库连接（Connection）玩转的。

如果说上一题是“怎么用”，这道题就是考你“看没看过源码，懂不懂核心机制”。

---

### 1. 核心结论 (Core Conclusion)

Spring 声明式事务的底层实现依赖于 **三大核心组件的无缝协作**：
1. **AOP 动态代理（拦截机制）：** 负责在方法执行前后进行拦截。
2. **PlatformTransactionManager（事务管理器）：** 负责执行具体的开启、提交和回滚动作。
3. **ThreadLocal（连接绑定）：** 负责将同一个数据库连接（Connection）绑定到当前执行的线程上，确保多个 DAO/Mapper 操作使用的是同一个事务。

**一句话总结：** Spring 通过 **AOP** 拦截方法，利用 **TransactionManager** 从数据源获取连接并关闭自动提交，最后把这个连接挂在 **ThreadLocal** 上，让方法内所有的 SQL 都跑在这个连接里，搞定后统一提交或回滚。

---

### 2. 深度解析 (Detailed Breakdown)

我们可以把整个底层执行流程拆解为 **5 个标准步骤**。

#### 2.1 步骤一：生成代理对象 (AOP)
在 Spring 容器启动时，它会扫描所有的 Bean。如果发现类或方法上有 `@Transactional` 注解，Spring 就会利用 JDK 动态代理或 CGLIB 生成一个代理子类。
* 你通过 `@Autowired` 注入的 `UserService`，其实已经不是原来的类了，而是 Spring 包装过的 `UserServiceProxy`。

#### 2.2 步骤二：TransactionInterceptor 拦截 (拦截器)
当你调用代理对象的目标方法时，请求首先会被核心拦截器 `TransactionInterceptor` 拦截。它的 `invoke()` 方法是整个事务控制的“总导演”。

#### 2.3 步骤三：开启事务并绑定线程 (ThreadLocal 的妙用 - 最核心！)
“总导演”会调用 `PlatformTransactionManager`（比如针对 MySQL 的 `DataSourceTransactionManager`）。
1. 管理器从数据库连接池（如 HikariCP）里拿出一个 `Connection`。
2. 执行 `connection.setAutoCommit(false)`，开启原生 JDBC 事务。
3. **关键动作：** Spring 会将这个 `Connection` 对象放入一个名为 `TransactionSynchronizationManager` 的类中，这个类底层使用的是 **`ThreadLocal`**。
 * *为什么要用 ThreadLocal？* 因为你的 Service 方法里可能会调用 `订单Mapper` 和 `库存Mapper`。为了保证它们在同一个事务里，必须让这两个 Mapper 拿到**同一个数据库 Connection**。挂在当前线程的 ThreadLocal 上，大家去拿的时候拿到的就是同一个！

#### 2.4 步骤四：执行业务逻辑
你的真实业务代码开始执行。当 MyBatis 或 JdbcTemplate 要执行 SQL 时，它们不会自己去连接池拿新连接，而是先去 `TransactionSynchronizationManager`（当前线程的 ThreadLocal）里找。一看有现成的连接，就直接用这个连接执行 SQL。

#### 2.5 步骤五：提交或回滚
业务逻辑执行完毕：
* **成功：** 拦截器捕获到方法正常返回，调用 `Manager.commit()`，底层执行 `connection.commit()`。
* **抛出异常：** 拦截器 catch 到异常，检查是否符合 `@Transactional(rollbackFor=...)` 的配置。如果符合，调用 `Manager.rollback()`，底层执行 `connection.rollback()`。
* **清理现场：** 无论成功失败，最后都会将 Connection 从 ThreadLocal 中解绑，恢复 `autoCommit=true`，并归还给连接池。

---

### 3. 面试官视角 (The Interviewer's Lens)

**高频追问与致命陷阱（进阶必问）：**

1. **陷阱：如果在 `@Transactional` 方法里新开了一个子线程（比如用线程池或 `new Thread()`）去执行数据库操作，这个子线程受事务控制吗？**
 * *满分回答：* **不受控制！** 极其容易引发数据不一致。因为 Spring 事务的 Connection 是绑定在当前线程（ThreadLocal）上的。子线程是一个全新的线程，它的 ThreadLocal 是空的，它会去连接池拿一个全新的 Connection，完全脱离了外部的事务管控。
2. **追问：事务管理器（TransactionManager）为什么是一个接口？**
 * *设计模式考点：* 这是典型的**策略模式**。Spring 不知道你底层用的是什么持久层框架。如果你用 MyBatis/JDBC，它就给你注入 `DataSourceTransactionManager`；如果你用 Hibernate/JPA，它就给你注入 `JpaTransactionManager`。Spring 只定规矩，不写死实现。

---

### 4. 源码级伪代码演示 (Code Snippet)

如果你能在面试中手写出 `TransactionInterceptor` 的核心逻辑（哪怕是伪代码），面试官绝对会对你刮目相看：

```java
// 这是 Spring 源码中 TransactionAspectSupport 的 invokeWithinTransaction 方法的极简提炼版本
public Object invokeWithinTransaction(Method method, Class<?> targetClass, InvocationCallback invocation) {
    
    // 1. 获取事务管理器 (PlatformTransactionManager)
    TransactionManager tm = getTransactionManager();
    
    // 2. 获取事务属性 (如传播行为、隔离级别、rollbackFor)
    TransactionAttribute txAttr = getTransactionAttribute(method);
    
    // 3. 开启事务 (从连接池拿 Connection，setAutoCommit(false)，并绑定到 ThreadLocal)
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, method);
    
    Object retVal = null;
    try {
        // 4. 放行！执行你写的真实业务逻辑 (比如你的 UserService.createOrder)
        retVal = invocation.proceedWithInvocation();
    } catch (Throwable ex) {
        // 5. 抓到异常！检查配置并决定是否回滚
        completeTransactionAfterThrowing(txInfo, ex);
        throw ex; // 把异常重新抛出，不能吞掉
    } finally {
        // 清理线程绑定的 TransactionInfo (防止 ThreadLocal 内存泄漏)
        cleanupTransactionInfo(txInfo);
    }
    
    // 6. 正常结束，执行提交
    commitTransactionAfterReturning(txInfo);
    
    return retVal;
}
```

---

### 5. 总结 (Summary)

* **AOP 是骨架：** 通过动态代理生成拦截器，包裹业务代码，无侵入式地织入事务逻辑。
* **ThreadLocal 是灵魂：** 巧妙解决同一个线程跨多个 DAO 层方法共享同一个 Connection 的问题，这是事务“同生共死”的物理基础。
* **Manager 是打工人：** 根据底层的持久化框架（MyBatis/JPA），执行原生的 JDBC `commit` 和 `rollback`。

---

**面试官点评：** “这道题你能把 `ThreadLocal` 的作用点出来，并说明多线程环境下事务失效的原因，说明你不仅懂理论，而且具备排查复杂线上 Bug 的能力。”

**既然提到了同一个事务跨越多个方法，我们来聊聊“Spring 事务的传播机制（Propagation）”吧！你知道 `REQUIRED` 和 `REQUIRES_NEW` 有什么区别吗？**
