在 Java 中，垃圾回收（Garbage Collection, GC）是 JVM 自动管理内存的核心机制。它让开发者从繁琐的“手动释放内存”（如 C++ 的 `free/delete`）中解放出来，专注于业务逻辑。

---

### 1\. 核心结论 (Core Conclusion)

**垃圾回收（GC）** 是 JVM 定期自动识别并回收不再使用的对象内存的过程。 其触发时机通常由 JVM 根据内存使用情况自主决定，主要场景包括 **Eden 区满（Minor GC）** 或 **老年代满（Full GC）**。

---

### 2\. 深度剖析 (Deep Breakdown)

#### A. 怎么判断对象是“垃圾”？

在回收之前，JVM 必须先确定哪些对象已经“死”了。

1.  **引用计数法（Reference Counting）：** 给对象添加一个引用计数器。**缺点：** 无法解决循环引用问题（A 引用 B，B 引用 A，除此之外无外部引用），因此主流 JVM 不使用。
    
2.  **可达性分析算法（Reachability Analysis）：** 从一系列称为 **"GC Roots"** 的根对象开始向下搜索。如果一个对象到 GC Roots 没有任何引用链相连，则证明此对象不可用。
    

-   *常见的 GC Roots：* 虚拟机栈中引用的对象、方法区中类静态属性引用的对象、本地方法栈（Native 方法）引用的对象。
    

#### B. 垃圾回收的触发时机 (When is it triggered?)

GC 的触发通常是由于 **“内存分配失败”** 导致的：

1.  **Minor GC (Young GC) 触发：**
    

-   当新生代的 **Eden 区** 空间不足以分配给新创建的对象时，JVM 会触发 Minor GC。这是最频繁的 GC。
    

2.  **Major GC / Full GC 触发：**
    

-   **老年代空间不足：** 对象从新生代晋升到老年代时，发现老年代没地方了。
    
-   **元空间 (Metaspace) 空间不足。**
    
-   **System.gc()：** 程序显式调用，**建议** JVM 进行 GC（但不保证立即执行）。
    
-   **空间分配担保失败：** 统计发现历次晋升到老年代的平均大小大于当前老年代剩余空间。
    

#### C. 回收算法（简述）

-   **复制算法 (Copying)：** 用于新生代。将活对象从 Eden 复制到 Survivor 区，清理 Eden。
    
-   **标记-清除 (Mark-Sweep)：** 用于老年代。标记后直接清除，会产生碎片。
    
-   **标记-整理 (Mark-Compact)：** 用于老年代。标记后将活对象向一端移动，解决碎片问题。
    

---

### 3\. 面试官视角 (Interviewer's Lens)

**经典追问一：**`System.gc()` **真的有用吗？**

-   *答案：* 它只是一个“提示”。调用后，JVM 会尽力去回收，但不保证一定会发生。在生产环境中，通常建议通过 `-XX:+DisableExplicitGC` 禁用它，防止开发人员误调用导致频繁的 **Stop The World (STW)**。
    

**经典追问二：什么是 Stop The World (STW)？**

-   *答案：* 在执行垃圾回收算法时，Java 应用程序的所有线程都会被挂起（除了垃圾回收线程）。STW 时间越短，程序的响应性能越好。这也是各种 GC 收集器（如 G1, ZGC）不断进化的目标。
    

**典型陷阱 (Pitfalls)：**

-   误认为对象被置为 `null` 就会立即被回收。实际上，`null` 只是断开了引用，回收时机仍由 JVM 控制。
    
-   分不清**内存泄漏 (Memory Leak)** 和 **内存溢出 (OOM)**。内存泄漏是指对象不再使用但由于被错误引用导致 GC 无法回收；内存溢出是内存确实不够用了。
    

---

### 4\. 代码示例：模拟 GC 触发

```java
public class GCTest {
    public static void main(String[] args) {
        // 假设 JVM 参数：-Xms20M -Xmx20M -Xmn10M (Eden 8M)
        byte[] allocation1 = new byte[2000 * 1024]; // 2M
        byte[] allocation2 = new byte[2000 * 1024]; // 2M
        byte[] allocation3 = new byte[2000 * 1024]; // 2M
        
        // 此时 Eden 已占用 6M。再分配 3M 时，Eden 不够了
        byte[] allocation4 = new byte[3000 * 1024]; // 触发 Minor GC
    }
}
```

---

### 5\. 总结 (Summary)

1.  **判断：** 走 **可达性分析**（GC Roots），连不上的就是垃圾。
    
2.  **触发：** 主要是 **“位置不够了”**（Eden 满触发 Minor，老年代满触发 Full）。
    
3.  **影响：** GC 期间可能产生 **STW**，导致系统瞬时卡顿。
    

---

**面试官点评：** 你对 GC 的触发逻辑理解很清晰。在实际工作中，我们更多是去**调优** GC，减少 Full GC 的频率。

**延伸讨论：** “**如果你的生产系统频繁发生 Full GC，你会从哪些维度去排查和优化？**” （这是一个非常体现水平的排查实战题，涉及到堆转储分析、JVM 参数调整等，需要我为你梳理排查思路吗？）