###  一、 @PostConstruct注解

#### 1、核心作用

核心作用是：**标记一个方法，让这个方法在 Bean 实例化完成、所有依赖注入（DI）完成之后，且在 Bean 正式投入使用之前执行**。

#### 2、`@PostConstruct` 的执行时机

想明确 `@PostConstruct` 的执行时机是 “当前类依赖的 Bean 创建完” 还是 “整个 Spring 服务所有 Bean 创建完”，答案很明确：它只会在**当前类自身实例化完成、且当前类所依赖的 Bean 都已创建并注入完成后** 执行，而非等待整个 Spring 容器中所有 Bean 都创建完毕。

- 核心逻辑拆解

Spring 容器创建 Bean 是**按需 / 按依赖顺序**进行的，而非一次性创建所有 Bean，`@PostConstruct` 的执行完全跟随单个 Bean 的生命周期：

1. Spring 要创建 Bean A → 发现 A 依赖 Bean B → 先创建并初始化 Bean B；
2. Bean B 初始化完成（包括 B 的 `@PostConstruct` 执行）→ 将 B 注入到 A 中；
3. A 自身实例化 + 依赖注入完成 → 执行 A 的 `@PostConstruct` 方法；
4. 此时容器中可能还有其他 Bean（如 C、D）尚未创建，A 的 `@PostConstruct` 不会等待这些 Bean。

简单说：`@PostConstruct` 只保证**当前 Bean 及其直接 / 间接依赖的 Bean 都就绪**，不关心容器中其他无关 Bean 的状态。

#### 3、代码验证

通过两个无依赖的 Bean 演示，能清晰看到各自的 `@PostConstruct` 独立执行，不会等待对方

```java
import org.springframework.stereotype.Component;
import javax.annotation.PostConstruct;

// Bean A：无依赖
@Component
public class BeanA {
    @PostConstruct
    public void initA() {
        System.out.println("BeanA 的 @PostConstruct 执行");
    }
}

// Bean B：无依赖，与 BeanA 无关联
@Component
public class BeanB {
    @PostConstruct
    public void initB() {
        System.out.println("BeanB 的 @PostConstruct 执行");
    }
}

// 启动类
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
public class SpringApp {
    public static void main(String[] args) {
        System.out.println("开始初始化 Spring 容器");
        AnnotationConfigApplicationContext context = 
            new AnnotationConfigApplicationContext("com.example");
        System.out.println("Spring 容器初始化完成");
    }
}
```

**执行结果（核心顺序）**：

```java
开始初始化 Spring 容器
BeanA 的 @PostConstruct 执行  // A 就绪后立即执行，不等 B
BeanB 的 @PostConstruct 执行  // B 就绪后立即执行，不等其他
Spring 容器初始化完成
```

如果 BeanA 依赖 BeanB，执行顺序会变成：

```java
BeanB 的 @PostConstruct 执行  // 先初始化依赖的 B
BeanA 的 @PostConstruct 执行  // B 注入后，A 才执行
```

#### 4、扩展

1. 若想在**整个 Spring 容器所有 Bean 都创建完成后**执行逻辑，不能用 `@PostConstruct`，需要用 `ApplicationListener<ContextRefreshedEvent>` 或 `@EventListener(ContextRefreshedEvent.class)`；
2. `@PostConstruct` 是单个 Bean 生命周期的环节，而容器刷新完成是整个上下文的生命周期环节，二者层级不同。

