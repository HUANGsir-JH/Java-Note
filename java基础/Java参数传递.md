在 Java 中，**参数传递始终是按值传递（Pass-by-Value）**，这是 Java 语言设计的核心原则之一。即使传递的是对象，也并不是传递对象本身，而是传递对象引用的**副本**。

我们可以从两个角度来理解：

---

### ✅ 1. 基本数据类型（Primitive Types）

对于 `int`, `double`, `char`, `boolean` 等基本类型，传递的是该值的**副本**。

```java
public class Example {
    public static void main(String[] args) {
        int a = 10;
        changeValue(a);
        System.out.println(a); // 输出：10，未改变
    }

    public static void changeValue(int x) {
        x = 20; // 只修改了副本
    }
}
```

> 🔍 **结论**：方法内对参数的修改不会影响原始变量。

---

### ✅ 2. 引用类型（Reference Types）——对象

对于 `String`, `ArrayList`, 自定义类等对象，传递的是**对象引用的副本**（即指向堆中对象的地址的副本），而不是对象本身。

```java
public class Example {
    public static void main(String[] args) {
        StringBuilder sb = new StringBuilder("Hello");
        changeReference(sb);
        System.out.println(sb); // 输出：Hello World —— 对象内容被修改了！

        changeObject(sb);
        System.out.println(sb); // 输出：Hello World —— 引用未改变
    }

    public static void changeReference(StringBuilder s) {
        s.append(" World"); // 修改的是同一个对象的内容
    }

    public static void changeObject(StringBuilder s) {
        s = new StringBuilder("New Object"); // 修改的是引用的副本
        s.append("!!!"); // 这个修改只影响副本，不影响原引用
    }
}
```

#### 📌 关键点解析：

| 操作  | 是否影响原始对象 |
| --- | --- |
| 修改对象的内容（如 `sb.append()`） | ✅ 会，因为引用副本指向同一个对象 |
| 修改引用本身（如 `s = new StringBuilder(...)`） | ❌ 不会，因为只是修改了副本，原引用仍指向旧对象 |

> 💡 可以想象：你有一把房子的钥匙（引用），你把钥匙的复印件（副本）交给别人。
> 
> -   如果对方用复印件开门，把房子刷漆了 → 你家的房子里也变了 ✅
>     
> -   如果对方把复印件扔了，拿了一把新钥匙（指向新房子）→ 你家的房子没变 ❌
>     

---

### ❌ 常见误解：“Java 是按引用传递”

很多人误以为 Java 是“按引用传递”，因为对象内容可以被修改。但实际上：

> **Java 传递的是引用的值（即地址的副本），不是引用本身。**

这与 C++ 的 `int&` 引用传递有本质区别：

```cpp
// C++ 示例：真正的引用传递
void change(int& x) { x = 100; } // 直接修改原变量
```

Java 没有这种机制。

---

### ✅ 总结：Java 参数传递的本质

| 类型  | 传递内容 | 是否影响原变量 |
| --- | --- | --- |
| 基本类型 | 值的副本 | ❌ 不影响 |
| 引用类型 | 对象引用的副本 | ✅ 可修改对象内容<br>❌ 不能修改原引用指向 |

> ✅ **一句话总结**： **Java 中所有参数传递都是按值传递，即使是对象，传递的也是引用的“值”（地址副本）。**

---

### 💡 小贴士

-   如果你想在方法中“改变”一个对象引用（让它指向新对象），并让调用者感知到，可以：
    
-   返回新对象（推荐）
    
-   使用包装类（如 `AtomicReference<T>`）
    
-   将对象封装在容器中（如 `List<T>`、`Map<String, T>`）
    

```java
public static StringBuilder changeAndReturn(StringBuilder s) {
    return new StringBuilder("New Object");
}

// 调用时：
sb = changeAndReturn(sb); // 显式赋值，才能生效
```

这样更清晰、符合 Java 的设计哲学。

---

## 一、基本类型参数传递（int）

### 代码

```java
int a = 10;
change(a);

static void change(int x) {
    x = 20;
}
```

---

### ① 调用前的内存状态

```text
栈（Stack）
┌─────────────┐
│ a = 10      │
└─────────────┘

堆（Heap）
（无）
```

---

### ② 调用 change(a) 之后

> **值传递：拷贝一份给 x**

```text
栈（Stack）
┌─────────────┐
│ a = 10      │  ← main 方法
│─────────────│
│ x = 10      │  ← change 方法
└─────────────┘
```

---

### ③ 方法内部修改 x

```text
栈（Stack）
┌─────────────┐
│ a = 10      │
│─────────────│
│ x = 20      │
└─────────────┘
```

➡️ **a 和 x 完全没关系** ➡️ 方法结束，`x` 出栈

---

### ④ 最终结果

```text
a = 10
```

---

## 二、引用类型：修改对象属性（重点）

### 代码

```java
Person p = new Person("Tom");
changeName(p);

static void changeName(Person p) {
    p.name = "Jerry";
}
```

---

### ① 创建对象

```text
栈（Stack）
┌─────────────┐
│ p ─────┐    │
└────────┼────┘
         │
堆（Heap）│
┌────────▼────────┐
│ Person          │
│ name = "Tom"    │
└─────────────────┘
```

📌 栈里放的是 **引用（地址）** 📌 堆里放的才是 **真正的对象**

---

### ② 调用方法（拷贝引用的值）

```text
栈（Stack）
┌─────────────┐
│ p ─────┐    │  ← main
│────────┼────│
│ p ─────┘    │  ← changeName
└─────────────┘
         │
堆（Heap）│
┌────────▼────────┐
│ Person          │
│ name = "Tom"    │
└─────────────────┘
```

⚠️ 注意：

-   **两个 p**
    
-   但 **指向同一个堆对象**
    

---

### ③ 修改对象属性

```text
堆（Heap）
┌─────────────────┐
│ Person          │
│ name = "Jerry"  │  ← 被修改
└─────────────────┘
```

➡️ 不管通过哪个 p，改的都是**同一个对象**

---

### ④ 方法结束

```text
changeName 的 p 出栈
main 的 p 还在
```

结果：

```java
p.name == "Jerry"
```

---

## 三、引用类型：重新指向新对象（不会影响外部）

### 代码

```java
Person p = new Person("Tom");
changePerson(p);

static void changePerson(Person p) {
    p = new Person("Jerry");
}
```

---

### ① 调用前

```text
栈（Stack）
┌─────────────┐
│ p ─────┐    │
└────────┼────┘
         │
堆（Heap）│
┌────────▼────────┐
│ Person          │
│ name = "Tom"    │
└─────────────────┘
```

---

### ② 方法内部 new 新对象

```text
栈（Stack）
┌─────────────┐
│ p ─────┐    │  ← main
│────────┼────│
│ p ─────┐    │  ← changePerson（指向新对象）
└────────┼────┘
         │
堆（Heap）│
┌────────▼────────┐   ┌─────────────────┐
│ Person (Tom)    │   │ Person (Jerry)  │
└─────────────────┘   └─────────────────┘
```

📌 **两个对象** 📌 **两个不同的引用**

---

### ③ 方法结束

```text
changePerson 的 p 出栈
```

```text
栈（Stack）
┌─────────────┐
│ p ─────┐    │
└────────┼────┘
         │
堆（Heap）│
┌────────▼────────┐
│ Person (Tom)    │
└─────────────────┘
```

➡️ main 里的 `p` **完全没变**

---

## 四、终极记忆口诀（强烈建议背）

> 🔥 **Java 方法参数 = 栈中变量的值拷贝**
> 
> -   基本类型：拷贝的是数值
>     
> -   引用类型：拷贝的是地址
>     
> -   改属性 ✔
>     
> -   换对象 ✘
>     

---

## 五、面试一句话版（超稳）

> Java 中不存在引用传递，<br>方法参数传递的是 **引用变量中保存的地址值的副本**，<br>因此可以修改对象内容，但不能改变外部引用指向。