在 Spring Boot 开发中，过滤器（Filter）和拦截器（Interceptor）都是处理**横切关注点**（如日志、权限、限流）的利器，但它们所属的“阵营”和作用的层级完全不同。

---

### 1\. 核心结论 (Core Conclusion)

-   **过滤器（Filter）**：属于 **Servlet 容器**（如 Tomcat），是 Web 应用的第一道防线，作用于 `DispatcherServlet` 之前。
    
-   **拦截器（Interceptor）**：属于 **Spring MVC 框架**，作用于 `DispatcherServlet` 之后、控制器（Controller）处理前后。
    

**一句话总结：** 过滤器关注的是“请求进来没”，拦截器关注的是“业务怎么处理”。

---

### 2\. 深度解析：异同点对比 (Detailed Breakdown)

| 特性  | 过滤器 (Filter) | 拦截器 (Interceptor) |
| --- | --- | --- |
| **所属标准** | Servlet 标准 (javax.servlet) | Spring MVC 框架 |
| **底层原理** | 基于**函数回调** | 基于**Java 反射（动态代理）** |
| **容器支持** | 依赖 Servlet 容器（Tomcat/Jetty） | 依赖 Spring 框架，不依赖 Servlet |
| **作用范围** | 几乎可以过滤所有请求（静态资源等） | 只拦截匹配 Handler 的请求（通常是 Controller） |
| **访问能力** | 只能访问 Request/Response，无法获取 Spring 环境 | **能力更强**：可以获取目标 Controller 对象、AOP 增强等 |
| **生命周期** | 由 Servlet 容器管理 | 由 Spring 容器管理（可直接 `@Autowired`） |

#### 执行顺序（“U型”结构）：

1.  **Request** -> Filter -> Servlet -> Interceptor -> **Controller**
    
2.  **Response** <- Filter <- Servlet <- Interceptor <- **Controller**
    

---

### 3\. 应用场景 (Application Scenarios)

#### 过滤器（Filter）的常见用途：

-   **敏感词/脱敏过滤**：对请求体进行全局包装和修改（使用 `HttpServletRequestWrapper`）。
    
-   **字符编码设置**：设置 `request.setCharacterEncoding("UTF-8")`。
    
-   **跨域处理 (CORS)**：在最外层设置允许跨域的 Header。
    
-   **Spring Security 权限控制**：安全框架本质上就是一串长长的过滤器链。
    

#### 拦截器（Interceptor）的常见用途：

-   **业务权限校验**：判断当前用户是否有权访问某个 Controller 方法（RBAC）。
    
-   **性能监控**：计算方法执行时间（在 `preHandle` 记录开始时间，`afterCompletion` 计算差值）。
    
-   **日志记录**：记录请求的具体接口、参数及耗时。
    
-   **通用属性填充**：将用户信息注入到 Request 域中，供后续业务使用。
    

---

### 4\. 面试官视角：深度追问 (Interviewer's Lens)

#### 问：拦截器里能注入 Service 吗？过滤器里呢？

-   **答：**
    
-   **拦截器**完全可以，因为它是由 Spring 容器管理的 Bean。
    
-   **过滤器**默认不行，因为它是 Tomcat 管理的。但可以通过 `FilterRegistrationBean` 注册过滤器，或者在 Filter 中手动通过 `ApplicationContext` 获取 Bean。
    

#### 问：如果 Controller 抛出异常，拦截器的 `postHandle` 会执行吗？

-   **答：** **不会**。如果 Controller 发生异常，会直接跳转到异常处理逻辑。但是 `afterCompletion` **一定会执行**，因为它常用于资源清理。
    

#### 问：为什么修改请求参数通常在 Filter 中做？

-   **答：** 因为 Filter 接收的是原始的 `ServletRequest`。通过包装模式（Wrapper），我们可以重写 `getParameter` 等方法，从而在请求到达 Controller 之前实现参数篡改（如过滤 SQL 注入关键字）。拦截器虽然也能拿到 Request，但此时参数可能已经被封装到参数对象中了，修改起来很麻烦。
    

---

### 5\. 代码演示 (Code Snippet)

#### 拦截器定义示例：

```java
@Component
public class MyInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        System.out.println("拦截器：请求前校验");
        return true; // 继续执行
    }
}

// 注册拦截器
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private MyInterceptor myInterceptor;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(myInterceptor).addPathPatterns("/api/**");
    }
}
```

---

### 6\. 总结 (Summary)

1.  **层级不同**：Filter 外（容器级），Interceptor 内（框架级）。
    
2.  **粒度不同**：Filter 粗（过滤 URL 模式），Interceptor 细（能拿到具体哪个方法在执行）。
    
3.  **能力不同**：Interceptor 更懂 Spring（能拿 Bean，懂 AOP），Filter 更懂 HTTP（能改 Request/Response 本体）。
    

---

**面试建议：** 如果你能提到 `OncePerRequestFilter`（确保一次请求只经过一次过滤器，解决 Servlet Forward 导致的多次过滤问题），面试官会觉得你对 Spring 源码细节掌握得非常扎实。

**关于 Web 层开发，你是否想聊聊：Spring Boot 是如何处理全局异常的（**`@ControllerAdvice`**）？或者是想深入探讨一下 Spring Security 的过滤器链工作原理？**