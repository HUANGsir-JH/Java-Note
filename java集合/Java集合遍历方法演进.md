
在 Java 中，遍历集合的方法非常多。这并不是为了把简单问题复杂化，而是随着 Java 版本的演进，为了适应**不同的数据结构、不同的性能需求以及不同的编程范式（如函数式编程）**而产生的。

以下是详细的梳理：

---

### 1. 常用的遍历方法

#### ① 普通 For 循环（基于下标）
```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```
* **适用**：实现了 `RandomAccess` 接口的集合（如 `ArrayList`）。

#### ② 迭代器（Iterator）
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String val = it.next();
    if (val.equals("delete")) it.remove(); // 安全删除
}
```
* **适用**：所有 `Collection` 子类。

#### ③ 增强 For 循环（foreach）
```java
for (String s : list) {
    System.out.println(s);
}
```
* **本质**：它是 `Iterator` 的**语法糖**。代码编译后会被转成 Iterator。

#### ④ Java 8 `forEach` + Lambda
```java
list.forEach(item -> System.out.println(item));
// 或者使用方法引用
list.forEach(System.out::println);
```
* **适用**：简洁的代码编写，适合 Consumer 操作。

#### ⑤ Stream 流遍历
```java
list.stream().filter(s -> s.startsWith("a")).forEach(System.out::println);
```
* **适用**：需要对数据进行过滤、转换、聚合等复杂操作。

#### ⑥ ListIterator（List 特有）
* **特点**：支持双向遍历（向前/向后），支持在遍历时修改、添加元素。

---

### 2. 为什么存在这么多方法？（演进史）

1. **早期（Java 1.0/1.2）**：只有 `Enumeration`（已废弃）和 `Iterator`。目的是提供一种标准方式访问容器，而不必暴露容器的内部结构。
2. **可读性需求（Java 5）**：引入了**增强 For 循环**。因为写 `Iterator` 太繁琐了，开发者想要更简洁的语法。
3. **性能优化需求**：对于 `ArrayList`，下标访问最快；对于 `LinkedList`，用下标访问会导致 $O(n^2)$ 的灾难性能，必须用 `Iterator`。
4. **函数式编程浪潮（Java 8）**：引入了 **Lambda 和 Stream**。
 * 为了支持**声明式编程**（告诉程序“我要做什么”，而不是“怎么去做”）。
 * 为了支持**并行处理**（`parallelStream`），利用多核 CPU 优势。

---

### 3. 异同点与性能对比

| 遍历方式 | 能够删除元素 | 性能特点 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **普通 For** | 可以（但需手动处理下标） | **ArrayList 极快**；LinkedList 极慢。 | 数组结构、需要操作下标。 |
| **Iterator** | **可以（安全删除）** | 稳定，各种结构表现均衡。 | 遍历过程中需要删除元素。 |
| **增强 For** | **不可以**（会抛异常） | 与 Iterator 一致。 | 最常用的普通遍历。 |
| **forEach** | 不可以 | 略慢于增强 For（多一层方法调用）。 | 简单的打印或处理，代码美观。 |
| **Stream** | 不可以 | 单线程略慢；**大数据量并行极快**。 | 复杂过滤、转换、并行计算。 |

---

### 4. 核心考点：Fail-Fast 机制（并发修改异常）

这是面试最常问的：**为什么不能在 foreach 循环里调用 `list.remove()`？**

* **现象**：如果你在增强 For 循环里调用 `list.remove()`，会抛出 `ConcurrentModificationException`。
* **原因**：
 1. foreach 背后是 `Iterator`。
 2. `Iterator` 内部维护了一个 `expectedModCount`（预期的修改次数）。
 3. 当你直接调用 `list.remove()` 时，List 的 `modCount` 增加了，但 `Iterator` 里的 `expectedModCount` 没变。
 4. 下次循环调用 `it.next()` 时，发现两个 Count 不相等，判定有其它线程或逻辑在乱改数据，于是“愤而自杀”（抛出异常）。
* **解决方案**：使用 `Iterator.remove()`。这个方法在删除后会同步更新 `expectedModCount`，所以是安全的。

---

### 5. 总结建议

1. **最通用**：用**增强 For 循环**（foreach），代码最整洁。
2. **要删除**：必须用 **Iterator**。
3. **要处理逻辑/过滤**：用 **Stream**。
4. **追求极致性能**：如果是 `ArrayList` 且循环次数极多，用**普通 For 循环**。
5. **反面教材**：千万不要用普通 For 循环去遍历 `LinkedList`（面试官会直接让你回家）。
