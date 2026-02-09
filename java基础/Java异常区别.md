在 Java 中，`Exception` 和 `RuntimeException` 都是异常类的子类，但它们在**处理机制、使用场景和 Spring Boot 中的应用**上有本质区别。理解这些区别，对编写健壮、可维护的 Spring Boot 应用至关重要。

---

## ✅ 一、核心区别：受检异常（Checked） vs 非受检异常（Unchecked）

| 特性  | `Exception`（一般指受检异常） | `RuntimeException`（非受检异常） |
| --- | --- | --- |
| **父类** | `java.lang.Exception` | `java.lang.RuntimeException`（继承自 `Exception`） |
| **是否必须处理** | ✅ **必须处理**（编译时检查） | ❌ **无需强制处理**（运行时才抛出） |
| **俗称** | 受检异常（Checked Exception） | 非受检异常（Unchecked Exception） |
| **典型例子** | `IOException`, `SQLException`, `ClassNotFoundException` | `NullPointerException`, `IllegalArgumentException`, `ArithmeticException`, `IndexOutOfBoundsException` |
| **设计哲学** | “你可以预见并恢复” | “这是程序逻辑错误，应该被修复” |

> 📌 **关键记忆点**： `RuntimeException` **是** `Exception` **的子类**，但它被 Java 设计为“不需要强制捕获”。

---

## ✅ 二、代码示例对比

### 1\. 受检异常 —— 必须处理

```java
import java.io.FileReader;
import java.io.FileNotFoundException;

public class CheckedExample {
    public void readFile() {
        FileReader file = new FileReader("config.txt"); // 编译报错！
        // 错误：Unreported exception FileNotFoundException; must be caught or declared to be thrown
    }

    // ✅ 正确写法1：try-catch
    public void readFile1() {
        try {
            FileReader file = new FileReader("config.txt");
        } catch (FileNotFoundException e) {
            System.err.println("文件未找到：" + e.getMessage());
        }
    }

    // ✅ 正确写法2：向上抛出
    public void readFile2() throws FileNotFoundException {
        FileReader file = new FileReader("config.txt");
    }
}
```

### 2\. 非受检异常 —— 可不处理（但不推荐忽略）

```java
public class UncheckedExample {
    public void divide(int a, int b) {
        int result = a / b; // 如果 b=0，抛出 ArithmeticException
        // 编译通过！无需 try-catch 或 throws
    }

    public void accessArray(int[] arr, int index) {
        int value = arr[index]; // 如果 index 越界，抛出 ArrayIndexOutOfBoundsException
        // 无需处理，但运行时会崩溃
    }
}
```

> 💡 虽然 `RuntimeException` 不强制处理，但**良好的实践是预防它们**（如校验参数、判空）。

---

## ✅ 三、在 Spring Boot 中的应用

Spring Boot 作为企业级框架，对异常处理有完善的机制，但对两类异常的处理策略不同：

### 🔹 1. 受检异常（Exception）—— 通常用于“可恢复的外部错误”

#### ✅ 适用场景：

-   数据库连接失败（`SQLException`）
    
-   文件读写失败（`IOException`）
    
-   远程服务调用超时（`RemoteAccessException`）
    
-   配置文件缺失
    

#### 💡 Spring Boot 处理方式：

-   通常在 **Service 层抛出**，由 **Controller 层统一捕获**
    
-   常封装成 **自定义业务异常**（继承 `Exception`），便于统一响应
    

```java
// 自定义受检异常（推荐继承 Exception）
public class UserNotFoundException extends Exception {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// Service 层
@Service
public class UserService throw UserNotFoundException {
    public User findById(Long id) {
        User user = userRepository.findById(id);
        if (user == null) {
            throw new UserNotFoundException("用户不存在: " + id);
        }
        return user;
    }
}

// Controller 层：使用 @ControllerAdvice 统一处理
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException e) {
        ErrorResponse error = new ErrorResponse("USER_NOT_FOUND", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

> ✅ **优势**：调用方必须处理，强制开发者考虑“用户不存在”这类业务逻辑，提高程序健壮性。

---

### 🔹 2. 非受检异常（RuntimeException）—— 通常用于“程序错误”或“意外情况”

#### ✅ 适用场景：

-   参数非法：`IllegalArgumentException`
    
-   空指针：`NullPointerException`
    
-   类型转换错误：`ClassCastException`
    
-   数组越界：`IndexOutOfBoundsException`
    
-   业务规则违反：自定义 `BusinessException`（继承 `RuntimeException`）
    

#### 💡 Spring Boot 处理方式：

-   **大量使用** `@ControllerAdvice` **统一捕获**，避免堆栈信息暴露给客户端
    
-   常定义**自定义运行时异常**，用于业务逻辑控制
    

```java
// 自定义业务运行时异常（推荐！）
public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}

// Service 层
@Service
public class AccountService {
    public void withdraw(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId);
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("余额不足，当前余额：" + account.getBalance());
        }
        account.withdraw(amount);
    }
}

// 全局异常处理器
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientBalanceException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientBalance(InsufficientBalanceException e) {
        ErrorResponse error = new ErrorResponse("INSUFFICIENT_BALANCE", e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(Exception.class) // 捕获所有未处理的异常
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        ErrorResponse error = new ErrorResponse("INTERNAL_ERROR", "服务器内部错误");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

> ✅ **为什么用 RuntimeException？** 因为“余额不足”是业务逻辑的一部分，不是“程序错误”，但你**不想强迫每个调用者都 try-catch**。 使用运行时异常，可以让异常自然向上冒泡，由统一入口处理，**代码更简洁、业务更清晰**。

---

## ✅ 四、Spring Boot 最佳实践总结

| 场景  | 推荐异常类型 | 说明  |
| --- | --- | --- |
| **数据库连接失败、文件读取失败等外部依赖问题** | ✅ `Exception`（受检） | 强制调用方处理，体现“可恢复性” |
| **参数校验失败、业务规则违反（如余额不足、库存不足）** | ✅ `RuntimeException`（自定义） | 更符合“业务异常”语义，避免污染调用链 |
| **空指针、数组越界、类型转换错误** | ❌ 不要抛出，应**预防** | 这是代码缺陷，应通过 `Objects.requireNonNull()`、`@Valid` 等避免 |
| **所有异常统一返回 JSON 格式** | ✅ 使用 `@ControllerAdvice` | 无论受检/非受检，统一捕获，返回标准错误结构 |
| **REST API 响应状态码** | ✅ 用 HTTP 状态码表达语义 | 如 404（NotFound）、400（Bad Request）、500（Internal Error） |

---

## ✅ 五、推荐的异常设计模式（Spring Boot 高频模式）

### ✅ 模式一：自定义业务运行时异常（最常用）

```java
public class BusinessException extends RuntimeException {
    private final String code;

    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }

    public String getCode() { return code; }
}

// 使用
if (user == null) {
    throw new BusinessException("USER_NOT_FOUND", "用户不存在");
}
```

### ✅ 模式二：参数校验（推荐用 Bean Validation，避免手动抛 RuntimeException）

```java
@RestController
public class UserController {

    @PostMapping("/users")
    public User createUser(@Valid @RequestBody UserRequest request) {
        // 如果 request 不合法，Spring 会自动抛出 MethodArgumentNotValidException
        return userService.create(request);
    }
}

@RestControllerAdvice
public class ValidationExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining("; "));
        ErrorResponse error = new ErrorResponse("VALIDATION_FAILED", message);
        return ResponseEntity.badRequest().body(error);
    }
}
```

> ✅ 使用 `@Valid` + `@NotBlank`、`@Min` 等注解，比手动 `if (name == null)` 更优雅。

---

## ✅ 六、常见误区

| 误区  | 正确做法 |
| --- | --- |
| ❌ “所有异常都用 RuntimeException” | ✅ 业务逻辑异常用 `RuntimeException`，外部依赖异常用 `Exception` |
| ❌ “受检异常太麻烦，全改成 RuntimeException” | ✅ 受检异常能强制你思考错误处理，是安全的保障 |
| ❌ `throw new Exception("出错了")` | ✅ 定义**语义明确的自定义异常类**，带错误码和日志 |
| ❌ 捕获异常后什么都不做 | ✅ 至少记录日志：`log.error("发生错误", e);` |
| ❌ 在 Controller 里写 try-catch | ✅ 全局用 `@ControllerAdvice` 统一处理 |

---

## ✅ 总结一句话：

> **在 Spring Boot 中：**`Exception` **用于“外部可恢复错误”（如 IO、DB），强制处理；**`RuntimeException` **用于“业务逻辑错误”（如参数非法、余额不足），统一捕获，优雅响应。无论哪种，都推荐使用自定义异常 +** `@ControllerAdvice` **实现标准化错误响应。**

---

## 🎯 最佳实践清单（Spring Boot 开发者必备）

1.  ✅ 所有业务异常继承 `RuntimeException`（如 `BusinessException`）
    
2.  ✅ 外部依赖异常（如数据库、网络）可保留或包装为 `BusinessException`
    
3.  ✅ 使用 `@Valid` 做参数校验，代替手动 `if (xxx == null)`
    
4.  ✅ 用 `@ControllerAdvice` 统一处理所有异常，返回结构化 JSON
    
5.  ✅ 日志记录异常堆栈，但**不向客户端暴露敏感信息**
    
6.  ✅ 为每个异常定义**唯一错误码**（如 `ERR_001`），便于前端和监控系统识别
    

遵循这些原则，你的 Spring Boot 应用将**健壮、清晰、专业**。