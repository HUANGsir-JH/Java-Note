这是一个非常经典的并发编程面试题。在 Java 面试中，这道题不仅考察你对基本锁的使用，更考察你对 JVM 底层优化和 JUC（Java.util.concurrent）包核心组件（如 AQS）的理解深浅。

---

### 1\. 核心结论 (Conclusion First)

`synchronized` 是 Java **内置的关键字**（JVM 层面），简单易用、自动释放锁；而 `ReentrantLock` 是 JUC 包下的**一个类**（API 层面），提供了比 `synchronized` 更强大的灵活性（如响应中断、超时尝试、公平锁、多条件变量），但需要手动释放锁。

---

### 2\. 深度解析 (Detailed Breakdown)

我们可以从以下 5 个维度来深度对比：

#### ① 实现层次与底层原理

-   `synchronized`**：** 是 JVM 级别的关键字。底层依赖操作系统的 Mutex Lock 实现。编译后会在代码块前后生成 `monitorenter` 和 `monitorexit` 指令。
    
-   `ReentrantLock`**：** 是 JDK 级别的类。底层基于 **AQS (AbstractQueuedSynchronizer)** 和 **CAS (Compare And Swap)** 操作实现，完全由 Java 代码控制。
    

#### ② 灵活性与高级功能（ReentrantLock 的杀手锏）

-   **等待可中断：** `synchronized` 一旦等待锁，就会一直死等，无法被中断。`ReentrantLock` 提供了 `lockInterruptibly()`，允许在等待锁的过程中响应中断（别人叫你别等了，你可以放弃等待）。
    
-   **非阻塞尝试拿锁：** `ReentrantLock` 提供了 `tryLock()`（尝试拿锁，拿到返回 true，拿不到立刻返回 false）和 `tryLock(time, unit)`（带超时的尝试）。
    
-   *生活类比：* 去餐厅吃饭，`synchronized` 是死心眼，没有座位就一直在门口傻等；`ReentrantLock` 是聪明人，可以设定等 10 分钟（超时），或者看一眼没座位直接去别家吃（非阻塞）。
    

#### ③ 公平锁 vs 非公平锁

-   `synchronized`**：** 只能是**非公平锁**（新来的线程可能插队，效率高，但可能导致老线程饿死）。
    
-   `ReentrantLock`**：** 默认是非公平锁，但可以在构造函数中传入 `true` 来实现**公平锁**（严格按照排队顺序拿锁）。
    

#### ④ 条件变量（精确唤醒）

-   `synchronized`**：** 配合 `Object` 的 `wait()/notify()` 使用，只能唤醒一个或全部唤醒（相当于只有一个休息室）。
    
-   `ReentrantLock`**：** 配合 `Condition` 使用，可以创建多个条件变量（多个休息室）。比如可以做到“只唤醒消费者线程”而不是唤醒所有线程，这在复杂并发场景下非常有用。
    

#### ⑤ 锁的释放

-   `synchronized`**：** 发生异常或代码块执行完毕时，**JVM 会自动释放锁**。
    
-   `ReentrantLock`**：** 必须在 `finally` 块中手动调用 `unlock()`。如果不写，一旦发生异常，就会造成死锁。
    

---

### 3\. 面试官视角 (The "Interviewer's Lens")

**追问 1：既然** `ReentrantLock` **这么强大，是不是应该完全取代** `synchronized`**？/ 性能谁更好？**

-   **坑：** 很多老旧资料会说 `synchronized` 性能差（重量级锁）。
    
-   **真相：** 自从 Java 1.6 对 `synchronized` 进行了大量优化（引入了偏向锁、轻量级锁、锁消除、锁粗化等机制）后，两者的**性能在一般情况下几乎持平**。
    
-   **官方建议：** 除非你需要 `ReentrantLock` 的特有功能（如超时限制、公平锁、Condition），否则**优先使用** `synchronized`。因为它代码更简洁，且不用担心忘记释放锁导致死锁，JVM 未来也更有可能针对内置关键字进行深度优化。
    

**追问 2：讲讲 AQS 是什么？（高频进阶题）**

-   **考察点：** 这是 `ReentrantLock` 的核心。你需要点出 AQS 是一个基于 `volatile int state`（代表同步状态）和一个 FIFO 的双向链表（等待队列）来实现的同步框架。
    

---

### 4\. 代码示例 (Code Snippet)

对比一下两者的典型写法，注意 `ReentrantLock` 的标准范式：

```java
// 1. synchronized 写法 (简洁明了)
public void syncMethod() {
    synchronized (this) {
        // 业务逻辑
    } // 自动释放锁
}

// 2. ReentrantLock 写法 (严谨范式)
private final ReentrantLock lock = new ReentrantLock(); // 默认非公平锁

public void lockMethod() {
    lock.lock(); // 阻塞式获取锁
    try {
        // 业务逻辑
    } finally {
        // 🚨 绝对不可遗漏：必须在 finally 中释放锁！
        lock.unlock(); 
    }
}
```

---

### 5\. 核心总结 (Summary)

1.  **本质区别：** 一个是自动挡（JVM 内置、自动释放），一个是手动挡（API 类、手动加解锁）。
    
2.  **功能差异：** `ReentrantLock` 胜在功能丰富，支持公平锁、响应中断、超时尝试、多条件变量。
    
3.  **选型原则：** 默认首选 `synchronized`（简单安全、性能已优化），遇到复杂并发控制需求时才上 `ReentrantLock`。
    

**如果你准备挑战更高薪资的岗位，面试官大概率会顺着这个问题追问：“Java 1.6 对** `synchronized` **做了哪些锁升级的优化？”需要我带你梳理一下“无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁”的升级过程吗？或者你想直接进入【模拟面试】模式？**