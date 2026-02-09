如果把 Java 并发包（JUC）比作一座大厦，那么 **AQS (AbstractQueuedSynchronizer)** 就是这座大厦的**地基**。理解了 AQS，你就理解了 Java 中几乎所有同步工具（`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock`）的核心逻辑。

---

### 1\. 核心结论 (Core Conclusion)

AQS 是一个用于构建锁和同步器的**框架**。它利用一个 `volatile int state`**（同步状态）** 和一个 **FIFO（先进先出）的双向队列**，通过 **CAS** 操作来实现线程对资源的访问控制。它采用了**模板方法模式**，将通用的线程排队、阻塞、唤醒逻辑封装好，而将具体的资源获取逻辑（如何定义 state）交给子类实现。

---

### 2\. 深度解析：AQS 的“三大支柱” (Detailed Breakdown)

AQS 的内部实现可以概括为：**状态、CAS、队列、模式**。

#### ① 状态控制：volatile int state

这是同步器的核心。不同的子类对 `state` 有不同的定义：

-   **ReentrantLock**：`state` 表示锁的持有状态（0-未占用，1-已占用，>1-重入次数）。
    
-   **Semaphore**：`state` 表示可用许可的数量。
    
-   **CountDownLatch**：`state` 表示还需要等待的倒计时次数。
    

#### ② 原子操作：CAS

AQS 所有的状态修改都使用 `Unsafe` 类的 **CAS** 操作。比如抢锁时，多个线程同时尝试 `compareAndSetState(0, 1)`，只有一个人能成功。

#### ③ 核心数据结构：CLH 双向队列

当线程抢不到锁（CAS 失败）时，AQS 会将其包装成一个 **Node** 节点，并放入同步队列的尾部。

-   **为什么是双向的？**：
    
-   **节点取消**：如果一个线程在等待时超时或被中断，它需要从队列中移除，双向链表方便找到前驱和后继节点。
    
-   **唤醒机制**：当头节点（Head）释放资源后，它需要从前往后寻找第一个没有被取消的节点进行唤醒。
    

#### ④ 两种运行模式

-   **独占模式 (Exclusive)**：同一时间只能有一个线程持有（如 `ReentrantLock`）。
    
-   **共享模式 (Shared)**：同一时间可以有多个线程持有（如 `Semaphore`、`CountDownLatch`）。
    

---

### 3\. AQS 的工作流程 (Flow)

1.  **尝试获取 (tryAcquire)**：子类实现此方法。如果成功，直接干活。
    
2.  **入队 (addWaiter)**：如果失败，线程被封装成 Node 存入队列尾部。
    
3.  **自旋与阻塞 (acquireQueued)**：
    

-   入队后，线程会检查它的前驱节点是不是 Head。如果是，它会**再次尝试**抢锁（自旋）。
    
-   如果还是抢不到，或者前驱不是 Head，则调用 `LockSupport.park()` 挂起线程，等待唤醒。
    

4.  **释放与唤醒 (release)**：
    

-   持有锁的线程完成后，改写 `state`。
    
-   调用 `LockSupport.unpark()` 唤醒 Head 后的第一个有效节点。
    

---

### 4\. 面试官视角 (Interviewer's Lens)

#### 进阶追问：

-   **追问 1：AQS 为什么使用模板方法模式？**
    
-   *核心回答*：同步器的逻辑极其复杂（涉及线程调度、队列管理），但“如何判断抢锁成功”的业务逻辑却很简单。AQS 负责复杂的脏活累活，子类只需要实现 `tryAcquire` / `tryRelease` 几个简单的方法即可。
    
-   **追问 2：AQS 队列里的第一个节点（Head）存的是什么？**
    
-   *核心回答*：Head 节点是一个“哑节点”（Dummy Node）或者代表当前正持有锁的线程。它不存储等待的线程，真正等待的线程是从 `Head.next` 开始的。
    
-   **追问 3：为什么 AQS 的自旋只判断** `predecessor == head`**？**
    
-   *核心回答*：为了公平和效率。只有 Head 的后继节点才有资格尝试抢锁，防止整个队列的线程都在自旋浪费 CPU（这就是所谓的“惊群效应”）。
    

---

### 5\. 代码示意 (Template Pattern)

子类实现 `ReentrantLock` 的大致逻辑：

```java
// AQS 子类内部实现
protected boolean tryAcquire(int arg) {
    if (getState() == 0 && compareAndSetState(0, arg)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}

// 外部调用 lock() 时
// 1. 调用 AQS 的 acquire(1)
// 2. acquire 会内部调用子类的 tryAcquire
// 3. 失败则入队、挂起
```

---

### 6\. 总结 (Summary)

1.  **State** 是信号灯，**CAS** 是变灯的动作。
    
2.  **Node + 队列** 是排队领号的队伍。
    
3.  **Template Pattern** 是为了让开发者只写“变灯逻辑”，不写“排队逻辑”。
    

---

**面试官点评：** 讲 AQS 时，如果你能画出 **CLH 队列的节点指向**（Prev/Next）以及 **State 变化过程**，面试官会对你的理解深度非常认可。

![pasted_image_fceba703-e3c2-4d94-800b-39aaa945c212.png](file://D:\CherryStudio\Data\Files\fceba703-e3c2-4d94-800b-39aaa945c212.png)

**学完了 AQS 这种“悲观锁”的基础，想了解一下它是如何支撑起** `ReentrantReadWriteLock`**（读写锁）实现“读读共享、读写互斥”的吗？**