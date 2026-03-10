这是一个非常高频的面试题，也是实际开发中很多 Bug 的来源。

### 1\. 核心结论 (Core Conclusion)

**会。** 如果单例 Bean 中存在**可变的成员变量**（有状态的 Bean），那么多个线程同时访问并修改这个变量时，就会产生线程安全问题。因为单例 Bean 在 JVM 中只存在一个实例，其成员变量是存放在堆内存中的**共享资源**。

---

### 2\. 深度剖析 (Deep Breakdown)

#### A. “有状态” vs “无状态” Bean

-   **无状态 Bean (Stateless)：** 类中没有成员变量，或者成员变量也是单例且无状态的（如 Service 注入的 Mapper）。大部分的 Controller 和 Service 都是无状态的。
    
-   *结论：* **线程安全**。因为大家只是调用方法，方法内部的局部变量在每个线程的栈帧里，互不干扰。
    
-   **有状态 Bean (Stateful)：** 类中定义了成员变量，并且有方法去修改这个变量的值（例如计数器、用户 Session 信息）。
    
-   *结论：* **线程不安全**。多线程环境下会出现竞态条件（Race Condition）。
    

#### B. 为什么 Spring 默认单例却不处理线程安全？

-   **性能考量：** 单例可以减少创建对象的开销，减少 GC。
    
-   **设计原则：** Spring 认为 Service 和 Controller 应该被设计成**无状态**的逻辑处理单元，而不是数据承载单元。
    

---

### 3\. 面试官视角 (Interviewer's Lens)

**经典追问一：如何解决单例 Bean 的线程安全问题？**

1.  **尽量设计为无状态：** 这是最推荐的方案。数据通过方法参数传递，不要存在成员变量里。
    
2.  **使用 ThreadLocal：** 将成员变量包装在 `ThreadLocal` 中，为每个线程提供独立的变量副本。
    
3.  **改变 Bean 的作用域：** 使用 `@Scope("prototype")`，每次请求都创建一个新对象。
    
4.  **加锁（Synchronized/Lock）：** 虽能解决问题，但会极大降低系统并发性能，通常不建议。
    

**经典追问二：既然单例不安全，为什么平时写 Service 没出过事？**

-   *答案：* 因为我们平时在 Service 里注入的 `Mapper`、`DAO` 或者其他的 `Service` 本身也是单例且无状态的。我们处理业务逻辑用的是方法内的**局部变量**，它们分配在线程私有的**虚拟机栈**中，自然没有安全问题。
    

---

### 4\. 代码案例：翻车现场

```java
@Service
public class CounterService {
    private int count = 0; // 这是一个有状态的成员变量

    public int addAndGet() {
        // 多个线程同时执行 count++ 
        // count++ 不是原子操作，包含读取-修改-写入三个步骤
        return ++count; 
    }
}
```

*在并发环境下，上述代码返回的结果极大概率会小于预期的执行次数。*

**修正方案（使用 ThreadLocal）：**

```java
@Service
public class SecureCounterService {
    private ThreadLocal<Integer> threadLocalCount = ThreadLocal.withInitial(() -> 0);

    public int addAndGet() {
        threadLocalCount.set(threadLocalCount.get() + 1);
        return threadLocalCount.get();
    }
}
```

---

### 5\. 总结 (Summary)

-   **本质原因：** 单例 Bean 是堆内存中的共享资源，成员变量随对象存在堆中。
    
-   **风险点：** 带有可变成员变量（Stateful）的 Bean。
    
-   **最佳实践：** 坚持 **“无状态设计”**，如果非要存状态，优先考虑 `ThreadLocal` 或 `prototype` 作用域。
    

---

**面试官点评：** 你能区分“有状态”和“无状态”的概念，并联想到虚拟机栈和堆的区别，这说明你的 Java 体系已经串联起来了。

**顺着 ThreadLocal 往下挖：** 你刚才提到了用 `ThreadLocal` 来解决问题。那如果我们在**线程池**环境（比如 Tomcat 的处理线程）中使用 `ThreadLocal` 变量，会有什么隐患吗？（提示：这和“线程复用”以及“内存泄漏”有关）。