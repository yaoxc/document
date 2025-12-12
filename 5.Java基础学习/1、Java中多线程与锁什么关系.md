## 1、Java中多线程与锁什么关系

可以概括为：**锁是用来解决多线程并发访问共享资源时可能引发的数据不一致问题的同步机制。**

#### 一句话总结：

> **多线程是并发执行的主体，锁是保障并发安全的工具。**

#### 详细解释：

##### ✅ 多线程带来的问题：

当多个线程**同时访问和修改同一个共享变量**时，如果没有同步机制，就会出现**竞态条件（race condition）**，导致数据不一致、逻辑错误，甚至程序崩溃。

##### ✅ 锁的作用：

锁（如 `synchronized`、`ReentrantLock` 等）用于**确保同一时刻只有一个线程能访问临界区（critical section）**，从而避免竞态条件，保证**线程安全**。

## 2、锁与juc是什么关系

锁是 JUC 的“地基”之一，JUC 把最原始的 synchronized 关键字抽象、扩展成了一整套**锁框架**（Lock 接口 + AQS + 并发容器 + 原子变量 + 线程池），让 Java 并发从“只能用的语法”变成“可以写的库”。

## 3、多线程和juc是什么关系

一句话：  
**多线程是 Java 并发要解决的问题“场景”，JUC 是 Java 官方给出的“解决方案包”。**

| 维度        | 多线程（Multithreading）                                     | JUC（java.util.concurrent）                                  |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 本质        | 操作系统/语言特性：允许一段程序同时跑多条执行路径            | 一个标准库包（+ 并发框架），专门用来**写多线程程序**的工具箱 |
| 关系        | **需求方** —— 业务代码说“我要并发”                           | **供给方** —— 把线程安全、高效率、易维护的并发工具一次性给齐 |
| 没有 JUC 时 | 只能自己写 `Thread` + `synchronized` + `wait/notify`，易错、低效 | ——                                                           |
| 有了 JUC 后 | 直接用 `Executor` 线程池、`ConcurrentHashMap`、`CountDownLatch`、`CompletableFuture`… 少写底层同步代码 | 提供**线程池、锁、原子变量、并发集合、同步器、异步框架**等   |
| 类比        | 要“做饭”这件事                                               | 提供“整套厨房电器+食谱”                                      |

## 4、juc就是Java提供用来解决多线程中对共享变量访问的问题么？

可以这么理解，但还不够完整。  
JUC 确实**核心目标**之一就是“安全且高效地解决多线程访问共享变量/资源的问题”，不过它给出的手段远不止“锁”或“同步”这么简单，而是**一整套并发编程的“工具链”**，覆盖了：

1. 原子性（atomic）  
2. 可见性 + 有序性（volatile + final + AQS 内部内存语义）  
3. 线程调度与任务拆分（线程池、Fork/Join、CompletableFuture）  
4. 高并发数据结构（并发 Map、Queue、Deque…）  
5. 多线程协作（CountDownLatch、CyclicBarrier、Semaphore、Phaser）  

因此，更准确的说法是：

> **JUC 是 Java 官方提供的“高并发工具箱”，用来系统性地解决“多线程环境下对共享状态/任务/资源的正确性、吞吐、可扩展性”三大问题，而不仅仅是“共享变量访问”。**

## 5.线程创建方式

Java 里“怎么把一条代码路径变成多条并发路径”只有两大类、四种写法，记住一句话：  
**万变不离“Thread”与“Runnable”——其他都是语法糖或线程池封装。**

| 创建方式                  | 关键代码骨架                                                 | 关键解释（必须理解）                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. 继承 Thread            | `class T extends Thread{ public void run(){ ... } }`         | 单继承局限；run 里写任务逻辑；`new T().start()` 才开新线程，**直接调 run 只是普通方法调用** |
| 2. 实现 Runnable          | `class R implements Runnable{ public void run(){ ... } }`    | 解耦“任务”与“线程”；可传给 Thread、线程池、Executor；**Java 8 以后直接用 λ 表达式** |
| 3. 实现 Callable + Future | `class C implements Callable<String>{ public String call()throws Exception{ ... } }` | 任务可抛受检异常、可带返回值；**必须借助线程池或 FutureTask 启动**，Thread 构造器不接受 Callable |
| 4. 线程池/Executor 框架   | `Executors.newFixedThreadPool(4).submit(runnable/callable)`  | 实际开发 99% 场景；**线程复用+队列+拒绝策略**，屏蔽手动 new Thread 的开销与风险 |

一句话背下来：  
> **“任务”用 Runnable/Callable 描述，“线程”用 Thread/线程池 驱动；别手动 new Thread，请交线程池。**

## 6、分别给出具有实际意义的应用代码样例

下面给出 4 种创建方式各自**“能直接搬到生产”**的代码片段，每个都解决一个真实小问题，拿来即可跑。

---

### ① 继承 Thread：后台打印小票
**场景**：餐饮 POS 机结账后，**立即**把“后厨小票”推给打印机，**不能阻塞结账界面**。

```java
/** 后厨小票线程 */
public class KitchenTicketThread extends Thread {
    private final String content;

    public KitchenTicketThread(String content) {
        this.content = content;
    }

    @Override
    public void run() {
        // 真实场景这里走串口/USB 打印
        System.out.println(LocalTime.now() + " 后厨打印 >>> " + content);
    }

    public static void main(String[] args) {
        // 结账按钮点击触发
        new KitchenTicketThread("2号桌 宫保鸡丁×1 米饭×2").start();
        System.out.println("结账界面立即返回，不等待打印完成");
    }
}
```

---

### ② 实现 Runnable：Tomcat 请求任务
**场景**：自己写个**迷你 Web 容器**接收请求，用线程池执行 `Runnable` 任务。

```java
public class TinyHttpServer {
    private static final ExecutorService pool = Executors.newCachedThreadPool();

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(8080);
        while (true) {
            Socket client = server.accept();
            // 每个请求一个 Runnable
            pool.execute(new RequestTask(client));
        }
    }

    static class RequestTask implements Runnable {
        private final Socket socket;
        RequestTask(Socket socket) { this.socket = socket; }
        @Override
        public void run() {
            try (BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                 PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {
                in.readLine(); // 简单读一行
                out.println("HTTP/1.1 200 OK\r\n\r\nHello, world!");
            } catch (IOException e) { e.printStackTrace(); }
        }
    }
}
```

---

### ③ Callable + Future：并行拉取**多源**价格
**场景**：电商商品页需要**同时**调用内部服务、第三方比价接口，**最长 2 s 返回**。

```java
public class PriceCollator {

    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(2);

        Callable<BigDecimal> innerTask = () -> {
            Thread.sleep(800);          // 模拟内部服务
            return new BigDecimal("99");
        };
        Callable<BigDecimal> outerTask = () -> {
            Thread.sleep(1200);         // 模拟第三方
            return new BigDecimal("95");
        };

        // 把任务 innerTask 扔进线程池的队列，立刻返回一个 Future 对象；线程池里的某个工作线程会在后台真正执行 innerTask.call()。
        Future<BigDecimal> f1 = pool.submit(innerTask);
        Future<BigDecimal> f2 = pool.submit(outerTask);
		
        // f1.get(2, TimeUnit.SECONDS) 的含义： 最多等 2 秒拿到结果，超时立即抛 TimeoutException，防止无限阻塞。
        BigDecimal lowest = f1.get(2, TimeUnit.SECONDS) .min(f2.get(2, TimeUnit.SECONDS));
        System.out.println("最低报价 = " + lowest);
        pool.shutdown();
    }
}
```

---

### ④ 线程池：异步写**埋点日志**
**场景**：下单接口**不能卡顿**，但要把埋点写到本地队列，再批量刷盘。

```java
public class LoggerPool {
    private static final ExecutorService ioPool =
            new ThreadPoolExecutor(1, 1,
                    0L, TimeUnit.MILLISECONDS,
                    new LinkedBlockingQueue<>(1000),
                    new ThreadPoolExecutor.DiscardPolicy()); // 队列满直接丢弃，避免拖垮业务

    public static void log(String msg) {
        ioPool.execute(() -> {
            // 真实写盘 / 写 Kafka
            System.out.println(LocalDateTime.now() + " 埋点 ==> " + msg);
        });
    }

    public static void main(String[] args) {
        // 业务线程完全不感知耗时
        log("orderId=12345,amount=199");
        System.out.println("下单接口立即返回");
    }
}
```

---

### 一张图总结
| 方式            | 生产场景                                         | 关键口诀                              |
| --------------- | ------------------------------------------------ | ------------------------------------- |
| 继承 Thread     | 必须**new 独立对象**且**立即启动**的简单后台任务 | “用完即走，不复用”                    |
| 实现 Runnable   | 任务与线程解耦，交给**线程池或框架**             | “任务只写 run，线程别人给”            |
| Callable+Future | 需要**返回值或抛异常**的并行调用                 | “异步 = 提交 Callable，聚合 = Future” |
| 线程池          | 高频、异步、批量任务；**绝对不要 new Thread**    | “池化 = 复用线程 + 队列 + 拒绝策略”   |



## 7、CopyOnWriteArrayList有什么特性？

解释

- **Copy** = 复制
- **On** = 在……时候
- **Write** = 写入

`CopyOnWriteArrayList` 的核心特性一句话总结：

> **“写时复制”——** 任何修改操作（add/set/remove）都先**复制整个底层数组**，在新数组上改，改完再**原子替换引用**；读取全程**无锁、快照一致性**。

---

### 1. 线程安全
- 不加锁，利用 `volatile` 数组 + **写时复制**保证并发安全。
- 读线程**永远不会阻塞写**，写线程之间通过 **ReentrantLock** 互斥。

### 2. 读性能极高
- `get()`、`iterator()` 直接访问数组，**无同步开销**，适合**读远多于写**的场景（黑白名单、事件监听器列表）。

### 3. 写性能差
- 每次修改都 **new 一个全新数组**，内存拷贝 + 双倍内存占用；数据量大或写频繁时 GC 压力大。

### 4. 快照一致性迭代器
- `iterator()` 返回的是**创建迭代器瞬间数组的快照**，遍历期间原数组如何增删都不会抛 `ConcurrentModificationException`，也**看不到**后续改动。

### 5. 内存可见性
- 底层数组引用被 `volatile` 修饰，**写改完替换引用**后，读线程立即见到新数组。

---

### 典型用法
```java
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();

// 发布事件：读多
for (Listener l : listeners) {          // 无锁、无 CME
    l.onEvent(event);
}

// 注册/注销：写少
listeners.add(newListener);
listeners.remove(oldListener);
```

---

### 口诀
> **读疯写少用 COW，写疯读少用同步/并发队列；迭代要快照，内存要翻倍。**



##  8、CopyOnWriteArrayList的读写过程用图解表达出来

下面用“时间线 + 内存图”方式，把 CopyOnWriteArrayList 一次“写-读”过程画成 3 张快照。  
（纯文本，复制到记事本也能看，真正跑的就是这个顺序）

```

初始时刻 T0
+-----------------------------+
| 数组引用 array（volatile） |----► [A, B, C]   ←---- 所有读线程直接读这里
+-----------------------------+        老数组
       ▲
       │ 读线程 R1、R2 正在遍历
       │ 读线程 R3 正在 get(1) → 返回 B

==========================================================================

写线程 W 开始 add("D")  ——  T1
+-----------------------------+
| 数组引用 array（volatile） |----► [A, B, C]   ←---- 读线程仍读老数组，无锁
+-----------------------------+
       │
       │ W 获得 ReentrantLock
       │ 复制老数组 → 新数组 [A, B, C, D]   （内存里出现两份）
       ▼
+-----------------------------+
| 数组引用 array（volatile） |----► [A, B, C]   （引用暂未变）
+-----------------------------+
       ▲
       │ 读线程 R1、R2、R3 继续读老数组，**看不到 D**，无阻塞

==========================================================================

写线程 W 提交修改  ——  T2
+-----------------------------+    原子写
| 数组引用 array（volatile） |-----► [A, B, C, D]  新数组立刻可见
+-----------------------------+    释放锁
       ▲
       │ 后续所有读线程直接读新数组，**能看到 D**
       │ 老数组 [A, B, C] 不再被引用，等待 GC

==========================================================================

结论速记
读：永远**当前数组**，无锁无复制。
写：加锁 → **复制** → 改副本 → 原子替换引用。
老数组成为**快照**，迭代器全程无 ConcurrentModificationException。
```

把 3 张图按 T0→T1→T2 顺序看，就是 CopyOnWriteArrayList 一次完整生命周期。

---

读线程**直接读当前的“原”数组**，没有任何复制动作；  
只有**写线程**在修改时才会**复制一份新数组**，改完后把引用切到新数组，此后读线程就**自然看到新数组**了。

流程再梳理一遍（CopyOnWriteArrayList 底层）：

1. 数组引用被声明为 `volatile`，所有线程都能看到最新引用。  
2. **读操作**（`get(int index)` / 迭代器）  
   - 拿到当前数组引用 → 直接按索引访问 → **无锁、无复制、无阻塞**。  
3. **写操作**（`add/remove/set`）  
   - 加 `ReentrantLock` → **复制整个数组** → 在新数组上修改 → 改完后把 `volatile` 数组引用指向新数组 → 释放锁。  
4. 后续读线程看到的就是**新数组**，老数组将被 GC 回收。

所以：  
> **读线程永远只读“当前”数组，从不复制；复制只发生在写线程内部。**

## 9、ReentrantLock是什么，有什么特性

`ReentrantLock` 是 Java 在 `java.util.concurrent.locks` 包里提供的**“可重入互斥锁”**——作用跟 `synchronized` 一样，但功能更强、使用更灵活。

### 一句话定义

> **“可重入” + “可中断” + “可公平” + “可多条件”** 的显式锁；必须**手动加锁、手动解锁**。
>
> > 四个特性翻译成“人话”就是：**让“排队拿锁”这件事变得更灵活、更公平、更不容易死机，同时还能精准叫醒不同的人**。

### 1、典型模板代码

```java
class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count;

    void increment() {
        lock.lock();          // ① 加锁
        try {
            count++;          // ② 临界区
        } finally {
            lock.unlock();    // ③ 一定释放
        }
    }
}
```

---

### 2、多条件示例（生产者-消费者）

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();
Condition notEmpty = lock.newCondition();

// 生产者
lock.lock();
try {
    while (queue.isFull()) notFull.await();  // 队列满就等
    queue.offer(x);
    notEmpty.signal();                       // 叫醒消费者
} finally { lock.unlock(); }
```

---

### 3、“可重入” + “可中断” + “可公平” + “可多条件” 的 具体案例

下面用**一个“排队打印店”故事**把 `ReentrantLock` 的四大特性串起来：

- 老板（线程）**可重入**——自己拿钥匙多次进屋
- 顾客**可中断**——等得不耐烦走人
- 排队机**可公平**——先来先服务
- 多个等候区**可多条件**——彩印/黑白/装订各自唤醒对应人群

####  1. 可重入 —— “老板拿钥匙”

```java
class PrintShop {
    private final ReentrantLock lock = new ReentrantLock();

    public void printInvoice() {
        lock.lock();          // 第一次加锁
        try {
            printDetail();    // 内部再调自己，依旧能拿到锁
        } finally { lock.unlock(); }
    }

    private void printDetail() {
        lock.lock();          // 同一线程再次 lock → 计数器+1
        try { System.out.println("明细页"); }
        finally { lock.unlock(); }
    }
}
```

**解释**：同一条线程可多次 `lock()`，**计数器减到 0 才真正释放**；避免自己锁死自己。

------

#### 2. 可中断 —— “顾客等烦了走人”

```java
public void cancelablePrint() throws InterruptedException {
    try {
        lock.lockInterruptibly();   // 可被打断的等待
        System.out.println("开始打印");
        Thread.sleep(5000);         // 模拟长时间打印
    } finally {
        if (lock.isHeldByCurrentThread()) lock.unlock();
    }
}

// 另一位线程
public void timeoutCancel() {
    Thread t = new Thread(() -> {
        try { cancelablePrint(); }
        catch (InterruptedException e) {
            System.out.println("不打了，走人！");
        }
    });
    t.start();
    Thread.sleep(2000);
    t.interrupt();      // 2 秒后打断，lockInterruptibly 会抛 IE
}
```

**解释**：`lockInterruptibly()` 让**等待锁的过程**能响应中断，避免永久阻塞。

------

#### 3. 可公平 —— “先来先服务”

```java
private final ReentrantLock fairLock = new ReentrantLock(true); // 公平模式

public void fairPrint(String name) {
    fairLock.lock();
    try {
        System.out.println(name + " 正在打印");
        Thread.sleep(1000);
    } catch (InterruptedException ignored) {
    } finally {
        fairLock.unlock();
    }
}

// 启动 5 个线程，按先后顺序排队
IntStream.rangeClosed(1, 5).forEach(i ->
        new Thread(() -> fairPrint("顾客" + i), "T" + i).start()
);
```

**输出**（始终固定顺序）：

```
顾客1 正在打印
顾客2 正在打印
顾客3 正在打印
顾客4 正在打印
顾客5 正在打印
```

**解释**：构造 `ReentrantLock(true)` 后，线程**按到达顺序**获得锁，防止“插队”饥饿。

------

#### 4. 可多条件 —— “彩印/黑白各自等候区”

```java
class PrintShop {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition colorQueue  = lock.newCondition();  // 彩印等候
    private final Condition bwQueue     = lock.newCondition();  // 黑白等候
    private boolean colorPrinterIdle = true;
    private boolean bwPrinterIdle    = true;

    public void colorPrint(String doc) throws InterruptedException {
        lock.lock();
        try {
            while (!colorPrinterIdle) colorQueue.await(); // 进彩印区等待
            colorPrinterIdle = false;
            System.out.println(doc + " 彩印中");
            Thread.sleep(1000);
            colorPrinterIdle = true;
            colorQueue.signal();     // 只唤醒彩印区下一位
        } finally { lock.unlock(); }
    }

    public void bwPrint(String doc) throws InterruptedException {
        lock.lock();
        try {
            while (!bwPrinterIdle) bwQueue.await();  // 进黑白区等待
            bwPrinterIdle = false;
            System.out.println(doc + " 黑白打印中");
            Thread.sleep(1000);
            bwPrinterIdle = true;
            bwQueue.signal();          // 只唤醒黑白区下一位
        } finally { lock.unlock(); }
    }
}
```

**解释**：一个锁创建多个 `Condition`，**精确唤醒不同业务队列**，避免“notifyAll”惊群。

------

#### 一张图背下来

| 特性   | 故事关键词          | 对应 API                   |
| :----- | :------------------ | :------------------------- |
| 可重入 | 老板多次拿钥匙      | `lock.lock()` 同线程可重入 |
| 可中断 | 顾客等烦走人        | `lockInterruptibly()`      |
| 可公平 | 先来先服务          | `new ReentrantLock(true)`  |
| 多条件 | 彩印/黑白各自等候区 | `lock.newCondition()`      |

## 10、“可重入” + “可中断” + “可公平” + “可多条件” 意义是什么

把四个特性翻译成“人话”就是一句话：  
**让“排队拿锁”这件事变得更灵活、更公平、更不容易死机，同时还能精准叫醒不同的人。**

拆成四条意义：

1. 可重入 —— **“自己锁自己”不会卡死**  
   递归、回调、锁嵌锁场景下，同一条线程可多次加锁，代码随便写，不会把自己憋死。

2. 可中断 —— **“等锁”也能随时撤票**  
   等待锁的过程中可以响应中断，线程不会被永久挂起，系统可控、可降级、可超时。

3. 可公平 —— **“插队”被禁止，防止饥饿**  
   先来先服务，写高并发服务器时，避免某些线程老被后来的抢走锁，长期得不到 CPU。

4. 可多条件 —— **“一锁多等候室”，精准唤醒**  
   一个锁带多个 Condition，生产者只叫醒消费者，不吵醒其他生产者，减少无效竞争，吞吐量直线上升。

合起来：  
**ReentrantLock = 安全 + 灵活 + 高性能 + 可运维**，  
让并发代码既能跑得快，又能在极端场景下“踩刹车”“让排队”“分窗口叫号”，而 `synchronized` 只能“一刀切”。

## 11、ExecutorService 线程池的好处

`ExecutorService`（线程池）一句话总结：  
**“不用白建新线程，任务来了有人接，高峰不会爆内存，空闲还能及时退。”**

具体 6 大好处：

1. 降低资源开销  
   复用已建好的线程，**避免频繁 new/销毁**线程对象（系统线程=1 MB 栈+内核对象）。

2. 提升响应速度  
   任务到达时**直接复用空闲线程**，省去创建时间，TP99 明显下降。

3. 提高系统稳定性  
   通过**队列+拒绝策略**把并发量“削峰填谷”，防止把 CPU、内存、连接数瞬间打满。

4. 统一管理与监控  
   可以**动态调参**（corePoolSize、maxPoolSize、keepAliveTime）并实时查看队列长度、活跃线程数，方便运维。

5. 功能更丰富  
   支持**定时/周期执行**（`ScheduledThreadPoolExecutor`）、**异步结果**（`submit`+`Future`）、**批量执行**（`invokeAll`/`invokeAny`）。

6. 代码更优雅  
   业务只写 `Runnable/Callable`，**线程生命周期完全托管**给框架，消灭手写 `new Thread(...).start()` 的野指针。

一句话记忆：  
**线程池 = 复用线程 + 削峰填谷 + 可观测 + 功能全家桶**，  
让并发代码写得又快又稳还能上线睡觉。

## 12、削峰填谷怎么做的？

线程池（ExecutorService）的“削峰填谷”不是靠人工调参，而是靠**三套自动机制**协同完成：

1. 队列当“水库”——先拦洪峰  
   任务突增时，线程池优先把任务放进**阻塞队列**（`LinkedBlockingQueue`、`ArrayBlockingQueue` 等），而不是立即新开线程，避免瞬间把 CPU/内存 打满。

2. 弹性线程数——按需开闸放水  
   当队列快满时，池子会**逐步扩容**到 `maximumPoolSize`，让更多工作线程并发处理；高峰期过后，**空闲线程超过 `keepAliveTime`** 就自动回收，把资源让回系统。

3. 拒绝策略——最后一道闸门  
   如果`队列`和`最大线程`都满了，线程池会触发**拒绝策略**（`AbortPolicy`、`CallerRunsPolicy`、`DiscardPolicy` 等），**丢掉或回退**部分任务，保证整个应用不崩溃。

形象流程：  
**洪峰来了 → 水库（队列）先蓄水 → 水坝按需加闸门（扩容线程） → 旱季关闸放水（回收线程） → 实在装不下就溢洪（拒绝策略）**。  
整个流程全自动，无需人工干预，就能把流量“尖峰”削平，资源“低谷”填满。

### 1、LinkedBlockingQueue、ArrayBlockingQueue 有什么特性

| 特性     | ArrayBlockingQueue                              | LinkedBlockingQueue                                  |
| -------- | ----------------------------------------------- | ---------------------------------------------------- |
| 类路径   | java.util.concurrent.ArrayBlockingQueue         | java.util.concurrent.LinkedBlockingQueue             |
| 底层结构 | **固定长度的数组**                              | **单向链表**（默认 Integer.MAX_VALUE）               |
| 容量     | **有界**（构造时必须指定）                      | **可选有界/无界**（空参=无界）                       |
| 锁算法   | **一把全局 ReentrantLock**<br>读写共用 → 竞争多 | **双锁（takeLock + putLock）**<br>读、写几乎互不阻塞 |
| 吞吐量   | 中                                              | **高**（并发读写分离）                               |
| 内存     | 提前一次性分配数组，**无节点对象**              | 每入队一个节点就 new Node，**GC 压力略大**           |
| 使用场景 | 需要**强制限定最大队列长度**，防止内存爆        | 任务峰值不确定，**希望大部分时候无界**；**吞吐优先** |

一句话记忆：  
> **Array 强制有界、单锁快但抢锁频；Linked 可无界、双锁高吞吐、内存略涨。**

### 2、**拒绝策略**`AbortPolicy`、`CallerRunsPolicy`、`DiscardPolicy`解析

JDK 线程池（`ThreadPoolExecutor`）在**队列满 + 最大线程也满**时会触发“拒绝策略”，系统内置 4 种实现，含义一句话就能记住：

| 策略类                    | 触发时行为                                     | 通俗解释                                               |
| ------------------------- | ---------------------------------------------- | ------------------------------------------------------ |
| `AbortPolicy`（**默认**） | 直接抛 `RejectedExecutionException`            | **抛异常，让调用方立刻知道任务被拒绝了**               |
| `CallerRunsPolicy`        | **由提交任务的线程自己执行**这个任务           | **拖慢调用者**，相当于“降级同步”，给线程池争取喘息时间 |
| `DiscardPolicy`           | 悄悄把任务**丢掉**，**不抛异常也不执行**       | **佛系丢弃**，适合非关键日志、埋点等                   |
| `DiscardOldestPolicy`     | 把**队列里最老的任务丢掉**，再尝试提交当前任务 | **牺牲老任务**，尽量保证新任务被处理                   |

---

#### 代码速览
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        4, 6,
        60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10),
        new ThreadPoolExecutor.CallerRunsPolicy()); // 换成任意策略
```

---

#### 一句话记忆
> **Abort 抛、Caller 拖、Discard 扔、DiscardOldest 牺牲老的保新的**。



## 13、ThreadPoolExecutor 参数详解

把这一行拆成 7 个参数，**按顺序**逐个说人话：

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        4,                  // ① corePoolSize
        6,                  // ② maximumPoolSize
        60L,                // ③ keepAliveTime
        TimeUnit.SECONDS,   // ④ unit
        new ArrayBlockingQueue<>(10), // ⑤ workQueue
        new ThreadPoolExecutor.CallerRunsPolicy()); // ⑥ rejectedExecutionHandler
```

| 序号 | 参数名            | 本例取值                   | 含义与实战注意点                                             |
| ---- | ----------------- | -------------------------- | ------------------------------------------------------------ |
| ①    | `corePoolSize`    | 4                          | **“常设窗口”**：线程池**始终保持** 4 条线程，即使它们空闲也不会被回收（除非设置 `allowCoreThreadTimeOut(true)`）。 |
| ②    | `maximumPoolSize` | 6                          | **“应急窗口”**：当队列满且任务继续涌入，池子会**再开 2 条临时线程**（总线程数 ≤ 6）。 |
| ③    | `keepAliveTime`   | 60L                        | 临时线程（> corePoolSize 的那部分）**空闲超过 60 秒**就被回收，释放资源。 |
| ④    | `unit`            | TimeUnit.SECONDS           | 配合③的时间单位，这里写秒；也可以用毫秒、分钟等。            |
| ⑤    | `workQueue`       | `ArrayBlockingQueue<>(10)` | **任务队列**：最多装 10 个任务。第 11 个任务来时，若当前线程数<6 会新建临时线程；否则触发拒绝策略。 |
| ⑥    | `handler`         | `CallerRunsPolicy()`       | **拒绝策略**：队列满 + 线程数已达 6，**由提交任务的 main 线程自己执行**，相当于“降级同步”，给池子缓冲时间。 |

---

### 运行流程图解（数字对应参数）
1. 任务数 ≤ 4 → 复用常设线程。  
2. 任务数 5~14 → 塞进队列（容量 10）。  
3. 任务数 15~16 → 队列满，**再开临时线程 5、6**（总线程=6）。  
4. 任务数 ≥ 17 → 队列满 + 线程满 → **触发 CallerRunsPolicy**，main 线程亲自跑任务，速度变慢但不会丢。  

记住口诀：  
> **常设 4，应急 6，队列 10，超时 60 秒，满了调用者自己跑！**



## 14、创建线程池有哪些方式？

在 Java 里“弄一个线程池”只有两大类 7 条路子，记住 **“5 个工厂 + 1 个裸 new + 1 个 Spring”** 即可：

---

### 一、JDK 工厂方法（5 条）——快速使用
1. `Executors.newFixedThreadPool(n)`  
   固定并发数，无界队列。

2. `Executors.newCachedThreadPool()`  
   0 核心、可无限扩容，60 s 回收，适合短任务。

3. `Executors.newSingleThreadExecutor()`  
   单后台线程，顺序执行。

4. `Executors.newScheduledThreadPool(n)`  
   定时 & 周期执行。

5. `Executors.newWorkStealingPool()`（JDK 8）  
   工作窃取，默认 CPU 核数。

---

### 二、裸 new——定制所有参数
6. `new ThreadPoolExecutor(core, max, keepAlive, unit, queue, handler)`  
   自己指定队列长度、拒绝策略、线程名、回退方案等，**生产环境推荐**。

---

### 三、Spring 托管（可选）
7. `ThreadPoolTaskExecutor`（Spring 容器）  
   通过 `@Bean` 配置，支持 `@Async`、优雅关闭、队列/拒绝策略全定制，**Web 项目标配**。

---

### 四、总结（用法建议）
> **demo 用工厂，上线用裸 new `ThreadPoolExecutor`，Spring 项目用 `ThreadPoolTaskExecutor`**——就这 3 句。



## 15、7种创建线程池方式配置参数详细说明

（底层都是 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor`）。记住这张表就够了：

| 工厂类 / 构造器                              | 创建的线程池类型              | 简要说明                                                |
| -------------------------------------------- | ----------------------------- | ------------------------------------------------------- |
| `Executors.newFixedThreadPool(int n)`        | `ThreadPoolExecutor`          | 固定大小，无界 `LinkedBlockingQueue`                    |
| `Executors.newCachedThreadPool()`            | `ThreadPoolExecutor`          | 0 核心，最大 `Integer.MAX_VALUE`，60 s 回收；适合短任务 |
| `Executors.newSingleThreadExecutor()`        | `ThreadPoolExecutor`          | 单后台线程，顺序执行任务                                |
| `Executors.newScheduledThreadPool(int core)` | `ScheduledThreadPoolExecutor` | 支持定时 & 周期性执行                                   |
| `Executors.newWorkStealingPool()`（JDK 8+）  | `ForkJoinPool`                | 工作窃取，默认 `Runtime.availableProcessors()` 个线程   |

### 1. newFixedThreadPool(int n)

**固定大小，无界队列**

| 参数            | 值                                        |
| --------------- | ----------------------------------------- |
| corePoolSize    | n                                         |
| maximumPoolSize | n                                         |
| keepAliveTime   | 0L                                        |
| workQueue       | `LinkedBlockingQueue<Runnable>()`（无界） |
| handler         | `AbortPolicy`                             |

```java
// 创建最多同时 4 条工作线程的线程池，任务队列无界
ExecutorService fix = Executors.newFixedThreadPool(4);
// 一口气提交 10 个Lambda 任务（编号 0-9）。
// 池子先把任务塞进队列，4 条线程抢任务执行，因此输出顺序乱序，但同一时间最多 4 个任务在跑。
IntStream.range(0, 10).forEach(i -> fix.submit(() ->
        System.out.println(Thread.currentThread().getName() + " 执行任务" + i)));
// 优雅关闭：不再接受新任务，等队列里现有任务全部完成后池子才真正退出
fix.shutdown();
```

运行效果（示例，每次顺序不同）：

```java
pool-1-thread-2 执行任务1
pool-1-thread-4 执行任务3
pool-1-thread-1 执行任务0
pool-1-thread-3 执行任务2
...
```

**结论**：用 4 条线程并发跑完 10 个任务，主代码**不手动 new Thread**，全部交给线程池调度。

---

### 2. newCachedThreadPool()
**无限扩容 + 60 s 回收**

| 参数            | 值 (默认值)                   |
| --------------- | ----------------------------- |
| corePoolSize    | 0                             |
| maximumPoolSize | `Integer.MAX_VALUE`           |
| keepAliveTime   | 60L                           |
| unit            | `SECONDS`                     |
| workQueue       | `SynchronousQueue`（容量为0） |
| handler         | `AbortPolicy`                 |

```java
// 效果 = “来一个任务就建一条线程，高峰期瞬间膨胀，空闲 60 秒后全部回收

ExecutorService cache = Executors.newCachedThreadPool();
// 1.循环 0-99，每次 submit 相当于把任务塞过去
// 2.因为 SynchronousQueue 容量为 0，如果当前没有空闲线程，池子会立即 new 一条新线程来接手
// 3.100 个任务几乎同时到达，于是池子会在瞬间开到 100 条线程（名字依次为 pool-1-thread-1 … pool-1-thread-100）
// 4.每条线程只睡 100 ms 就结束，任务完成线程回到池里；60 秒内没有新任务进来，这 100 条线程会被全部回收，整个池子恢复成 0 线程
IntStream.range(0, 100).forEach(i -> cache.submit(() -> {
    try { Thread.sleep(100); } catch (InterruptedException ignored) {}
    System.out.println("任务" + i);
}));
// 不再接受新任务，等待现有任务执行完后完全释放线程
cache.shutdown();
```

---

### 3. newSingleThreadExecutor()
**单后台线程，顺序执行**

| 参数            | 值                            |
| --------------- | ----------------------------- |
| corePoolSize    | 1                             |
| maximumPoolSize | 1                             |
| keepAliveTime   | 0L                            |
| workQueue       | `LinkedBlockingQueue`（无界） |
| handler         | `AbortPolicy`                 |

```java
ExecutorService single = Executors.newSingleThreadExecutor();
single.submit(() -> System.out.println("顺序 1"));
single.submit(() -> System.out.println("顺序 2"));
single.submit(() -> System.out.println("顺序 3"));
single.shutdown();
```

---

### 4. newScheduledThreadPool(int core)
**定时 & 周期执行**

| 参数            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| corePoolSize    | core                                                         |
| maximumPoolSize | `Integer.MAX_VALUE`                                          |
| keepAliveTime   | 0L                                                           |
| unit            | `NANOSECONDS`  //  是 `TimeUnit` 枚举里的一个常量，表示“纳秒”，意思**空闲线程立即回收** |
| workQueue       | `DelayedWorkQueue`（按时间排序）// `ScheduledThreadPoolExecutor` 的专用延迟阻塞队列，队列容量自动扩容 |
| handler         | `AbortPolicy`                                                |

```java
ScheduledExecutorService sched = Executors.newScheduledThreadPool(2);
// 延迟 1 秒执行
sched.schedule(() -> System.out.println("一次性延迟任务"), 1, TimeUnit.SECONDS);
// 初始 0 秒，之后每 500 ms 执行一次
sched.scheduleAtFixedRate(() -> System.out.println("周期任务"), 0, 500, TimeUnit.MILLISECONDS);
Thread.sleep(3000);
sched.shutdown();
```

---

### 5. newWorkStealingPool()（JDK 8）
**工作窃取，默认 CPU 核数**

| 参数        | 值                                               |
| ----------- | ------------------------------------------------ |
| 实现类      | `ForkJoinPool`                                   |
| parallelism | `Runtime.getRuntime().availableProcessors()`     |
| 队列        | `ForkJoinTask` 内部 dequeue 数组（工作窃取队列） |
| 线程名      | `ForkJoinPool-worker-*`                          |

```java
ExecutorService steal = Executors.newWorkStealingPool(); // 默认并行=CPU核
// 大任务拆分
class SumTask extends RecursiveTask<Long> {
    private final long[] arr; private final int start, end;
    SumTask(long[] arr, int start, int end) { this.arr = arr; this.start = start; this.end = end; }
    @Override
    protected Long compute() {
        if (end - start <= 1000) { // 阈值
            return Arrays.stream(arr, start, end).sum();
        }
        int mid = (start + end) >>> 1;
        SumTask left  = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();
        return right.compute() + left.join();
    }
}
long[] data = new long[10_000_000];
Arrays.setAll(data, i -> i + 1);
long result = ((ForkJoinPool) steal).invoke(new SumTask(data, 0, data.length));
System.out.println("sum=" + result);
steal.shutdown();
```

---

### 6. 裸 new ThreadPoolExecutor（生产推荐）
**全部参数自己控**

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        4,                              // core
        6,                              // max
        60L, TimeUnit.SECONDS,          // 空闲回收时间
        new ArrayBlockingQueue<>(1000), // 有界队列，防内存爆
        new ThreadFactory() {           // 自定义线程名
            private final AtomicInteger seq = new AtomicInteger(1);
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("biz-pool-" + seq.getAndIncrement());
                return t;
            }
        },
        new ThreadPoolExecutor.CallerRunsPolicy()); // 拒绝策略：调用者执行

// 使用
IntStream.range(0, 2000).forEach(i -> pool.submit(() -> {
    try { Thread.sleep(100); } catch (InterruptedException ignored) {}
    System.out.println(Thread.currentThread().getName() + " 处理 " + i);
}));
pool.shutdown();
```

---

### 7. Spring ThreadPoolTaskExecutor（Web 场景）
**容器托管、优雅关闭、配置外部化**

```java
@Configuration
@EnableAsync
public class PoolConfig {
    @Bean("asyncPool")
    public ThreadPoolTaskExecutor executor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(8);
        exec.setQueueCapacity(200);
        exec.setKeepAliveSeconds(60);
        exec.setThreadNamePrefix("spring-async-");
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setWaitForTasksToCompleteOnShutdown(true); // 优雅
        exec.setAwaitTerminationSeconds(30);
        exec.initialize();
        return exec;
    }
}

// 业务层直接注入
@Service
public class AsyncService {
    @Async("asyncPool")   // 指定上面定义的池
    public CompletableFuture<String> sendEmail() {
        try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
        return CompletableFuture.completedFuture("email sent");
    }
}
```

---

### 一张表速记
| 创建方式                       | 默认队列                   | 默认线程数 | 适用场景                |
| ------------------------------ | -------------------------- | ---------- | ----------------------- |
| `newFixedThreadPool(n)`        | 无界 `LinkedBlockingQueue` | 固定 n     | 任务量平稳、CPU -bound  |
| `newCachedThreadPool()`        | `SynchronousQueue`         | 0~∞        | 短、异步任务，I/O 密集  |
| `newSingleThreadExecutor()`    | 无界 `LinkedBlockingQueue` | 1          | 顺序执行、单后台线程    |
| `newScheduledThreadPool(core)` | `DelayedWorkQueue`         | core~∞     | 定时/周期任务           |
| `newWorkStealingPool()`        | 内部 steal 队列            | CPU 核数   | 大任务拆分、递归        |
| `new ThreadPoolExecutor(...)`  | 自定义                     | 自定义     | 生产环境，全部可控      |
| `ThreadPoolTaskExecutor`       | 自定义                     | 自定义     | Spring 容器，可外部配置 |



## 16、 线程池的shutdown()方法

#### 1、调用shutdown();方法，后续还可以向线程池提交任务么？

不可以。

`cache.shutdown()` 会先把线程池切换到 **SHUTDOWN** 状态：

- **已提交的任务仍会被执行**（包括队列里等待的）  
- **但再调用 `execute/submit` 会立刻抛 `RejectedExecutionException`**，提示线程池已关闭。

如果想**等待正在跑的任务结束**，再追加新任务，只能用 `awaitTermination` 阻塞，或者**先判断 `isShutdown()`/`isTerminated()`** 再决定是否新建一个池。

一句话：  
> **shutdown ≠ 暂停，而是“打烊”——进门就拒客，里面吃完再走。**



#### 2、那线程池就永远无法接受新任务了么？

对当前这个池子来说，**“永远”不再接受新任务**。

`shutdown()` 会把状态永久设为 **SHUTDOWN**（或 `shutdownNow()` 设为 **STOP**），内部逻辑就是：

> 一旦进入这两种终止态，**任何 `execute`/`submit` 都直接走拒绝策略**，不可能再回到 **RUNNING**。

所以：

- > 旧池：生命周期结束，**别再往里丢任务**；  
  >
  > 新任务：只能**新建一个线程池**，或者提前把旧池注入/托管到外部，由框架重新创建。

一句话：  
> **线程池是“一次性”的，shutdown 后就成“僵尸池”，再提交任务只会吃 `RejectedExecutionException`。**

（除非你自己重写一个能回滚状态的实现，否则 JDK 官方实现**没有“重启”**能力。）



#### 3、那这个线程池就相当于没有用了么？

**不是“完全没用”**，而是“**不能再接新任务**”，但**已提交的任务仍会继续跑完**，最后池子自己会被 JVM 回收。

1. 已入队的任务  
   `shutdown()` 会**保证把队列里剩余的任务全部执行完毕**。

2. 正在跑的任务  
   线程池会等它们**正常结束**（`run()` 方法返回），不会强制中断。

3. 资源回收  
   当队列和线程都空后，池子状态变成 **TIDYING → TERMINATED**，内部线程死亡，对象变成普通垃圾，**等待 GC 回收**。

4. 再提交？  
   **只能新建一个池**，旧池无法“复活”。

一句话：  
> **shutdown 后的线程池 = “营业结束，打扫干净再关门”**，**旧任务继续服务，新任务请去别家**。



#### 4、实际工程项目中，会调用线程池的shutdown方法么

会，而且**必须调**，但**不是业务代码里随手调**，而是交给**“容器生命周期”**统一管：

1. Spring / Spring Boot  
   官方 `ThreadPoolTaskExecutor` 已实现 `DisposableBean`，**应用关闭时自动 `shutdown()`**；  
   你自己定义裸 `ThreadPoolExecutor` 也要用 `@PreDestroy` 或 `DisposableBean` 显式调，否则 JVM 会被**残留线程**卡住，**优雅停机失败**。

2. Web 容器（Tomcat、Undertow）  
   注册 `ServletContextListener` 的 `contextDestroyed()` 里关闭线程池，**防止热部署内存泄露**。

3. 命令行 / 定时任务一次性程序  
   `main` 方法最后 `pool.shutdown();` + `awaitTermination()`，**等任务全部落地再退出**；否则主线程一跑完就强杀后台线程，日志还没写盘。

4. 微服务滚动发布  
   Kubernetes 先发 `SIGTERM`，**shutdown → awaitTermination → 退出**，**保证正在处理的请求不被中断**。

结论：  
> **生产代码一定要调 shutdown**，但**写在“关闭钩子”里**，业务层日常**只负责 submit，不负责关门**；**忘记调 = 优雅停机失败 + 线程泄露 + 热部署内存暴涨**。



## 17、线程池是单进程下的技术么

线程池是**单进程内**的并发技术。  
它只负责把**同一 JVM 进程**里的任务调度到若干**用户态线程**上执行，既跨不了进程，也跨不了机器。  
想实现**多进程**或**多机**并发，必须借助**消息队列**、**RPC**、**分布式任务框架**等进程间通信手段。

















