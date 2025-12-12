## 一、ApplicationListener作用及常见用法

简单来说，它是 Spring 事件监听机制的核心接口，用于**监听 Spring 容器或业务中发布的事件，并在事件触发时执行自定义逻辑**，是实现 “发布 - 订阅” 设计模式的关键组件（**Spring 4.2+ 推荐使用 `@EventListener` 注解替代实现接口，写法更简洁**）。

### 1、核心作用

1. **监听容器生命周期事件**：比如容器启动完成、刷新完成、关闭等，实现全局初始化 / 清理逻辑；
2. **监听自定义业务事件**：比如用户注册成功、订单支付完成等，**实现 “发布 - 订阅” 模式，解耦业务逻辑**（如注册后发送短信、记录日志）；

### 2、核心原理（发布 - 订阅模型）

```xml
┌─────────────────────────────────────────────────────────┐
│                    Spring 容器（ApplicationContext）     │
│  ┌───────────────┐        ┌───────────────┐             │
│  │ 事件发布者    │        │ 事件广播器    │             │
│  │ (ApplicationEventPublisher)                 │             │
│  └───────┬───────┘        └───────┬───────┘             │
│          │                        │                     │
│          ▼                        ▼                     │
│  ┌───────────────┐        ┌───────────────┐             │
│  │ 应用事件      │        │ 事件监听器    │             │
│  │ (ApplicationEvent)     │ (ApplicationListener)       │
│  └───────┬───────┘        └───────┬───────┘             │
│          │                        │                     │
│          └────────────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

### 组件说明

1. **事件发布者（ApplicationEventPublisher）**
   - 是 Spring 容器的核心接口之一，`ApplicationContext`本身实现了该接口，因此容器可直接作为事件发布者。
   - 核心方法：`publishEvent(Object event)`，用于发布自定义或内置的应用事件。
2. **应用事件（ApplicationEvent）**
   - 所有事件的父类，封装了事件源（`source`）和事件发生时间（`timestamp`）。
   - 常见实现：`ContextRefreshedEvent`（容器刷新完成）、`ContextClosedEvent`（容器关闭）、自定义业务事件（如`OrderCreatedEvent`）。
3. **事件广播器（ApplicationEventMulticaster）**
   - 是事件分发的核心中间件，默认实现为`SimpleApplicationEventMulticaster`。
   - 作用：接收发布者的事件，根据事件类型匹配对应的监听器，并将事件分发给监听器处理（支持同步 / 异步分发）。
4. **事件监听器（ApplicationListener）**
   - 泛型接口，需指定监听的事件类型，核心方法`onApplicationEvent(E event)`用于处理事件。
   - 注册方式：`@Component`注解、`@EventListener`注解、手动注册到容器。
5. ApplicationListener 发布订阅执行流程图
   1. **事件触发**：业务代码或 Spring 容器生命周期（如容器刷新、关闭）触发事件发布。
   2. **事件发布**：通过`ApplicationContext.publishEvent()`将`ApplicationEvent`对象发送给事件广播器。
   3. **事件分发**：事件广播器根据事件类型，筛选出所有匹配的`ApplicationListener`（泛型匹配）。
   4. **事件处理**：广播器调用监听器的`onApplicationEvent()`方法，执行事件处理逻辑（默认同步，可配置异步）。
   5. **处理完成**：监听器处理完毕，整个发布订阅流程结束。

### 3、常见用法（分场景示例）

前置说明: `ApplicationListener` 是泛型接口，泛型**参数指定要监听的事件类型**，核心方法：

```java
void onApplicationEvent(E event); // 事件触发时执行的逻辑
```

#### 场景 1：监听 Spring 容器生命周期事件（最常用）

比如监听容器刷新完成（所有 Bean 初始化完毕），执行全局初始化逻辑（替代 `@PostConstruct` 的全局场景）。

```java
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;

// 1. 实现 ApplicationListener，指定监听的事件类型为 ContextRefreshedEvent
@Component // 必须交给 Spring 管理，否则无法被容器识别
public class ContextRefreshedListener implements ApplicationListener<ContextRefreshedEvent> {

    // 2. 重写 onApplicationEvent 方法，实现容器刷新完成后的逻辑
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // ContextRefreshedEvent：Spring 容器刷新完成（所有 Bean 初始化+依赖注入完成）触发
        System.out.println("=== Spring 容器刷新完成（所有 Bean 已创建）===");
        // 执行全局初始化逻辑：比如加载缓存、初始化第三方客户端、校验配置等
        initGlobalCache();
    }

    private void initGlobalCache() {
        System.out.println("全局缓存预热完成");
    }
}
```

**常用容器事件类型**：

| 事件类型                | 触发时机                             | 典型用途                     |
| ----------------------- | ------------------------------------ | ---------------------------- |
| `ContextRefreshedEvent` | 容器刷新完成（所有 Bean 初始化完毕） | 全局初始化、缓存预热         |
| `ContextStartedEvent`   | 容器启动（调用 `start()` 方法）      | 启动后台线程、定时任务       |
| `ContextClosedEvent`    | 容器关闭（调用 `close()` 方法）      | 释放资源、关闭连接、保存数据 |
| `ContextStoppedEvent`   | 容器停止（调用 `stop()` 方法）       | 暂停业务、停止定时任务       |

#### 场景 2：监听自定义业务事件（解耦业务逻辑）

比如用户注册成功后，发布 “用户注册事件”，监听器执行发送短信、记录日志等逻辑，无需侵入注册接口。

##### 步骤 1：定义自定义事件（承载数据）

```java
import org.springframework.context.ApplicationEvent;

// 自定义事件：继承 ApplicationEvent
public class UserRegisterEvent extends ApplicationEvent {
    // 事件携带的数据（比如用户ID、手机号）
    private Long userId;
    private String phone;

    // 构造方法必须调用父类构造方法（传入事件源）
    public UserRegisterEvent(Object source, Long userId, String phone) {
        super(source);
        this.userId = userId;
        this.phone = phone;
    }

    // getter 方法
    public Long getUserId() { return userId; }
    public String getPhone() { return phone; }
}
```

##### 步骤 2：发布自定义事件

```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
public class UserService {
    // 注入事件发布器（Spring 容器自动注入）
    private final ApplicationEventPublisher publisher;

    // 构造方法注入
    public UserService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    // 用户注册方法
    public void register(Long userId, String phone) {
        // 1. 执行注册核心逻辑
        System.out.println("用户 " + userId + " 注册成功");

        // 2. 发布自定义事件（解耦后续逻辑）
        publisher.publishEvent(new UserRegisterEvent(this, userId, phone));
    }
}
```

##### 步骤 3：实现监听器监听自定义事件

```java
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

// 监听器1：发送短信
@Component
public class SendSmsListener implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent event) {
        Long userId = event.getUserId();
        String phone = event.getPhone();
        System.out.println("给用户 " + userId + "（手机号：" + phone + "）发送注册成功短信");
    }
}

// 监听器2：记录日志（可新增多个监听器，无需修改注册逻辑）
@Component
public class LogRegisterListener implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent event) {
        System.out.println("记录用户 " + event.getUserId() + " 注册日志：注册时间=" + System.currentTimeMillis());
    }
}
```

##### 步骤 4：测试

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;

@ComponentScan("com.example")
public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(App.class);
        UserService userService = context.getBean(UserService.class);
        // 调用注册方法
        userService.register(1001L, "13800138000");
        context.close();
    }
}
```

**执行结果**：

```plaintext
用户 1001 注册成功
给用户 1001（手机号：13800138000）发送注册成功短信
记录用户 1001 注册日志：注册时间=1735632000000
=== Spring 容器刷新完成（所有 Bean 已创建）===
全局缓存预热完成
```

#### 场景 3：进阶用法（泛型事件 / 多事件监听）

如果需要监听多个事件，可使用泛型通配符 `?`，并在方法内判断事件类型：

```java
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

@Component
public class MultiEventListener implements ApplicationListener<ApplicationEvent> {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ContextRefreshedEvent) {
            System.out.println("监听到容器刷新事件");
        } else if (event instanceof UserRegisterEvent) {
            System.out.println("监听到用户注册事件");
        } else if (event instanceof ContextClosedEvent) {
            System.out.println("监听到容器关闭事件");
        }
    }
}
```

### 4、简化写法-->推荐（@EventListener 注解）

Spring 4.2+ 提供了更简洁的 `@EventListener` 注解，无需实现 `ApplicationListener` 接口，直接标记方法即可：

```java
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class AnnotationBasedListener {

    // 监听用户注册事件
    @EventListener(UserRegisterEvent.class)
    public void handleUserRegister(UserRegisterEvent event) {
        System.out.println("注解式监听器：给用户 " + event.getUserId() + " 发送邮件");
    }

    // 监听容器关闭事件
    @EventListener(ContextClosedEvent.class)
    public void handleContextClosed() {
        System.out.println("注解式监听器：容器关闭，释放数据库连接");
    }
}
```

### 5、总结

1. `ApplicationListener` 是 Spring 事件监听的核心接口，实现 “发布 - 订阅” 模式，解耦代码；
2. 核心用法分两类：监听容器生命周期事件（全局初始化 / 清理）、监听自定义业务事件（解耦业务逻辑）；
3. Spring 4.2+ 推荐使用 `@EventListener` 注解替代实现接口，写法更简洁。

### 6、基于 `ApplicationListener` 实现 “订单支付成功后触发库存扣减 + 消息通知” 的完整业务示例

#### 1.核心流程

1. 用户完成订单支付，系统发布「订单支付成功事件」；
2. 监听器 1：监听该事件，执行库存扣减逻辑；
3. 监听器 2：监听该事件，执行消息通知（如短信 / 站内信）逻辑；
4. 所有逻辑解耦，支付核心代码无需修改，新增 / 移除监听逻辑不影响主流程。

#### 2、完整代码实现

##### 1. 定义核心实体类（订单 / 商品）

```java
// 商品实体
public class Product {
    private Long productId;
    private String productName;
    private Integer stock; // 库存

    // 构造器、getter/setter
    public Product(Long productId, String productName, Integer stock) {
        this.productId = productId;
        this.productName = productName;
        this.stock = stock;
    }

    public Long getProductId() { return productId; }
    public String getProductName() { return productName; }
    public Integer getStock() { return stock; }
    public void setStock(Integer stock) { this.stock = stock; }
}

// 订单实体
public class Order {
    private Long orderId;
    private Long productId;
    private Integer buyNum; // 购买数量
    private String status; // 订单状态：UNPAID(未支付)/PAID(已支付)

    // 构造器、getter/setter
    public Order(Long orderId, Long productId, Integer buyNum, String status) {
        this.orderId = orderId;
        this.productId = productId;
        this.buyNum = buyNum;
        this.status = status;
    }

    public Long getOrderId() { return orderId; }
    public Long getProductId() { return productId; }
    public Integer getBuyNum() { return buyNum; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}
```

##### 2. 定义自定义事件（订单支付成功事件）

继承 `ApplicationEvent`，承载订单支付成功的核心数据：

```java
import org.springframework.context.ApplicationEvent;

// 订单支付成功事件
public class OrderPaidEvent extends ApplicationEvent {
    private Order order; // 支付成功的订单
    private String payTime; // 支付时间

    // 构造器必须调用父类构造器（传入事件源）
    public OrderPaidEvent(Object source, Order order, String payTime) {
        super(source);
        this.order = order;
        this.payTime = payTime;
    }

    // getter
    public Order getOrder() { return order; }
    public String getPayTime() { return payTime; }
}
```

##### 3. 模拟商品库存服务（库存扣减逻辑）

```java
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

// 商品库存服务（模拟数据库存储）
@Service
public class ProductStockService {
    // 模拟商品库存数据
    private static final Map<Long, Product> PRODUCT_MAP = new HashMap<>();

    // 初始化商品数据
    static {
        PRODUCT_MAP.put(1001L, new Product(1001L, "华为Mate70", 100));
        PRODUCT_MAP.put(1002L, new Product(1002L, "iPhone16", 200));
    }

    // 扣减库存
    public boolean deductStock(Long productId, Integer buyNum) {
        Product product = PRODUCT_MAP.get(productId);
        if (product == null) {
            System.out.println("商品不存在：productId=" + productId);
            return false;
        }
        if (product.getStock() < buyNum) {
            System.out.println("库存不足：商品=" + product.getProductName() + "，剩余库存=" + product.getStock() + "，购买数量=" + buyNum);
            return false;
        }
        product.setStock(product.getStock() - buyNum);
        System.out.println("库存扣减成功：商品=" + product.getProductName() + "，扣减数量=" + buyNum + "，剩余库存=" + product.getStock());
        return true;
    }

    // 查询商品库存（测试用）
    public Product getProduct(Long productId) {
        return PRODUCT_MAP.get(productId);
    }
}
```

##### 4. 模拟消息通知服务（短信 / 站内信）

```java
import org.springframework.stereotype.Service;

// 消息通知服务
@Service
public class MessageNotifyService {
    // 发送支付成功通知
    public void sendPaidNotify(Order order, String payTime) {
        System.out.println("【消息通知】订单" + order.getOrderId() + "于" + payTime + "支付成功，已为您发货，请注意查收！");
    }
}
```

##### 5. 订单支付核心服务（事件发布者）

```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

// 订单服务（核心业务，发布事件）
@Service
public class OrderService {
    // 注入事件发布器（Spring容器自动提供）
    private final ApplicationEventPublisher eventPublisher;

    // 构造器注入
    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    // 订单支付方法（核心逻辑）
    public void payOrder(Order order) {
        // 1. 执行支付核心逻辑（模拟：修改订单状态为已支付）
        order.setStatus("PAID");
        String payTime = "2025-12-10 15:30:00"; // 模拟支付时间
        System.out.println("订单" + order.getOrderId() + "支付成功，支付时间：" + payTime);

        // 2. 发布「订单支付成功事件」（解耦后续逻辑）
        eventPublisher.publishEvent(new OrderPaidEvent(this, order, payTime));
    }
}
```

##### 6. 实现事件监听器（库存扣减 + 消息通知）

```java
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

// 监听器1：订单支付成功 → 扣减库存
@Component // 必须交给Spring管理
public class OrderPaidDeductStockListener implements ApplicationListener<OrderPaidEvent> {
    // 注入库存服务
    private final ProductStockService stockService;

    // 构造器注入
    public OrderPaidDeductStockListener(ProductStockService stockService) {
        this.stockService = stockService;
    }

    @Override
    public void onApplicationEvent(OrderPaidEvent event) {
        // 获取事件中的订单数据
        Order order = event.getOrder();
        // 执行库存扣减
        stockService.deductStock(order.getProductId(), order.getBuyNum());
    }
}

// 监听器2：订单支付成功 → 发送消息通知
@Component
public class OrderPaidNotifyListener implements ApplicationListener<OrderPaidEvent> {
    // 注入消息通知服务
    private final MessageNotifyService notifyService;

    // 构造器注入
    public OrderPaidNotifyListener(MessageNotifyService notifyService) {
        this.notifyService = notifyService;
    }

    @Override
    public void onApplicationEvent(OrderPaidEvent event) {
        // 获取事件中的订单和支付时间
        Order order = event.getOrder();
        String payTime = event.getPayTime();
        // 发送通知
        notifyService.sendPaidNotify(order, payTime);
    }
}
```

##### 7. 配置 Spring 上下文 + 测试类

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

// Spring配置类（扫描组件）
@Configuration
@ComponentScan("com.example") // 替换为你的实际包名
public class SpringConfig {
}

// 测试主类
public class OrderPaidEventTest {
    public static void main(String[] args) {
        // 1. 初始化Spring容器
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class);

        // 2. 获取订单服务
        OrderService orderService = context.getBean(OrderService.class);

        // 3. 模拟创建未支付订单（订单10001，购买商品1001，数量2）
        Order order = new Order(10001L, 1001L, 2, "UNPAID");

        // 4. 执行订单支付
        orderService.payOrder(order);

        // 5. 关闭容器
        context.close();
    }
}
```

#### 4、执行结果

运行测试类后，控制台输出如下（按执行顺序）：

```plaintext
订单10001支付成功，支付时间：2025-12-10 15:30:00
库存扣减成功：商品=华为Mate70，扣减数量=2，剩余库存=98
【消息通知】订单10001于2025-12-10 15:30:00支付成功，已为您发货，请注意查收！
```

#### 5、关键说明

1. **解耦核心**：订单支付的核心代码（`OrderService.payOrder`）只负责发布事件，库存扣减、消息通知的逻辑都在监听器中，新增 / 删除监听器无需修改支付代码；
2. **多监听器执行**：Spring 会自动识别所有`@Component`标注的监听器，当事件发布时，所有监听该事件的监听器都会执行；
3. **扩展性**：如果需要新增 “支付成功后记录日志”“支付成功后积分增加” 等逻辑，只需新增一个监听器类，无需改动原有代码，符合 “开闭原则”。

#### 6、总结

1. 基于`ApplicationListener`实现业务解耦的核心是：定义自定义事件→在核心业务中发布事件→编写监听器处理后续逻辑；
2. 订单支付场景中，支付核心逻辑只负责发布事件，库存扣减、消息通知等附属逻辑通过监听器实现，降低代码耦合度；
3. 所有组件需交给 Spring 容器管理（`@Service`/`@Component`），事件发布器由 Spring 自动注入。

#### 7、优化：异步处理

添加`@Async`注解实现监听器异步执行（避免库存扣减 / 消息通知阻塞支付主流程）

##### 7-1、核心改造思路

1. 开启 Spring 异步支持（添加 `@EnableAsync` 注解）；
2. 在监听器的 `onApplicationEvent` 方法上添加 `@Async` 注解，让监听逻辑异步执行；
3. 验证异步效果：支付主流程快速完成，库存扣减 / 消息通知在后台线程执行。

##### 7-2、完整改造后代码

###### 1. 新增异步配置（开启 @Async 支持）

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

// 开启Spring异步支持，这是@Async生效的前提
@Configuration
@EnableAsync
public class AsyncConfig {
    // 可选：自定义线程池（避免使用默认线程池的弊端）
    /*
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5); // 核心线程数
        executor.setMaxPoolSize(10); // 最大线程数
        executor.setQueueCapacity(20); // 队列容量
        executor.setThreadNamePrefix("async-listener-"); // 线程名前缀
        executor.initialize();
        return executor;
    }
    */
}
```

---

###### 2. 改造监听器（添加 @Async 注解）

```java
import org.springframework.context.ApplicationListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

// 监听器1：异步扣减库存
@Component
public class OrderPaidDeductStockListener implements ApplicationListener<OrderPaidEvent> {
    private final ProductStockService stockService;

    public OrderPaidDeductStockListener(ProductStockService stockService) {
        this.stockService = stockService;
    }

    // 添加@Async，让该方法异步执行
    @Override
    @Async
    public void onApplicationEvent(OrderPaidEvent event) {
        // 模拟耗时操作（比如远程调用库存服务）
        try {
            Thread.sleep(2000); // 睡眠2秒，模拟库存扣减耗时
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        Order order = event.getOrder();
        stockService.deductStock(order.getProductId(), order.getBuyNum());
        System.out.println("【异步执行】库存扣减线程：" + Thread.currentThread().getName());
    }
}

// 监听器2：异步发送消息通知
@Component
public class OrderPaidNotifyListener implements ApplicationListener<OrderPaidEvent> {
    private final MessageNotifyService notifyService;

    public OrderPaidNotifyListener(MessageNotifyService notifyService) {
        this.notifyService = notifyService;
    }

    // 添加@Async，异步执行
    @Override
    @Async
    public void onApplicationEvent(OrderPaidEvent event) {
        // 模拟耗时操作（比如调用短信网关）
        try {
            Thread.sleep(1000); // 睡眠1秒，模拟消息发送耗时
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        Order order = event.getOrder();
        String payTime = event.getPayTime();
        notifyService.sendPaidNotify(order, payTime);
        System.out.println("【异步执行】消息通知线程：" + Thread.currentThread().getName());
    }
}
```

###### 3. 其他代码（实体 / 服务 / 测试类）保持不变

> 注：之前的 `Product`、`Order`、`OrderPaidEvent`、`ProductStockService`、`MessageNotifyService`、`OrderService`、`SpringConfig`、`OrderPaidEventTest` 代码无需修改，只需确保 `SpringConfig` 扫描到 `AsyncConfig` 所在包。

---

##### 7-3、执行结果与异步效果验证

运行 `OrderPaidEventTest` 后，控制台输出顺序如下（核心差异：主流程不阻塞）：

```plaintext
订单10001支付成功，支付时间：2025-12-10 15:30:00
// 主流程直接完成，库存扣减和消息通知在后台异步执行
【异步执行】消息通知线程：async-listener-1
【消息通知】订单10001于2025-12-10 15:30:00支付成功，已为您发货，请注意查收！
【异步执行】库存扣减线程：async-listener-2
库存扣减成功：商品=华为Mate70，扣减数量=2，剩余库存=98
```

关键对比（异步 vs 同步）

| 模式 | 主流程耗时                | 体验                 | 适用场景               |
| ---- | ------------------------- | -------------------- | ---------------------- |
| 同步 | 主流程 = 所有逻辑耗时总和 | 接口响应慢，用户等待 | 必须同步执行的核心逻辑 |
| 异步 | 主流程仅耗时核心逻辑      | 接口响应快，用户无感 | 非核心的附属逻辑       |

###### `主流程` = `所有逻辑耗时总和`，主流程怎么理解

> 答案是：**在测试类的 main 方法场景中，主流程初期由 main 线程执行；但在实际 Spring Boot 应用的 Web 场景中，主流程由 Tomcat 线程池的工作线程执行（而非 main 线程）**。我会分场景拆解，让你清晰理解不同环境下的线程执行逻辑。

3、先明确核心概念

- **主流程**：指触发核心业务的线程执行路径（比如测试类中调用`orderService.payOrder()`，或 Web 场景中处理接口请求的线程）；
- **同步执行**：监听器的`onApplicationEvent`方法会在**发布事件的线程**中执行，因此主流程线程会阻塞等待所有监听器逻辑完成，总耗时 = 主流程耗时 + 所有监听器耗时总和。

3、分场景验证线程归属

场景 1：测试类（main 方法）场景（你之前的示例）

在`OrderPaidEventTest`的`main`方法中，主流程完全由**main 线程**执行，同步模式下所有逻辑都在 main 线程中完成。

- 代码验证（添加线程名打印）:  修改`OrderService`的`payOrder`方法和监听器的`onApplicationEvent`方法，打印当前线程名：

```java
// 1. 修改OrderService的payOrder方法
public void payOrder(Order order) {
    // 打印当前线程名（主流程线程）
    System.out.println("【主流程】支付逻辑执行线程：" + Thread.currentThread().getName());
    
    order.setStatus("PAID");
    String payTime = "2025-12-10 15:30:00";
    System.out.println("订单" + order.getOrderId() + "支付成功，支付时间：" + payTime);

    // 发布事件（同步模式下，监听器会在当前线程执行）
    eventPublisher.publishEvent(new OrderPaidEvent(this, order, payTime));
}

// 2. 修改库存扣减监听器（同步模式，去掉@Async）
@Override
public void onApplicationEvent(OrderPaidEvent event) {
    System.out.println("【监听器1】库存扣减执行线程：" + Thread.currentThread().getName());
    Order order = event.getOrder();
    stockService.deductStock(order.getProductId(), order.getBuyNum());
}

// 3. 修改消息通知监听器（同步模式，去掉@Async）
@Override
public void onApplicationEvent(OrderPaidEvent event) {
    System.out.println("【监听器2】消息通知执行线程：" + Thread.currentThread().getName());
    Order order = event.getOrder();
    String payTime = event.getPayTime();
    notifyService.sendPaidNotify(order, payTime);
}
```

- 执行结果（同步模式）

```plaintext
【主流程】支付逻辑执行线程：main
订单10001支付成功，支付时间：2025-12-10 15:30:00
【监听器1】库存扣减执行线程：main
库存扣减成功：商品=华为Mate70，扣减数量=2，剩余库存=98
【监听器2】消息通知执行线程：main
【消息通知】订单10001于2025-12-10 15:30:00支付成功，已为您发货，请注意查收！
```

**结论**：测试类的 main 方法中，主流程（支付）和所有监听器逻辑都在`main`线程执行，同步模式下 main 线程会阻塞等待所有监听器完成，总耗时 = 支付耗时 + 库存扣减耗时 + 消息通知耗时。

---

场景 2：实际 Spring Boot Web 场景（生产环境）

在 Spring Boot Web 应用中，用户通过 HTTP 接口调用订单支付接口，此时主流程线程不是`main`线程，而是**Tomcat 线程池的工作线程**（命名格式如`http-nio-8080-exec-1`）。

- 代码示例（Web 接口）

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    // 支付接口
    @GetMapping("/order/pay/{orderId}")
    public String payOrder(@PathVariable Long orderId) {
        // 打印接口处理线程（Tomcat工作线程）
        System.out.println("【Web接口】处理线程：" + Thread.currentThread().getName());
        // 模拟订单
        Order order = new Order(orderId, 1001L, 2, "UNPAID");
        // 执行支付
        orderService.payOrder(order);
        return "支付成功";
    }
}
```

- 执行结果（同步模式）

```plaintext
【Web接口】处理线程：http-nio-8080-exec-1
【主流程】支付逻辑执行线程：http-nio-8080-exec-1
订单10001支付成功，支付时间：2025-12-10 15:30:00
【监听器1】库存扣减执行线程：http-nio-8080-exec-1
库存扣减成功：商品=华为Mate70，扣减数量=2，剩余库存=98
【监听器2】消息通知执行线程：http-nio-8080-exec-1
【消息通知】订单10001于2025-12-10 15:30:00支付成功，已为您发货，请注意查收！
```

**结论**：Web 场景中，主流程线程是 Tomcat 的工作线程，同步模式下该线程会阻塞等待所有监听器完成，导致接口响应慢（用户等待时间长）；而异步模式下，监听器会在`@Async`指定的线程池线程中执行，Tomcat 工作线程快速返回，提升接口响应速度。

---

3、同步 vs 异步线程执行对比

| 模式 | 主流程线程                          | 监听器执行线程   | 主流程耗时                  | 接口响应速度 |
| ---- | ----------------------------------- | ---------------- | --------------------------- | ------------ |
| 同步 | main（测试）/Tomcat 工作线程（Web） | 与主流程线程相同 | 主流程 + 所有监听器耗时总和 | 慢           |
| 异步 | main（测试）/Tomcat 工作线程（Web） | 异步线程池线程   | 仅主流程耗时                | 快           |

总结

1. 测试类的 main 方法中，主流程由`main`线程执行，同步模式下所有监听器逻辑也在`main`线程执行，总耗时是所有逻辑之和；
2. 实际 Web 场景中，主流程由 Tomcat 工作线程执行，同步模式下监听器也在该线程执行，会阻塞接口响应；
3. `@Async`的核心价值是让监听器在独立的异步线程执行，释放主流程线程（main/Tomcat 工作线程），避免阻塞。



##### 7-4、异步监听器的关键注意事项

1. **@EnableAsync 必须加**：这是 `@Async` 生效的前提，需添加在配置类上；

2. **异常处理**：异步方法的异常不会直接抛到主流程，需手动捕获处理（比如记录日志、重试）：

   ```java
   @Override
   @Async
   public void onApplicationEvent(OrderPaidEvent event) {
       try {
           // 业务逻辑
       } catch (Exception e) {
           System.out.println("库存扣减失败：" + e.getMessage());
           // 可选：重试逻辑/告警
       }
   }
   ```

3. **自定义线程池（推荐）**：默认线程池（`SimpleAsyncTaskExecutor`）无队列、每次创建新线程，高并发下易耗尽资源，建议像 `AsyncConfig` 中注释的那样自定义线程池；

4. **事务一致性**：异步逻辑若涉及数据库操作，需注意事务边界（比如主流程提交事务后，异步逻辑再执行，避免读不到未提交数据）。

##### 7-5、总结

1. 给监听器方法添加 `@Async` 并开启 `@EnableAsync`，即可实现监听逻辑异步执行，避免阻塞主流程；
2. 异步执行能显著提升核心接口的响应速度，适合库存扣减、消息通知等非核心附属逻辑；
3. 异步场景需重点关注异常处理、线程池配置和事务一致性问题。

#### 8、优化：添加自定义线程池和异步异常的全局处理机制

添加**自定义线程池**和**异步异常全局处理机制**，这样能解决默认线程池的弊端，同时统一处理异步执行中抛出的异常，保证系统稳定性。

##### 8-1、核心改造目标

1. **自定义线程池**：替代 Spring 默认的`SimpleAsyncTaskExecutor`（无队列、每次创建新线程），避免高并发下线程耗尽；
2. **全局异常处理**：通过`AsyncUncaughtExceptionHandler`捕获所有异步方法的未处理异常，统一日志记录、告警，避免异常丢失。

##### 8-2、完整改造后代码

###### 1、自定义异步配置（线程池 + 全局异常处理）

```java
import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.lang.reflect.Method;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

// 开启异步 + 实现AsyncConfigurer自定义线程池和异常处理器
@Configuration
@EnableAsync
public class CustomAsyncConfig implements AsyncConfigurer {

    /**
     * 自定义异步线程池（核心配置）
     */
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 核心线程数：默认一直存活，除非设置allowCoreThreadTimeOut=true
        executor.setCorePoolSize(5);
        // 最大线程数：核心线程满+队列满后，新建线程的上限
        executor.setMaxPoolSize(10);
        // 队列容量：核心线程满后，任务进入队列等待
        executor.setQueueCapacity(20);
        // 线程空闲超时时间：非核心线程空闲超过该时间会被销毁（默认60秒）
        executor.setKeepAliveSeconds(60);
        // 线程名前缀：便于日志排查线程归属
        executor.setThreadNamePrefix("order-async-");
        // 拒绝策略：队列满+最大线程数满时，如何处理新任务
        // CallerRunsPolicy：由提交任务的线程（如Tomcat线程）执行，避免任务丢失
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 初始化线程池
        executor.initialize();
        return executor;
    }

    /**
     * 全局异步异常处理器（捕获所有@Async方法的未处理异常）
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }

    /**
     * 自定义异步异常处理实现类
     */
    static class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
        @Override
        public void handleUncaughtException(Throwable ex, Method method, Object... params) {
            // 1. 打印详细异常日志（方法名、参数、异常栈）
            System.err.println("===== 异步方法异常全局捕获 =====");
            System.err.println("异常方法：" + method.getDeclaringClass().getName() + "." + method.getName());
            System.err.println("异常参数：" + (params.length > 0 ? params[0].toString() : "无"));
            System.err.println("异常信息：" + ex.getMessage());
            ex.printStackTrace(); // 打印异常栈，便于排查

            // 2. 可选：添加告警逻辑（如发送邮件、钉钉/企业微信通知）
            // sendAlarm(ex, method);

            // 3. 可选：异常重试逻辑（根据业务场景）
            // retryFailedTask(method, params);
        }

        // 示例：告警方法（可根据实际需求实现）
        /*
        private void sendAlarm(Throwable ex, Method method) {
            // 调用告警接口或发送消息
            System.out.println("【告警】异步方法" + method.getName() + "执行失败：" + ex.getMessage());
        }
        */
    }
}
```

###### 2. 改造监听器（模拟异常场景）

为了验证全局异常处理，在库存扣减监听器中模拟一个异常（比如库存不足）：

```java
import org.springframework.context.ApplicationListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

// 监听器1：异步扣减库存（模拟异常）
@Component
public class OrderPaidDeductStockListener implements ApplicationListener<OrderPaidEvent> {
    private final ProductStockService stockService;

    public OrderPaidDeductStockListener(ProductStockService stockService) {
        this.stockService = stockService;
    }

    // 使用自定义线程池异步执行
    @Override
    @Async
    public void onApplicationEvent(OrderPaidEvent event) {
        System.out.println("【异步执行】库存扣减线程：" + Thread.currentThread().getName());
        Order order = event.getOrder();
        // 模拟库存不足异常（购买数量100，超过初始库存100）
        boolean success = stockService.deductStock(order.getProductId(), 100);
        if (!success) {
            // 抛出运行时异常，会被全局异常处理器捕获
            throw new RuntimeException("库存扣减失败：商品ID=" + order.getProductId() + "，购买数量=" + 100);
        }
    }
}

// 监听器2：异步发送消息通知（正常执行）
@Component
public class OrderPaidNotifyListener implements ApplicationListener<OrderPaidEvent> {
    private final MessageNotifyService notifyService;

    public OrderPaidNotifyListener(MessageNotifyService notifyService) {
        this.notifyService = notifyService;
    }

    @Override
    @Async
    public void onApplicationEvent(OrderPaidEvent event) {
        System.out.println("【异步执行】消息通知线程：" + Thread.currentThread().getName());
        Order order = event.getOrder();
        String payTime = event.getPayTime();
        notifyService.sendPaidNotify(order, payTime);
    }
}
```

###### 3. 其他核心代码（实体 / 服务 / 测试类）

> 注：`Product`、`Order`、`OrderPaidEvent`、`ProductStockService`、`MessageNotifyService`、`OrderService`、`SpringConfig` 代码保持不变，只需确保 `SpringConfig` 扫描到 `CustomAsyncConfig` 所在包。

###### 4. 测试类（验证效果）

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.example") // 替换为你的实际包名
public class OrderPaidEventTest {
    public static void main(String[] args) throws InterruptedException {
        // 1. 初始化Spring容器
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(OrderPaidEventTest.class);

        // 2. 获取订单服务
        OrderService orderService = context.getBean(OrderService.class);

        // 3. 模拟创建未支付订单（订单10001，购买商品1001，数量2）
        Order order = new Order(10001L, 1001L, 2, "UNPAID");

        // 4. 执行订单支付
        orderService.payOrder(order);

        // 等待异步线程执行完成（测试用，生产环境无需等待）
        Thread.sleep(3000);

        // 5. 关闭容器
        context.close();
    }
}
```

##### 8-3、执行结果与验证

###### 1. 控制台输出（关键信息）

```plaintext
订单10001支付成功，支付时间：2025-12-10 15:30:00
【异步执行】消息通知线程：order-async-1
【消息通知】订单10001于2025-12-10 15:30:00支付成功，已为您发货，请注意查收！
【异步执行】库存扣减线程：order-async-2
库存不足：商品=华为Mate70，剩余库存=100，购买数量=100
===== 异步方法异常全局捕获 =====
异常方法：com.example.OrderPaidDeductStockListener.onApplicationEvent
异常参数：com.example.OrderPaidEvent@7a79be86
异常信息：库存扣减失败：商品ID=1001，购买数量=100
java.lang.RuntimeException: 库存扣减失败：商品ID=1001，购买数量=100
	at com.example.OrderPaidDeductStockListener.onApplicationEvent(OrderPaidDeductStockListener.java:26)
	...（异常栈省略）
```

###### 2. 关键验证点

- **线程名验证**：异步线程名以`order-async-`开头，证明使用了自定义线程池；
- **异常捕获验证**：库存扣减的异常被`CustomAsyncExceptionHandler`全局捕获，打印了详细的方法名、参数、异常栈；
- **主流程不阻塞**：支付主流程快速完成，异常不影响主流程，且异常未丢失（被全局处理）。

##### 8-4、核心配置说明

###### 1. 自定义线程池关键参数

| 参数                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| corePoolSize             | 核心线程数，默认一直存活，保证基础并发能力                   |
| maxPoolSize              | 最大线程数，核心线程满 + 队列满后，新建线程的上限（避免线程无限创建） |
| queueCapacity            | 任务队列容量，核心线程满后，任务进入队列等待（减少线程创建开销） |
| keepAliveSeconds         | 非核心线程空闲超时时间，超时销毁，节省资源                   |
| threadNamePrefix         | 线程名前缀，便于日志排查（如定位哪个线程执行的库存扣减）     |
| rejectedExecutionHandler | 拒绝策略：`CallerRunsPolicy` 让提交任务的线程执行，避免任务丢失 |

###### 2. 全局异常处理的价值

- **异常不丢失**：异步方法的异常不会抛到主流程，若不处理会静默丢失，全局处理器确保所有异常被捕获；
- **统一处理逻辑**：无需在每个异步方法中 try-catch，降低代码冗余；
- **可扩展告警**：可在处理器中添加钉钉 / 邮件告警，及时发现线上异步任务异常。

##### 8-5、总结

1. 通过实现`AsyncConfigurer`可以自定义异步线程池和全局异常处理器，解决默认线程池的弊端和异步异常丢失问题；
2. 自定义线程池的核心参数需根据业务并发量调整（如核心线程数、队列容量），拒绝策略优先选择`CallerRunsPolicy`避免任务丢失；
3. 全局异常处理器能统一捕获异步方法的未处理异常，便于日志排查和告警，提升系统稳定性。



### 7、分布式微服务下，实现ApplicationListener功能，常用什么方案

