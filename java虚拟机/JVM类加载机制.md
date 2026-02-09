这是一个非常经典的 JVM 基础面试题。类加载器（ClassLoader）是 Java 语言能够实现“动态性”的核心机制，也是理解热部署、框架插件化、甚至是安全机制的基础。

---

### 1\. 核心结论 (Core Conclusion)

Java 的类加载器主要分为四类，并遵循 **“双亲委派模型” (Parents Delegation Model)**。

1.  **Bootstrap ClassLoader**（启动类加载器）
    
2.  **Extension ClassLoader**（扩展类加载器）
    
3.  **Application ClassLoader**（应用程序类加载器）
    
4.  **Custom ClassLoader**（自定义类加载器）
    

---

### 2\. 深度剖析 (Deep Breakdown)

#### A. 四种类加载器各司其职

-   **Bootstrap ClassLoader (启动类加载器)**
    
-   *负责范围：* 加载 Java 的核心库（`$JAVA_HOME/lib` 目录下的 `rt.jar`、`resources.jar` 等）。
    
-   *特殊性：* 它是用 **C++ 编写**的，不是 `java.lang.ClassLoader` 的子类。因此在 Java 代码中尝试获取它时，会返回 `null`。
    
-   **Extension ClassLoader (扩展类加载器)**
    
-   *负责范围：* 加载 Java 的扩展库（`$JAVA_HOME/lib/ext` 目录下的 jar 包）。
    
-   *注意：* 在 Java 9 及以后版本中，它被更名为 **Platform ClassLoader**（平台类加载器）。
    
-   **Application ClassLoader (应用程序类加载器)**
    
-   *负责范围：* 加载用户类路径（ClassPath）上指定的类库。
    
-   *地位：* 它是程序中默认的类加载器，平时我们写的代码大多由它加载。
    
-   **Custom ClassLoader (自定义类加载器)**
    
-   *实现方式：* 继承 `java.lang.ClassLoader` 并重写 `findClass` 方法。
    
-   *应用场景：* 加密后的类文件加载、从网络/数据库读取类、热部署、隔离不同版本的依赖。
    

#### B. 双亲委派模型 (Parents Delegation Model)

**工作原理：**

1.  当一个类加载器收到加载请求时，它首先不会自己尝试加载，而是**把请求委托给父类加载器**去执行。
    
2.  每一层都是如此，直到传到最顶层的启动类加载器（Bootstrap）。
    
3.  只有当父加载器反馈无法完成此加载请求时（它的搜索范围中没有找到所需的类），子加载器才会尝试自己去加载。
    

**为什么要这么做？**

1.  **安全性：** 防止核心 API 被篡改。例如你伪造一个 `java.lang.String`，如果没有双亲委派，系统可能会加载你的伪造类，导致整个 JVM 崩溃。有了这个模型，`String` 永远由 Bootstrap 加载。
    
2.  **避免重复加载：** 保证同一个类在同一个类加载器体系下只被加载一次。
    

---

### 3\. 面试官视角 (Interviewer's Lens)

**经典追问一：如何破坏双亲委派模型？为什么要破坏？**

-   *答案：* 通过重写 `loadClass` 方法（而不是 `findClass`）。
    
-   *原因：*
    
-   **SPI (Service Provider Interface)：** 比如 JDBC。核心库 `java.sql` 由 Bootstrap 加载，但具体的驱动（如 MySQL）由 AppClassLoader 加载。Bootstrap 必须调用子类加载器加载的实现类，这违反了由上而下的委派，通过 **线程上下文类加载器 (Thread Context ClassLoader)** 解决。
    
-   **热部署/热插拔：** 如 Tomcat 为了隔离不同的 Web 应用，或者 OSGi 这种模块化技术。
    

**经典追问二：如果你自己写一个** `java.lang.String` **类，能运行吗？**

-   *答案：* **不能。** 由于双亲委派，它会被委托给 Bootstrap 加载器，而 Bootstrap 发现已经加载过核心 `String` 类了，所以它不会加载你写的那个。如果你强行用自定义加载器加载，JVM 也会报错。
    

---

### 4\. 代码案例：观察类加载器

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // 1. AppClassLoader
        ClassLoader appLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println("AppClassLoader: " + appLoader);

        // 2. ExtClassLoader (PlatformClassLoader in Java 9+)
        ClassLoader extLoader = appLoader.getParent();
        System.out.println("ExtClassLoader: " + extLoader);

        // 3. BootstrapClassLoader
        ClassLoader bootstrapLoader = extLoader.getParent();
        System.out.println("BootstrapClassLoader: " + bootstrapLoader); // 输出为 null
    }
}
```

---

### 5\. 总结 (Summary)

1.  **分级加载：** 从核心库（Bootstrap）到扩展库（Ext），再到用户库（App）。
    
2.  **向上委派：** 优先请示上级，上级搞不定，下级再动手。
    
3.  **安全第一：** 核心类保护神，确保 Java 环境的稳定和一致性。
    

---

**面试官点评：** 你对类加载器的分工和双亲委派的意义理解得很透彻。能提到 Java 9 的变化以及“破坏双亲委派”的真实场景（如 JDBC 和 Tomcat），这是区分“背题选手”和“实战选手”的关键点。

**关联知识深度探测：** 既然说到了 Tomcat 的类加载器，你是否了解： “**为什么 Tomcat 要自定义类加载器？为什么两个 Web 应用里有同名且同路径的类，Tomcat 却不会把它们搞混？**” （这涉及到了类加载器的**隔离性**，是 Web 开发中的重难点。）