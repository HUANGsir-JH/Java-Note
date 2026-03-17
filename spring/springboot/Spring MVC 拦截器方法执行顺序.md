在 Spring MVC 中，拦截器接口 `HandlerInterceptor` 定义了三个核心方法。理解这三个方法的**执行时机**以及在**异常发生时**的表现，是区分初级和中高级开发者的关键。

---

### 1\. 核心结论 (Core Conclusion)

拦截器的执行流程遵循\*\*“前置拦截 -> 业务处理 -> 后置拦截 -> 完成回调”\*\*。这三个方法分别是：`preHandle`、`postHandle` 和 `afterCompletion`。它们共同构成了一个环绕 Controller 的逻辑圈。

---

### 2\. 深度解析：三个方法与执行时机 (Detailed Breakdown)

#### ① `preHandle` (前置处理)

-   **执行时机**：在 `DispatcherServlet` 收到请求，经 `HandlerMapping` 匹配到具体的 Controller 方法，但**在调用 Controller 方法之前**。
    
-   **返回值 (**`boolean`**)**：
    
-   `true`：放行，继续执行后续的拦截器或 Controller。
    
-   `false`：拦截，请求直接结束，不会进入 Controller。
    
-   **应用场景**：权限校验（是否登录）、白名单过滤、参数预处理。
    

#### ② `postHandle` (后置处理)

-   **执行时机**：在 **Controller 方法调用完之后**，但在 `DispatcherServlet` **渲染视图之前**。
    
-   **特点**：
    
-   可以访问到 Controller 返回的 `ModelAndView` 对象，甚至可以修改它（在前后端分离中，这个方法用得相对较少）。
    
-   **注意**：如果 Controller 抛出了异常，该方法**不会**被执行。
    
-   **应用场景**：对渲染前的数据进行统一处理，或在日志中记录 Controller 的执行结果。
    

#### ③ `afterCompletion` (完成回调)

-   **执行时机**：在**整个请求处理完成后**（即视图渲染结束，或 REST 接口数据已写回客户端）。
    
-   **特点**：
    
-   **必然执行**：只要对应的 `preHandle` 返回了 `true`，无论 Controller 是否抛出异常，`afterCompletion` 都会执行。
    
-   可以获取到抛出的异常对象。
    
-   **应用场景**：**资源清理**（如释放线程变量 `ThreadLocal`）、统计请求耗时、统一异常日志记录。
    

---

### 3\. 多拦截器下的执行顺序 (Execution Order)

如果有多个拦截器（Interceptor A 和 Interceptor B），其顺序如下：

1.  **preHandle A**
    
2.  **preHandle B**
    
3.  **Controller 业务逻辑**
    
4.  **postHandle B** (逆序)
    
5.  **postHandle A** (逆序)
    
6.  **渲染视图 / 响应**
    
7.  **afterCompletion B** (逆序)
    
8.  **afterCompletion A** (逆序)
    

> **面试加分点**：这种执行结构被称为\*\*“栈式执行”\*\*（后进先出）。

---

### 4\. 面试官视角：深度考点 (Interviewer's Lens)

#### 问：如果 Controller 抛出异常，哪个方法会执行？

-   **答**：`postHandle` **不会**执行，但 `afterCompletion` **会**执行（前提是 `preHandle` 已经放行）。
    

#### 问：为什么 `ThreadLocal` 的清理一定要放在 `afterCompletion` 里？

-   **答**：因为如果在 `postHandle` 中清理，一旦 Controller 抛出异常，`postHandle` 就会被跳过，导致 `ThreadLocal` 无法清理，从而产生**内存泄漏**。由于 `afterCompletion` 具有“保底执行”的特性，它是清理资源的唯一安全位置。
    

#### 问：`preHandle` 返回 `false` 后，`afterCompletion` 还会执行吗？

-   **答**：**不会**。只有当前拦截器的 `preHandle` 返回 `true` 时，它对应的 `afterCompletion` 才会进入执行链。
    

---

### 5\. 代码演示 (Code Snippet)

```java
public class PerformanceInterceptor implements HandlerInterceptor {

    // 使用 ThreadLocal 记录起始时间
    private ThreadLocal<Long> startTimeThreadLocal = new ThreadLocal<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        startTimeThreadLocal.set(System.currentTimeMillis());
        return true; // 继续执行
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) {
        // Controller 成功执行后，可以在这里修改模型数据
        System.out.println("Controller 执行成功，准备渲染视图");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            long startTime = startTimeThreadLocal.get();
            long endTime = System.currentTimeMillis();
            System.out.println("接口耗时：" + (endTime - startTime) + "ms");
            if (ex != null) {
                System.out.println("捕获到异常：" + ex.getMessage());
            }
        } finally {
            // 必须手动移除，防止线程池场景下的内存泄漏
            startTimeThreadLocal.remove();
        }
    }
}
```

---

### 6\. 总结 (Summary)

1.  **preHandle**：Controller 之前，用于“准入控制”。
    
2.  **postHandle**：Controller 之后，视图渲染前，用于“模型修正”（异常时不执行）。
    
3.  **afterCompletion**：整个请求结束后，用于“资源清理”和“异常记录”（无论是否有异常都会执行）。
    

**关于拦截器的应用，你是否想了解：它在 Spring Security 这种复杂的安全框架中是如何嵌套工作的？或者是想聊聊它与 AOP（切面编程）在性能监控上的优劣对比？**