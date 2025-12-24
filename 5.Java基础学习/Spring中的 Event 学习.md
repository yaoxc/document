

# 一、观察者模式

观察者模式 =「一个对象状态变了，自动通知所有关心它的人」
角色只有 2 个：

1. Subject（被观察者）——维护一份观察列表，状态变更时循环通知。
2. Observer（观察者）——收到通知后各自干活。

流程（3 步）：
注册 → 状态变化 → 推送通知。

# 二、Spring EventListener中谁是subject、谁是观察者

一句话对应  
- **Subject（被观察者）** → `ApplicationEventMulticaster`（由其托管所有监听器列表，并在 `publishEvent()` 时循环通知）。  
- **Observer（观察者）** → 所有被 `@EventListener` 注解的方法（背后包装成 `ApplicationListenerMethodAdapter`）。  

流程再压缩：  
任何 Bean 调 `ApplicationEventPublisher.publishEvent()` → 转给 `ApplicationEventMulticaster`（Subject） → 遍历注册的 `ApplicationListener`（Observer） → 反射调用目标方法。

---

# 三、ApplicationEventPublisher 相当于什么角色

`ApplicationEventPublisher` 只是 **“广播遥控器”** —— 它接口里只有一个 `publishEvent(Object event)` 方法，**本身既不保存监听器列表，也不负责遍历通知**。

角色定位：
- 相当于 **Subject 的“对外窗口”/门面（Facade）**；
- 真正的“被观察”逻辑（列表管理 + 通知循环）在 **ApplicationEventMulticaster** 里完成；
- 在 Spring 容器里，`ApplicationContext` 实现了 `ApplicationEventPublisher`，所以任何 Bean 都能通过注入的 `ApplicationEventPublisher` 或直接把事件扔给 `ctx.publishEvent()`，最终都转发到底层的 **ApplicationEventMulticaster** 去干活。

一句话：  
**ApplicationEventPublisher = 门面接口；ApplicationEventMulticaster = 实际 Subject。**

---

# 四、EventListener使用详解，包含为什么这么使用的原理

下面给出一份「能直接跑」的完整示例，把  
① 为什么用 @EventListener（原理）  
② 定义事件 → 发送事件 → 处理事件 全链路  
③ 同步 / 异步 / 条件过滤 用法  
一次讲透。代码基于 Spring Boot 3.x，Java 17，复制即可运行。

--------------------------------------------------
1、先搞懂原理——它到底怎么被调起来的？
--------------------------------------------------
1. 观察者模式  
   Spring 内置「事件广播器」`ApplicationEventMulticaster`。当任何 Bean 调用 `ApplicationEventPublisher#publishEvent()` 时，广播器会遍历所有匹配的监听器，反射调用目标方法 。

2. 注解解析时机  
   容器启动后，`EventListenerMethodProcessor` 把所有 `@EventListener` 方法包装成 `ApplicationListenerMethodAdapter` 并注册到广播器；发布事件时，根据方法参数类型做匹配 。

3. 同步 / 异步  
   默认同步执行；如果想异步，再加 `@Async` 并在启动类打开 `@EnableAsync`，广播器就会用线程池回调 。

> 1. 启动阶段：Spring 把每个 `@EventListener` 方法包装成 `ApplicationListenerMethodAdapter`，统一注册到 `ApplicationEventMulticaster`。【Multicaster：多路广播器】
> 2. 发布阶段：任何 Bean 调 `publishEvent()` → 广播器遍历匹配 → 反射调用。
> 3. 同步/异步分水岭：有无 `@Async` 决定「当前线程直接调用」还是「扔进线程池」。

--------------------------------------------------
2、完整代码（单模块，Spring Boot 3.2）
--------------------------------------------------
build.gradle 关键依赖

```
implementation 'org.springframework.boot:spring-boot-starter-web'
```

### 1、定义事件（可任意类，不强制继承 ApplicationEvent）

```java
public class OrderCreatedEvent {
    private final Long orderId;
    private final boolean urgent;
    public OrderCreatedEvent(Long orderId, boolean urgent) {
        this.orderId = orderId;
        this.urgent = urgent;
    }
    public Long getOrderId() { return orderId; }
    public boolean isUrgent() { return urgent; }
}
```

### 2、发送事件（Service）

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public void createOrder() {
        long id = System.currentTimeMillis();          // 模拟主键
        publisher.publishEvent(new OrderCreatedEvent(id, false)); // 普通
        publisher.publishEvent(new OrderCreatedEvent(id, true));  // 加急
    }
}
```

###  3、处理事件（三种写法）

```java
@Component
public class OrderEventListener {

    /* ① 同步监听：事件一发布就执行 */
    @EventListener
    public void onNormal(OrderCreatedEvent event) {
        if (event.isUrgent()) return;          // 加急事件交给下面方法
        doBusiness(event.getOrderId());
    }

    /* ② 条件过滤：只处理 urgent = true */
    @EventListener(condition = "#event.urgent")
    public void onUrgent(OrderCreatedEvent event) {
        doBusiness(event.getOrderId());
    }

    /* ③ 异步监听：不阻塞主线程 */
    @EventListener
    @Async   // 需要 @EnableAsync
    public void onAsync(OrderCreatedEvent event) {
        doBusiness(event.getOrderId());
    }

    private void doBusiness(Long orderId) {
        System.out.println(LocalDateTime.now() + " 处理订单 " + orderId);
    }
}
```

4. 启动类

```java
@SpringBootApplication
@EnableAsync   // 打开异步
public class Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx =
            SpringApplication.run(Application.class, args);
        // 测试：容器就绪后发一条事件
        ctx.publishEvent(new OrderCreatedEvent(1L, false));
    }
}
```

--------------------------------------------------
3、运行效果
--------------------------------------------------
启动后控制台立即输出：

```
2025-12-24T15:10:42 处理订单 1          ← 同步监听
2025-12-24T15:10:42 处理订单 1          ← 异步监听（线程名不同）
```

调用 `POST http://localhost:8080/order` 会再打印两条（普通+加急）。

--------------------------------------------------
4、常见坑 & 最佳实践
--------------------------------------------------
1. 默认同步，重 IO 任务请加 `@Async` 避免阻塞主线程 。  
2. 想控制顺序用 `@Order(数值越小越先执行)` 。  
3. 条件过滤用 SpEL，`condition = "#event.urgent"` 只处理加急 。  
4. 跨进程/重启不丢消息请把事件落库后再发 MQ，本地 `@EventListener` 仅做进程内解耦 。

--------------------------------------------------
总结：  
`@EventListener` 让你用**一个注解**就能实现**发布-订阅，同步解耦、异步提速**、条件过滤一条龙；搞懂上面 50 行核心代码，90 % 业务场景直接套用 。



# 五、ApplicationEvent 事件可以跨进程么

**不能跨进程**。

Spring 的 `@EventListener`（以及 `ApplicationEvent` 体系）本质是一套**进程内**的观察者机制：  
- 事件发布器 `ApplicationEventPublisher` 只会把消息广播给**同一 JVM 里**的 Spring 容器所管理的监听器；  
- 底层没有任何网络协议、消息持久化或跨进程路由能力，应用重启后未处理的事件也会直接丢失 。

因此，在微服务或分布式部署场景下，如果 A 服务发布事件，B 服务（不同进程）是**感知不到**的。  
要实现“跨进程/跨服务”事件驱动，需要引入**消息中间件**（RabbitMQ、Kafka、RocketMQ 等）或**分布式事件总线**（Spring Cloud Bus、Redis Pub/Sub 等），把本地事件桥接到 MQ 主题，再由下游服务订阅消费 。

一句话总结：  
**同进程解耦用 `@EventListener`，跨系统通信靠 MQ** 。

# 六、那如果服务重启，没有处理的事件不就丢失了么？

是的——**默认情况下，只要进程挂掉，所有还没处理完的 `@EventListener` 事件都会直接消失**，因为 Spring 的事件机制是纯内存的：

1. 事件对象保存在 JVM 堆内存的阻塞队列里；  
2. 消费者就是同一个进程里的 `@EventListener` 方法；  
3. 应用重启 → 队列清空 → 事件永久丢失。

也就是说，`@EventListener` 只提供「线程级/进程级」的解耦，**不保证可靠性**（既不持久化，也不重试）。  
如果业务需要「服务重启也不丢事件」，就必须把事件落到**持久化介质**（DB、Redis、MQ）里，再交给**可靠消息中间件**或**定时任务**去重新投递。常见做法：

- 本地事务 + 事件表（Outbox 模式）  
  业务表与事件表在同一个本地事务提交；后台线程或 MQ-Connector 把事件表里的记录推到 Kafka/RabbitMQ，下游服务订阅即可。  

- 直接发 MQ（事务消息/发布确认）  
  **用 RocketMQ 事务消息、Kafka 事务生产者、RabbitMQ 发布确认**，把“业务落库”和“消息落地”做成原子性；消费者端做幂等。

- Redis 延迟队列 + RDB/AOF 持久化  
  适合延迟事件，但需打开 Redis 持久化并评估恢复时间窗。

一句话：**`@EventListener` 只负责“进程内实时”，想“重启不丢”就让事件先落盘再进 MQ**。

# 七、RocketMQ、RabbitMQ 可以做持久化么？服务重启未处理的消息还能找到么

可以，而且**「持久化」正是 RocketMQ 与 RabbitMQ 的核心能力之一**。  
只要按规范开启持久化开关，即使整个集群重启，**已落盘且未被消费的消息依旧能找回并继续投递**。下面把两者机制、配置要点和注意事项一次说清。

--------------------------------------------------
一、RocketMQ 的持久化与重启恢复
--------------------------------------------------
1. 存储机制  
- 所有消息顺序写入 `CommitLog` 文件（默认 1 GB 滚动），并同步刷盘或异步刷盘；  
- 消费进度（`ConsumerQueue`）也落盘，重启后根据进度继续拉取。  

2. 刷盘方式（broker.conf）  
```properties
# SYNC_FLUSH：每条消息都落盘才返回；ASYNC_FLUSH：先写内存再定时刷盘
flushDiskType=SYNC_FLUSH
```

3. 主从同步  
- 同步复制（`brokerRole=SYNC_MASTER`）保证「主+从」都写盘成功才 ack；  
- 即使主节点物理机掉电，从节点也能在重启后接管并恢复数据。

4. 结果  
只要 broker 在崩溃前**成功刷盘**（或同步复制到从节点），服务重启后未消费的消息**零丢失**。

--------------------------------------------------
二、RabbitMQ 的持久化与重启恢复
--------------------------------------------------
RabbitMQ 需要**三个层面同时开启**持久化，缺一不可：

1. Queue 持久化  
```java
QueueBuilder.durable("order.queue").build()
```
或 Web 界面创建时勾选 **Durable**。

2. Exchange 持久化  
```java
ExchangeBuilder.directExchange("order.exchange").durable(true).build()
```

3. Message 持久化  
发送时指定 `deliveryMode = 2`（PERSISTENT）：
```java
message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
```
只有**同时满足**这三点，消息才会在内存+磁盘各存一份；节点重启后，队列、交换器、消息都能恢复。

4. 镜像队列（高可用）  
生产环境建议再配 **mirror 策略**（`ha-mode:all`），让队列在多个节点各存一份；即使某节点物理损坏，其他节点依旧能提供完整数据。

--------------------------------------------------
三、常见误区 & 避坑
--------------------------------------------------
1. 「只开队列持久化」≠「消息持久化」  
   若消息本身未标记 `PERSISTENT`，重启后队列还在，但消息会清空。  
2. 刷盘瞬间宕机  
   无论是 RocketMQ 的 `CommitLog` 还是 RabbitMQ 的 `msg_store`，**正在写入而未完成**的那一批消息仍可能丢失；对绝对零丢失场景，RocketMQ 用「同步刷盘+同步复制」，RabbitMQ 用「镜像队列+生产者确认」组合。  
3. 性能权衡  
   持久化会降低吞吐；RocketMQ 异步刷盘可做到 10 万级 QPS，而同步刷盘大约折半；RabbitMQ 镜像队列也会带来 20%~30% 性能下降。

--------------------------------------------------
结论
--------------------------------------------------
- **RocketMQ**：只要 `flushDiskType=SYNC_FLUSH` 或主从同步复制成功，服务重启**未消费消息可找回**。  
- **RabbitMQ**：同时开启 **队列+交换器+消息** 持久化，并配镜像队列，重启后**消息零丢失**。  

因此，把「本地事务 + 可靠持久化 MQ」作为事件存储，就能彻底解决「应用重启导致事件丢失」的问题。
