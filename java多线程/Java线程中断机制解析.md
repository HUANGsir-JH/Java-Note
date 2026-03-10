在 Java 多线程面试中，`interrupt`（中断）是一个极具迷惑性的知识点。很多初学者认为调用 `interrupt()` 就是“杀死”线程，其实不然。

以下是关于 **Java 线程中断机制** 的深度解析：

---

### 1\. 核心结论 (Core Conclusion)

`interrupt()` 是一种**协作机制（Cooperative Mechanism）**，而不是一种“强制命令”。它仅仅是给目标线程打了一个\*\*“中断标记”\*\*，告诉它：“有人希望你停下来。” 至于线程停不停、什么时候停，完全由线程自己（的代码逻辑）决定。

-   **对比**：`stop()` 方法是暴力的，会直接掐断线程，可能导致锁不释放、数据错乱（已废弃）。`interrupt()` 是绅士的，通过打招呼的方式让线程优雅地收尾。
    

---

### 2\. 深度解析 (Detailed Breakdown)

#### ① 中断的三种状态/方法

1.  `interrupt()`：实例方法。设置目标线程的中断标志位为 `true`。
    
2.  `isInterrupted()`：实例方法。判断目标线程的中断标志位是否为 `true`，**不清除**标志位。
    
3.  `Thread.interrupted()`：**静态方法**。判断当前线程是否中断，并**清除**中断标志位（即将标志位设回 `false`）。
    

#### ② 两种不同的处理场景

根据线程当前的状态，中断的表现不同：

-   **场景 A：线程在运行中（Runnable）**
    
-   此时调用 `interrupt()`，标志位变 `true`。线程如果不去检查这个标志位，它会**继续跑，完全没影响**。
    
-   *代码逻辑*：必须在循环中用 `Thread.currentThread().isInterrupted()` 检查标志。
    
-   **场景 B：线程在阻塞中（Waiting/Sleeping/Joined）**
    
-   此时调用 `interrupt()`，线程会立刻抛出 `InterruptedException`。
    
-   **关键点**：一旦抛出异常，JVM 会自动将该线程的中断标志位**重置为** `false`。
    

#### ③ 为什么要重置标志位？

这是为了给线程一个**复原的机会**。抛出异常是为了唤醒阻塞的线程，让它处理“被中断”这件事（比如关流、放锁）。如果线程决定不退出，重置标志位可以让它继续正常工作。

---

### 3\. 面试官视角 (Interviewer's Lens)

#### 进阶追问（高频陷阱）：

-   **追问 1：你在 Catch 块里抓到了** `InterruptedException` **后，通常怎么处理？**
    
-   *错误回答*：打个日志就完了。
    
-   *高级回答（必杀技）*：**“恢复中断状态”**。因为抛出异常时标志位被清除了，如果你在 catch 里不往外抛异常，你应该再次调用 `Thread.currentThread().interrupt()`。这样上层代码（如线程池）才能感知到这个线程曾经被中断过，从而做出正确反应。
    
-   **追问 2：**`Thread.interrupted()` **和** `isInterrupted()` **有什么区别？**
    
-   *核心点*：前者是静态的，检查当前线程且会**清除状态**；后者是实例方法，检查任意线程且**不改状态**。
    

---

### 4\. 代码示例 (Code Snippet)

展示**优雅地中断**一个线程的标准模板：

```java
public class InterruptTemplate implements Runnable {
    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                // 模拟正常业务逻辑
                System.out.println("Thread is working...");
                
                // 模拟阻塞操作
                Thread.sleep(1000); 
            }
        } catch (InterruptedException e) {
            // 场景：在 sleep 时被中断
            System.err.println("Thread was interrupted while sleeping!");
            // 【重要】恢复中断状态，通知上层调用者
            Thread.currentThread().interrupt();
        } finally {
            // 收尾工作：释放资源、关闭连接
            System.out.println("Cleaning up resources...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new InterruptTemplate());
        t.start();
        Thread.sleep(2500);
        t.interrupt(); // 发起中断信号
    }
}
```

---

### 5\. 总结 (Summary)

-   **本质**：中断是一个**信号量**，通过修改一个布尔值标志位来实现。
    
-   **原则**：线程应该通过不断的检查标志位，或者处理 `InterruptedException` 来实现**两阶段终止**。
    
-   **忌讳**：**吞掉中断异常**。如果在 catch 块里什么都不做，中断信号就会“消失”，可能导致线程该停时停不下来。
    

---

**面试官点评：** 理解 `interrupt` 标志位在“阻塞态”和“运行态”下的不同表现，以及“中断恢复”的操作，是区分初级和中高级开发者的分水岭。

**接下来你可能感兴趣的话题：**

1.  **既然提到阻塞，你知道线程有哪些状态（Life Cycle）吗？**
    
2.  **线程池（ThreadPoolExecutor）在调用** `shutdown()` **和** `shutdownNow()` **时，分别是如何处理中断的？**