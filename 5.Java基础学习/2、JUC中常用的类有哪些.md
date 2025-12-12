## 0、juc是单进程下的并发框架么

是的，JUC（`java.util.concurrent`）是**单进程内**的并发框架。

它所有组件（线程池、锁、原子变量、并发容器等）都基于**JVM 级别的线程机制**实现，只在**同一个 JVM 进程**里解决多线程并发问题；**不涉及跨进程通信，也不依赖外部中间件**。  
要实现跨进程/多机并发，必须借助**消息队列**、**RPC**、**分布式锁**等进程间通信技术，这些已超出 JUC 的范畴。

## 一、JUC中常用的类有哪些

JUC（`java.util.concurrent`）里常用的类按功能分组，记住这张“七张王牌”即可：

### 1、锁  
`ReentrantLock` / `ReentrantReadWriteLock` / `StampedLock`

### 2、原子变量  
`AtomicInteger` / `AtomicLong` / `AtomicReference` / `LongAdder`

###  3、并发容器  
`ConcurrentHashMap` / `CopyOnWriteArrayList` / `CopyOnWriteArraySet` / `ConcurrentLinkedQueue`

###  4、线程池  
`ThreadPoolExecutor` / `ScheduledThreadPoolExecutor` / `Executors`（工厂）

### 5、同步工具  
`CountDownLatch` / `CyclicBarrier` / `Semaphore` / `Phaser`

### 6、异步结果  
`Future` / `CompletableFuture` / `ExecutorService`

###  7、阻塞队列  
`ArrayBlockingQueue` / `LinkedBlockingQueue` / `SynchronousQueue` / `DelayQueue` / `LinkedTransferQueue`

> 日常开发先拿 **锁、原子变量、ConcurrentHashMap、ThreadPoolExecutor、CompletableFuture** 这五样，就能解决 90 % 并发问题。



## 二、原子变量 `AtomicInteger` / `AtomicLong` / `AtomicReference` / `LongAdder 有什么特性`

以下是 Java 中 `AtomicInteger`、`AtomicLong`、`AtomicReference` 和 `LongAdder` 的核心特性对比：

| 特性维度     | `AtomicInteger`                              | `AtomicLong`       | `AtomicReference`                          | `LongAdder`                                  |
| ------------ | -------------------------------------------- | ------------------ | ------------------------------------------ | -------------------------------------------- |
| **数据类型** | `int`                                        | `long`             | 任意引用类型                               | `long`                                       |
| **实现机制** | CAS + volatile                               | CAS + volatile     | CAS + volatile                             | 分片 CAS（base + Cell[]）                    |
| **并发性能** | 低竞争时高效；**高竞争时自旋重试，性能下降** | 同 `AtomicInteger` | 同 `AtomicInteger`                         | **高并发写性能极佳**，读性能略低             |
| **内存占用** | 低（单个 `int`）                             | 低（单个 `long`）  | 低（单个引用）                             | 高（base + Cell 数组）                       |
| **一致性**   | 强一致性                                     | 强一致性           | 强一致性                                   | 最终一致性（`sum()` 非原子快照）             |
| **功能接口** | 丰富原子操作（如 `compareAndSet`）           | 同 `AtomicInteger` | 支持引用原子更新（如 `compareAndSet`）     | 仅支持累加（`add`、`increment`），不支持 CAS |
| **典型场景** | 计数器、状态标志、序列号生成                 | 同 `AtomicInteger` | 无锁数据结构、乐观锁、引用原子更新         | 高并发统计（如 QPS、点击量）                 |
| **ABA 问题** | 存在（可用 `AtomicStampedReference` 解决）   | 存在               | 存在（可用 `AtomicStampedReference` 解决） | 不涉及（仅累加）                             |

---

### 使用建议：
- **低竞争或需精确控制**：选 `AtomicInteger` / `AtomicLong` / `AtomicReference`。
- **高并发写多读少**：选 `LongAdder`，牺牲强一致性换取吞吐量。

以下给出 4 类原子类在真实业务中的**最小可运行**示例，每个例子都突出其**典型场景 + 核心 API**。

---

### 1. AtomicInteger —— 高频、低竞争的**全局计数器**

场景：接口 QPS 实时看板，要求**强一致**且**读远多于写**。  
```java
public class QpsCounter {
    // AtomicInteger：线程安全的“整数盒子”，内部用 CAS 保证原子性
    // new AtomicInteger(0)：把盒子初始值设为 0
    private static final AtomicInteger QPS = new AtomicInteger(0);

    // throws InterruptedException：主线程可能被打断，先声明抛出去，省得 try-catch
    public static void main(String[] args) throws InterruptedException {
        // 模拟 10 个线程（主线程 for 循环瞬间创建 10 个线程，它们并发跑）
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                // 每线程 100 万次调用
                for (int j = 0; j < 1_000_000; j++) {
                    // 原子地把盒子里的值 +1，并返回加完后的结果
                    QPS.incrementAndGet();
                }
            }).start();
        }

        // 主线程睡 3 秒，给那 10 个工人时间干活。
		// 3 秒足够它们全部跑完（100 万 × 10 ≈ 1 000 万 次 CAS，现代 CPU 几秒搞定）
        Thread.sleep(3_000);
        System.out.println("实时 QPS = " + QPS.get()); // 永远=10 000 000
    }
}
```

一句话总结：
10 个线程并发地各累加 100 万次，利用 `AtomicInteger` 的 CAS 保证**一滴都不漏**，最后打印总数。

---

### 2. AtomicLong —— **单机 ID 生成器**（mini版**雪花算法**（Snowflake））
场景：订单号需要**递增、无跳号、可反解时间戳**。  
```java
public class IdGenerator {
    private static final AtomicLong SEQ = new AtomicLong(
            System.currentTimeMillis() / 1000 << 20   // 29-bit 秒级时间戳
    );

    public static long nextId() {
        // incrementAndGet：“先 +1，再返回新值；整个动作原子，多线程不会串票。”
        return SEQ.incrementAndGet();                 // 后 20 bit 做序列
    }

    public static void main(String[] args) {
        IntStream.range(0, 10).parallel()
                 .forEach(i -> System.out.println(nextId()));
    }
}
```
输出示例：  
```
348966092390932481
348966092390932482
...
```
保证**递增 + 无锁** 。

#### incrementAndGet()解析

是 Java 原子类（`AtomicInteger` / `AtomicLong`）里**最常用**的一个方法.

方法签名：

```java
public final long incrementAndGet()   // AtomicLong
public final int  incrementAndGet()   // AtomicInteger
```

#### incrementAndGet底层等价伪代码

```java
public final long incrementAndGet() {
    long old, newVal;
    do {
        old   = value;      // volatile 读
        newVal = old + 1;   // 加 1
    } while (!compareAndSet(old, newVal)); // CAS 失败就重试
    return newVal;          // 返回加完后的值
}
```

- **没有锁**，靠 CAS 自旋。
- **返回值是加 1 后的结果**，不是旧值。

#### 与 `getAndIncrement` 的区别

| 方法                | 返回     | 语义     |
| ------------------- | -------- | -------- |
| `incrementAndGet()` | **新值** | 先加再拿 |
| `getAndIncrement()` | **旧值** | 先拿再加 |

```java
AtomicLong l = new AtomicLong(0);
System.out.println(l.incrementAndGet()); // 打印 1
System.out.println(l.getAndIncrement()); // 打印 1，此时内部值已变 2
```

---

### 3. AtomicReference —— **无锁切换配置**
场景：热更新线程池参数，要求**并发读不阻塞、替换原子**。  
```java
public class HotConfig {
    static class Conf {
        final int core, max;
        Conf(int c, int m) { this.core = c; this.max = m; }
    }

    private static final AtomicReference<Conf> CONF =
            new AtomicReference<>(new Conf(4, 8));

    public static void update(int c, int m) {
        Conf old, neu;
        do {
            old = CONF.get();
            neu = new Conf(c, m);
        } while (!CONF.compareAndSet(old, neu));   // CAS 避免ABA
    }

    public static void main(String[] args) {
        // 线程 1 定时读
        new Thread(() -> {
            while (true) {
                Conf c = CONF.get();
                System.out.println("当前 core=" + c.core + ", max=" + c.max);
                LockSupport.parkNanos(Duration.ofSeconds(1).toNanos());
            }
        }).start();

        // 线程 2 模拟运维发配置
        LockSupport.parkNanos(Duration.ofSeconds(3).toNanos());
        update(8, 16);
    }
}
```
输出：  
```
当前 core=4, max=8
...
当前 core=8, max=16   ← 3 秒后平滑切换，无锁
```


---

### 4. LongAdder —— **高并发写、低频读**的**流量统计**
场景：网关收集 10 万并发请求的**总字节数**，读仅每 5 秒上报一次。  
```java
public class NetByteCounter {
    private static final LongAdder BYTES = new LongAdder();

    public static void main(String[] args) throws InterruptedException {
        // 10 线程，每线程 10 万次写入
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 100_000; j++) {
                    BYTES.add(ThreadLocalRandom.current().nextInt(512, 2048));
                }
            }).start();
        }

        // 后台定时读（可容忍延迟）
        Executors.newSingleThreadScheduledExecutor()
                 .scheduleAtFixedRate(() ->
                         System.out.println("5 秒累计流量=" + BYTES.sum() + " bytes"),
                         0, 5, TimeUnit.SECONDS);

        Thread.sleep(15_000);
        System.out.println("最终总量=" + BYTES.sum());
    }
}
```
利用**分片 Cell** 将写压力打散，**QPS 提升 3~5 倍**于 `AtomicLong` 。

---

### 一张图总结用法
| 原子类          | 核心动词          | 业务含义             | 选型口诀                   |
| --------------- | ----------------- | -------------------- | -------------------------- |
| AtomicInteger   | `incrementAndGet` | 实时、强一致计数     | 低竞争 + 读多写少          |
| AtomicLong      | `incrementAndGet` | 序列号、ID、自旋锁   | 需要 CAS 语义              |
| AtomicReference | `compareAndSet`   | 配置、状态、缓存     | 引用级原子更新             |
| LongAdder       | `add` / `sum`     | 监控、指标、流量统计 | 高并发写、可接受最终一致性 |

按需复制即可直接落地。



##  三、CAS 详解

下面用“一步一图”的方式，把 CAS 从**硬件原语**讲到**Java 源码**，再讲到**ABA 与解决方案**，一次性讲透。

---

### 1 硬件层：LOCK CMPXCHG
x86 提供的**原子指令**  
```
LOCK CMPXCHG  reg, [mem]   ; 伪汇编
```
执行逻辑（单条 CPU 指令完成）  
```
if (EAX == [mem])        // EAX 是旧值
    [mem] = reg;         // 写入新值
else
    EAX = [mem];         // 返回最新值
```
LOCK 前缀保证**总线锁**或**缓存锁**，**整个比较-替换**原子完成。  
> 多核并发时，硬件通过 **MESI 协议**把其它核心的对应缓存行置为 Invalid，实现“独占”。

---

### 2 Java 层：sun.misc.Unsafe
HotSpot 把指令封装成 CPU 相关的**intrinsic**（JIT 直接生成机器码）。  
```java
public final native boolean compareAndSwapInt( Object o, long offset, int expected, int x);
```
- o + offset → 内存地址  
- expected → 旧值（EAX）  
- x → 新值（reg）  
返回值 true/false 即是否替换成功。

---

### 3 源码走读：以 AtomicInteger 为例（**）
```java
public class AtomicInteger {
    private volatile int value;          // 保证可见性
    private static final long VALUE = Unsafe.objectFieldOffset(...);

    public final boolean compareAndSet(int expect, int update) {
        return U.compareAndSwapInt(this, VALUE, expect, update);
    }

    public final int incrementAndGet() {
        int old, neu;
        do {
            old = value;                 // 读 volatile
            neu = old + 1;
        } while (!compareAndSet(old, neu)); // 自旋重试
        return neu;
    }
}
```
流程图  

```
读旧值 → 计算新值 → CAS[CompareAndSet] → 失败？→ 重试
```

---

#### compareAndSet() 自旋解析(**重点理解**)

compareAndSet的职责可以用一句话概括： “当且仅当当前值是我期望的旧值时，才把它替换成新值；否则什么都不做，并告诉我成没成功。”

> public final boolean compareAndSet(int expect, int update)

   - expect：你认为现在应该是多少（旧值）。
   - update：你想把它改成多少（新值）。
   - 返回值：true 表示替换成功；false 表示“预期值不对”，没动
   - 底层伪代码：

> ```java
> 
>     // 由 CPU 一条指令保证原
> if (currentValue == expect) {
>     currentValue = update;
>     return true;
> } else {
>     return false;
> }
> ```

---

### 4 性能特征
- **无锁**：线程不会阻塞，不会陷入内核态。  
- **自旋**：高竞争时 CPU 空转，**饥饿**可能。  
- **规模效应**：核心数越多、竞争越激烈，重试次数指数级上升 → **性能崩塌**。

---

### 5 ABA 问题
时间线  
```
T1  读 A
T2  把 A→B→A
T1  CAS(A, C)  仍然成功，但中间状态 B 被“无视”
```
危害：  
- 链表/栈的**节点被回收重用**，CAS 后指针指向已释放内存 → **段错误**。  
- 金融场景**余额重复扣款**。

---

### 6 ABA 解决
#### 6.1 版本号（AtomicStampedReference）
```java
AtomicStampedReference<V> ref = new AtomicStampedReference<>(val, 0);

boolean cas(V expectedRef, int expectedStamp,
            V newRef,     int newStamp)
```
CAS 时必须 **ref + stamp** 同时匹配。  
stamp 32 位，**溢出**后归零（业务允许）。

#### 6.2 延迟回收（GC 语言偷懒法）
Java 依靠 Safe GC：  
- 只要存在可达引用，对象**不会被回收**，自然不会有“旧地址重用”问题。  
- 但**逻辑 ABA**（值相同，语义不同）仍需版本号。

---

### 7 进阶：CPU 缓存伪共享 & 优化
Atomic* 变量多核竞争时，缓存行来回失效 → **伪共享**。  
LongAdder 通过 **Cell 数组** 把计数拆到不同缓存行，**降低冲突**，思想与 CAS 互补。

---

### 8 一张图总结
```
┌--------------┐
|  业务代码    |  compareAndSet(expect, update)
└------┬-------┘
       | JNI / intrinsic
┌------▼-------┐
|  HotSpot     |  Unsafe_CompareAndSwapInt
└------┬-------┘
       | JIT 生成
┌------▼-------┐
|  汇编指令     |  LOCK CMPXCHG
└------┬-------┘
       | 总线/缓存锁
┌------▼-------┐
|  CPU 硬件    |  MESI 协议保证原子
└--------------┘
```
- 失败 → 自旋重试（用户态循环）  
- ABA → 版本号或延迟回收  
- 高竞争 → LongAdder/分片

掌握这条链路，CAS 再无黑盒。



## 四、有哪些方法走CAS

只有**“修改”**操作（`compareAndSet`、`incrementAndGet`、`addAndGet` …）才会走 CAS；**“只读”**操作（`get`、`getAcquire`）就是普通的 volatile 读，**没有任何 CAS 指令**。

再细化一层：

#### 1、读  

```java
public final int get()          // 普通 volatile 读，无 CAS
public final int getAcquire()   // 比 volatile 再弱一点的顺序保证，也无 CAS
```

#### 2、写  

```java
public final void set(int newValue)        // 普通 volatile 写，无 CAS
public final void setRelease(int newValue) // 释放顺序的 volatile 写，无 CAS
public final void lazySet(int newValue)    // Ordered/Lazy 写，仍无 CAS
```

#### 3、读-改-写（RMW）  

下面这组才**真正触发 CAS 自旋**：

```java
compareAndSet(expect, update)
weakCompareAndSet(expect, update)
incrementAndGet() / decrementAndGet()
addAndGet(delta) / getAndAdd(delta)
getAndIncrement() / getAndDecrement()
updateAndGet(op) / getAndUpdate(op)
accumulateAndGet(x, f) / getAndAccumulate(x, f)
```

#### 4、批量  

`AtomicIntegerArray`、`AtomicReferenceArray` 里  

- `get(int i)` / `set(int i, …)` →  volatile 访问，**无 CAS**  
- `compareAndSet(i, …)` / `addAndGet(i, …)` → 定位到数组元素后再 CAS

#### 5、特殊  

`LongAdder` / `DoubleAdder` 的 `add()` 底层是 **“分片 CAS”**（base + Cell[]），**不是单次 CAS**；  
`sum()` 只是普通累加，**无 CAS**。

---

一句话总结  
> **只有“先比较、再更新”的复合动作才用 CAS；纯读、纯写都不碰 CAS，成本和普通 volatile 一样低。**



## 五、并发容器

JUC（`java.util.concurrent`）包里的**所有并发容器**（`ConcurrentHashMap`、`CopyOnWriteArrayList`、`ConcurrentLinkedQueue` 等）**默认就是线程安全的**，**无需额外加锁**即可在多线程环境下直接使用。

| 容器                         | 并发策略                                                     | 读写并发                       | 迭代器              | 时间复杂度                 | 内存开销               | 一句话场景                           | 绝对禁忌                    |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------ | ------------------- | -------------------------- | ---------------------- | ------------------------------------ | --------------------------- |
| **ConcurrentHashMap**        | 桶数组 + synchronized + CAS（JDK8）<br>Node 数组 + 红黑树    | 读完全无锁<br>写只锁头节点     | 弱一致（fail-safe） | 读 O(1)<br>写 O(1)（平均） | 正常                   | 高并发缓存、计数、索引               | 需要 **强一致** 快照        |
| **CopyOnWriteArrayList**     | 数组**快照复制**                                             | 读**无锁**<br>写**整数组复制** | 强一致（快照）      | 读 O(1)<br>写 O(n)         | **爆内存**（每次写×2） | **读远多于写**的黑白名单、监听器列表 | 高频写或大数组              |
| **CopyOnWriteArraySet** 同上 | 内部就是 **CopyOnWriteArrayList**（去重靠 `addIfAbsent`：**只有当元素不存在时才添加**） | 同上                           | 同上                | 同上                       | 同上                   | 同上                                 | 同上                        |
| **ConcurrentLinkedQueue**    | **无锁** Michael-Scott 链表（CAS 节点）                      | 读/写都用 CAS                  | 弱一致              | 入队 O(1)<br>出队 O(1)     | 正常                   | 高并发**任务队列**、消息传递         | 需要 **阻塞** 生产者/消费者 |

---

### 1 ConcurrentHashMap（JDK 8+）
- 结构：Node[] + 链表 + 红黑树；桶首节点 synchronized + CAS。  
- 并发度：桶级锁，16 条线程同时写 16 个不同桶**互不阻塞**。  
- 扩容：多线程协同迁移，**无 stop-the-world**。  
- 示例：秒杀商品库存缓存，**百万 QPS** 读，几千次写/秒。

### 2 CopyOnWriteArrayList
- 写：复制全新数组 → 替换引用 → 老数组瞬间失效。  
- 读：永远在读**当前数组快照**，**无锁、无 CAS**。  
- 迭代器：直接持有老数组引用，**遍历期间容器被改也安全**，但数据是“旧”的。  
- 禁忌：写频繁或大数组（一次复制几百兆直接 **FGC**）。

### 3 CopyOnWriteArraySet
- 内部 `add` 调用 `CopyOnWriteArrayList.addIfAbsent(e)`，遍历数组去重，**写 O(n)**。  
- 功能等价 `Collections.newSetFromList(new CopyOnWriteArrayList<>())`，**只是套壳**。

### 4 ConcurrentLinkedQueue
- 算法：Michael-Scott 无锁链表，**头、尾双指针 + CAS**。  
- 特点：  
  - `offer`/`poll`/`peek` 全无锁。  
  - `size()` 要**遍历整个队列**（O(n)），**慎用**；用 `isEmpty()` 代替。  
- 示例：Netty 的 I/O 任务队列、RPC 异步请求管道。

---

### 总结： 4 句话背下来
1. **高频读写** → `ConcurrentHashMap`  
2. **读多写极少**且**数组小** → `CopyOnWriteArrayList/Set`  
3. **无锁队列**传递任务 → `ConcurrentLinkedQueue`  
4. 需要**阻塞等待** → 换 `ArrayBlockingQueue` / `LinkedBlockingQueue`（不是上面四个）

## 六、addIfAbsent解析

#### 1、行为拆解

| 当前集合状态 | 调用 `addIfAbsent(e)` | 返回值 | 集合是否变化 |
| ------------ | --------------------- | ------ | ------------ |
| 不包含 e     | 把 e 加入             | true   | 是           |
| 已包含 e     | 什么都不做            | false  | 否           |

#### 2 实现源码（JDK 17）

```java
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();        // 拿到当前数组快照
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false : addIfAbsent(e, snapshot);       // 真正复制添加
}
```

- 先**线性扫描**数组判重（O(n)）。
- 若不存在，再复制新数组并替换引用——**整表复制**，所以写操作仍昂贵。

#### 3 与 `add()` 的区别

- `add()` 同样调 `addIfAbsent()`，**语义完全一样**；
  因此 `CopyOnWriteArraySet` 的 `add` 就是**去重安全版**，绝不会出现重复元素。

#### 4 一句话记忆

**Absent = “缺位才加”，已存在就无视。**

## 七、线程池三剑客 `ThreadPoolExecutor` / `ScheduledThreadPoolExecutor` / `Executors`

### 1 类图速览
```
java.util.concurrent  
├─ ThreadPoolExecutor                 ← 普通池  
├─ ScheduledThreadPoolExecutor       ← 定时池（继承 ThreadPoolExecutor）  
└─ Executors                          ← 静态工厂（只是封装构造参数，真正 new 的还是上面两个）
```

---

### 2 对比表（先给结论）
| 维度       | ThreadPoolExecutor                                    | ScheduledThreadPoolExecutor       | Executors 工厂方法                                  |
| ---------- | ----------------------------------------------------- | --------------------------------- | --------------------------------------------------- |
| 功能       | **一次性**任务                                        | **延迟/周期**任务                 | 快速拿到池子                                        |
| 底层队列   | LinkedBlockingQueue / SynchronousQueue 等             | 自定义 **DelayedWorkQueue**（堆） | 由方法决定                                          |
| 核心参数   | 7 大参数（见下）                                      | 复用父类 7 参数                   | 隐藏参数，**可能埋雷**                              |
| 扩展钩子   | `beforeExecute()` / `afterExecute()` / `terminated()` | 同上                              | 无                                                  |
| 拒绝策略   | 4 种（可自定义）                                      | 同上                              | 同上                                                |
| 默认线程名 | `pool-%d-thread-%d`                                   | 同上                              | 同上                                                |
| 使用建议   | **手动 new** 可控                                     | **手动 new** 可控                 | **生产环境别用**（阻塞队列无界 + 线程数无界 = OOM） |

---

### 3 ThreadPoolExecutor  7 大参数真身
```java
public ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数（懒创建，永不回收）
    int maximumPoolSize,     // 最大线程数（任务爆满是再启临时工）
    long keepAliveTime,      // 临时工闲置多久被辞退
    TimeUnit unit,           // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列（选错=OOM）
    ThreadFactory threadFactory,        // 线程命名、异常回调
    RejectedExecutionHandler handler)   // 拒绝策略（4 种）
```
图示流程  
```
提交任务 → 核心线程满？→ 队列满？→ 最大线程满？→ 拒绝策略
```

4 种拒绝策略（JDK 自带）  
1. AbortPolicy（默认）抛异常  
2. CallerRunsPolicy 退给调用线程跑  
3. DiscardPolicy 直接丢弃  
4. DiscardOldestPolicy 丢弃队列最老任务

---

### 4 ScheduledThreadPoolExecutor 专属能力
| 方法                              | 说明                                       |
| --------------------------------- | ------------------------------------------ |
| `schedule(Runnable, delay, unit)` | 一次性延迟执行                             |
| `scheduleAtFixedRate(...)`        | 固定**频率**（周期=period；任务开始→开始） |
| `scheduleWithFixedDelay(...)`     | 固定**间隔**（任务结束→开始）              |

内部队列：  
**DelayedWorkQueue** = 基于 **最小堆** 的优先级队列，按 **下次执行时间** 排序；  
因此定时精度 ≈ **毫秒级**，**不可能**做到硬实时。

---

### 5 Executors 工厂黑名单（生产勿用）
| 方法                                     | 背后实现                                        | 坑点                       |
| ---------------------------------------- | ----------------------------------------------- | -------------------------- |
| `Executors.newFixedThreadPool(n)`        | `LinkedBlockingQueue` 无界                      | 任务无限堆积 → OOM         |
| `Executors.newCachedThreadPool()`        | `SynchronousQueue` + 最大线程=Integer.MAX_VALUE | 线程无限创建 → 打爆 CPU/OS |
| `Executors.newSingleThreadExecutor()`    | 队列无界 + 单线程                               | 任务堆积照样 OOM           |
| `Executors.newScheduledThreadPool(core)` | 延迟队列无界                                    | 同上                       |

---

### 6 正确姿势（生产代码模板）
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        8,                                      // core
        16,                                     // max
        60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(2000),        // 有界队列，拒绝 OOM
        new ThreadFactoryBuilder()
                .setNameFormat("biz-pool-%d")   // Guava 小工具
                .setUncaughtExceptionHandler((t, e) -> log.error(e))
                .build(),
        new ThreadPoolExecutor.CallerRunsPolicy() // 限流回退
);

ScheduledThreadPoolExecutor sched = new ScheduledThreadPoolExecutor(
        4, new ThreadFactoryBuilder().setNameFormat("sched-%d").build()
);
```

---

### 7 一张图背下来
```
任务特征
├─ 一次性、突发流量  →  ThreadPoolExecutor（手动 7 参数）
├─ 延迟/周期执行     →  ScheduledThreadPoolExecutor
└─ 快速 demo         →  Executors（**仅限本地测试**）
```

记住：**工厂类是“教学版”，生产环境请 `new ThreadPoolExecutor` 自己填参数！**



## 八、同步工具 `CountDownLatch` / `CyclicBarrier` / `Semaphore` / `Phaser`

四个工具都是 **“线程之间约个暗号”** 的同步器，区别只在于 **暗号次数、能否循环、能否加人**。

一句话速记：

| 工具               | 暗号逻辑             | 能否重用               | 典型场景                   |
| ------------------ | -------------------- | ---------------------- | -------------------------- |
| **CountDownLatch** | 倒计时到 0 放人      | **一次性**             | 主线程等**所有**子任务完成 |
| **CyclicBarrier**  | 人满发车（计数归 0） | **可循环**             | 多线程**分阶段**计算       |
| **Semaphore**      | 拿到通行证才进       | **可循环**             | 限流、控并发数             |
| **Phaser**         | 人满 + 阶段号匹配    | **可循环、可动态加人** | 分阶段 + 中途加人          |

---

### 1 CountDownLatch（一次性倒计时）
```java
CountDownLatch latch = new CountDownLatch(3); // 主线程等 3 件事
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown();   // 做完 -1
    }).start();
}
latch.await();               // 到 0 才继续
System.out.println("全部干完");
```
- 倒计时器只能**用一次**；  
- 主线程**阻塞**在 `await()`，子线程只做“减法”。

---

### 2 CyclicBarrier（人满发车，可再来一轮）
```java
CyclicBarrier barrier = new CyclicBarrier(4, () -> System.out.println("阶段完成"));
for (int i = 0; i < 4; i++) {
    new Thread(() -> {
        compute();
        barrier.await();   // 人到齐才继续
        compute2();
        barrier.await();   // 第二轮
    }).start();
}
```
- 每个线程都**等待**；人到 0 触发**回调**，然后**自动重置**进入下一轮。  
- 经典用法：**MapReduce 分阶段计算**、**并行迭代算法**。

---

### 3 Semaphore（通行证，限流）
```java
Semaphore permit = new Semaphore(10); // 最多 10 并发
public void handle(Request req) throws InterruptedException {
    permit.acquire();   // 拿不到就阻塞
    try {
        doService(req);
    } finally {
        permit.release(); // 归还
    }
}
```
- 控制**同时进入临界区的线程数**；  
- 可用 `tryAcquire(timeout)` 做**弹性限流**。

```xml
permit.acquire();   // 拿不到就阻塞 这个地方是 拿什么？
```

拿的是 **“通行证”** —— 也就是 `Semaphore` 内部那个**计数器**。

- `new Semaphore(10)` 表示 **只有 10 张通行证**（计数器 = 10）。  
- `permit.acquire()` 把计数器 **-1**；  
  - 如果计数器 **> 0**，立刻拿到，线程继续；  
  - 如果计数器 **= 0**，线程就**阻塞排队**，直到别的线程执行 `release()` 把通行证**还回来**（计数器 +1）。

所以 **“拿不到就阻塞”** 指的是 **拿不到“一张空通行证”**，而不是拿某个具体对象。

---

### 4 Phaser（可动态加人的分阶段屏障）
```java
Phaser phaser = new Phaser(1); // 注册一个“主线程”
for (int i = 0; i < 3; i++) {
    phaser.register();           // 动态加人
    new Thread(() -> {
        phaseCompute(1);
        phaser.arriveAndAwaitAdvance(); // 等阶段 1 完成
        phaseCompute(2);
        phaser.arriveAndAwaitAdvance(); // 等阶段 2 完成
        phaser.arriveAndDeregister();   // 注销
    }).start();
}
phaser.arriveAndAwaitAdvance(); // 主线程也一起阶段推进
System.out.println("全部阶段结束");
```
- 支持**中途注册/注销**、**多阶段**（阶段号自动 +1）。  
- 适用：**ForkJoin 池任务**、**树形分治算法**。

---

### 5 一张图背下来
```
倒计时→CountDownLatch  
循环发车→CyclicBarrier  
通行证→Semaphore  
动态阶段→Phaser
```

记住暗号逻辑，四个工具再也不会混。



## 九、异步结果是什么？`Future` / `CompletableFuture` / `ExecutorService`

一句话：它们都是 **“把任务扔给别的线程，先拿张‘提货单’，稍后凭单取结果”** 的异步凭证，但**灵活度、组合能力、阻塞策略**逐级升级。

---

### 1 三代“提货单”对比表

| 维度             | `Future`                | `CompletableFuture`                             | `ExecutorService`                    |
| ---------------- | ----------------------- | ----------------------------------------------- | ------------------------------------ |
| **角色定位**     | **第一代**异步结果凭证  | **第二代**异步结果 + 回调 + 组合                | **线程池**（真正执行任务）           |
| **提交方式**     | `executor.submit(task)` | `CompletableFuture.supplyAsync(task, executor)` | `executor.submit(...)` 返回 `Future` |
| **取结果**       | **阻塞** `get()`        | **非阻塞** `thenApply/thenAccept` 链式回调      | 不负责取结果，只返回 `Future`        |
| **回调**         | ❌ 不支持                | ✅ 支持链式、组合、异常处理                      | ❌ 无                                 |
| **多个结果组合** | ❌ 自己写                | ✅ `thenCompose` / `allOf` / `anyOf`             | ❌ 无                                 |
| **异常处理**     | 自己 `try-catch`        | `exceptionally` / `handle`                      | ❌ 无                                 |

---

### 2 代码 30 秒看懂

#### ① 原生 Future（阻塞取货）
```java
ExecutorService pool = Executors.newFixedThreadPool(2);
Future<Integer> f = pool.submit(() -> {
    sleep(1); return 42;
});
// 干点别的
Integer ans = f.get(); // 阻塞直到 42 到手
```

#### ② CompletableFuture（链式非阻塞）
```java
CompletableFuture<Integer> f =
        CompletableFuture.supplyAsync(() -> 20, pool)
        .thenApply(x -> x * 2)      // 20->40
        .thenApply(x -> x + 2)      // 40->42
        .exceptionally(e -> -1);    // 异常返回 -1

f.thenAccept(System.out::println); // 42 到手后自动打印，**不阻塞当前线程**
```

#### ③ ExecutorService 本身
它只是**线程池**，用来**执行任务**；提交后返回 `Future`（或 `CompletableFuture`）给你去跟踪结果，它**不保存也不组合**结果。

---

### 3 一句话背下来
> **ExecutorService 负责“线程池”，Future 是“第一代提货单”（只能阻塞拿），CompletableFuture 是“第二代提货单”——支持链式回调、组合、异常处理，全程异步不阻塞。**

###  4、实际开发最佳实践

实际开发 = **“线程池 + CompletableFuture”** 组合打法；  
`Future` 只出现在老代码/旧库，`CompletableFuture` 是**事实标准**。

---

#### 1 生产级模板（复制即用）
```java
// ① 自定义线程池（拒绝默认Executors）
private static final ExecutorService POOL =
        new ThreadPoolExecutor(8, 16, 60, TimeUnit.SECONDS,
                               new ArrayBlockingQueue<>(2000),
                               new ThreadFactoryBuilder()
                                   .setNameFormat("biz-pool-%d").build(),
                               new ThreadPoolExecutor.CallerRunsPolicy());

// ② 并行调3个接口，全部完成再汇总
// supplyAsync 直译： “异步地供应（结果）”
// 即：把一段能产生结果的代码异步执行，最终把结果交回给 CompletableFuture
CompletableFuture<UserInfo> f1 = CompletableFuture
        .supplyAsync(() -> userService.get(uid), POOL);
CompletableFuture<OrderInfo> f2 = CompletableFuture
        .supplyAsync(() -> orderService.list(uid), POOL);
CompletableFuture<CouponInfo> f3 = CompletableFuture
        .supplyAsync(() -> couponService.valid(uid), POOL);

// ③ 等待全部完成 & 组装结果
Result result = CompletableFuture.allOf(f1, f2, f3)
        .thenApply(v -> Result.builder()
                     .user(f1.join())    // user(f1.join()) 等价于：把异步查询到的用户资料取出来，塞进 Result 的 user 字段。
                   						// 如果 f1 还没跑完，当前线程会阻塞等待直到结果就位
                     .orders(f2.join())
                     .coupons(f3.join())
                     .build())
        .get(); // 只在最后阻塞（或 thenAccept 异步回调）
```

##### - 可能的 **Result 模型**（Lombok 版）

```java
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class Result {
    // 用户基础信息
    private UserInfo user;
    // 订单列表
    private OrderInfo orders;
    // 可用优惠券
    private CouponInfo coupons;
}
```

其中三个子模型示例：

```java
@Data
public class UserInfo {
    private Long uid;
    private String name;
    private Integer age;
}

@Data
public class OrderInfo {
    private Integer total;
    private List<OrderItem> items;
}

@Data
public class CouponInfo {
    private Integer usableCount;
    private List<Coupon> list;
}
```

> 字段可随业务扩展，但 **Result 顶层永远只包含** `user`、`orders`、`coupons` **三个聚合对象**，与 `CompletableFuture.allOf(...)` 的 join 顺序一一对应。

---

#### 2 为什么不用 Future？
- `Future.get()` **必阻塞**，不能链式回调；  
- 不能组合多个结果 (`allOf`/`anyOf`)；  
- 异常只能 `try-catch` 手动处理。

---

#### 3 背下来
> **新代码 100% `CompletableFuture` + 自定义线程池；`Future` 只在老 SDK 里被动接收，绝不主动使用。**



## 十、CompletableFuture 、Future 是对任务的包装么？

不是“包装任务”，而是**“任务结果的提货单”**（凭证）。

| 维度                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| **任务本身**                 | 你提交的 `Runnable`/`Callable` 才是任务                      |
| **Future/CompletableFuture** | 只是**返回凭证**，让你**以后取结果**、**链回调**、**组合多个结果** |
| **类比**                     | 任务 = 工厂生产；Future = 快递单号；凭单号可查物流、收货、转发 |

所以它们**不包任务，只包“结果引用”**。
