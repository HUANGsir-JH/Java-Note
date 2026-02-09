
这是一道 Java 并发编程中**基础但极其高频**的面试题。虽然它们都能让线程暂停执行，但其底层的设计哲学和运行机制完全不同。

---

### 1. 核心结论 (Core Conclusion)
**`sleep()` 是“休眠”，`wait()` 是“等待”**。
最本质的区别在于：**`sleep()` 不释放锁，而 `wait()` 会释放锁**。
* `sleep()` 是 `Thread` 类的静态方法，用于控制线程自身的运行节拍。
* `wait()` 是 `Object` 类的方法，用于线程间的通信和同步。

---

### 2. 深度解析 (Detailed Breakdown)

#### ① 锁的持有情况 (核心差异)
* **`Thread.sleep()`**：线程虽然睡觉了，但它依然**死死抱着锁不放**。就像你抱着厕所钥匙在门口睡觉，别人依然进不去。
* **`Object.wait()`**：线程进入等待池前，会**主动交出持有的锁**。就像你发现厕所没纸了，把钥匙还给管理员，然后去休息室坐着等纸来，这时别人可以用这把钥匙进厕所。

#### ② 所属类与适用范围
* **`sleep()`**：来自 `java.lang.Thread` 类。可以在**任何地方**使用。
* **`wait()`**：来自 `java.lang.Object` 类。必须在 **`synchronized` 代码块或方法**中使用，否则会抛出 `IllegalMonitorStateException`。

#### ③ 唤醒条件
* **`sleep(ms)`**：时间到了自动醒，或者被 `interrupt()` 中断抛出异常。
* **`wait()`**：需要别人调用同一个对象的 `notify()` 或 `notifyAll()` 才能醒。当然也可以用 `wait(ms)` 设置超时时间。

#### ④ 线程状态切换
* `sleep()` 使线程进入 **TIMED_WAITING** 状态。
* `wait()` 使线程进入 **WAITING**（或带时间的 TIMED_WAITING）状态。

---

### 3. 面试官视角 (Interviewer's Lens)

#### 进阶追问：
* **追问 1：为什么 `wait()` 方法定义在 `Object` 类中，而不是 `Thread` 类中？**
 * *核心回答*：因为 Java 的锁（Monitor）是绑定在**对象实例**上的，而不是绑定在线程上的。`wait()` 的本质是让出对象锁的控制权，所以它必须由对象（Object）来操作。
* **追问 2：调用 `wait()` 之后，线程被 `notify()` 唤醒，它会立即执行吗？**
 * *核心回答*：**不会**。它首先会从“等待集（Wait Set）”移动到“入口集（Entry Set）”，去重新竞争锁。只有重新抢到锁后，它才会从上次 `wait()` 的地方继续向下执行。
* **追问 3：sleep(0) 有什么作用？**
 * *核心回答*：它触发了一次操作系统层面的**线程调度（Yield）**。它告诉 CPU：“我现在可以交出时间片，让其他同优先级的线程跑一下”，但它不保证一定会切换。

---

### 4. 代码示例 (Code Snippet)

展示 `wait` 释放锁而 `sleep` 占用锁的现象：

```java
public class SleepWaitDemo {
    private static final Object LOCK = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            synchronized (LOCK) {
                try {
                    System.out.println("T1 拿到锁了");
                    // 换成 Thread.sleep(2000)，T2 必须等 2 秒后才能拿到锁
                    // 换成 LOCK.wait(2000)，T2 会立即拿到锁
                    LOCK.wait(2000); 
                    System.out.println("T1 重新苏醒");
                } catch (InterruptedException e) { e.printStackTrace(); }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("T2 拿到锁了，说明 T1 刚才释放了锁！");
            }
        });

        t1.start();
        Thread.sleep(100); // 确保 T1 先运行
        t2.start();
    }
}
```

---

### 5. 总结 (Summary)

| 特性 | Thread.sleep() | Object.wait() |
| :--- | :--- | :--- |
| **锁释放** | **不释放锁** | **释放锁** |
| **所属类** | `Thread` 静态方法 | `Object` 实例方法 |
| **同步要求** | 无要求 | 必须在 `synchronized` 块中 |
| **主要用途** | 暂停执行、控制频率 | 线程间通信（协作） |

---

**面试官点评：**
能回答出“锁释放”是及格，能解释清楚“为什么 wait 在 Object 中”是优秀。

**想挑战更高难度的题目吗？** 比如：**“既然 wait/notify 可以通信，为什么 Java 还要引入 Condition (ReentrantLock)？”** 或者 **“如何防止虚假唤醒（Spurious Wakeup）？”**
