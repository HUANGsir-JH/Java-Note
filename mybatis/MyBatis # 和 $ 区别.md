
这是一个 MyBatis 面试中的**必考题**。它不仅考察你对 MyBatis 框架的熟悉程度，还考察你对 **JDBC 底层原理**以及 **SQL 注入**等安全问题的理解。

---

### 1. 核心结论 (Core Conclusion)
* **`#` (占位符)**：采用 **预编译（PreparedStatement）** 方式，将参数设为 `?`，能有效防止 SQL 注入，是绝大多数场景下的首选。
* **`$` (拼接符)**：直接进行 **字符串替换**，存在 SQL 注入风险，仅用于动态传入表名、字段名或排序方式（Order By）等无法预编译的场景。

---

### 2. 深度解析 (Detailed Breakdown)

#### ① 运行机制的区别
* **使用 `#`**：MyBatis 会将 SQL 中的 `#{}` 替换为 `?`。在执行时，调用 JDBC 的 `PreparedStatement.setXXX()` 方法来赋值。
 * *过程*：`select * from user where id = #{id}` -> `select * from user where id = ?`
* **使用 `$`**：MyBatis 直接将参数的值字面量拼接到 SQL 字符串中。
 * *过程*：`select * from user where name = '${name}'` -> `select * from user where name = 'Jack'`

#### ② 安全性 (SQL 注入)
* **`#` 是安全的**。由于使用了预编译，数据库会将 `?` 处的内容看作纯粹的**列值**，而不是 SQL 指令。即使传入 `' or 1=1 --`，也只会被当做一个普通的字符串搜索。
* **`$` 是危险的**。它会引发 SQL 注入。
 * *攻击示例*：如果代码是 `where password = '${pwd}'`，黑客传入 `' or '1'='1`，SQL 就会变成 `where password = '' or '1'='1'`，从而绕过登录。

#### ③ 自动类型处理
* **使用 `#`**：MyBatis 会自动处理数据类型。比如传入 String 类型，生成的 SQL 会自动加上单引号；传入 Date，会自动格式化。
* **使用 `$`**：必须手动处理。如果你拼的是字符串，必须写成 `'${name}'`，否则 SQL 会报错。

---

### 3. 面试官视角：深度考点 (Interviewer's Lens)

#### 问：既然 `$` 不安全，为什么 MyBatis 还要提供它？
* **答：** 因为 **JDBC 的预编译（`?`）只能代表值，不能代表 SQL 关键字或元数据**。
 * **动态表名**：`select * from ${tableName}`（预编译不支持 `from ?`）。
 * **动态排序**：`order by ${columnName}`（如果用 `#`，SQL 会变成 `order by 'name'`，这在 SQL 中是无效的，必须是 `order by name`）。

#### 问：从性能角度看，这两者有什么区别？
* **答：** **`#` 的性能通常更好**。因为数据库会对预编译的 SQL 进行缓存（执行计划缓存）。如果 SQL 结构不变（只是 `?` 的值在变），数据库可以跳过解析、优化阶段，直接执行。而使用 `$` 每次生成的 SQL 字符串可能都不同，数据库需要重新解析。

#### 核心陷阱 (Pitfalls)：
* **模糊查询**：很多新手写 `LIKE '%#{name}%'`，这会报错。
 * *正确写法*：使用 `LIKE concat('%', #{name}, '%')` 或者 `LIKE "%"#{name}"%"`。

---

### 4. 代码演示 (Code Snippet)

```xml
<!-- 推荐：安全搜索 -->
<select id="findUserById" resultType="User">
  SELECT * FROM users WHERE id = #{id}
</select>

<!-- 特殊：动态列名排序（必须用 $） -->
<select id="findUsersOrder" resultType="User">
  SELECT * FROM users ORDER BY ${sortColumn} ${direction}
</select>

<!-- 错误示例：模糊查询不能直接拼在引号里 -->
<!-- SELECT * FROM users WHERE name LIKE '%#{name}%' (错误) -->
<select id="findUserLike" resultType="User">
  SELECT * FROM users WHERE name LIKE CONCAT('%', #{name}, '%')
</select>
```

---

### 5. 总结 (Summary)

1. **能用 `#` 绝对不用 `$`**：这是为了防御 SQL 注入的最高优先级原则。
2. **`$` 仅用于元数据**：如表名、字段名、排序关键字。
3. **底层原理**：`#` 映射 JDBC 的 `PreparedStatement`，`$` 映射 JDBC 的 `Statement`。

---

**面试小技巧：**
在回答时，提到 **“执行计划缓存”** 或 **“预编译过程”**，能体现你对数据库底层性能优化的理解。

**关于 MyBatis，你是否想了解一下：它的二级缓存机制是如何设计的？或者是它的分页插件 `PageHelper` 的底层拦截器原理？**
