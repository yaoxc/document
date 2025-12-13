## 一、Aop与代理是什么关系？

> 一句话：
> **AOP 是“想法”——把通用功能横插到业务里；代理是“手段”——Spring 用来实现这个想法的底层技术。**

### 1、代码层面

Spring AOP 只支持**方法级**拦截，做法就是：

- 1.启动时扫描切面 → 
- 2.给目标 Bean 生成一个**代理对象**（JDK 动态代理 或 CGLIB）→ 
- 3.外界注入/获取的实际上是代理 → 
- 4. 代理先执行切面逻辑，再调原始对象。

因此：**代理是 AOP 的落地载体；没有代理，Spring AOP 就玩不转。**

### 2、图例

```xml
AOP 概念（切面/切点/通知）
        │
        ▼
Spring AOP 运行时
        │
        ├─> 有接口 ──JDK 动态代理──> 代理对象
        └─> 无接口 ──CGLIB 子类───> 代理对象
        │
        ▼
代理对象负责：先执行增强逻辑，再转发给原始对象
```

所以：
**AOP 是目标，代理是路线；**
**AOP 说“我要加功能”，代理说“我来帮你加”。**



## 二、切面/切点/通知 都是什么？

把 AOP 想象成“**给方法加套餐**”的过程，一句话先记住：

> 切面 = 套餐菜单
> 切点 = 哪类订单适用
> 通知 = 具体加的菜（什么时候加）

------

### 1、切面（Aspect）——“套餐菜单”

一个类，里面装着**同一类横切功能**的所有代码。
示例：

```java
@Aspect
@Component
public class LogAspect { … }   // 整个 LogAspect 就是“日志套餐”
```

------

### 2、切点（Pointcut）——“哪类订单适用”

>  一条**匹配规则**，用来挑方法；只有被挑中的方法才享受套餐。
> 写法：`execution(* com.example.service.*.*(..))`
> 读法：service 包下所有 public 方法。

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}   // 起个名字，方便后面引用
```

------

### 3、通知（Advice）——“具体加的菜 & 什么时候加”

真正要执行的代码，分 5 种：

| 通知注解          | 执行时机                   | 特点                                     |
| :---------------- | :------------------------- | :--------------------------------------- |
| `@Before`         | 方法执行前                 | 无返回值，不能阻止方法执行               |
| `@AfterReturning` | 方法正常执行后             | 可获取方法返回值（通过 returning 属性）  |
| `@AfterThrowing`  | 方法抛出异常时             | 可获取异常对象（通过 throwing 属性）     |
| `@After`          | 方法执行后（无论是否异常） | 无返回值，相当于 finally 块              |
| `@Around`         | 方法执行前后（最灵活）     | 可控制方法是否执行、修改返回值、统计时长 |

示例：

```java
@Before("serviceLayer()")
public void before(JoinPoint jp) {
    System.out.println("【开始】" + jp.getSignature().getName());
}
```

------

### 4、图解关系

```
┌--------- 切面（套餐菜单）---------┐
│  ┌-------------------------------┐ │
│  │ 切点：service包下所有方法      │ │←---“哪些订单”
│  └-------------------------------┘ │
│  ┌-------------------------------┐ │
│  │ @Before  →  贴标签             │ │←---“什么时候加什么菜”
│  │ @Around  →  全程陪同           │ │
│  └-------------------------------┘ │
└------------------------------------┘
```

------

### 5、JoinPoint

> 连接点 = **“程序执行过程中** **能够被 AOP 插一刀的** **所有** **‘时机’**”

一句话记忆:  JoinPoint 是**候选名单**；Pointcut 是从名单里**挑出**来要动刀的那些。

在 Spring AOP 里，名单只包含：

> **“public 方法调用”**
> （因为 Spring 用代理，只能拦截方法）

---

生活例子——外卖订单

- **连接点**：订单的整个生命周期——下单、烹饪、打包、派送、签收、差评（**一个业务类中的所有public方法**）。
- **切点**：我们只拦截“派送”这个连接点，做“加餐具”逻辑（**在业务类中过滤的需要做增强的具体方法**）。
- **通知**：在“派送”时真正贴条、塞餐具。

```java
@Before("execution(* service..*(..))")
public void before(JoinPoint joinPoint) {   // ← 这就是当前被拦截的“连接点”对象
    String methodName = joinPoint.getSignature().getName();
    Object[] args     = joinPoint.getArgs();
    System.out.println("准备进入方法：" + methodName);
}
```

### 6、总结

把 AOP 想成“**给方法加套餐**”的流水线，4 个词一口气串下来：

1. 连接点 JoinPoint  
   候选名单——**所有能下刀的方法**（**Spring 里仅指 public 方法**）。
2. 切点 Pointcut  
   匹配规则——**从名单里挑出“哪些方法”真正动刀**；写成正则就是 `execution(* com.service.*.*(..))`。
3. 通知 Advice  
   具体动作——**在挑中的方法“什么时候”执行什么代码**；@Before、@After、@Around 等 5 种。
4. 切面 Aspect  
   整套套餐——**一个类里同时装着“切点 + 通知”**，即“在哪切 + 怎么切”。



## 三、为什么切面也需要家@Component注解？

“切面类”本身只是 **普通 Java 对象**，Spring AOP 的底层是 **代理模式**，而 Spring 容器只会对**它管理的 Bean** 做代理。  
因此：  

> 如果切面类**不是 Spring Bean**（没被 `@Component` 等注解标上），Spring 压根扫描不到，也就**无法为它`产生关联的业务类`创建代理对象**，结果 —— 所有 `@Before/@Around` 全部失效，控制台还看不出报错，表现为“AOP 不生效”。

加了 `@Component`（或 `@Configuration` 等）后：  
1. 切面类加 `@Component`  
   → 切面本身**只是**一个普通 Spring Bean，**不会被代理**。

2. Spring 后置处理器发现这个 Bean 带有 `@Aspect`  
   → 把它注册成**“切面定义源”**（存着切点、通知的模板）。

3. 容器**对“匹配切点的业务 Bean”**（Service、Controller…）创建代理对象  
   → 代理对象 = **业务 Bean 的包装**（JDK 或 CGLIB）。

4. 代理对象里**持有第 2 步的切面实例**；每次方法调用时  
   → 先执行切面实例里的通知逻辑，再转发到**原始业务实例**。

一句话：**切面必须先进 Spring 的“员工名单”，才能拿到“工牌”去拦截别人。**



## 四、aop编程，是给切面类(Aspect) 动态创建代理对象么?

不是。  Spring AOP 的代理对象是**“被切面拦截的业务 Bean”**的代理，而不是“切面类（Aspect）”本身的代理。

顺序再捋一次：

1. 切面类（带 `@Aspect`）先被 Spring 扫描成一个**普通 Bean**，用来存放“切点 + 通知”的定义。  
2. Spring 后置处理器读取这些定义，对**匹配切点的业务类**（Service、Controller …）创建代理对象（JDK 或 CGLIB）**【因为切面类的定义，决定给哪些业务类创建代理对象】**。  
3. 代理对象内部持有**原始业务实例**和**切面实例**；每次调用时，先执行切面实例里的通知逻辑，再转发给原始实例。

所以：  
- **代理目标 = 业务 Bean**  
- **切面 Bean = 工具人**，只负责提供拦截逻辑，自己不生成代理。

一句话：  **AOP 是给被切的类造代理，不是给切面本身造代理。**



## 五、Spring AOP 实战：用注解快速实现（开发中常用）

Spring AOP 简化了 AOP 的实现，不需要手动写动态代理，只需要通过**注解配置切面**即可。下面用 Spring Boot 项目演示，实现日志监控功能。

### 1、环境准备

创建一个 Spring Boot 项目，引入**spring-boot-starter**依赖（默认包含 AOP 相关依赖）；

### 2、项目结构

```plaintext
src/main/java/com/aop/demo/
├── AopDemoApplication.java // 启动类
├── service/ // 核心业务层
│   ├── UserService.java
│   └── impl/UserServiceImpl.java
└── aspect/ // 切面类
    └── LogAspect.java
```

#### 步骤 1：编写核心业务代码

```java
// 1. 业务接口
package com.aop.demo.service;

public interface UserService {
    void createUser(String username);
    void deleteUser(Long id);
}

// 2. 业务实现类
package com.aop.demo.service.impl;

import com.aop.demo.service.UserService;
import org.springframework.stereotype.Service;

@Service // 交给Spring容器管理
public class UserServiceImpl implements UserService {
    @Override
    public void createUser(String username) {
        System.out.println("【核心业务】创建用户：" + username);
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void deleteUser(Long id) {
        System.out.println("【核心业务】删除用户：" + id);
        // 模拟异常
        // if (id < 0) {
        //     throw new RuntimeException("用户ID不能为负数");
        // }
    }
}
```

#### 步骤 2：编写切面类（核心：用注解定义切面）

```java
package com.aop.demo.aspect;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Arrays;

/**
 * 日志切面类：封装日志监控的横切逻辑
 * 注解说明：
 * @Aspect：标记这是一个切面类
 * @Component：交给Spring容器管理（否则Spring无法识别）
 */
@Aspect
@Component
public class LogAspect {

    /**
     * 定义切入点：指定哪些方法需要执行切面逻辑
     * 表达式解释：execution(* com.aop.demo.service.*.*(..))
     * - execution：匹配方法执行的连接点
     * - *：返回值任意
     * - com.aop.demo.service.*：service包下的所有类
     * - *：所有方法
     * - (..)：参数任意
     */
    @Pointcut("execution(* com.aop.demo.service.*.*(..))")
    public void servicePointcut() {
        // 这是一个空方法，仅用于承载@Pointcut注解
    }

    /**
     * 前置通知：方法执行前执行
     * @param joinPoint 连接点对象，可获取方法信息、参数等
     */
    @Before("servicePointcut()")
    public void beforeAdvice(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName(); // 获取方法名
        Object[] args = joinPoint.getArgs(); // 获取方法参数
        System.out.println("【前置日志】方法" + methodName + "开始执行，参数：" + Arrays.toString(args));
    }

    /**
     * 后置通知：方法正常执行后执行（异常时不执行）
     * @param joinPoint 连接点
     */
    @AfterReturning("servicePointcut()")
    public void afterReturningAdvice(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("【后置日志】方法" + methodName + "执行成功");
    }

    /**
     * 异常通知：方法抛出异常时执行
     * @param joinPoint 连接点
     * @param e 异常对象（需和注解的throwing属性名一致）
     */
    @AfterThrowing(value = "servicePointcut()", throwing = "e")
    public void afterThrowingAdvice(JoinPoint joinPoint, Exception e) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("【异常日志】方法" + methodName + "执行失败，原因：" + e.getMessage());
    }

    /**
     * 最终通知：方法执行后无论是否异常，都会执行
     * @param joinPoint 连接点
     */
    @After("servicePointcut()")
    public void afterAdvice(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("【最终日志】方法" + methodName + "执行结束");
    }

    /**
     * 环绕通知：最强大的通知，可控制方法的执行时机（前置+后置+异常+最终）
     * @param proceedingJoinPoint 可执行目标方法的连接点
     * @return 目标方法的返回值
     * @throws Throwable 异常
     */
    @Around("servicePointcut()")
    public Object aroundAdvice(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        String methodName = proceedingJoinPoint.getSignature().getName();
        long startTime = System.currentTimeMillis();
        Object result = null;

        try {
            // 执行前置逻辑（相当于@Before）
            // 执行目标方法
            result = proceedingJoinPoint.proceed();
            // 执行后置逻辑（相当于@AfterReturning）
        } catch (Exception e) {
            // 执行异常逻辑（相当于@AfterThrowing）
            throw e;
        } finally {
            // 执行最终逻辑
            long endTime = System.currentTimeMillis();
            System.out.println("【环绕日志】方法" + methodName + "执行时长：" + (endTime - startTime) + "ms");
        }

        return result;
    }
}
```

#### 步骤 3：编写启动类并测试

```java
package com.aop.demo;

import com.aop.demo.service.UserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class AopDemoApplication {
    public static void main(String[] args) {
        // 启动Spring容器
        ApplicationContext context = SpringApplication.run(AopDemoApplication.class, args);

        // 获取UserService的代理对象（Spring自动生成）
        UserService userService = context.getBean(UserService.class);

        // 调用方法，自动执行切面逻辑
        userService.createUser("张三");
        System.out.println("------------------------");
        userService.deleteUser(1L);

        // 测试异常场景
        // userService.deleteUser(-1L);
    }
}
```

### 3、执行结果

```plaintext
【前置日志】方法createUser开始执行，参数：[张三]
【核心业务】创建用户：张三
【后置日志】方法createUser执行成功
【最终日志】方法createUser执行结束
【环绕日志】方法createUser执行时长：102ms
------------------------
【前置日志】方法deleteUser开始执行，参数：[1]
【核心业务】删除用户：1
【后置日志】方法deleteUser执行成功
【最终日志】方法deleteUser执行结束
【环绕日志】方法deleteUser执行时长：0ms
```

### 4、关键说明：Spring AOP 的通知类型

| 通知注解          | 执行时机                   | 特点                                     |
| ----------------- | -------------------------- | ---------------------------------------- |
| `@Before`         | 方法执行前                 | 无返回值，不能阻止方法执行               |
| `@AfterReturning` | 方法正常执行后             | 可获取方法返回值（通过 returning 属性）  |
| `@AfterThrowing`  | 方法抛出异常时             | 可获取异常对象（通过 throwing 属性）     |
| `@After`          | 方法执行后（无论是否异常） | 无返回值，相当于 finally 块              |
| `@Around`         | 方法执行前后（最灵活）     | 可控制方法是否执行、修改返回值、统计时长 |

## 六、Spring AOP 进阶：常用场景实战

### 场景 1：权限校验（前置通知）

需求：对`admin`开头的方法，校验用户是否为管理员，非管理员则抛出异常。

```java
// 在LogAspect中添加以下代码
/**
 * 权限校验切入点：匹配所有admin开头的方法
 */
@Pointcut("execution(* com.aop.demo.service.*.admin*(..))")
public void adminPointcut() {}

/**
 * 前置通知：权限校验
 */
@Before("adminPointcut()")
public void checkPermission(JoinPoint joinPoint) {
    // 模拟获取当前用户（实际项目中可从Session、Token中获取）
    String currentUser = "user"; // 非管理员
    // String currentUser = "admin"; // 管理员

    if (!"admin".equals(currentUser)) {
        throw new RuntimeException("权限不足：非管理员不能执行" + joinPoint.getSignature().getName() + "方法");
    }
    System.out.println("【权限校验】用户" + currentUser + "有权限执行方法");
}

// 在UserService中添加admin方法
public interface UserService {
    // ... 原有方法
    void adminDeleteAllUser();
}

// 在UserServiceImpl中实现
@Override
public void adminDeleteAllUser() {
    System.out.println("【核心业务】管理员删除所有用户");
}

// 测试
// userService.adminDeleteAllUser(); // 非管理员会抛异常
```

### 场景 2：事务管理（环绕通知，Spring 底层实现）

Spring 的`@Transactional`注解就是通过 AOP 实现的，核心逻辑是：

1. 前置：开启事务；
2. 执行目标方法；
3. 后置：提交事务；
4. 异常：回滚事务。

我们可以模拟实现一个简单的事务注解：

```java
// 1. 自定义事务注解
package com.aop.demo.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyTransactional {
}



// 2. 事务切面类
package com.aop.demo.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class TransactionAspect {

    // 切入点：匹配标注了@MyTransactional的方法
    @Pointcut("@annotation(com.aop.demo.annotation.MyTransactional)")
    public void transactionPointcut() {}

    // 环绕通知：实现事务逻辑
    @Around("transactionPointcut()")
    public Object transactionAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = null;
        try {
            System.out.println("【事务】开启事务");
            // 执行目标方法
            result = joinPoint.proceed();
            System.out.println("【事务】提交事务");
        } catch (Exception e) {
            System.out.println("【事务】回滚事务，原因：" + e.getMessage());
            throw e;
        }
        return result;
    }
}

// 3. 在业务方法上添加注解
@Override
@MyTransactional
public void createUser(String username) {
    System.out.println("【核心业务】创建用户：" + username);
    // 模拟异常，触发事务回滚
    // if ("李四".equals(username)) {
    //     throw new RuntimeException("用户已存在");
    // }
}

// 测试
// userService.createUser("李四"); // 触发事务回滚
```

## 七、AOP 的优缺点和适用场景

### 1. 优点

- **代码解耦**：核心业务和横切逻辑分离，便于维护；
- **代码复用**：横切逻辑（日志、事务）只需写一次，作用于多个方法；
- **灵活性高**：通过配置即可修改横切逻辑的作用范围，无需修改业务代码。

### 2. 缺点

- **增加复杂度**：动态代理会增加少量性能开销（微乎其微）；
- **调试难度提升**：代理对象的执行流程比原生对象复杂。

### 3. 适用场景

- **日志记录**：方法执行的日志、参数、返回值记录；
- **性能监控**：统计方法执行时长、接口响应时间；
- **事务管理**：方法执行的事务开启、提交、回滚；
- **权限校验**：方法执行前的用户权限验证；
- **缓存处理**：方法执行前先查缓存，执行后更新缓存；
- **异常处理**：统一的异常捕获和处理。