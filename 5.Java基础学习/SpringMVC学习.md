## 一、Spring MVC 是什么？

>  **Spring MVC**（Spring Model-View-Controller）是 Spring 框架的一个**核心模块**。
>
> - 基于 MVC 设计模式的**轻量级 Web 框架**；
> - 用于构建灵活、松耦合的 Java Web 应用程序。

### 1、MVC 设计模式的职责划分

Spring MVC 严格遵循 MVC 模式，将不同职责的代码分离：

1. **Model（模型）**：处理业务逻辑和数据，通常是实体类（POJO）、Service、DAO 层，负责数据的封装、存储和处理。
2. **View（视图）**：展示数据给用户，通常是 HTML、JSP、Thymeleaf 等页面模板。
3. **Controller（控制器）**：接收客户端请求，协调 Model 处理业务，最终将数据传递给 View 展示（也可直接返回 JSON/XML 数据）。

### 2、Spring MVC 的核心组件

Spring MVC 的请求处理流程依赖以下核心组件，它们协同工作完成请求的处理：

1. **DispatcherServlet（前端控制器）**：整个 Spring MVC 的核心，负责接收所有客户端请求，分发到对应的处理器（Controller），是请求的 “中央调度器”。
2. **HandlerMapping（处理器映射器）**：根据请求的 URL、请求方式等信息，找到对应的 Controller 方法（处理器）。
3. **HandlerAdapter（处理器适配器）**：适配并执行找到的 Controller 方法，处理方法的参数、返回值等。
4. **Controller（处理器）**：业务处理的核心，定义请求处理的方法，返回数据或视图信息。
5. **ViewResolver（视图解析器）**：根据 Controller 返回的视图名称，解析成具体的视图对象（如 JSP、Thymeleaf 页面）。

### 3、Spring MVC 的请求处理流程

1. 客户端发送 HTTP 请求到 **DispatcherServlet**。
2. DispatcherServlet 调用 **HandlerMapping**，获取对应 Handler（Controller 方法）。
3. DispatcherServlet 通过 **HandlerAdapter** 执行 Handler 方法，得到处理结果（ModelAndView 或 JSON 数据）。
4. 若返回的是视图，DispatcherServlet 调用 **ViewResolver** 解析视图，渲染数据后返回给客户端；若返回的是 JSON 等数据，则直接返回给客户端。

### 4、Spring MVC 的典型使用场景

- 开发传统的服务端渲染页面的 Web 应用（如后台管理系统）。
- 开发 RESTful API 接口（前后端分离项目的后端接口）。
- 整合其他 Spring 生态组件（如 Spring Security 做权限控制、MyBatis 做数据持久化）。

### 5、补充：Spring MVC 与 Spring Boot 的关系

- **Spring MVC** 是**Web 框架**，专注于 Web 层的请求处理。
- **Spring Boot** 是**快速开发框架**，基于 Spring 框架，通过**自动配置**简化了 Spring MVC 的配置（如内置 Tomcat、自动配置 DispatcherServlet 等），让开发者可以 “零配置” 快速搭建 Spring MVC 项目。

简单来说：**Spring Boot 是脚手架，Spring MVC 是其中的核心 Web 组件**。



## 二、springmvc和tomcat的关系

> 简单来说：**Tomcat 是承载 Spring MVC 应用的运行容器，Spring MVC 是运行在 Tomcat 中的 Web 框架**。
>
> 二者协同工作支撑 Java Web 应用的运行。

### 1、二者的协同工作流程

Spring MVC 应用最终会被打包成 WAR 包（或 Spring Boot 内置 Tomcat 时为 JAR 包），部署到 Tomcat 中运行，具体流程如下：

1. **启动阶段**
   - Tomcat 启动后，会加载 Web 应用的配置（如 `web.xml` 或 Spring Boot 自动配置的 `DispatcherServlet`）。
   - Tomcat 会初始化 `DispatcherServlet`（Spring MVC 的前端控制器，本质是一个 Servlet），并触发 Spring IOC 容器的初始化，加载所有 Controller、Service 等 Bean。
2. **请求处理阶段**
   - 客户端发送 HTTP 请求到 Tomcat 的 8080 端口，Tomcat 解析 HTTP 请求后，根据请求路径将其转发给 `DispatcherServlet`。
   - `DispatcherServlet` 作为 Spring MVC 的核心，会通过 `HandlerMapping` 找到对应的 Controller 方法，调用 `HandlerAdapter` 执行方法，处理业务逻辑。
   - Controller 处理完成后，返回数据（如 JSON）或视图名称，`DispatcherServlet` 再将结果通过 Tomcat 转换为 HTTP 响应，发送给客户端。
3. **销毁阶段**
   - Tomcat 关闭时，会销毁 `DispatcherServlet`，进而触发 Spring IOC 容器的销毁，释放资源。

### 2、关键关联点：DispatcherServlet（Spring MVC）与 Servlet 容器（Tomcat）

Spring MVC 的核心是 `DispatcherServlet`，而 `DispatcherServlet` 是**标准的 Servlet 实现类**，这是二者能够集成的关键：

- Tomcat 作为 Servlet 容器，只认 Servlet 规范，它并不感知 Spring MVC 的存在，只负责调用 `DispatcherServlet` 的 `service()` 方法处理请求。
- Spring MVC 则在 `DispatcherServlet` 内部封装了复杂的 MVC 逻辑，将 Servlet 底层的 API（如 `HttpServletRequest`、`HttpServletResponse`）封装成更易用的对象（如 `ModelAndView`、`@RequestParam` 注解），简化开发。

### 3、Spring Boot 中内置 Tomcat 与 Spring MVC 的关系

在 Spring Boot 中，默认会**内置 Tomcat 依赖**，并通过自动配置完成以下工作，让二者的集成更便捷：

1. 自动注册 `DispatcherServlet` 到内置 Tomcat 中，无需手动配置 `web.xml`。
2. 自动配置 Tomcat 的端口、线程池等参数（可通过 `application.properties` 自定义）。
3. 启动 Spring Boot 应用时，内置 Tomcat 会被启动，同时初始化 Spring MVC 容器，实现 “一键运行”。

### 4、总结

- **Tomcat 是 “容器”**：负责底层的网络通信、Servlet 生命周期管理，是 Spring MVC 应用的运行载体。
- **Spring MVC 是 “应用逻辑层”**：基于 Servlet 规范构建，利用 Tomcat 提供的 Servlet 运行环境，实现 Web 请求的业务处理。
- 二者的关系：**Spring MVC 依赖 Tomcat 提供的 Servlet 运行环境，Tomcat 承载并执行 Spring MVC 的核心组件**，共同完成 Web 应用的开发与运行。

## 三、SpringMVC线程安全问题

### 1、产生原因

**SpringMVC线程安全问题是由于发者在使用时`未遵循单例的设计规范`导致的**。在 Spring MVC 中，**Controller 默认是单例 Bean**（由 Spring IOC 容器管理，整个应用生命周期内只有一个实例）, Tomcat 处理请求时，会为每个 HTTP 请求分配一个独立的线程来执行 Controller 的方法。

这就意味着：**多个请求线程会同时操作同一个 Controller 实例的成员变量**，如果成员变量是**可变的**（如普通的 `int`、`StringBuilder`、自定义对象等），就会引发线程安全问题（数据错乱、覆盖、脏读等）。

####  SpringMVC 默认单例验证

##### 1. 核心依赖（Maven）

确保 `pom.xml` 中引入 Spring Web 起步依赖，无需额外 Tomcat 依赖（Spring Boot 内置）：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

##### 2. 验证 Controller 代码

通过打印 Controller 实例的 `hashCode` 判断是否为单例，多次请求对比哈希值：

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SingletonCheckController {

    // 无参构造函数，启动时打印实例哈希值
    public SingletonCheckController() {
        System.out.println("Controller 实例创建，hashCode: " + this.hashCode());
    }

    @GetMapping("/check-singleton")
    public String checkSingleton() {
        String result = "当前 Controller 实例 hashCode: " + this.hashCode();
        System.out.println(result);
        return result;
    }
}
```

##### 3. Spring Boot 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SingletonCheckApplication {
    public static void main(String[] args) {
        SpringApplication.run(SingletonCheckApplication.class, args);
    }
}
```

##### 4. 测试步骤与结果

1. 启动项目，控制台会输出 **1 次** Controller 实例的 `hashCode`（证明容器启动时仅创建 1 个实例）。
2. 多次访问 `http://localhost:8080/check-singleton`（浏览器、Postman 或 curl 均可）。
3. 观察控制台和接口返回值：**所有请求的 `hashCode` 完全相同**，验证 Controller 默认是单例。

##### 5. 多例模式验证（可选）

在 Controller 类上添加 `@Scope("prototype")` 注解，重复测试：

```java
import org.springframework.context.annotation.Scope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@Scope("prototype")
public class SingletonCheckController {
    // 其余代码不变
}
```

此时**每次请求都会创建新的 Controller 实例**，控制台会输出不同的 `hashCode`

### 2、解决单例带来的线程安全问题

> 核心原则是 **避免在 Controller 中定义可变成员变量**。

#### 1、**优先使用局部变量**

局部变量存储在栈中，每个线程有独立的栈空间，不会存在线程安全问题，这是最推荐的做法。

```java
@RestController
public class SafeController {
    @GetMapping("/safe")
    public String safeMethod() {
        // 局部变量，线程安全
        String localVar = "thread-safe";
        return localVar;
    }
}
```

#### 2、**使用线程安全的成员变量**

若必须定义成员变量，需使用 JDK 提供的线程安全容器或原子类，如 `ConcurrentHashMap`、`AtomicInteger`。

```java
@RestController
public class SafeController {
    // 原子类，保证操作原子性
    private AtomicInteger count = new AtomicInteger(0);

    @GetMapping("/count")
    public int count() {
        // 原子自增，线程安全
        return count.incrementAndGet();
    }
}
```

#### 3、**使用 ThreadLocal 隔离线程数据**

`ThreadLocal` 可为每个线程提供独立的变量副本，避免多线程共享数据冲突。

```java
@RestController
public class SafeController {
    private ThreadLocal<String> threadLocal = new ThreadLocal<>();

    @GetMapping("/thread-local")
    public String threadLocalTest() {
        threadLocal.set("data-" + Thread.currentThread().getId());
        String data = threadLocal.get();
        // 务必移除，防止内存泄漏
        threadLocal.remove();
        return data;
    }
}
```

#### 4、**将 Controller 改为多例（不推荐）**

给 Controller 添加 `@Scope("prototype")` 注解，让每次请求创建新实例。此方案会增加对象创建和销毁的开销，影响性能，仅适用于特殊场景。

### 3、最佳实践总结

1. **核心原则**：**不要在 Controller 中定义可变的成员变量**，这是避免线程安全问题的根本。
2. **优先方案**：使用局部变量处理业务数据，天然线程安全。
3. **特殊场景**：
   - 若需全局共享数据（如计数），使用原子类（`AtomicXXX`）或并发容器（`ConcurrentHashMap`）。
   - 若需线程内共享数据（如用户上下文），使用 `ThreadLocal` 并记得调用 `remove()`。
4. **避免方案**：尽量不要将 Controller 改为多例，除非有特殊业务需求（如依赖非线程安全的第三方类，且无法修改）。

### 4、补充：Spring MVC 其他组件的线程安全

除了 Controller，Spring MVC 的其他核心组件（如 `DispatcherServlet`、`HandlerMapping`、`ViewResolver`）也都是单例的，但这些组件**没有可变的成员变量**，因此天然线程安全。这也印证了：**单例本身不是问题，问题在于单例中的可变共享数据**。



## 四、前后端分离架构，给出请求从发出到处理全景图

在前后端分离架构下，请求处理流程的核心变化是**服务端不再负责视图渲染，仅返回 JSON 数据，View 层职责转移到前端**。

### 1、请求处理全流程全景图（组件 + 步骤）

整个流程分为**前端层**、**Tomcat 底层处理层**、**Spring MVC 核心处理层**、**后端业务层**四个部分，涉及的核心组件及执行步骤如下：

```plaintext
1、前端客户端（Vue/React/Postman）
    ↓（发送AJAX/HTTP请求：如GET http://localhost:8080/api/user/1）
（HTTP请求通过网络传输到Tomcat服务器）
2. Tomcat底层网络/线程组件
   - Connector（NIO模式）：监听8080端口，接收请求并解析为HttpServletRequest/HttpServletResponse
   - ThreadPool（nio-8080-exec-*）：为每个请求分配独立工作线程
   - Engine/Host/Context：匹配请求的Spring Boot应用（Context）
    ↓（Tomcat线程将请求转发给Spring MVC核心入口）
3. Spring MVC核心组件（运行在Tomcat线程中）
   - DispatcherServlet（前端控制器）：接收请求，作为总调度器
   - HandlerMapping（处理器映射器）：根据URL匹配Controller方法（如/api/user/{id} → UserController.getUser()）
   - HandlerInterceptor（拦截器）：请求处理前后的切面处理（登录校验、日志、跨域）
   - HandlerAdapter（处理器适配器）：处理参数绑定（@PathVariable/@RequestParam）、执行Controller方法
   - Controller（处理器）：接收参数，调用Service层，返回数据对象
   - HttpMessageConverter（消息转换器）：将Java对象序列化为JSON（默认MappingJackson2HttpMessageConverter）
   - ExceptionHandler（全局异常处理器）：处理请求过程中的异常，返回统一错误JSON
    ↓（JSON响应返回Tomcat）
4. Tomcat将JSON响应转为HTTP响应，通过网络返回给前端
    ↓
5. 前端接收JSON数据，通过框架渲染页面（前端View层）
```

### 2、各组件核心职责表（前后端分离专属版）

| 阶段            | 组件                          | 核心职责（前后端分离特性）                                   |
| --------------- | ----------------------------- | ------------------------------------------------------------ |
| 前端层          | Axios/Vue/React               | 发起异步请求，接收 JSON 数据，渲染前端视图（接管传统 V 层）  |
| Tomcat 底层     | Connector（NIO）              | 解析 HTTP 请求为 Servlet 标准对象，区别于传统模式的是请求为 AJAX 异步请求 |
| Tomcat 底层     | ThreadPool                    | 分配 nio-8080-exec-* 线程，处理请求（与传统模式一致）        |
| Spring MVC 核心 | DispatcherServlet             | 总调度器，区别于传统模式的是最终调用消息转换器而非视图解析器 |
| Spring MVC 核心 | HandlerMapping                | 匹配 RESTful API 路径（如 /api/user/{id}）到 Controller 方法 |
| Spring MVC 核心 | HandlerInterceptor            | 处理跨域、登录校验（前后端分离中跨域是高频需求）             |
| Spring MVC 核心 | HandlerAdapter                | 支持 RESTful 参数绑定（如 @PathVariable、@RequestBody）      |
| Spring MVC 核心 | Controller（@RestController） | 不再返回视图名称，直接返回 Java 对象（如 UserVO）            |
| Spring MVC 核心 | HttpMessageConverter          | 将 Java 对象序列化为 JSON（替代传统的 ViewResolver/View）    |
| Spring MVC 核心 | GlobalExceptionHandler        | 统一返回异常 JSON（前后端分离需标准化错误响应）              |

### 3、代码案例：前后端分离下的完整请求处理流程

我们基于 **Spring Boot + Spring MVC（内置 Tomcat）** 搭建案例，覆盖从前端请求到后端返回 JSON 的全流程。

#### 1. 依赖配置（pom.xml）

```xml
<dependencies>
    <!-- Spring Web（Spring MVC + 内置Tomcat） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Lombok（简化代码） -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <!-- Spring Boot测试（可选） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

#### 2. 全局配置：跨域处理（前后端分离必配）

```java
package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 跨域配置：解决前后端分离中前端域名与后端域名不一致的问题
 */
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        // 允许所有请求跨域
        registry.addMapping("/api/**")
                .allowedOriginPatterns("*") // 允许的前端域名（*表示所有，生产环境需指定具体域名）
                .allowedMethods("GET", "POST", "PUT", "DELETE") // 允许的请求方法
                .allowedHeaders("*") // 允许的请求头
                .allowCredentials(true) // 允许携带Cookie
                .maxAge(3600); // 预检请求的缓存时间（秒）
    }
}
```

#### 3. 自定义拦截器：登录校验 + 日志记录

```java
package com.example.demo.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

/**
 * Spring MVC拦截器：处理请求前后的逻辑（登录校验、日志）
 */
@Slf4j
@Component
public class LoginInterceptor implements HandlerInterceptor {

    // 1. 请求到达Controller方法前执行（核心：登录校验）
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestUrl = request.getRequestURL().toString();
        String threadId = String.valueOf(Thread.currentThread().getId());
        log.info("【preHandle】Tomcat线程ID：{}，请求URL：{}，请求方法：{}", threadId, requestUrl, request.getMethod());

        // 模拟登录校验：检查请求头中的token
        String token = request.getHeader("token");
        if ("admin123".equals(token)) {
            return true; // 放行
        }

        // 未登录：返回401错误，JSON格式
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":401,\"message\":\"未登录，请先登录\"}");
        return false; // 拦截
    }

    // 2. Controller方法执行完成后执行（前后端分离中极少用，因为无视图渲染）
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("【postHandle】Controller方法执行完成");
    }

    // 3. 整个请求处理完成后执行（资源清理）
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("【afterCompletion】请求处理全流程完成，响应状态码：{}", response.getStatus());
    }
}
```

#### 4. 注册拦截器

```java
package com.example.demo.config;

import com.example.demo.interceptor.LoginInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册拦截器，匹配所有/api/**请求
        registry.addInterceptor(loginInterceptor).addPathPatterns("/api/**");
    }
}
```

#### 5. 数据模型：Entity/VO/DTO（前后端分离的数据封装）

```java
package com.example.demo.entity;

import lombok.Data;

/**
 * 数据库实体：与user表一一对应（后端内部使用）
 */
@Data
public class User {
    private Long id;
    private String username;
    private String password; // 敏感字段，不返回给前端
    private Integer age;
    private String email;
}
```

```java
package com.example.demo.vo;

import lombok.Data;

/**
 * 前端输出VO：返回给前端的数据（隐藏敏感字段）
 */
@Data
public class UserVO {
    private Long id;
    private String username;
    private Integer age;
    private String email;
}
```

```java
package com.example.demo.dto;

import lombok.Data;

/**
 * 前端入参DTO：接收前端传递的参数
 */
@Data
public class UserQueryDTO {
    private Long id;
    private String username;
}
```

#### 6. 统一响应结果封装（前后端分离标准做法）

```java
package com.example.demo.common;

import lombok.Data;

/**
 * 统一JSON响应体：前后端约定的返回格式
 */
@Data
public class ResultVO<T> {
    // 响应码：200成功，400参数错误，401未登录，500服务器错误
    private Integer code;
    // 响应信息
    private String message;
    // 响应数据
    private T data;

    // 成功响应（带数据）
    public static <T> ResultVO<T> success(T data) {
        ResultVO<T> result = new ResultVO<>();
        result.setCode(200);
        result.setMessage("操作成功");
        result.setData(data);
        return result;
    }

    // 成功响应（无数据）
    public static <T> ResultVO<T> success() {
        return success(null);
    }

    // 失败响应
    public static <T> ResultVO<T> fail(Integer code, String message) {
        ResultVO<T> result = new ResultVO<>();
        result.setCode(code);
        result.setMessage(message);
        result.setData(null);
        return result;
    }
}
```

#### 7. 全局异常处理器（前后端分离必配）

```java
package com.example.demo.handler;

import com.example.demo.common.ResultVO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/**
 * 全局异常处理器：统一处理后端异常，返回标准JSON
 */
@Slf4j
@RestControllerAdvice // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // 处理运行时异常
    @ExceptionHandler(RuntimeException.class)
    public ResultVO<?> handleRuntimeException(RuntimeException e) {
        log.error("运行时异常：", e);
        return ResultVO.fail(500, e.getMessage());
    }

    // 处理所有异常（兜底）
    @ExceptionHandler(Exception.class)
    public ResultVO<?> handleException(Exception e) {
        log.error("系统异常：", e);
        return ResultVO.fail(500, "服务器内部错误，请联系管理员");
    }
}
```

#### 8. Service 层：业务逻辑处理

```java
package com.example.demo.service;

import com.example.demo.dto.UserQueryDTO;
import com.example.demo.entity.User;
import com.example.demo.vo.UserVO;
import org.springframework.stereotype.Service;
import org.springframework.beans.BeanUtils;

/**
 * 业务逻辑层：处理核心业务（模拟数据库查询）
 */
@Service
public class UserService {

    public UserVO getUserInfo(UserQueryDTO queryDTO) {
        // 1. 模拟数据库查询（实际项目中调用Mapper/DAO层）
        User user = new User();
        user.setId(queryDTO.getId());
        user.setUsername("张三");
        user.setPassword("123456"); // 敏感字段
        user.setAge(20);
        user.setEmail("zhangsan@example.com");

        // 2. 业务逻辑校验
        if (user.getAge() < 0) {
            throw new RuntimeException("年龄不能为负数");
        }

        // 3. 转换为VO（隐藏敏感字段）
        UserVO userVO = new UserVO();
        BeanUtils.copyProperties(user, userVO);

        return userVO;
    }
}
```

#### 9. Controller 层：RESTful API 提供（核心）

```java
package com.example.demo.controller;

import com.example.demo.common.ResultVO;
import com.example.demo.dto.UserQueryDTO;
import com.example.demo.service.UserService;
import com.example.demo.vo.UserVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

/**
 * 前后端分离的Controller：返回JSON数据（@RestController）
 */
@RestController // = @Controller + @ResponseBody，所有方法返回JSON
@RequestMapping("/api/user") // 接口统一前缀
public class UserController {

    @Autowired
    private UserService userService;

    // GET请求：根据ID查询用户
    @GetMapping("/{id}")
    public ResultVO<UserVO> getUser(@PathVariable Long id, @RequestParam(required = false) String username) {
        // 1. 封装前端参数
        UserQueryDTO queryDTO = new UserQueryDTO();
        queryDTO.setId(id);
        queryDTO.setUsername(username);

        // 2. 调用Service层处理业务逻辑
        UserVO userVO = userService.getUserInfo(queryDTO);

        // 3. 返回统一JSON响应
        return ResultVO.success(userVO);
    }

    // POST请求：新增用户（接收JSON参数）
    @PostMapping
    public ResultVO<?> addUser(@RequestBody UserVO userVO) {
        // 模拟新增业务
        log.info("新增用户：{}", userVO);
        return ResultVO.success("用户新增成功");
    }
}
```

#### 10. Spring Boot 启动类

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringMvcSeparateDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringMvcSeparateDemoApplication.class, args);
    }
}
```

###  4、全流程执行演示（以前端请求`/api/user/1`为例）

#### 步骤 1：前端发起请求

Vue 页面挂载后，通过 Axios 发送 GET 请求：`http://localhost:8080/api/user/1?username=张三`，请求头携带`token: admin123`。

#### 步骤 2：Tomcat 底层处理

1. **Connector（NIO）**：监听 8080 端口，接收 HTTP 请求，解析为`HttpServletRequest`和`HttpServletResponse`对象。
2. **ThreadPool**：分配工作线程（如`nio-8080-exec-1`，线程 ID：17）处理该请求。
3. **Context**：匹配到 Spring Boot 应用（根路径`/`），将请求转发给`DispatcherServlet`（Spring MVC 核心 Servlet）。

#### 步骤 3：Spring MVC 核心处理

1. **DispatcherServlet**：接收请求，调用`HandlerMapping`（`RequestMappingHandlerMapping`）。
2. **HandlerMapping**：根据 URL`/api/user/{id}`和 GET 方法，匹配到`UserController.getUser()`方法，返回`HandlerExecutionChain`（包含 Handler 和拦截器）。
3. **拦截器`preHandle`**：检查请求头的 token=admin123，返回 true 放行。
4. **DispatcherServlet**：调用`HandlerAdapter`（`RequestMappingHandlerAdapter`）。
5. **HandlerAdapter**：处理参数绑定（`@PathVariable Long id`→1，`@RequestParam String username`→张三），执行`getUser()`方法。
6. **Controller 层**：封装参数为`UserQueryDTO`，调用`UserService.getUserInfo()`。
7. **Service 层**：模拟数据库查询，转换为`UserVO`返回。
8. **Controller 层**：返回`ResultVO.success(userVO)`。
9. **HttpMessageConverter**：将`ResultVO<UserVO>`序列化为 JSON 数据（`MappingJackson2HttpMessageConverter`处理）。
10. **拦截器`postHandle`**：Controller 方法执行完成（无视图渲染，仅记录日志）。
11. **拦截器`afterCompletion`**：请求处理完成，记录响应状态码。

#### 步骤 4：响应返回

1. Spring MVC 将 JSON 数据传递给 Tomcat 的`HttpServletResponse`。
2. Tomcat 将 JSON 转为 HTTP 响应，通过 Connector 返回给前端 Axios。
3. 前端接收 JSON 数据，渲染用户信息页面。

#### 后端控制台日志（关键输出）

```plaintext
【preHandle】Tomcat线程ID：17，请求URL：http://localhost:8080/api/user/1，请求方法：GET
【postHandle】Controller方法执行完成
【afterCompletion】请求处理全流程完成，响应状态码：200
```

#### 后端返回的 JSON 数据

```json
{
  "code": 200,
  "message": "操作成功",
  "data": {
    "id": 1,
    "username": "张三",
    "age": 20,
    "email": "zhangsan@example.com"
  }
}
```

### 5、前后端分离架构的关键特性总结

1. **V 层完全迁移**：前端框架（Vue/React）接管 View 层，负责数据渲染和用户交互，后端仅提供 JSON 数据。
2. **接口标准化**：后端通过`ResultVO`封装统一的 JSON 响应格式，前后端约定响应码、消息、数据字段。
3. **跨域处理**：后端通过 CORS 配置解决前端跨域问题，这是前后端分离的必备配置。
4. **Token 认证**：替代传统的 Session-Cookie 认证，前端请求头携带 Token，后端拦截器校验。
5. **消息转换器替代视图解析器**：`HttpMessageConverter`将 Java 对象序列化为 JSON，替代传统的`ViewResolver`和`View`组件。
6. **全局异常处理**：后端通过`@RestControllerAdvice`统一处理异常，返回标准 JSON 错误信息，前端统一接收处理。

这种架构的核心优势是**前后端解耦**：前端专注于用户体验，后端专注于业务逻辑和数据处理，便于团队协作和项目维护。



## 五、跨域问题

> 前后端分离项目中，前端域名与后端域名不一致，会导致跨域问题。

### 1、什么叫“前后端域名一致”

1. 协议相同（http vs https）
2. 主机相同（a.com vs b.com，或同一条子域）
3. 端口相同（80/443 可省略，但 8080 与 80 算不同）

只要三项中任何一项不同，都算“跨域”，浏览器就会走 CORS 逻辑。

---

### 2、域名一致的典型场景（不会触发 CORS）

场景 A：最传统的“一把梭”部署
前端静态文件和后端接口都由同一台 Web 容器（Nginx/Tomcat/Apache）在同一端口提供。
浏览器地址： [https://shop.example.com](https://shop.example.com/)
Ajax 调用： https://shop.example.com/api/login
结果：同源，无需任何跨域配置。

场景 B：同主域 + 同端口，仅路径不同
浏览器地址： [http://localhost:3000](http://localhost:3000/)
Ajax 调用： http://localhost:3000/api/user
结果：同源，前端直接 fetch('/api/user') 即可。

### 3、域名不一致的典型场景（必须做 CORS 或代理）

场景 1：端口不同
浏览器地址： [http://localhost:3000](http://localhost:3000/)  （前端 dev-server）
后端接口： http://localhost:8080/api/list
→ 端口不同 → 跨域。

场景 2：子域不同
浏览器地址： [https://app.example.com](https://app.example.com/)
接口地址： [https://api.example.com](https://api.example.com/)
→ 主机不同 → 跨域。

场景 3：协议不同
浏览器地址： [http://localhost:3000](http://localhost:3000/)
接口地址： [https://api.example.com](https://api.example.com/)
→ 协议不同 → 跨域。

场景 4：纯前后端分离上线
浏览器地址： [https://www.example.com](https://www.example.com/)  （静态文件放 CDN 或 Nginx）
接口地址： [https://service.example.com](https://service.example.com/)  （后端集群）
→ 主机不同 → 跨域。

---

### 4、总结
“只要浏览器地址栏里的‘协议+主机+端口’与 Ajax 访问的接口地址三者有任何一项不同，就是前后端域名不一致，需要 CORS 或代理来解决跨域；完全一致则天然同源，无需额外配置。”



## 六、URL中主机是哪一段?

在一条 URL 里，“主机（host）”就是“协议”后面、第一个单斜杠 `/` 前面的那一整段，它由 **主机名（hostname）** + **端口（port）** 两部分组成（端口省略时走默认 80/443）。

标准格式：

```
协议 :// 用户名:密码@ 主机名 :端口 /路径?查询#片段
        ↑-----------------↑
            这一段叫“主机”
```

举例拆解：

1. `https://api.example.com:8080/v1/user/list?age=18`
   主机 = `api.example.com:8080`
   主机名 = `api.example.com`
   端口 = `8080`
2. `http://localhost:3000/index.html`
   主机 = `localhost:3000`
   主机名 = `localhost`
   端口 = `3000`

1. `https://www.example.com/search`
   主机 = `www.example.com` （默认 443 省略）
   主机名 = `www.example.com`
   端口 = `443`

因此，只要记住：
“协议后面、第一个 `/` 前面”那一截就是主机；冒号后面如果还有数字，就是端口。