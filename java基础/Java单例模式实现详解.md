单例模式（Singleton Pattern）是一种常用的软件设计模式，其核心目标是**确保一个类只有一个实例，并提供一个全局访问点**。

在 Java 中，实现单例模式通常有以下几种方式，每种方式都有其优缺点：

---

### 1\. 饿汉式（Eager Initialization）—— 线程安全

在类加载时就完成了初始化，所以类加载较慢，但获取对象的速度快。

-   **优点：** 线程安全，实现简单。
    
-   **缺点：** 不支持懒加载（Lazy Loading），如果该实例一直没被使用，会浪费内存。
    

```java
public class Singleton {
    // 1. 私有化构造方法，防止外部 new
    private Singleton() {}

    // 2. 内部创建静态实例
    private static final Singleton instance = new Singleton();

    // 3. 提供静态公共方法获取实例
    public static Singleton getInstance() {
        return instance;
    }
}
```

---

### 2\. 懒汉式（Lazy Initialization）—— 线程不安全

只有在第一次调用 `getInstance()` 时才创建实例。

-   **优点：** 实现了懒加载。
    
-   **缺点：** **线程不安全**。在多线程环境下，可能会创建出多个实例。
    

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

---

### 3\. 懒汉式（线程安全版）

通过在方法上加 `synchronized` 锁来保证线程安全。

-   **优点：** 线程安全。
    
-   **缺点：** 每次调用都要同步，**性能极差**。
    

```java
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

---

### 4\. 双重检查锁定（Double-Checked Locking, DCL）—— 推荐

结合了懒加载和高性能。

-   **关键点：** 必须使用 `volatile` 关键字，防止指令重排序。
    
-   **优点：** 线程安全，性能高，懒加载。
    

```java
public class Singleton {
    // 使用 volatile 关键字防止指令重排
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查：避免不必要的同步
            synchronized (Singleton.class) {
                if (instance == null) { // 第二次检查：确保只创建一个实例
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

---

### 5\. 静态内部类（Static Inner Class）—— 推荐

利用了 Java 类加载机制来保证线程安全。

-   **优点：** 既实现了懒加载，又保证了线程安全，代码简洁。
    

```java
public class Singleton {
    private Singleton() {}

    // 静态内部类只有在第一次被引用时才会被加载
    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

---

### 6\. 枚举（Enum）—— 最推荐

这是《Effective Java》作者 Joshua Bloch 提倡的方式。

-   **优点：** 默认线程安全，**绝对防止多次实例化**，甚至能防止反射和序列化破坏单例。
    

```java
public enum Singleton {
    INSTANCE;

    public void someMethod() {
        // 业务逻辑
    }
}

// 使用方式：
Singleton.INSTANCE.someMethod();
```

---

### 总结与选择

| 实现方式 | 线程安全 | 懒加载 | 性能  | 备注  |
| --- | --- | --- | --- | --- |
| **饿汉式** | 是   | 否   | 高   | 简单，适用于必用的对象 |
| **懒汉式** | 否   | 是   | 高   | **不可在多线程环境下使用** |
| **DCL(双重检查)** | 是   | 是   | 高   | 复杂，但性能好 |
| **静态内部类** | 是   | 是   | 高   | **推荐使用**，代码优雅 |
| **枚举** | 是   | 否   | 高   | **最安全**，防止反射破坏 |

**建议：**

-   如果没有特殊的懒加载需求，使用**饿汉式**。
    
-   如果需要懒加载，推荐使用**静态内部类**。
    
-   如果涉及到序列化反序列化或需要绝对防御反射攻击，使用**枚举**。