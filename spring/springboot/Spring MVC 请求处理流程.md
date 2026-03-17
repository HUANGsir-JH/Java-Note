这是 Spring MVC 最经典的一张架构图所描述的内容。我们可以通过\*\*“中央调度、各司其职”\*\*来理解这个流程。

### 1\. 核心结论 (Core Conclusion)

Spring MVC 的核心是 **DispatcherServlet（前端控制器）**。它不负责具体的业务处理，而是作为一个**总调度中心**，协调各个组件（处理器映射、适配器、视图解析器等）来共同完成请求的处理和响应。

---

### 2\. Spring MVC 处理流程图 (Flowchart)

```text
sequenceDiagram
    participant User as 用户 (Browser)
    participant DS as DispatcherServlet (前端控制器)
    participant HM as HandlerMapping (处理器映射器)
    participant HA as HandlerAdapter (处理器适配器)
    participant C as Controller (后端控制器/Handler)
    participant VR as ViewResolver (视图解析器)
    participant V as View (视图/JSON)

    User->>DS: 1. 发送请求 (HTTP Request)
    DS->>HM: 2. 请求获取 Handler
    HM-->>DS: 3. 返回 HandlerExecutionChain (包含拦截器)
    DS->>HA: 4. 请求执行 Handler
    HA->>C: 5. 适配并调用具体方法 (业务逻辑)
    C-->>HA: 6. 返回结果 (ModelAndView 或 数据)
    HA-->>DS: 7. 返回结果给总控
    
    alt 是传统页面跳转 (SSR)
        DS->>VR: 8. 请求解析视图
        VR-->>DS: 9. 返回 View 对象
        DS->>V: 10. 渲染视图 (填充模型数据)
        V-->>DS: 11. 返回渲染后的 HTML
    else 是 RESTful 接口 (JSON)
        DS->>DS: 8. 通过 HttpMessageConverter 转换数据
    end

    DS-->>User: 12. 响应结果 (Response)
```

![pasted_image_0e7eca04-b99b-4956-985f-2d326c023bbd.png](file://D:\CherryStudio\Data\Files\0e7eca04-b99b-4956-985f-2d326c023bbd.png)

---

### 3\. 步骤详细拆解 (Detailed Breakdown)

1.  **DispatcherServlet (前端控制器)**：所有请求的入口，负责拦截请求并分发。
    
2.  **HandlerMapping (处理器映射器)**：根据 URL 查找对应的 `Controller` 方法。它会返回一个“执行链” `HandlerExecutionChain`，里面不仅有 Controller，还有配置的 **Interceptor（拦截器）**。
    
3.  **HandlerAdapter (处理器适配器)**：这是典型的**适配器模式**。因为 Controller 有很多种（注解式的、实现接口的等），DS 不需要知道怎么调，直接交给适配器，适配器知道如何执行对应的 Controller。
    
4.  **Handler (后端控制器/Controller)**：开发者写的业务代码，处理逻辑并返回数据。
    
5.  **ViewResolver (视图解析器)**：将逻辑视图名（如 `"user_list"`）解析成物理视图（如 `"/WEB-INF/jsp/user_list.jsp"`）。
    
6.  **HttpMessageConverter (消息转换器)**：**（现代开发重点）** 如果方法上有 `@ResponseBody`，则跳过视图解析，直接由转换器（如 Jackson）将对象转为 JSON 写入响应体。
    

---

### 4\. 面试官视角：深度考点 (Interviewer's Lens)

#### 问：拦截器（Interceptor）在哪个阶段执行？

-   **答：**
    
-   `preHandle`：在 `HandlerAdapter` 执行 Controller **之前**。
    
-   `postHandle`：在 Controller 执行完，返回 `ModelAndView` **之后**，但在视图渲染之前。
    
-   `afterCompletion`：整个请求结束，视图渲染完 **之后**。
    

#### 问：HandlerMapping 和 HandlerAdapter 为什么都要有？

-   **答：** 为了**解耦和扩展**。
    
-   `Mapping` 只负责“找人”：根据 URL 找到对应的类和方法。
    
-   `Adapter` 只负责“干活”：不管你这个 Handler 是怎么定义的（是 `@RequestMapping` 还是老的 `HttpRequestHandler`），适配器都能统一调用。这体现了 Spring 的极强兼容性。
    

#### 问：如果发生了异常，流程怎么走？

-   **答：** Spring 会交给 `HandlerExceptionResolver`（全局异常处理器）处理。它会尝试寻找匹配的 `@ExceptionHandler` 方法，重新包装结果返回。
    

---

### 5\. 总结 (Summary)

-   **一核心**：`DispatcherServlet` 调度一切。
    
-   **两映射**：`HandlerMapping` 找方法，`HandlerAdapter` 调方法。
    
-   **两出口**：要么走 `ViewResolver` 出 HTML 页面，要么走 `HttpMessageConverter` 出 JSON 数据。
    

---

**面试小技巧：** 如果你能主动提到 `HandlerExecutionChain` 包含了拦截器，以及在前后端分离中 `HttpMessageConverter` 的重要性，面试官会认为你对源码和现状都有深入的理解。

**既然聊到了 JSON 转换，你对 Jackson 或者 Fastjson 的一些安全问题或底层原理感兴趣吗？或者我们回过头去看看 Spring 容器是如何管理这些 Controller 对象的（作用域问题）？**