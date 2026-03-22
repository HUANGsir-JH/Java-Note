
作为资深面试官，我会从**底层原理、攻击手段、防范方案**这三个维度来考察你对 SQL 注入（SQL Injection）的理解。这不仅是一个安全问题，更是对“数据与指令边界”理解的深度体现。

---

### 💡 核心结论
**SQL 注入**是指攻击者通过在应用程序的输入字段中插入恶意的 SQL 代码片段，骗过后端程序，使其将这些**恶意代码当做 SQL 命令执行**。
* **本质原因**：程序没有对用户输入进行严格过滤或转义，导致**“数据”被误认为是“指令”**。

---

### 🔍 深度拆解：它是如何发生的？

#### 1. 场景还原：脆弱的代码
假设后端有一段简单的登录校验代码（伪代码）：
```sql
-- 后端拼接出来的 SQL 语句
SELECT * FROM users WHERE username = '$user_input' AND password = '$password_input';
```

#### 2. 攻击者手段：改变逻辑
如果攻击者在 `username` 框输入：`admin' OR '1'='1`，而 `password` 随便填。
最终拼出来的 SQL 变成：
```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = 'xxx';
```
* **结果**：由于 `'1'='1'` 永远为真（True），这个 `WHERE` 子句总能通过校验。攻击者无需密码即可登录 `admin` 账号。

#### 3. 更严重的后果：拖库与删库
攻击者可以使用 `UNION` 操作符或堆叠查询：
* **获取数据库版本**：`' UNION SELECT 1, version(), 3 -- `
* **删除表**：`'; DROP TABLE users; -- `

---

### 🛡️ 如何防范：三道防线

这是面试中的重点，你需要给出**工程化**的解决方案。

#### 1. 预编译语句 (Prepared Statements / Parameterized Queries) —— **终极武器**
这是目前最有效、最标准的防御方式（如 Java 的 `PreparedStatement`，Go 的 `db.Query`）。
* **原理**：先发送 SQL 模板给数据库进行编译，然后再发送参数。
* **关键点**：数据库在执行时，已经确定了 SQL 的逻辑结构。**参数部分只会被当成纯字符串处理**，绝对不会被当成指令执行。
* *例子*：即便输入 `OR '1'='1'`，数据库也只会去查找用户名真的叫 `OR '1'='1'` 的人。

#### 2. 使用 ORM 框架 (MyBatis, Hibernate, GORM)
* **优势**：成熟的 ORM 默认使用预编译。
* **注意**：要警惕框架中的“原始拼接”。比如 MyBatis 中，**使用 `#{}` 是安全的（预编译）**，而使用 `${}` 是直接拼接字符串（不安全）。

#### 3. 输入合法性校验 & 最小权限原则
* **校验**：对于 ID 字段，只允许输入数字；对于日期，只允许日期格式。
* **最小权限**：Web 应用的数据库账号不应拥有 `DROP TABLE` 等高级权限。

---

### 🧐 面试官视角：进阶陷阱 (The "Interviewer's Lens")

#### 1. “既然有了预编译，为什么还会发生 SQL 注入？”
* **深度回答**：有些场景无法使用预编译。比如 **`ORDER BY` 后的字段名**、**表名**。预编译只能替换“值”，不能替换“结构件”。
* **对策**：这类场景必须使用**白名单校验**（只允许输入固定的几个排序字段名）。

#### 2. “什么是‘盲注’ (Blind SQL Injection)？”
* **核心回答**：当页面不直接回显数据库报错信息时，攻击者通过页面的**响应时间（时间盲注）**或**页面内容微小差异（布尔盲注）**来推测数据。
* *例子*：`IF(ASCII(SUBSTR(password,1,1))=104, SLEEP(5), 0)`。如果服务器卡了 5 秒才返回，说明密码第一个字母是 'h'。

#### 3. “前端做过滤能防住 SQL 注入吗？”
* **避坑指南**：**完全不能！** 前端校验只能提升用户体验。攻击者可以轻易绕过浏览器，直接用 `Postman` 或 `curl` 向后端接口发送恶意字符串。**防御必须在后端闭环。**

---

### 💻 极客演示：安全 vs 不安全 (Java 示例)

**❌ 危险写法（字符串拼接）：**
```java
String sql = "SELECT * FROM users WHERE id = " + request.getParameter("id");
Statement statement = connection.createStatement();
ResultSet rs = statement.executeQuery(sql); // 极易被注入
```

**✅ 安全写法（预编译）：**
```java
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement pstmt = connection.prepareStatement(sql);
pstmt.setInt(1, Integer.parseInt(request.getParameter("id"))); // 强制类型转换 + 预编译
ResultSet rs = pstmt.executeQuery();
```

---

### 📝 一分钟总结 (Summary)

1. **定义**：将恶意代码混入数据，欺骗数据库执行非法指令。
2. **根源**：信任了不该信任的用户输入。
3. **防范核心**：**强制使用预编译语句（Prepared Statement）**，杜绝字符串拼接。

---

**掌握了 SQL 注入，你已经具备了基础的安全意识。想聊聊其他 Web 安全问题吗？**
A. **XSS (跨站脚本攻击)**：如何防止别人在你的网页里执行 JS？
B. **CSRF (跨站请求伪造)**：如何防止别人冒充你的身份发帖？
C. **JWT 安全性**：如果 Token 被截获了，攻击者能做什么？
