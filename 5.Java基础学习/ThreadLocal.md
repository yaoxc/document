## 一、什么是ThreadLocal

> ThreadLocal（线程本地变量）是 Java `java.lang` 包下的一个核心工具类，其核心作用是**为每个线程创建独立的变量副本**，让每个线程只能访问和修改自己的副本，完全隔离不同线程之间的变量访问，从而从根本上避免多线程共享变量带来的线程安全问题。

简单来说，ThreadLocal 就像给每个线程分配了一个 “专属储物柜”：线程只能操作自己的储物柜里的东西，不会和其他线程的储物柜混淆，无需依赖`synchronized`等锁机制（本质是**空间换时间**的设计思想）。



### 1、ThreadLocal 的核心原理（内存结构与存储逻辑）

ThreadLocal 本身**不存储任何数据**，它只是一个 “工具类”，真正的存储容器是**线程（Thread）对象内部的 `ThreadLocalMap`**（ThreadLocal 的静态内部类）。其核心存储逻辑如下：

> ThreadLocalMap是什么： ThreadLocal 的静态内部类。

1. **每个 Thread 对象持有一个 ThreadLocalMap 成员变量**（默认值为 `null`，首次调用 `set()`/`get()` 时初始化）；
2. **ThreadLocalMap 是一个自定义哈希表**，键（Key）是 ThreadLocal 实例的**弱引用**，值（Value）是线程的变量副本（强引用）；
3. 当调用 `threadLocal.set(value)` 时，本质是：以当前 ThreadLocal 实例为 Key，以 value 为 Value，将键值对存入**当前线程的 ThreadLocalMap**；
4. 当调用 `threadLocal.get()` 时，本质是：以当前 ThreadLocal 实例为 Key，从**当前线程的 ThreadLocalMap** 中取出对应的 Value。

### 2、可视化内存结构

```xml
┌─────────────────────────────────────────┐
│ Thread 线程对象（如 Thread-0、main 线程）		   │
│  ├── ThreadLocalMap threadLocals           │
│  │    ├── Entry[] table（哈希表数组）         │
│  │    │    ├── Entry 1:                     │
│  │    │    │   Key = ThreadLocal实例（弱引用）│
│  │    │    │   Value = 线程1的变量副本        │
│  │    │    ├── Entry 2:                     │
│  │    │    │   Key = 另一个ThreadLocal实例    │
│  │    │    │   Value = 线程1的另一个副本      │
│  │    │    └── ...                          │
│  └── 其他线程成员变量                         │
└─────────────────────────────────────────┘
```

**关键结论**：同一个 ThreadLocal 实例，在不同线程的 ThreadLocalMap 中对应不同的 Value 副本，实现线程隔离。

### 3、代码示例(同一个ThreadLocal实例，在不同线程绑定)

```java
public class ThreadLocalDemo {
    // 1. 创建ThreadLocal实例，声明为static final是规范写法（减少对象创建）
    private static final ThreadLocal<String> USER_THREAD_LOCAL = new ThreadLocal<String>() {
        // 重写initialValue，设置默认初始值
        @Override
        protected String initialValue() {
            return "默认用户";
        }
    };

    public static void main(String[] args) {
        // 线程1：设置并获取自己的副本
        new Thread(() -> {
          // set方法中会去主动获取Thread对象  
            USER_THREAD_LOCAL.set("用户A");
            System.out.println(Thread.currentThread().getName() + ": " + USER_THREAD_LOCAL.get()); // 输出：Thread-0: 用户A
            USER_THREAD_LOCAL.remove(); // 用完必须清理
        }, "Thread-0").start();

        // 线程2：获取默认值并修改
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + ": " + USER_THREAD_LOCAL.get()); // 输出：Thread-1: 默认用户
          // set方法中会去主动获取Thread对象  
          USER_THREAD_LOCAL.set("用户B");
            System.out.println(Thread.currentThread().getName() + ": " + USER_THREAD_LOCAL.get()); // 输出：Thread-1: 用户B
            USER_THREAD_LOCAL.remove(); // 用完必须清理
        }, "Thread-1").start();
    }
}
```



## 二、ThreadLocal实例 是如何与当前线程绑定的

### 1、基本前提

- `private static final ThreadLocal<String> userNameTl = new ThreadLocal<>();`时，**仅仅是创建了一个 ThreadLocal 对象实例，此时它还没有和任何线程绑定**。
- 真正的 “绑定” 发生在你调用`userNameTl.set(...)`或`userNameTl.get()`方法时，核心是通过**Thread 对象内部的 ThreadLocalMap**完成线程与值的关联，具体过程分为「初始化绑定容器」和「存储 / 读取值并绑定」两个阶段。
- ThreadLocal 本身不存储任何数据，它只是一个 **“工具类”**，真正的存储容器是**当前线程（Thread）的`threadLocals`成员变量（类型为 ThreadLocalMap）**



### 2、绑定的本质

#### 1、存入：

	- 以 ThreadLocal 实例为**Key**
	- 以要存储的 Value 为**值**
	- 将键值对存入**当前线程的 ThreadLocalMap**中

#### 2、读取

- 以 ThreadLocal 实例为 Key
- 当前线程的 ThreadLocalMap 中取出对应 Value

#### 3、绑定示例(set())

我们结合 JDK 源码，拆解`userNameTl.set("张三")`时的绑定逻辑：

##### 步骤 1：获取当前线程（Thread.currentThread ()）

这是绑定的**核心前提**，因为所有数据都存在当前线程的容器中，源码片段：

```java
public void set(T value) {
    // 1. 获取当前执行的线程对象（这就是要绑定的线程）
    Thread t = Thread.currentThread();
    // 2. 获取当前线程的ThreadLocalMap容器
    ThreadLocalMap map = getMap(t);
    // 3. 容器存在则存值，不存在则创建容器并存值
    if (map != null) {
        map.set(this, value); // this就是当前ThreadLocal实例（userNameTl）
    } else {
        createMap(t, value); // 初始化ThreadLocalMap并绑定到线程
    }
}
```

##### 步骤 2：获取 / 创建线程的 ThreadLocalMap 容器

每个 Thread 对象都有两个 ThreadLocalMap 类型的成员变量（`threadLocals`和`inheritableThreadLocals`），默认值为`null`，源码在 Thread 类中：

```java
public class Thread implements Runnable {
    // 存储普通ThreadLocal的键值对，默认null
    ThreadLocal.ThreadLocalMap threadLocals = null;
    // 存储可继承的ThreadLocal的键值对，默认null
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    // ... 其他成员变量
}
```

- 当第一次调用`set()`/`get()`时，线程的`threadLocals`为`null`，此时会执行`createMap(t, value)`创建容器并绑定到线程：

  ```java
  // ThreadLocal的createMap方法
  void createMap(Thread t, T firstValue) {
      // 初始化线程的threadLocals成员变量，将当前ThreadLocal和value作为第一个键值对存入
      t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  ```

  

- 后续调用`set()`时，直接使用已存在的`threadLocals`容器。

##### 步骤 3：将 ThreadLocal 实例和 Value 存入 ThreadLocalMap

ThreadLocalMap 是 ThreadLocal 的静态内部类，本质是一个自定义的哈希表，其`set(ThreadLocal<?> key, Object value)`方法会：

1. 以 ThreadLocal 实例的`threadLocalHashCode`（哈希值，通过黄金分割数生成，保证均匀分布）为依据，计算出键值对在哈希表数组中的下标；
2. 用**线性探测法**处理哈希冲突（如果下标已被占用，依次往后找空位置）；
3. 将键值对封装为`Entry`（Key 是 ThreadLocal 的弱引用，Value 是强引用），存入数组对应位置。

此时，**当前线程的 ThreadLocalMap 中就有了以 userNameTl 为 Key、以 "张三" 为 Value 的键值对**，也就完成了 ThreadLocal 与当前线程的 “绑定”。

#### 4、绑定示例(get())

当调用`userNameTl.get()`时，本质是反向从当前线程的容器中读取，过程如下：

```java
public T get() {
    // 1. 还是先获取当前线程
    Thread t = Thread.currentThread();
    // 2. 获取线程的ThreadLocalMap容器
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 3. 以当前ThreadLocal实例为Key，取出Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            // 4. 取出Value并返回（完成读取绑定）
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 5. 容器为空或无对应Key时，初始化值（调用initialValue()）并绑定
    return setInitialValue();
}
```

其中`setInitialValue()`方法的逻辑和`set()`类似：如果线程没有 ThreadLocalMap，就创建并初始化；如果有，就存入`initialValue()`返回的默认值（默认是`null`，可重写该方法自定义默认值）。



## 三、线程绑定多个Value（ThreadLocal）

ThreadLocal **本身单个实例只能绑定一个线程本地的 Value**，但可以通过

- **创建多个 ThreadLocal 实例**、
- **封装复合对象**、
- **使用 ThreadLocalMap 直接操作（不推荐）** 三种方式，让单个线程存储多个 Value。

### 1、单个 ThreadLocal 实例：仅能存一个 Value（线程维度）

对于**一个 ThreadLocal 对象**（比如 `ThreadLocal<String> tl = new ThreadLocal<>()`），`每个线程`只能通过它存储**一个对应的 Value 副本**。

这是因为：

- ThreadLocal 的 `set(T value)` 方法会**覆盖当前线程中该 ThreadLocal 对应的旧 Value**，`get()` 方法也只能获取到最新设置的那个 Value。

> `Thread对象`的`ThreadLocalMap属性`，是以 `ThreadLocal 对象` 作为Key的，map.set(key,  value);  会覆盖旧值。

代码示例

```java
public class ThreadLocalSingleValue {
    // 单个 ThreadLocal 实例
    private static final ThreadLocal<String> tl = new ThreadLocal<>();

    public static void main(String[] args) {
        new Thread(() -> {
            // 第一次设置 Value
            tl.set("第一个值");
            System.out.println(Thread.currentThread().getName() + ": " + tl.get()); // 输出：第一个值

            // 第二次设置 Value，覆盖旧值
            tl.set("第二个值");
            System.out.println(Thread.currentThread().getName() + ": " + tl.get()); // 输出：第二个值

            tl.remove(); // 用完清理
        }, "线程1").start();
    }
}
```

### 2、存储多个 Value 的三种常用方式

#### 1. 方式一：创建多个 ThreadLocal 实例（推荐，最简洁）

这是**最常用、最符合 ThreadLocal 设计意图**的方式：为每个需要存储的变量创建独立的 ThreadLocal 实例。每个 ThreadLocal 实例对应 ThreadLocalMap 中的一个 Entry，彼此互不干扰。

##### 示例：多个 ThreadLocal 存储不同 Value

```java
public class ThreadLocalMultiInstance {
    // 第一个 ThreadLocal：存储用户名
    private static final ThreadLocal<String> userNameTl = new ThreadLocal<>();
    // 第二个 ThreadLocal：存储用户ID
    private static final ThreadLocal<Long> userIdTl = new ThreadLocal<>();
    // 第三个 ThreadLocal：存储用户角色
    private static final ThreadLocal<List<String>> userRoleTl = new ThreadLocal<>();

    public static void main(String[] args) {
        new Thread(() -> {
            // 为不同 ThreadLocal 设置不同 Value
            userNameTl.set("张三");
            userIdTl.set(1001L);
            userRoleTl.set(Arrays.asList("管理员", "普通用户"));

            // 分别获取 Value
            System.out.println("用户名：" + userNameTl.get()); // 张三
            System.out.println("用户ID：" + userIdTl.get()); // 1001
            System.out.println("用户角色：" + userRoleTl.get()); // [管理员, 普通用户]

            // 用完统一清理
            userNameTl.remove();
            userIdTl.remove();
            userRoleTl.remove();
        }, "线程2").start();
    }
}
```

**内存结构对应**：Thread 的 ThreadLocalMap 中有 3 个 Entry，Key 分别是三个 ThreadLocal 实例，Value 是对应的数据。

#### 2. 方式二：封装复合对象（推荐，适合相关联的多个变量）

如果需要存储的多个 Value 是**相关联的业务数据**（比如用户的姓名、ID、角色属于一组用户信息），可以将这些数据封装到一个自定义对象中，再通过**一个 ThreadLocal 实例存储这个复合对象**。这种方式的优点是便于统一管理（比如初始化、清理），减少 ThreadLocal 实例的数量。

##### 示例：封装对象存储多个 Value

```java
// 复合对象：用户信息
class UserContext {
    private String userName;
    private Long userId;
    private List<String> roles;

    // 构造器、getter/setter 省略
    public UserContext(String userName, Long userId, List<String> roles) {
        this.userName = userName;
        this.userId = userId;
        this.roles = roles;
    }

    // 省略 getter、setter
}

public class ThreadLocalCompositeObject {
    // 单个 ThreadLocal 存储复合对象
    private static final ThreadLocal<UserContext> userContextTl = new ThreadLocal<>();

    public static void main(String[] args) {
        new Thread(() -> {
            // 封装复合对象并设置
            UserContext userContext = new UserContext("李四", 1002L, Arrays.asList("普通用户"));
            userContextTl.set(userContext);

            // 获取复合对象并读取内部属性
            UserContext ctx = userContextTl.get();
            System.out.println("用户名：" + ctx.getUserName()); // 李四
            System.out.println("用户ID：" + ctx.getUserId()); // 1002
            System.out.println("角色：" + ctx.getRoles()); // [普通用户]

            // 统一清理
            userContextTl.remove();
        }, "线程3").start();
    }
}
```

#### 3. 方式三：直接操作 ThreadLocalMap（不推荐，底层 API，易出错）--略