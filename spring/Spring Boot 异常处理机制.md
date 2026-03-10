Spring Boot 的异常处理机制非常灵活，它既提供了一套**默认的“保底”方案**，又允许开发者通过**全局拦截**来实现高度自定义的响应。

---

### 1\. 核心结论 (Core Conclusion)

Spring Boot 处理异常遵循 **“就近原则”**：

1.  **首选：** 查找开发者定义的 `@RestControllerAdvice` 中的 `@ExceptionHandler` 方法。
    
2.  **次选：** 查找 Controller 内部定义的 `@ExceptionHandler`。
    
3.  **兜底：** 交给 Spring Boot 默认的 `BasicErrorController`，由它分发到 `/error` 页面（HTML）或返回默认的 JSON 数据。
    

---

### 2\. 深度解析：全局异常处理器是如何实现的？ (Detailed Breakdown)

#### ① 核心注解：`@RestControllerAdvice`

这是目前最通用的方案。它利用了 **Spring AOP（面向切面编程）** 的思想，在不改动业务逻辑的情况下，对 Controller 层的异常进行横切处理。

-   `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`。
    
-   它底层通过 `ExceptionHandlerExceptionResolver` 类来解析抛出的异常，并寻找匹配的处理方法。
    

#### ② 异常匹配逻辑

当一个异常被抛出时，Spring 会在全局处理器中寻找最匹配的类。

-   **精确匹配优先**：如果抛出 `UserNotFoundException`，它会优先匹配处理 `UserNotFoundException` 的方法，而不是 `Exception`。
    
-   **父类兜底**：如果没有精确匹配，则寻找其父类，直到 `RuntimeException` 或 `Exception`。
    

---

### 3\. 代码演示：标准全局异常处理实现 (Code Snippet)

这是在企业级项目中推荐的标准写法：

```java
// 1. 定义统一响应格式
public record Result<T>(int code, String message, T data) {}

// 2. 编写全局异常处理器
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 处理特定的业务异常
    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusinessException(BusinessException e) {
        return new Result<>(e.getCode(), e.getMessage(), null);
    }

    // 处理参数校验异常 (Validation 框架抛出的)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidationException(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return new Result<>(400, msg, null);
    }

    // 兜底：处理所有未预料到的系统异常
    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e) {
        logger.error("系统运行异常: ", e);
        return new Result<>(500, "服务器开小差了，请稍后再试", null);
    }
}
```

---

### 4\. 面试官视角：深度考点 (Interviewer's Lens)

#### 问：404 错误能被 `@ExceptionHandler(Exception.class)` 捕获吗？

-   **答：** **默认情况下不能**。404 意味着没有找到对应的 Handler（控制器方法），因此不会进入 Controller 层，也就不会触发 `@ExceptionHandler`。
    
-   **解决方案**：
    

1.  依靠 Spring Boot 默认的 `BasicErrorController` 处理。
    
2.  在配置文件中开启 `spring.mvc.throw-exception-if-no-handler-found=true` 和 `spring.web.resources.add-mappings=false`。
    

#### 问：异常处理器的执行顺序是由什么决定的？

-   **答：** 如果有多个类都使用了 `@RestControllerAdvice`，可以通过 `@Order` 注解来指定优先级。数值越小，优先级越高。
    

#### 问：既然有了全局异常处理，Controller 里还需要写 try-catch 吗？

-   **答：**
    
-   **通常不需要**。为了代码简洁和统一规范，应该让异常向上抛出，由全局处理器处理。
    
-   **例外**：当你需要捕获特定异常并进行**业务补偿**（比如重试逻辑、回滚特定的非事务资源）时，才在 Controller 或 Service 中使用 `try-catch`。
    

#### 问：Spring Boot 的默认错误处理流程是怎样的？

-   **答：**
    

1.  异常发生且未被捕获。
    
2.  Servlet 容器将其转发给 `/error` 请求。
    
3.  `BasicErrorController` 处理该请求。它会根据请求头（Accept）返回不同的内容：如果是浏览器请求（text/html），返回白页（Whitelabel Error Page）；如果是机器请求（application/json），返回 JSON 错误信息。
    

---

### 5\. 总结 (Summary)

1.  **机制**：利用 AOP 思想，通过 `HandlerExceptionResolver` 实现全局拦截。
    
2.  **核心**：`@RestControllerAdvice` + `@ExceptionHandler` 是一对黄金搭档。
    
3.  **分层**：业务异常精确处理，未知异常统一兜底，保证返回给前端的格式始终一致。
    

---

**面试小技巧：** 在回答时，如果能提到 **“参数校验异常（**`MethodArgumentNotValidException`**）”** 的处理，会显得你实战经验非常丰富，因为这是实际开发中最常需要自定义返回格式的场景。

**既然聊到了异常处理，你是否想了解一下：如何配合全局异常处理来实现 Spring Security 的无权访问（403）和未登录（401）响应？或者是对 Hibernate Validator 的自动校验机制感兴趣？**