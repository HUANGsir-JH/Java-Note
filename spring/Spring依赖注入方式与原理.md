
在 Spring 中，**依赖注入（Dependency Injection, DI）** 是 IoC 容器将一个对象所依赖的其他对象注入到其中的过程。它主要有三种常见的实现方式，以及底层依赖的核心技术。

---

### 1. 核心结论 (Core Conclusion)
Spring 主要通过 **构造器注入 (Constructor Injection)**、**Setter 方法注入 (Setter Injection)** 和 **字段注入 (Field Injection)** 三种方式实现依赖注入。

**官方推荐的做法：** 强制依赖使用构造器注入，可选依赖使用 Setter 注入。在实际开发中，字段注入（`@Autowired`）最常用，但并非最佳实践。

---

### 2. 深度解析 (Detailed Breakdown)

#### ① 构造器注入 (Constructor Injection)
* **实现：** 通过类的构造函数来完成注入。
* **优点：**
 1. **保证不变性：** 注入的对象可以声明为 `final`。
 2. **保证完整性：** 在对象被创建时，所有必需的依赖都已就绪，避免了 `NullPointerException`。
 3. **易于测试：** 纯 Java 实例化时非常直观。
* **缺点：** 依赖过多时，构造函数会变得臃肿。

#### ② Setter 方法注入 (Setter Injection)
* **实现：** 在类中提供 `setXXX` 方法，并在方法上标注 `@Autowired`。
* **优点：** 灵活性高，可以在对象创建后动态修改依赖，或者注入可选依赖。
* **缺点：** 对象可能在未完全初始化的情况下被使用，且无法声明为 `final`。

#### ③ 字段注入 (Field Injection)
* **实现：** 直接在变量上使用 `@Autowired` 或 `@Resource` 注解。
* **现状：** 最常用（代码最简洁），但 Spring 官方不推荐。
* **缺点：**
 1. **与容器强耦合：** 脱离 Spring 容器（如简单的单元测试）很难手动注入。
 2. **隐藏依赖：** 类的职责可能变得模糊，容易违反单一职责原则。

#### ④ 底层原理 (Behind the Scenes)
Spring 注入的本质是 **反射（Reflection）**。
1. **扫描：** Spring 启动时扫描注解（或 XML）。
2. **解析：** 通过 `BeanDefinition` 记录依赖关系。
3. **注入：** 在 Bean 的生命周期中，由 `AutowiredAnnotationBeanPostProcessor` 等后置处理器，通过反射调用构造函数、Setter 方法或直接操作私有字段。

---

### 3. 面试官视角：追问与坑点 (Interviewer's Lens)

#### 常见追问：
1. **`@Autowired` 和 `@Resource` 的区别是什么？**
 * *答：* `@Autowired` 是 Spring 提供的，按 **类型（byType）** 注入；`@Resource` 是 JDK (JSR-250) 提供的，默认按 **名称（byName）** 注入。
2. **如果有两个同类型的 Bean，Spring 怎么知道注入哪一个？**
 * *答：* 结合 `@Qualifier` 注解指定名称，或者在其中一个 Bean 上使用 `@Primary`。
3. **为什么不推荐使用字段注入？**
 * *答：* 除了上述解耦问题外，字段注入会使得类在没有 IoC 容器的情况下无法被直接实例化使用，增加了组件的脆弱性。

#### 潜在坑点 (Pitfalls)：
* **循环依赖：** 构造器注入不支持循环依赖（会抛出 `BeanCurrentlyInCreationException`），而 Setter 注入通过“三级缓存”可以解决大部分单例 Bean 的循环依赖问题。

---

### 4. 代码示例 (Code Snippet)

```java
@Service
public class OrderService {

    // 1. 字段注入 (虽然常用，但不推荐)
    @Autowired
    private PaymentService paymentService;

    private final InventoryService inventoryService;
    private LogService logService;

    // 2. 构造器注入 (Spring 官方强烈推荐)
    @Autowired // 如果只有一个构造函数，此注解可省略
    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    // 3. Setter 注入 (用于可选依赖)
    @Autowired
    public void setLogService(LogService logService) {
        this.logService = logService;
    }
}
```

---

### 5. 总结 (Summary)

* **构造器注入：** 强依赖、不可变、保证对象完整，首选。
* **Setter 注入：** 可选依赖、动态注入，备选。
* **字段注入：** 简单快捷，但在大型项目和高质量单元测试中应谨慎使用。

---

**面试官提示：**
如果你在面试中能主动提到“为什么官方不推荐字段注入”以及“构造器注入如何强制依赖”，会显得你对代码质量和 Spring 设计模式有更深的理解。

**接下来，你想聊聊如何解决“循环依赖”这个高频难题，还是想看看 Spring Bean 的生命周期？**
