
在 Spring Boot 3 中编写 Starter 与之前的版本（2.x）最大的区别在于 **自动配置文件的路径** 以及 **最低支持 Java 17**。

以下是创建一个名为 `hello-spring-boot-starter` 的完整流程。

---

### 1. 核心结论 (Core Conclusion)
编写 Starter 的本质是：**封装业务功能 + 定义配置属性 + 编写自动配置类 + 注册 SPI 接口**。
在 Spring Boot 3 中，必须使用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 来注册自动配置类。

---

### 2. 深度流程演示 (Full Process)

#### 第一步：创建 Maven 项目
新建一个 Maven 项目 `hello-spring-boot-starter`，`pom.xml` 核心依赖如下：

```xml
<dependencies>
    <!-- Spring Boot 自动配置核心依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
        <version>3.2.0</version>
    </dependency>
    <!-- 生成配置元数据，方便 IDE 提示 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <version>3.2.0</version>
        <optional>true</optional>
    </dependency>
    <!-- 业务代码可能需要的依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <version>3.2.0</version>
    </dependency>
</dependencies>
```

#### 第二步：编写业务 Service (Model)
这是你想要暴露给用户的核心功能。

```java
public class HelloService {
    private String prefix;
    private String suffix;

    public HelloService(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }

    public String sayHello(String name) {
        return prefix + " " + name + " " + suffix;
    }
}
```

#### 第三步：编写配置属性类 (Properties)
映射 `application.yml` 中的配置。

```java
@ConfigurationProperties(prefix = "hello.service")
public class HelloProperties {
    private String prefix = "Default-Hi"; // 默认值
    private String suffix = "Default-Bye";
    
    // Getter 和 Setter (必须提供)
    public String getPrefix() { return prefix; }
    public void setPrefix(String prefix) { this.prefix = prefix; }
    public String getSuffix() { return suffix; }
    public void setSuffix(String suffix) { this.suffix = suffix; }
}
```

#### 第四步：编写自动配置类 (AutoConfiguration)
这是 Starter 的“大脑”，决定什么时候创建 Bean。

```java
@AutoConfiguration // Spring Boot 3 推荐使用此注解替代 @Configuration
@EnableConfigurationProperties(HelloProperties.class) // 使配置类生效
@ConditionalOnClass(HelloService.class) // 只有类路径下有这个类时才加载
@ConditionalOnProperty(prefix = "hello.service", name = "enabled", havingValue = "true") // 开关控制
public class HelloAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean // 如果用户自己定义了，就用用户的
    public HelloService helloService(HelloProperties properties) {
        return new HelloService(properties.getPrefix(), properties.getSuffix());
    }
}
```

#### 第五步：注册 SPI 接口 (关键变动)
在 `src/main/resources` 下创建目录结构：`META-INF/spring/`。
在该目录下新建文件：`org.springframework.boot.autoconfigure.AutoConfiguration.imports`。
**内容为自动配置类的全路径：**
```text
com.example.hello.config.HelloAutoConfiguration
```

#### 第六步：发布 (Install)
执行 `mvn clean install` 将项目打包并安装到本地 Maven 仓库。

---

### 3. 如何使用这个 Starter

1. **在其他项目中引入依赖：**
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>hello-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

2. **在 `application.yml` 中配置：**
```yaml
hello:
  service:
    enabled: true
    prefix: "Hello"
    suffix: "Welcome to SB3"
```

3. **直接注入并使用：**
```java
@RestController
public class TestController {
    @Autowired
    private HelloService helloService;

    @GetMapping("/test")
    public String test() {
        return helloService.sayHello("Jack"); // 输出: Hello Jack Welcome to SB3
    }
}
```

---

### 4. 面试官视角：深度考点 (Interviewer's Lens)

#### 问：为什么 Spring Boot 3 弃用了 `spring.factories`？
* **答：** 主要是为了**性能优化**和**结构清晰**。老的 `spring.factories` 把所有扩展点（监听器、初始化器、自动配置）混在一起，启动时扫描压力大。新的 `.imports` 文件专为自动配置设计，支持更高效的按需加载。

#### 问：`@AutoConfiguration` 和 `@Configuration` 有什么区别？
* **答：** `@AutoConfiguration` 是专门给自动配置类用的。它内部整合了 `@Configuration`，并确保该配置类在所有用户自定义配置类**之后**执行，这样 `@ConditionalOnMissingBean` 才能准确判断用户是否已经定义了 Bean。

#### 问：如何保证 Starter 的配置有代码提示？
* **答：** 引入 `spring-boot-configuration-processor`。在编译时它会扫描 `@ConfigurationProperties` 类，并生成 `META-INF/spring-configuration-metadata.json` 文件。IDE（如 IDEA）读取这个文件后，用户在写 YML 时就会有自动补全。

---

### 5. 总结 (Summary)

1. **一个文件**：必须有 `META-INF/spring/xxx.imports`。
2. **两类注解**：`@AutoConfiguration` 控制装配逻辑，`@Conditional` 系列控制装配条件。
3. **三层解耦**：Properties（数据层）、Service（业务层）、Configuration（组装层）。

---

**面试建议：**
提到 Starter 时，顺便提一下 **“命名规范”**：官方推荐 `spring-boot-starter-xxx`，第三方推荐 `xxx-spring-boot-starter`。这显示了你对社区规范的了解。

**既然聊到了 Starter 和自动装配，你是否想了解一下：Spring Boot 是如何处理循环依赖的？或者是想看看 Spring Boot 3 的 AOT（原生镜像编译）对 Starter 有什么特殊要求？**
