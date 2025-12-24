# 一、什么是事务？

我们常说的事务，一般指的就是`数据库事务`。

数据库事务是指：  一个`逻辑工作单元`(逻辑，意思可能有多个步骤)中执行的一系列（数据库）操作，要么一起成功，要么一起失败。

（当工作单元中的所以操作全部正确完成时，工作单元里的操作才会生效。如果检测到一个错误，程序执行回滚操作，恢复程序原状。即要么都执行要么都不执行）

（逻辑工作单元就是一个不可分割的操作序列，操作序列就是一系列的数据库操作）

`Spring的事务`是对`数据库的事务`的封装，最后`本质`的实现还是在数据库，假如`数据库`不支持事务的话，Spring的事务是没有作用的。

数据库的事务有开启、执行（数据库操作）、提交、回滚，还有个关闭是数据库连接操作。

## 1、事务的核心特性：ACID

事务必须满足**ACID 四大特性**，这是事务的基础：

| 特性                      | 英文全称    | 含义说明                                                     |
| ------------------------- | ----------- | ------------------------------------------------------------ |
| **原子性（Atomicity）**   | Atomicity   | 事务是一个不可分割的操作单元，事务中的所有操作要么全部执行，要么全部不执行。例：转账时，A 扣钱和 B 加钱必须同时完成，否则都不完成。 |
| **一致性（Consistency）** | Consistency | 事务执行前后，数据库的**完整性约束**（如主键唯一、金额不为负、总金额不变）始终保持有效。例：转账前 A+B 总金额 = 1500，转账后总金额仍需为 1500。 |
| **隔离性（Isolation）**   | Isolation   | **多个事务并发执行时，彼此之间互不干扰**，每个事务都感觉不到其他事务的存在。（用于解决脏读、不可重复读、幻读等并发问题） |
| **持久性（Durability）**  | Durability  | 事务一旦提交，其修改会永久保存到数据库中，即使系统崩溃、断电，数据也不会丢失。 |

> **以上事务的四大特性，是事务的本质特征和最终目标，`不是实现方式`**

## 2、事务的分类（从`实现`角度）

### 1. 数据库原生事务

这是最基础的事务，由数据库直接支持（如 MySQL 的 InnoDB 引擎），通过 SQL 语句手动控制：

```sql
-- 开启事务
START TRANSACTION;
-- 执行操作
UPDATE account SET balance = balance - 100 WHERE name = '张三';
UPDATE account SET balance = balance + 100 WHERE name = '李四';
-- 提交事务（成功则持久化）
COMMIT;
-- 若出错，回滚事务（恢复到操作前）
-- ROLLBACK;
```

### 2. 编程式事务

开发者通过**代码手动控制事务的开启、提交、回滚**，灵活性高，适用于复杂业务场景。例如 Spring 中的`TransactionTemplate`（前文案例中的编程式事务部分）。

```java
TransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
TransactionStatus status = txManager.getTransaction(def);  // 开启
try {
    // 业务 SQL …
    txManager.commit(status);               // 提交
} catch (Exception e) {
    txManager.rollback(status);             // 回滚
}
```

### 3. 声明式事务

通过**注解或 XML 配置**实现事务管理，无需侵入业务代码，是 Spring 中最常用的方式。例如 Spring 的`@Transactional`注解（前文案例中的声明式事务部分）。

```java
@Transactional(propagation = Propagation.REQUIRED,
               isolation = Isolation.READ_COMMITTED,
               timeout = 10)
public void placeOrder(Order order) {
    // 纯业务代码，无 txManager 影子
}
```

背后流程（AOP 代理）：

1. 方法进入 → 拦截器解析 @Transactional 属性，封装成 TransactionDefinition；
2. 调用 txManager.getTransaction(def) 得到 TransactionStatus；
3. 业务执行完 → 拦截器根据异常与否调用 txManager.commit(status) 或 rollback(status)。

> 理解这一点，看源码或调 bug 时就能快速定位：注解失效 → 看 AOP 代理；回滚没生效 → 看 TransactionStatus 的 rollbackOnly 标记。

### 4、总结

三大基础设施是 Spring 提供的“通用事务零件”；
编程式事务 = 开发者自己组装三大零件；
声明式事务 = Spring 用 AOP 帮开发者自动组装三大零件。

## 3、Spring 事务与数据库事务的关系

Spring 事务**底层依赖数据库的原生事务**，它只是提供了更便捷的封装（如声明式事务），让开发者无需手动写`START TRANSACTION`/`COMMIT`/`ROLLBACK`，并扩展了事务的传播行为、隔离级别等特性。

例如：Spring 的`@Transactional`注解最终会通过 AOP 代理，在方法执行前调用数据库的`START TRANSACTION`，方法执行成功后调用`COMMIT`，异常时调用`ROLLBACK`。



# 二、Spring 中的事务

## 1、 三大基础设施（了解）

Spring 中对事务的支持提供了三大基础设施：

### 1、**PlatformTransactionManager**

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition);
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

定义：事务管理器接口，**`屏蔽不同持久化技术`**（JDBC、JPA、Hibernate、MyBatis…）的差异。
核心方法：getTransaction、commit、rollback。
实现类举例：

- DataSourceTransactionManager（原生 JDBC）
- JpaTransactionManager
- HibernateTransactionManager

### 2、**TransactionDefinition**

> 使用了编程式事务的话，直接使用 `DefaultTransactionDefinition` 即可。

定义：用来描述事务的具体规则，**也称作`事务的属性`**。告诉管理器“我要什么级别、什么传播行为、超时多久、是否只读”。

1. getName()，获取事务的名称
2. getIsolationLevel()，获取事务的隔离级别
3. getPropagationBehavior()，获取事务的传播性
4. getTimeout()，获取事务的超时时间
5. isReadOnly()，获取事务是否是只读事务

### 3、**TransactionStatus**

> TransactionStatus 可以直接理解为事务本身。

定义：一个“当前事务句柄”，保存了事务运行期状态（是否已提交、是否回滚、是否已完成、保存点等）。
由 TransactionManager 在 getTransaction() 时返回，后续 commit/rollback 都靠它。

```java
public interface TransactionStatus extends SavepointManager, Flushable {
    boolean isNewTransaction();
    boolean hasSavepoint();
    void setRollbackOnly();
    boolean isRollbackOnly();
    void flush();
    boolean isCompleted();
}
```

## 2、编程式事务

通过 `PlatformTransactionManager `或者 `TransactionTemplate `可以实现编程式事务。

在 Spring Boot 项目中，这两个对象 Spring Boot 会自动提供，我们直接使用即可。

### 1、PlatformTransactionManager实现编程式事务

```java
@Service
public class TransferService {
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Autowired
    PlatformTransactionManager txManager;

    public void transfer() {
        DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
        TransactionStatus status = txManager.getTransaction(definition);
        try {
            jdbcTemplate.update("update user set account=account+100 where username='zhangsan'");
            int i = 1 / 0;
            jdbcTemplate.update("update user set account=account-100 where username='lisi'");
            txManager.commit(status);
        } catch (DataAccessException e) {
            e.printStackTrace();
            txManager.rollback(status);
        }
    }
}
```

> 在 `try...catch...` 中进行业务操作，没问题就 commit，有问题就 rollback；如果我们需要配置事务的隔离性、传播性等，可以在 DefaultTransactionDefinition 对象中进行配置。

### 2、TransactionTemplate 实现编程式事务

```java
import com.waves.task.dao.TaskCopyDao;
import com.waves.task.domain.entity.TaskCopy;
import org.springframework.dao.DataAccessException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionCallbackWithoutResult;
import org.springframework.transaction.support.TransactionSynchronizationManager;
import org.springframework.transaction.support.TransactionTemplate;

import javax.annotation.Resource;
import java.util.concurrent.CompletableFuture;

/**
 * 编程式事务
 */
@Service
public class TxDService {

    @Resource
    private TaskCopyDao taskCopyDao;

    @Resource
    TransactionTemplate transactionTemplate;

    public void handle4(Integer id) {
        TaskCopy taskCopy = taskCopyDao.getById(id);
        taskCopy.setDescription("DService描述");
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                try {
                    taskCopyDao.updateById(taskCopy);
                    if (1 == 1) {
                        throw new RuntimeException();
                    }
                } catch (Exception e) {
                    status.setRollbackOnly();
                    e.printStackTrace();
                }
            }
        });
    }
}
```



>  编程式事务由于代码入侵太严重了，因为在实际开发中使用的很少，我们在项目中更多的是使用声明式事务.



### 3、声明式事务

声明式事务如果使用 `XML` 配置，可以做到无侵入；如果使用 `Java` 配置，也只有一个 `@Transactional` 注解侵入而已，相对来说非常容易。

#### 1、 底层实现

1.当目标类被Spring管理时，Spring 会为目标对象创建一个代理对象。代理对象负责拦截目标方法的调用，并在必要时应用事务管理

2.代理对象内部包含一个事务拦截器TransactionInterceptor，负责处理事务相关的逻辑

3.事务拦截器会检查方法上是否添加了@Transactional注解，来决定是否应用事务

4.事务拦截器在目标方法执行前后应用事务通知。在方法执行前，事务拦截器启动事务；在方法执行后，根据方法的执行结果决定事务的提交或回滚

（事务拦截器还负责处理事务的传播行为）

如果想更深入了解@Transactional，建议最好**先学习一下 AOP**

![img](https://cdn.nlark.com/yuque/0/2025/png/47719599/1738676960991-99ab4a72-f23d-4440-907e-cd10e1ca51b0.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_27%2Ctext_5LuZ5Y-v%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

#### 2、Java 配置来实现声明式事务

最最关键的就两个：

- 事务管理器 PlatformTransactionManager
- @EnableTransactionManagement 注解开启事务支持
  - 没有写 `basePackages` 时，`@ComponentScan` 默认以当前类所在包为起点。
  - `@EnableTransactionManagement` 会往容器里注册一个 `TransactionInterceptor` + 对应的切面，让 `@Transactional` 生效

```java
@Configuration  // 1. 标记“这是配置类”，Spring 会在这里解析 @Bean
@ComponentScan  // 2. 默认扫描当前包及其子包，凡带 @Component/@Service/@Repository/@Controller 都会注册
//开启事务注解支持
@EnableTransactionManagement // 3. 关键：开启事务注解驱动，相当于 xml 里的 <tx:annotation-driven/>
public class JavaConfig {
  
  	/**
  		使用 Spring 自带的最简单数据源 DriverManagerDataSource；
			优点：零依赖；缺点：无连接池，
			** 生产请换成 HikariCP/Druid。**
			url 写成 jdbc:mysql:///test01 等价于 jdbc:mysql://localhost:3306/test01，依赖本机 MySQL。
  	*/
    @Bean
    DataSource dataSource() {  // 数据源
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setPassword("123");
        ds.setUsername("root");
        ds.setUrl("jdbc:mysql:///test01?serverTimezone=Asia/Shanghai");
        ds.setDriverClassName("com.mysql.cj.jdbc.Driver");
        return ds;
    }

  	// 把上一步的 DataSource 注入进来，构造一个 JdbcTemplate 并放进容器。
		// 后续在 DAO/Service 里直接 @Autowired JdbcTemplate 即可执行 SQL。
    @Bean
    JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
		
  	// 因为数据源是纯 JDBC，所以用 DataSourceTransactionManager； 它实现了 PlatformTransactionManager 接口，是三大基础设施之一。
		// 注意这里又调了一次 dataSource() —— 由于 @Configuration 类会被 CGLIB 代理，Spring 会保证同一线程内多次调用只返回同一个单例 DataSource，不会产生多个连接池。
    @Bean
    PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

##### 1、常见坑提醒

- 没加 `@EnableTransactionManagement` → `@Transactional` 完全不起作用。
- 数据源没换成连接池 → 高并发下性能差；生产加 HikariCP：

```java
@Bean
DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:mysql://localhost:3306/test01");
    ds.setUsername("root");
    ds.setPassword("123");
    return ds;
}
```

- 事务管理器与数据访问技术必须匹配：
  JDBC/MyBatis → `DataSourceTransactionManager`；
  JPA/Hibernate → `JpaTransactionManager` / `HibernateTransactionManager`。

##### 2、总结

- `JavaConfig` 用 4 个 `@Bean` 把“数据源 + JdbcTemplate + 事务管理器”注入容器

- 再用 `@EnableTransactionManagement` 开启切面，让后续 `@Transactional` 可以全自动提交/回滚；
- 这就是 Spring 纯注解事务的最小可运行骨架。

#### 3、事务失效

1. **类的自调用**，如果我们不通过代理对象来调用，那代理对象内部的事务拦截器就不会拦截到这次行为，行为都没有获取到怎么可能对他应用事务
2. **在私有方法上**，添加 @Transactional 注解也不会生效，Spring 的事务管理是基于 AOP实现的，AOP 代理无法拦截 目标对象内部的私有方法调用
3. **使用了多线程**，在主线程中开启的事务不会自动传播到其创建并执行的子线程中
4. 事务回滚必须要有运行时异常，如果被**捕获**了自然也不会生效了
5. 使用了一些可以脱离当前线程的传播性行为，如REQUIRES_NEW和NOT_SUPPORTED

### 4、Spring事务的属性

![未命名文件 (14).png](https://cdn.nlark.com/yuque/0/2025/png/47719599/1743686666608-bfdebd52-6d8c-41e1-afa2-a7b5958d149d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_38%2Ctext_5LuZ5Y-v%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fformat%2Cwebp)

常用的五种事务属性，分别是隔离性、传播性、回滚规则、是否只读、超时时间。

我们说的事务属性，通常指的是 `Spring 事务属性`。

#### 1、隔离性

> 解决的核心问题是：**并发事务之间互相干扰导致的数据不一致**。

MySQL 中有四种不同的隔离级别  Spring 中默认的事务隔离级别是 default，即数据库本身的隔离级别是啥就是啥

1. **Isolation.DEFAULT：**默认的事务隔离级别，以连接的数据库的事务隔离级别为准
2. **Isolation.READ_UNCOMMITTED**：读未提交，可以读取到未提交的事务，存在脏读 【实际开发几乎不用】
3. <u>**Isolation.READ_COMMITTED**：读已提交，只能读取到已经提交的事务，解决了脏读，存在不可重复读 【实际开发常用】</u>
4. <u>**Isolation.REPEATABLE_READ**：可重复读，解决了不可重复读，但存在幻读（**MySQL 数据库默认**的事务隔离级别）【实际开发常用】</u>
5. **Isolation.SERIALIZABLE**：串行化，可以解决所有并发问题，但性能太低

> - 脏读：读到别人**未提交**的修改。
> - 不可重复读：读到别人**已提交**的**修改**（update），导致同一事务两次读`同一行`值不同。
> - 幻读：读到别人**已提交**的**新增/删除**（insert/delete），导致同一事务两次读`同一范围`行数不同。

#### 2、传播性

> 当一个`事务方法`调用另一个`事务方法`时，**传播行为决定了新方法的事务如何处理**。

当一个事务方法被另一个事务方法调用时（即 A 方法调用 B 方法），事务会以何种状态存在，这些规则就涉及到事务的传播性。三条规则来定义这7种传播性：

① 创建新事务还是加入当前事务 

② 多个事务并存的时候是否会互相影响

③ 有事务抛异常、还是没事务抛异常

Spring 定义了如下七种事务传播性：

```java
public enum Propagation {
    REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
    SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),
    MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),
    REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
    NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),
    NEVER(TransactionDefinition.PROPAGATION_NEVER),
    NESTED(TransactionDefinition.PROPAGATION_NESTED);
    private final int value;
    Propagation(int value) { this.value = value; }
    public int value() { return this.value; }
}
```

| 传播性           | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| REQUIRED（默认） | 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务 |
| SUPPORTS         | 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行 |
| MANDATORY        | 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常 |
| REQUIRES_NEW     | 创建一个新的事务，如果当前存在事务，则把当前事务挂起（脱离当前事务的影响） |
| NOT_SUPPORTED    | 以非事务方式运行，如果当前存在事务，则把当前事务挂起（脱离当前事务的影响） |
| NEVER            | 以非事务方式运行，如果当前存在事务，则抛出异常               |
| NESTED           | 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，就创建一个新的事务 |

#### 3、回滚规则

默认情况下，事务只有遇到运行期异常（RuntimeException 的子类）以及 Error 时才会回滚，在遇到检查型（Checked Exception）异常时不会回滚

例如 发生 IOException 并不会导致事务回滚。如果在发生 IOException 时也能触发事务回滚，可以按照如下方式配置：

```java
@Transactional(rollbackFor = IOException.class)
public void handle2() {
    jdbcTemplate.update("update user set money = ? where username=?;", 1, "zhangsan");
    accountService.handle1();
}
```

我们也可以指定在发生某些异常时不回滚，例如当系统抛出 ArithmeticException 异常并不触发事务回滚，配置方式如下：

```java
@Transactional(noRollbackFor = ArithmeticException.class)
public void handle2() {
    jdbcTemplate.update("update user set money = ? where username=?;", 1, "zhangsan");
    accountService.handle1();
}
```

#### 4、是否只读

一般用在一个业务方法中全部都是查询的代码，没有增删改。作用就是让这些相同的查询可以查到相同的结果

只要在 Transactional 注解中添加 readOnly = true，就能让这些相同的查询查到相同的结果

那这个不就是事务隔离级别中的可重复读吗？就算不加 readOnly = true 不也可以实现这个效果？确实加个 Transactional 就可以将这些相同的查询添加到一个事务中，利用隔离级别的可重复读也可以做到，但是加了readOnly = true，事务就会知道你这是一个只读事件，就可以获得数据库和驱动的优化，性能就会更好

当然了解一下就可以了，如果你的方法里，出现了大量的一样的查询，还是比较推荐的

```java
@Transactional(readOnly = true)
```

#### 5、 超时时间

超时时间是一个事务允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务

```java
@Transactional(timeout = 10)
```

在 `TransactionDefinition`中以 int 的值来表示超时时间，其单位是秒，默认值为-1



# 三、SpringCloud分布式事务

