太棒了！你能问出这个问题，说明你的 Java 并发编程已经从“会用 API”跨越到了“探究底层引擎”的阶段。

**AQS（AbstractQueuedSynchronizer，抽象队列同步器）** 是整个 Java 并发包（JUC）的**绝对核心**。如果没有 AQS，就不会有 `ReentrantLock`、`CountDownLatch`、`Semaphore`、`ThreadPoolExecutor` 等等。

理解 AQS，就是理解整个 Java 并发体系的基石。

---

### 1\. 核心结论 (Conclusion First)

**AQS 是一个用来构建锁和同步器的框架。** 它的核心思想非常简单：**“状态位 + 休息室”**。它使用一个 `volatile int state` 变量来表示同步状态（锁有没有被占用），并维护了一个 **FIFO（先进先出）的双向链表队列** 来管理那些抢不到锁而被阻塞的线程。

---

### 2\. 深度解析 (Detailed Breakdown)

为了让你彻底弄懂，我们用\*\*“餐厅就餐”\*\*来做一个完美类比： 餐厅里只有一个包厢（独占锁），门口有一个领位员（CAS 机制），和一排等候椅（FIFO 双向队列）。

#### ① 核心组件 1：`state` 变量（包厢状态牌）

-   **原理：** AQS 内部有一个被 `volatile` 修饰的 `int state` 变量。
    
-   **类比：** 包厢门上的状态牌。`0` 表示包厢空着，`1` 表示里面有人。如果是可重入锁，同一个人再进一次，`state` 就变成 `2`。
    
-   **操作：** AQS 强制通过 **CAS（Compare-And-Swap）** 操作去修改 `state`，保证多线程并发修改时绝对的原子性。
    

#### ② 核心组件 2：CLH 双向等待队列（等候椅）

-   **原理：** 如果线程抢锁失败（CAS 修改 `state` 失败），AQS 会把这个线程封装成一个 `Node` 节点，加入到一个 FIFO 的双向链表中。
    
-   **类比：** 顾客（线程）发现包厢有人，领位员就把他安排到门外的等候椅上坐下排队，并让他**睡觉**（通过 `LockSupport.park()` 阻塞线程）。
    
-   **双向链表的优势：** 节点之间有 `prev`（前驱）和 `next`（后继）指针。排队的人可以看看前面的人是不是已经放弃排队了，或者当包厢空出来时，前面的人可以**唤醒**后面的人（`LockSupport.unpark()`）。
    

#### ③ 核心机制 3：模板方法设计模式（定制化你的锁）

AQS 是一个**抽象类**，它把复杂的排队、阻塞、唤醒机制全都封装好了，子类（比如 `ReentrantLock`）只需要实现具体的“抢锁/释放锁”逻辑即可。

-   **独占模式（Exclusive）：** 只有一个线程能执行，如 `ReentrantLock`。子类需实现 `tryAcquire()` 和 `tryRelease()`。
    
-   **共享模式（Shared）：** 允许多个线程同时执行，如 `Semaphore`（信号量）、`CountDownLatch`。子类需实现 `tryAcquireShared()` 和 `tryReleaseShared()`。
    

![pasted_image_fceba703-e3c2-4d94-800b-39aaa945c212.png](file://D:\CherryStudio\Data\Files\fceba703-e3c2-4d94-800b-39aaa945c212.png)

\---

### 3\. 面试官视角 (The "Interviewer's Lens")

**追问 1：AQS 为什么要用“双向链表”而不是单向链表？**

-   **满分回答：**
    

1.  **寻找有效前驱：** 如果一个等待的线程被中断或超时取消了，它需要从队列中剔除。双向链表可以通过 `prev` 指针快速找到前面一个**还在正常排队**的节点，把它和后面的节点连起来。单向链表做不到高效回溯。
    
2.  **唤醒后继节点：** 释放锁时，需要通过 `next` 指针找到下一个节点并唤醒它。
    

**追问 2：公平锁和非公平锁在 AQS 中是怎么体现的？（高频考察点）**

-   **非公平锁（默认）：** 新来的线程（顾客）非常没素质，一上来不管三七二十一，**直接去 CAS 抢锁**。抢不到才去乖乖排队。
    
-   **公平锁：** 新来的线程会先调用 AQS 的 `hasQueuedPredecessors()` 方法，看一眼队列里**有没有人在排队**。如果有人排队，自己绝对不抢，老老实实去队尾排队。
    

---

### 4\. 代码片段 (Code Snippet)

看看 `ReentrantLock` 是如何通过极其简单的几行代码，借用 AQS 实现非公平锁的（伪代码浓缩版）：

```java
// ReentrantLock 内部的非公平同步器，继承了 AQS
static final class NonfairSync extends AbstractQueuedSynchronizer {
    
    // 抢锁逻辑 (面试官最爱看的重写 tryAcquire 方法)
    protected boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState(); // 获取 AQS 的 state
        
        if (c == 0) {
            // 包厢空着！直接 CAS 抢锁（体现了非公平，不管有没有人排队直接抢）
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current); // 标记当前线程占有了锁
                return true;
            }
        }
        // 如果包厢有人，且就是当前线程自己（可重入锁逻辑）
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires; // state 累加
            setState(nextc);
            return true;
        }
        
        // 抢锁失败，返回 false，AQS 会自动把你扔进双向队列排队并阻塞
        return false; 
    }
}
```

---

### 5\. 核心总结 (Summary)

面试时如果被问到 AQS，请直接抛出这三板斧：

1.  **数据结构：** `volatile int state`（同步状态） + 双向 FIFO 链表（等待队列）。
    
2.  **核心操作：** CAS 保证原子性修改状态，`LockSupport.park/unpark` 控制线程的休眠与唤醒。
    
3.  **设计模式：** 采用模板方法模式，将底层的排队阻塞机制与上层的加锁规则彻底解耦，极大地简化了 JUC 组件的开发。
    

**到这里，关于 AQS 的骨架你已经非常清晰了。由于 AQS 依赖 CAS 来保证原子操作，很多面试官在这里一定会紧接着问：“你刚才提到了 CAS，能讲讲什么是 CAS 吗？它有什么缺点（比如 ABA 问题）？” 需要我为你解答这个连环炮吗？**