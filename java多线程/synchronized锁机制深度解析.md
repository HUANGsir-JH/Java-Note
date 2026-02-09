在 Java 面试中，`synchronized` 是最常被问到的基础，但要答出“深度”，你需要从 **字节码、对象头、锁升级** 这三个层面递进回答。

---

### 1\. 核心结论 (Core Conclusion)

`synchronized` 是 Java 的**内置锁**，它通过 JVM 内部的 **Monitor（管程/监视器）** 实现。其本质是让对象充当“锁”的角色，保证同一时刻只有一个线程能进入临界区。

---

### 2\. 深度解析 (Detailed Breakdown)

我们将底层原理拆解为“三位一体”：

#### ① 字节码层面：实现标志

-   **同步代码块**：编译后会生成 `monitorenter` 和 `monitorexit` 指令。
    
-   `monitorenter` 指向同步块开始，尝试获取对象的 Monitor。
    
-   `monitorexit` 指向结束，释放 Monitor（通常有两个，一个用于正常退出，一个用于异常退出，保证锁一定会被释放）。
    
-   **同步方法**：依靠方法修饰符中的 `ACC_SYNCHRONIZED` 标志。JVM 看到这个标志后，会在方法调用前自动获取锁。
    

#### ② 内存层面：对象头 (Object Header)

锁的信息存在哪？存在 Java 对象的 **Mark Word** 区域中。

-   每个 Java 对象在内存中都有一个对象头。
    
-   Mark Word 记录了对象的哈希码、GC 分代年龄、以及最核心的 **“锁状态标志位”**。
    
-   锁的升级过程（无锁、偏向锁、轻量级锁、重量级锁）就是通过修改 Mark Word 中的位来记录的。
    

#### ③ 运行层面：Monitor (对象监视器)

这是最底层的实现（在 HotSpot JVM 中是 `ObjectMonitor`，C++ 实现）。

-   **Owner**：指向当前持有锁的线程。
    
-   **EntryList**：阻塞队列，存放等待获取锁的线程。
    
-   **WaitSet**：等待队列，存放调用了 `wait()` 方法的线程。
    
-   **Count**：计数器，实现**可重入性**（同一个线程每进入一次，计数器 +1，退出 -1）。
    

---

### 3\. 面试官视角：锁升级流程 (Crucial!)

为了优化性能，`synchronized` 并不一上来就是重量级的。

1.  **偏向锁 (Biased Locking)**：
    

-   *背景*：大多数情况下锁其实是由同一个线程多次获得的。
    
-   *动作*：在对象头记录线程 ID。下次该线程再来，直接进，开销极低。
    

2.  **轻量级锁 (Lightweight Locking)**：
    

-   *触发*：当有第二个线程来竞争时。
    
-   *动作*：撤销偏向锁，改用 **CAS (自旋)** 的方式获取锁。不挂起线程，因为挂起线程涉及内核态切换，很慢。
    

3.  **重量级锁 (Heavyweight Locking)**：
    

-   *触发*：竞争激烈，或自旋达到一定次数。
    
-   *动作*：线程挂起，进入阻塞状态，交给操作系统调度。
    

---

### 4\. 代码演示 (Code Snippet)

```java
public class SyncDemo {
    private final Object lock = new Object();

    // 1. 同步代码块：使用 monitorenter/exit
    public void blockMethod() {
        synchronized (lock) {
            System.out.println("Locked block");
        }
    }

    // 2. 同步方法：使用 ACC_SYNCHRONIZED 标志
    public synchronized void syncMethod() {
        System.out.println("Locked method");
    }
}
```

---

### 5\. 常见陷阱与追问 (Interviewer's Lens)

-   **追问 1：为什么** `wait/notify` **必须在** `synchronized` **里面？**
    
-   *核心点*：因为 `wait/notify` 依赖于 `ObjectMonitor`。只有先拿到了锁（`monitorenter`），才能去操作这个对象的 Monitor 对象（EntryList/WaitSet）。
    
-   **追问 2：**`synchronized` **是公平锁吗？**
    
-   *核心点*：**非公平锁**。新来的线程可能会直接尝试 CAS 抢锁，抢不到才进队列。这能提高吞吐量，但可能导致线程“饥饿”。
    
-   **追问 3：Lock 相比于 synchronized 有什么优势？**
    
-   *核心点*：Lock 是 API 层面的，支持公平性、超时、中断、多个 Condition。而 `synchronized` 是关键字，由 JVM 维护，使用更简单。
    

---

### 6\. 总结 (Summary)

1.  **本质**：基于 Monitor 对象的互斥机制。
    
2.  **存储**：锁状态记录在对象头的 Mark Word 中。
    
3.  **进化**：经历了从偏向锁到重量级锁的动态升级，解决了早期性能差的问题。
    

**面试官点评：** 能讲清楚 **Mark Word** 和 **锁升级** 过程的候选人，通常被认为对 JVM 底层有深入研究，这是区分初中级和高级 Java 开发者的分水岭。

**接下来，你想聊聊“CAS 与自旋锁的具体实现”，还是看看“Java 内存模型 (JMM) 如何保证可见性”？**