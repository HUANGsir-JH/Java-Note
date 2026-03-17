
这道题是 Spring 框架面试中的**必考题**，也是实际开发中最容易踩坑的地方。面试官问这个问题，不仅是看你知不知道怎么用，更核心的是想考察你对**底层原理（AOP）**和**避坑经验（事务失效场景）**的掌握程度。

---

### 1. 核心结论 (Core Conclusion)

Spring Boot 控制事务主要有两种方式：**编程式事务**（代码手动控制）和**声明式事务**（`@Transactional` 注解）。
日常开发中绝大多数使用**声明式事务**，它的底层核心原理是基于 **Spring AOP（动态代理）**。Spring 会为打了注解的类生成一个代理对象，在目标方法执行前后自动完成事务的开启、提交或回滚。

**一句话总结：** 你只需贴个 `@Transactional` 标签写业务逻辑，Spring 借助 AOP 代理帮你把“开启事务”和“提交/回滚事务”的脏活累活全包了。

---

### 2. 深度解析 (Detailed Breakdown)

#### 2.1 两种控制方式对比
1. **声明式事务 (`@Transactional`)**
 * **怎么做：** 直接在类或 public 方法上加 `@Transactional` 注解。
 * **优点：** 代码极其简洁，业务逻辑和事务逻辑完全解耦。
 * **缺点：** 粒度较粗（只能精确到方法级别），容易因为不懂 AOP 代理原理而踩坑（导致事务失效）。
2. **编程式事务 (`TransactionTemplate` 或 `PlatformTransactionManager`)**
 * **怎么做：** 在代码里注入 `TransactionTemplate`，手动包裹需要事务的代码块。
 * **优点：** 粒度超细（可以精确到某几行代码），**绝对不会发生代理失效的问题**。
 * **缺点：** 代码侵入性强，显得臃肿。

#### 2.2 底层原理：AOP 代理是怎么干活的？
* *生活化类比：* 假设你要买房（执行业务逻辑），你找了房产中介（**AOP 代理对象**）。
 1. 你告诉中介你要买房（调用方法）。
 2. 中介先去房管局**帮你垫资/开户（开启事务 `connection.setAutoCommit(false)`）**。
 3. 你签合同交钱（执行你的业务代码）。
 4. 如果一切顺利，中介去**办理过户（提交事务 `connection.commit()`）**。
 5. 如果中途你发现房子漏水掀桌子了（**抛出 RuntimeException**），中介负责帮你**退钱（回滚事务 `connection.rollback()`）**。

#### 2.3 核心配置属性
* **`rollbackFor`（重要）：** 默认情况下，Spring 事务只在遇到 `RuntimeException`（运行时异常）和 `Error` 时回滚。如果你抛出的是 `Exception`（如 `IOException`），它**不会回滚**！通常我们强制要求写成 `@Transactional(rollbackFor = Exception.class)`。
* **`propagation`（传播行为）：** 解决多个事务方法互相调用时，事务该怎么融合的问题。最常用的是 `REQUIRED`（默认，加入当前事务）和 `REQUIRES_NEW`（不管当前有没有事务，我都新建一个独立的）。
* **`isolation`（隔离级别）：** 默认使用底层数据库的隔离级别（MySQL 是 Repeatable Read）。

---

### 3. 面试官视角 (The Interviewer's Lens)

**高频追问与致命陷阱（划重点，必背！）：**

1. **陷阱 1：同一个类中，方法 A 没有加事务，方法 B 加了事务。方法 A 内部调用方法 B，事务会生效吗？**
 * *致命回答：* 会生效。 (❌ 直接被 Pass)
 * *满分回答：* **不会生效！** 因为 Spring 事务是基于 AOP 代理的。在同一个类内部互相调用（`this.methodB()`），调用的是目标对象本身的方法，**没有经过代理对象**，所以 AOP 拦截不到，事务直接失效。这就是著名的“**同类自调用失效**”问题。
 * *怎么解决：* 将方法 B 剥离到另一个 Service 类中被调用；或者在类中自己注入自己（`@Autowired` 本类代理对象）再调用。
2. **陷阱 2：我在方法里把异常 `try-catch` 吞掉了，事务还能回滚吗？**
 * *满分回答：* **不能。** AOP 代理是通过捕获你抛出的异常来决定是否回滚的。如果你在代码里 `catch` 了异常并且没有重新 `throw` 出去，代理对象以为方法执行得很顺利，就会把事务**提交**了。
3. **追问 3：为什么 `@Transactional` 只能加在 public 方法上？**
 * *满分回答：* 因为 Spring 默认使用基于 JDK 或 CGLIB 的动态代理，AOP 拦截器在拦截目标方法时，底层源码会检查方法的修饰符。如果不是 public，拦截器会直接放行，不会织入事务逻辑。

---

### 4. 代码演示 (Code Snippet)

**场景 1：最规范的声明式事务写法**
```java
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private StockMapper stockMapper;

    // 规范1：必须 public
    // 规范2：强烈建议加上 rollbackFor = Exception.class
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(Order order) {
        // 1. 扣减库存
        stockMapper.decrease(order.getProductId(), order.getCount());
        
        // 2. 创建订单
        orderMapper.insert(order);
        
        // 如果这里发生了异常（即使是非运行时异常），因为配置了 rollbackFor，库存和订单都会回滚
    }
}
```

**场景 2：解决长事务的杀手锏（编程式事务）**
如果一个方法里有大量耗时的网络调用（比如调用外部发短信接口），用 `@Transactional` 会导致数据库连接长时间被占用（长事务）。此时推荐用编程式事务：

```java
@Service
public class UserService {

    @Autowired
    private TransactionTemplate transactionTemplate; // 注入编程式事务模板

    public void registerUser() {
        // 1. 耗时的网络请求（不要放在事务里！）
        String rpcResult = callSlowRemoteApi(); 

        // 2. 只有这几行核心数据库操作，才放进事务里
        transactionTemplate.execute(status -> {
            try {
                insertUser();
                updateLog();
                return true;
            } catch (Exception e) {
                status.setRollbackOnly(); // 手动标记回滚
                return false;
            }
        });
    }
}
```

---

### 5. 总结 (Summary)

* **两大方式：** 日常爽用 `@Transactional`（声明式），精细化控制防长事务用 `TransactionTemplate`（编程式）。
* **核心原理：** 依赖 Spring AOP 动态代理生成代理类，环绕业务逻辑执行 `commit` 或 `rollback`。
* **避坑指南（事务失效三板斧）：** 别内部自调用、别把异常 catch 吞掉、别标在非 public 方法上、记得配 `rollbackFor = Exception.class`。

---

**面试官点评：** “很多候选人只会说加个注解，一问失效场景就懵。你能把 AOP 代理的本质说清楚，并主动指出 `this` 调用导致失效的底层原因，就已经超越了 70% 的面试者。最后如果能提到用编程式事务解决长事务问题，那就是架构师级别的思维了。”

**需要我顺便跟你过一下“Spring 事务的 7 种传播行为”吗？（比如经典场景：外层方法回滚，怎么保证内层记录日志的方法不回滚？）**
