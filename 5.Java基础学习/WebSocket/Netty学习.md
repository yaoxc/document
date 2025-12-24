## 一、什么是netty？

**Netty 是一个基于 Java NIO 的`高性能、异步事件驱动`的`网络应用程序框架`**，把它当成“JDK 原生 NIO 的超高性能封装版 + 一条可插拔的流水线”即可。

**底层：Java NIO、IO 多路复用**。

核心目标：**让程序员只关心“收到数据后我要做什么”，而不用操心怎么把数据从网卡里读出来、怎么拼包、怎么扛住 10 万个并发连接**。

---

### 1、出现背景

- BIO（一条连接占用一个线程）在 C10K 场景下直接跪。
- JDK 1.4 推出 NIO，但 API 难用、epoll bug、空轮询、断线重连、粘包拆包全是坑。
- Netty 把这部分全部打包解决：线程模型、内存池、零拷贝、协议编解码、链路心跳、流量整形、SSL/TLS、**WebSocket**、HTTP/2、**gRPC**……开箱即用。

------

### 2、核心概念（3 张图就能记住）
Channel      —— 一条 socket 抽象，可读可写。
EventLoop    —— 单线程 Reactor，负责一个或多个 Channel 的所有 IO 事件。
ChannelPipeline —— 双向链表，里面全是 ChannelHandler；数据进来从头流到尾，出去从尾流回头。
ByteBuf      —— 自研字节容器，引用计数、池化、零拷贝，比 NIO ByteBuffer 快且安全。
Future/ChannelPromise —— 所有 IO 都是异步，返回一个 Future，可以同步等待也可以回调。

------

### 3、运行模型（一句话）
“主从 **Reactor** 多线程”：
BossGroup 只负责 accept → 把已建立的 socket 注册到 WorkerGroup；
WorkerGroup 里的每条线程（即每个 EventLoop）不断 select → 拿到 IO 事件后沿 pipeline 丢给业务 handler。

#### 什么是Reactor ？

Reactor = “反应堆”式的 **事件驱动、单线程(池)多路复用** 模型。
一句话：一条线程（或一个线程池）同时 **“监听很多连接，哪个连接有事件就处理哪个”**，而不是像 BIO 那样“一条连接占一条线程”。

------

##### 为什么叫 Reactor

- 它把操作系统提供的多路复用 API（select/poll/epoll/kqueue）封装成“事件循环”：
  1. 注册通道（Channel）和关心的事件（OP_ACCEPT / OP_READ / OP_WRITE）。
  2. 循环调用 selector.select() → 阻塞等待内核通知“哪些通道已就绪”。
  3. 返回一批“就绪事件”，分发到对应的 handler 进行处理。
     这个过程像“反应堆不断探测并响应事件”，所以起名叫 Reactor。

------

##### 三种线程角色（Doug Lea 的经典分法）

###### 1、Single Reactor 单线程
accept + read + decode + compute + encode + send 都在一条线程里跑；
足够快、无锁，但 CPU 密集或慢 handler 会拖住整条循环。

###### 2、Reactor + Worker 线程池（主从）
Reactor 线程只负责 IO（accept/read/write），把耗时的 decode/compute/encode 任务丢给 Worker 线程池；
Netty 的 BossGroup + WorkerGroup 就是这种模型。

###### 3、Multiple Reactor
一个 MainReactor 只 accept，拿到 socket 后注册到 SubReactor；
每个 SubReactor 对应一条线程，管理若干连接的读写；
Netty 里“BossGroup=1 线程，WorkerGroup=N 线程”就是它的实现。

------

##### 和“Proactor”区别（面试常问）

- Reactor：内核告诉你“某个 fd 可以读/写了”，**应用自己**把数据从内核拷到用户空间再处理——**同步 IO**。
- Proactor：内核帮你把数据**已经拷好**，通知你“读完成/写完成”，应用直接处理——**异步 IO**（Windows IOCP 典型实现）。
  Linux 下真正的异步 AIO 支持有限，所以 Netty 等高性能框架依旧用 Reactor（epoll）。

------

##### 总结
Reactor = “事件来了再干活”的单线程(池)多路复用模型；
它用最少线程伺候最多连接，是 Netty、Redis、Nginx、Node.js 都能扛高并发的底层原因。

------

### 4、能做什么

- 自定义协议：RPC（Dubbo、gRPC）、游戏私有协议、物联网终端协议。
- 高频长连接：IM、实时行情、推送网关。
- 高并发短连接：API 网关、HTTP 静态资源服务器。
- 支持 WebSocket、HTTP/1.1、HTTP/2、MQTT、Redis Protocol 等官方编解码器，直接拼装即可。

------

### 5、与 Spring 的关系
Netty 完全不依赖 Spring，可以独立跑 main 方法；
Spring WebFlux、Spring Cloud Gateway 的底层 reactor-netty 就是 Netty 的封装版；
**Spring “官方” WebSocket 也能切换成 Netty 容器**，只是对业务代码透明。



## 二、Netty和WebSocket什么关系？

Netty 和 WebSocket 的关系可以概括为：

- **WebSocket 是一种基于 TCP 的应用层协议（用于浏览器与服务器的双向通信）**
- **而 Netty 是一个高性能的 Java NIO 网络编程框架，它提供了对 WebSocket 协议的完整支持，能让开发者快速搭建高并发的 WebSocket 服务器（或客户端）**。

所以：WebSocket 是 “通信规则”，Netty 是 “通信工具包”—— 遵循 WebSocket 规则的通信，能通过 Netty 这个工具包高效实现。



## 三、使用Netty实现WebSocket的优势

如果不用 Netty，直接用 Java 原生的 `javax.websocket`（JSR 356）也能实现 WebSocket，但 Netty 有明显优势。

### 1、优势

1. **高性能**：基于 NIO （Non-blocking I/O）模型和事件驱动，支持百万级并发连接（适合高流量场景，如直播弹幕、实时游戏）；
2. **灵活性**：可自定义协议处理逻辑（如添加心跳检测、消息加密、断线重连）；
3. **一站式解决方案**：Netty 不仅支持 WebSocket，还支持 TCP、UDP、HTTP/2 等协议，可统一技术栈；
4. **低延迟**：优化了数据传输的底层细节，减少了上下文切换和内存拷贝。

### 2、总结

1. **WebSocket 是协议**：定义了客户端与服务器双向通信的规则，解决了 HTTP 单向通信的问题；
2. **Netty 是框架**：提供了高性能的网络编程能力，内置了 WebSocket 协议的处理组件，简化了 WebSocket 服务器的开发；
3. **关系本质**：Netty 是 WebSocket 协议的**高性能实现载体**，开发者通过 Netty 可以快速搭建高并发的 WebSocket 应用，而无需关心底层的协议解析和 IO 处理。

简单来说：**要做 WebSocket 开发，Netty 是 Java 领域的首选工具之一**。



## 四、Netty从0️⃣学习

### 1、为什么需要Netty？

先想一个问题：Java 本身不是有 Socket（BIO）和 NIO 吗？为什么还要用 Netty？

#### 1. 传统 Socket（BIO）的痛点（举个栗子）

假设你开了一家小超市，**BIO 就像一个收银员对应一个顾客**：

- 顾客 A 来结账，收银员全程服务 A，直到 A 结完账，才能服务顾客 B；
- 如果顾客 A 结账时突然停下来找钱包（网络延迟 / IO 阻塞），收银员只能干等着，后面的顾客全排队卡死。

这就是 BIO 的**同步阻塞**问题：一个线程只能处理一个连接，连接多了线程数爆炸，性能急剧下降。

#### 2. Java NIO 的改进与痛点

Java NIO（非阻塞 IO）解决了阻塞问题，

但 Java NIO 有个致命问题：**API 复杂、难用，还需要自己处理各种细节**（比如选择器轮询、缓冲区操作、半包粘包、线程安全），新手很容易写出 bug。

#### 3. Netty 的定位：“NIO 懒人神器”

Netty 基于 Java NIO 封装了所有复杂细节，提供了简洁、易用的 API，同时还优化了性能（比如零拷贝、内存池）、解决了网络通信的常见问题（半包粘包、断连重连）。

简单说：**Netty = 高性能 NIO + 开箱即用的网络通信解决方案**。



### 2、Netty核心概念：用 “快递站” 案例理解

我们用**快递站的工作流程**来对应 Netty 的核心组件。

#### 1、场景假设

你是一个快递站的站长，需要处理大量快递的 “接收” 和 “发送”：

- 顾客（客户端）把快递送到快递站（服务端）；
- 快递站把快递分类、打包后，送到目的地（另一个客户端 / 服务端）。

#### 2、Netty 核心组件与快递站的对应关系

| Netty 组件                    | 快递站对应角色 / 功能                   | 核心作用                                                     |
| ----------------------------- | --------------------------------------- | ------------------------------------------------------------ |
| **Bootstrap/ServerBootstrap** | 快递站的 “开业筹备组”                   | 启动客户端 / 服务端，配置线程、端口、处理器等核心参数        |
| **EventLoopGroup**            | 快递站的 “员工团队”（分为两组）         | 一组负责**接收连接**（前台接待员），一组负责**处理业务**（后台分拣员） |
| **Channel**                   | 快递站的 “快递通道”（比如一条快递线）   | 代表一个网络连接（客户端与服务端的双向通信通道），所有数据都通过 Channel 传输 |
| **ChannelHandler**            | 快递站的 “处理工序”（分拣、打包、贴单） | 处理网络事件（接收数据、发送数据、异常处理），是**业务逻辑的核心** |
| **ChannelPipeline**           | 快递站的 “流水线”                       | 把多个 ChannelHandler 串起来，数据按顺序经过每个 Handler 处理 |
| **ByteBuf**                   | 快递站的 “快递包裹”                     | Netty 封装的缓冲区，替代 Java NIO 的 ByteBuffer，更易用、高效 |

#### 3、一句话总结核心流程

服务端启动后，EventLoopGroup 接收客户端连接，创建 Channel；数据通过 Channel 传输，经过 ChannelPipeline 中的一系列 ChannelHandler 处理（比如解码、业务逻辑、编码），最终完成通信。

##### -  ChannelPipeline 、ChannelHandler 关系图

![image-20251214202504445](/Users/felix/Library/Application Support/typora-user-images/image-20251214202504445.png)

### 3、搭建 Netty 开发环境

#### 1. JDK 要求

Netty 4.x 要求 JDK 1.7+（推荐用 JDK 8，兼容性最好）。

#### 2. 依赖引入（Maven）

在 `pom.xml` 中添加以下依赖：

```xml
<!-- Netty 核心依赖 -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.94.Final</version> <!-- 版本可换最新稳定版 -->
</dependency>
```

### 4、第一个 Netty 程序：简易聊天机器人（客户端 + 服务端）

我们先写一个最简单的案例：**客户端发送消息，服务端接收后回复固定消息**（类似聊天机器人）。这个案例能覆盖 Netty 的核心使用流程。

#### 步骤 1：编写服务端代码

服务端的作用：绑定端口，接收客户端连接，处理客户端消息并回复。

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

/**
 * Netty 服务端：聊天机器人
 */
public class NettyServer {
    // 服务端端口
    private final int port;

    public NettyServer(int port) {
        this.port = port;
    }

    public void start() throws InterruptedException {
        // 1. 创建两个 EventLoopGroup（员工团队）
        // bossGroup：负责接收客户端连接（前台接待）
      	// 显式指定 线程数 = 1；底层只建一条线程，专门给 BossGroup 用来“接客”（accept 新连接）
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // workerGroup：负责处理客户端的读写事件（后台处理）
        // 默认值是 Runtime.availableProcessors() * 2，即 CPU 核数×2；
        EventLoopGroup workerGroup = new NioEventLoopGroup(); // 默认线程数是 CPU 核心数 * 2

        try {
            // 2. 创建 ServerBootstrap（开业筹备组）
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup) // 绑定两个线程组
                    .channel(NioServerSocketChannel.class) // 指定服务端通道类型（NIO 模式）
              			// 只在 服务端 ServerBootstrap 里生效, 客户端 Bootstrap 里写这个选项会被忽略
              			// 告诉 操作系统内核：“当三次握手完成但应用还没 accept() 时，最多给我暂存 128 个连接。”
              			// 作用举例： backlog=128，1 万客户端同时打过来 → 只有 128 个能暂存，其余直接拒绝
                    .option(ChannelOption.SO_BACKLOG, 128) // 设置连接队列大小
                    .childOption(ChannelOption.SO_KEEPALIVE, true) // 保持长连接
              			// Netty 服务端在“生”出每一个新连接（SocketChannel）后， 给这条连接装配处理器的“流水线作业指导书”
                    // childHandler 只在 服务端 ServerBootstrap 出现
              
              			// 当 Boss 线程 accept 到一个新客户端连接时，Netty 会创建一个 SocketChannel 实例，
                    // 并回调这里的 initChannel() 方法，让你给这条新连接“穿衣服”——也就是组装它的 ChannelPipeline
                    .childHandler(new ChannelInitializer<SocketChannel>() { // 配置通道的处理器
                        
                      	// SocketChannel  --- 1:1 --->  ChannelPipeline
                      	@Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            // 3. 获取通道的流水线（Pipeline），添加处理器
                            // 每个 SocketChannel 自带一条空的 双向链表 管道，拿到引用后就可以往里面塞处理器
                            ChannelPipeline pipeline = ch.pipeline();

                            // 添加字符串编解码器：解决字节与字符串的转换
                            // 把 字节流 → 字符串 的解码器放到链表尾部
                            // 当收到数据时，入站事件从头往后传，先经过它，于是你的业务 Handler 直接拿到 String 而不是 ByteBuf
                            pipeline.addLast("decoder", new StringDecoder()); // 解码：字节→字符串
                          
                          	// 把 字符串 → 字节流 的编码器也放进去；
													 // 当你 ctx.writeAndFlush("hello") 时，出站事件从后往前传，它会自动把字符串编码成字节再发出去。
                            pipeline.addLast("encoder", new StringEncoder()); // 编码：字符串→字节

                            // 添加自定义的业务处理器：处理客户端消息
                            // 最后放你写的 业务处理器；因为前面已经“拆好包、转成字符串”，
                            // 所以你在 channelRead(ChannelHandlerContext ctx, Object msg) 里拿到的 msg 就是纯字符串，直接处理即可。
                            pipeline.addLast(new NettyServerHandler());
                        }
                    });

            System.out.println("Netty 服务端启动中，端口：" + port);

            // 4. 绑定端口并同步等待（启动服务）
            ChannelFuture future = serverBootstrap.bind(port).sync();

            // 5. 等待通道关闭（阻塞，直到服务端被关闭）
            future.channel().closeFuture().sync();
        } finally {
            // 6. 优雅关闭线程组
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 启动服务端，端口 8080
        new NettyServer(8080).start();
    }

    /**
     * 自定义业务处理器：处理客户端的消息
     * SimpleChannelInboundHandler：泛型指定处理的数据类型（这里是 String）
     */
    static class NettyServerHandler extends SimpleChannelInboundHandler<String> {

        // 当接收到客户端消息时触发
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
            // 获取客户端通道信息
            String clientId = ctx.channel().remoteAddress().toString();
            System.out.println("收到客户端[" + clientId + "]的消息：" + msg);

            // 回复客户端消息（聊天机器人逻辑）
            String response = "机器人回复：你说的是\"" + msg + "\"，我收到啦！";
            ctx.writeAndFlush(response); // 发送回复
        }

        // 当客户端连接建立时触发
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            System.out.println("客户端[" + ctx.channel().remoteAddress() + "]已连接");
        }

        // 当发生异常时触发
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close(); // 关闭通道
        }
    }
}
```

##### 1、每一个channel对应一个NettyServerHandler实例么?

是的。Netty 的 `ChannelPipeline` 是“**每条连接独享一条双向链表**”，链表上的每一个 `ChannelHandler` 默认也都是 **new 出来的独立实例**。

因此在你写的代码里：

```java
p.addLast(new NettyServerHandler());   // 每来一个连接就 new 一次
```

每建一条 `SocketChannel`，就会执行一次 `initChannel()`，于是：

> 1 个客户端连接 → 1 个 `ChannelPipeline` → 1 个 **新的** `NettyServerHandler` 实例。

---

**例外情况（共用实例）**
如果你把 handler 声明成 **@Sharable** 并在 `addLast` 里传入 **同一个** 单例：

```java
@Sharable
public class ShareServerHandler extends ChannelInboundHandlerAdapter { ... }

static final ShareServerHandler INSTANCE = new ShareServerHandler();

// 然后在 initChannel 里
p.addLast(INSTANCE);   // 所有连接都用这同一个对象
```

此时所有 Channel 才会共享同一个 handler 实例，但前提是你必须保证线程安全。
默认不加 `@Sharable` 时，Netty 禁止这么做，启动就会抛异常。

---

总结：

> 默认模式下：
> **一条连接 → 一个 ChannelPipeline → 一套全新的 Handler 实例**，所以每个 channel 对应自己的 `NettyServerHandler`，互不干扰。

---

##### 2. 上面代码装配的 Pipeline 解析

###### 1、Pipeline 顺序

> 首尾是 Netty 自带的 HeadContext & TailContext

```xml
HeadContext ←→ StringDecoder(解码：字节→字符串)  ←→  NettyServerHandler  ←→  StringEncoder(编码：字符串→字节)  ←→  TailContext
   ↑  (入站从头往后)                                                                                   (出站从后往前)   ↑
```

###### 2、入站（client → server）流程

1. 内核把 TCP 报文放到接收缓冲区
2. HeadContext 触发 `channelRead()`，消息是 `ByteBuf`
3. `ByteBuf` 继续往后传 → **StringDecoder**
   - 解码：把 `ByteBuf` 拆成完整帧 → 转成 `String`
   - 调用 `ctx.fireChannelRead("hello")` 继续往后
4. 传到 **NettyServerHandler**
   - 业务代码收到 `String msg = "hello"`
   - 假设我们回复客户端：
     `ctx.writeAndFlush("HELLO")`   // 注意这是出站操作
5. 写事件掉头往前传 → **StringEncoder**
   - 把 `String "HELLO"` 编码成 `ByteBuf`
6. 再往前 → **HeadContext**
   - 把 `ByteBuf` 写进内核发送缓冲区
7. 客户端收到响应 `"HELLO"`

------

###### 3、出站（server → client）流程
（任何 handler 只要调用 `ctx.writeAndFlush(obj)` 都会触发）

1. 当前调用点（示例在 NettyServerHandler）
2. 事件从 **当前节点的前一个节点** 开始往前传
   - 这里会经过 **StringEncoder**
   - 如果对象是 `String`，被编码成 `ByteBuf`
3. 继续往前到 **HeadContext** → 写 socket
4. 如果 write 的是 `ByteBuf`，StringEncoder 会直接透传

###### 4、Demo 数据流转案例（带打印）

0）Pipeline 装配代码（与你给出的一致）

```java
serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast("decoder", new StringDecoder());          // 1
        p.addLast("encoder", new StringEncoder());          // 2
        p.addLast("biz", new NettyServerHandler());         // 3
    }
});
```

1）自定义业务 handler（把入站、出站都打印）

```java
public class NettyServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("【ServerHandler-入站】收到字符串 = " + msg);
        // 业务：原样转大写并回写
        String resp = msg.toUpperCase();
        System.out.println("【ServerHandler-准备出站】写回 = " + resp);
        // 出站起点
        ctx.writeAndFlush(resp);   
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

2）客户端简单 telnet / nc 测试

```bash
$ nc localhost 8080
hello
```

3）服务端控制台打印（一行对应一次 handler 事件）

```
【StringDecoder】          解码前：ByteBuf(hello)
【StringDecoder】          解码后：String(hello) 并 fireChannelRead
【ServerHandler-入站】     收到字符串 = hello
【ServerHandler-准备出站】 写回 = HELLO
【StringEncoder】          编码：String(HELLO) -> ByteBuf(HELLO)
【HeadContext】            最终写出 ByteBuf 到 socket
```

客户端立即收到：

```
HELLO
```

###### 5、过程图解

```xml
client  hello ─►ByteBuf─►Head─►StringDecoder─►NettyServerHandler─►StringEncoder─►Head─►ByteBuf─►HELLO─►client
                      入站方向  →  →  →  →  →  →  →  →  →  →  →  →  →  →  →  →  →  →  出站方向
```

只要记住：

- **入站**从 Head 往后传，谁 `fire*` 谁就把事件递给下一个；
- **出站**从当前节点“前一个”开始往前传，谁 `write*` 谁就是起点；
- `StringDecoder` 只处理入站，`StringEncoder` 只处理出站，互不干扰

###### 6、为什么 NettyServerHandler 在 StringDecoder、StringEncoder 中间

真正决定“**谁在前谁在后**”的，不是画图好看，而是：

1. 数据必须先**解码**成业务对象 →
2. 业务 Handler 才能处理 →
3. 处理完要**编码**回字节才能发出去。

------

事件流向强制了顺序

入站（读）
Head → StringDecoder → NettyServerHandler → Tail
  ①   ②    ③

出站（写）
Head ← StringEncoder ← NettyServerHandler ← Tail
  ③   ②    ①

------

因此：

- 把 `StringDecoder` 放在 **NettyServerHandler 前面**，保证 `channelRead` 拿到的是 `String` 而不是 `ByteBuf`。
- 把 `StringEncoder` 放在 **NettyServerHandler 后面**，保证 `writeAndFlush(String)` 时编码器能拦截到并完成 `String → ByteBuf`。

> 只要满足“**解码→业务→编码**”这条流水线，外观顺序自然呈现为`StringDecoder → NettyServerHandler → StringEncoder`，
> 否则业务代码就要自己去跟 `ByteBuf` 打交道，失去解码/编码的意义。

---

##### 3、如何理解 serverBootstrap.bind(port).sync()

###### 1、这行代码会卡住么？

> ChannelFuture future = serverBootstrap.bind(port).sync(); 不会让程序卡住

不会“卡住”，`bind(port).sync()` 只做两件事：

1. **`同步`等待端口绑定完成**（通常几毫秒） ，返回的 `ChannelFuture` 是 **操作结果**，而不是“服务生命周期”。
2. 一旦绑定成功，主线程就继续往下走； 如果没有别的逻辑，main 方法结束，整个 JVM 会退出——**服务器瞬间又关了**。

生产代码都会在 `bind()` 之后再加一行：

```java
future.channel().closeFuture().sync();   // ①   这里才会卡住，保证 JVM 不会提前退出
```

- `closeFuture()` 拿到的是 **服务端 Channel 的“关闭事件” Future**

- 再 `.sync()` 会让 **主线程挂起**，直到 **有人把这条 Channel 关掉**

  > 调用 `close()` :  
  >
  > ```java
  > Channel serverChannel = serverBootstrap.bind(8080).sync().channel();   // 启动时保存起来
  > serverChannel.close();                                                // 触发 closeFuture
  > ```
  >
  > `进程收到 kill`:  重启服务

所以 **① 才是真正的阻塞点**，保证 JVM 不会提前退出，Netty 事件线程可以一直跑在后台。

---

###### 2、 服务端Channel 关闭事件

服务端 Channel 的“关闭事件”本质是一个 `ChannelFuture`，Netty 把它挂在 **服务端 Channel** 上，随时可以让你监听或同步等待。常用三种拿法：

1、同步阻塞（上面例子那样，main 线程一直挂起）

```java
ChannelFuture bindFuture = serverBootstrap.bind(8080).sync();
Channel serverChannel = bindFuture.channel();

// 当前线程阻塞在这里，直到 serverChannel.close() 被调用
serverChannel.closeFuture().sync();
```

------

2、异步监听（不阻塞启动线程，**适合集成 Spring 等**）

```java
Channel serverChannel = serverBootstrap.bind(8080).sync().channel();

serverChannel.closeFuture().addListener(f -> {
    // 回调在 EventLoop 线程里执行
    System.out.println("服务端 Channel 已关闭，准备回收资源");
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
});
```

------

3、主动关闭（代码里想停服时）

```java
// 在其他地方（例如停机钩子、管理线程）调用
Channel serverChannel = ...;   // 启动时保存起来
serverChannel.close();         // 触发 closeFuture
```

要点小结

- 只有 **服务端 Channel**（`NioServerSocketChannel`）才有意义，客户端 `SocketChannel` 关闭只影响那条连接。
- `closeFuture()` 返回的 `ChannelFuture` 是 **全局唯一** 的，可以重复 `addListener` 或 `sync`。
- 想优雅停机，通常先 `serverChannel.close()`，再 `bossGroup.shutdownGracefully()` / `workerGroup.shutdownGracefully()`。

---

#### 步骤 2：编写客户端代码

客户端的作用：连接服务端，发送消息，接收服务端回复。

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.util.Scanner;

/**
 * Netty 客户端：发送消息给服务端
 */
public class NettyClient {
    // 服务端地址和端口
    private final String host;
    private final int port;
    private Channel channel; // 客户端通道，用于发送消息

    public NettyClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void start() throws InterruptedException {
        // 1. 创建 EventLoopGroup（客户端只有一个线程组，因为不需要接收连接）
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            // 2. 创建 Bootstrap（客户端启动器）
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class) // 指定客户端通道类型
                    .option(ChannelOption.TCP_NODELAY, true) // 禁用 Nagle 算法，加快消息发送
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            // 3. 添加处理器（和服务端对应，编解码器+业务处理器）
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("decoder", new StringDecoder());
                            pipeline.addLast("encoder", new StringEncoder());
                            pipeline.addLast(new NettyClientHandler());
                        }
                    });

            System.out.println("连接服务端中：" + host + ":" + port);

            // 4. 连接服务端
            ChannelFuture future = bootstrap.connect(host, port).sync();
            // 获取通道
            channel = future.channel();
            System.out.println("客户端连接成功，可以开始发送消息（输入exit退出）");

            // 5. 控制台输入消息，发送给服务端
            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNextLine()) {
                String msg = scanner.nextLine();
                if ("exit".equals(msg)) {
                    channel.close(); // 关闭通道
                    break;
                }
                channel.writeAndFlush(msg); // 发送消息
            }

            // 6. 等待通道关闭
            channel.closeFuture().sync();
        } finally {
            // 7. 优雅关闭线程组
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 连接本地服务端，端口 8080
        new NettyClient("127.0.0.1", 8080).start();
    }

    /**
     * 客户端业务处理器：接收服务端的回复
     */
    static class NettyClientHandler extends SimpleChannelInboundHandler<String> {

        // 接收服务端回复的消息时触发
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
            System.out.println("收到服务端回复：" + msg);
        }

        // 连接异常时触发
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close();
        }
    }
}
```

#### 步骤 3：运行程序

1. 先启动 `NettyServer`，控制台输出：`Netty 服务端启动中，端口：8080`；
2. 再启动 `NettyClient`，控制台输出：`客户端连接成功，可以开始发送消息（输入exit退出）`；
3. 在客户端控制台输入消息（比如 “你好，Netty”），服务端会打印收到的消息，客户端会收到服务端的回复；
4. 输入 “exit”，客户端断开连接。

#### 代码解析（新手重点看）

1. **EventLoopGroup**：服务端用了两个（boss/worker），客户端只用了一个，因为客户端不需要接收其他连接；
2. **编解码器**：`StringDecoder` 和 `StringEncoder` 是 Netty 提供的字符串编解码器，帮我们把字节数据转换成字符串，避免手动处理字节；
3. **ChannelHandler**：自定义的 `NettyServerHandler` 和 `NettyClientHandler` 是业务逻辑的核心，重写 `channelRead0` 处理接收到的消息；
4. **writeAndFlush**：`ctx.writeAndFlush()` 用于发送消息，`write` 是把消息写入缓冲区，`flush` 是把缓冲区的消息发送出去（可以合并成 `writeAndFlush`）。
