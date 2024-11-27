---
title: Netty-2.md
published: 2024-10-26
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---
# Netty-2

## Netty概述

| 原生NIO                                                      | Netty                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **1、** NIO的类库和API繁杂，使用麻烦：需要熟练掌握Selector、ServerSocketChannel、SocketChannel、ByteBuffer等； | **1、** 设计优雅：适用于各种传输类型的统一API阻塞和非阻塞Socket；基于灵活且可扩展的事件模型，可以清晰地分离关注点；高度可定制的线程模型-单线程，一个或多个线程池； |
| **2、** 需要具备其他的额外技能：要熟悉Java多线程编程，因为NIO编程涉及到Reactor模式，你必须对多线程和网络编程非常熟悉，才能编写出高质量的NIO程序； | **2、** 使用方便：详细记录的Javadoc，用户指南和示例；没有其他依赖项，JDK5（Netty3.X）或6（Netty4.X）就足够了； |
| **3、** 开发工作量和难度都非常大：例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流的处理等等； | **3、** 高性能、吞吐量更高；延迟更低；减少资源消耗；最小化不必要的内存复制； |
| **4、** JDKNIO的Bug：例如臭名昭著EpollBug，它会导致Selector空轮询，最终导致CPU100%直到JDK1.7版本该问题仍然存在，没有被根本解决； | **4、** 安全：完成的SSL/TLS和StartTLS支持；                  |
|                                                              | **5、** 社区活跃、不断更新：社区活跃，版本迭代周期短，发现的Bug可以被及时修复，同时，更多的新功能会被加入； |



## 线程模型概述

#### 传统阻塞I/O服务模型

![ ](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202410261845757.png)

#### Reactor模式

![ ](https://www.ddkk.com/images/2023/9/11/1715/1694423708582.png)

核心概念：

- **Reactor**：负责监听和分发事件，将事件分发给相应的处理器（Handler）。
- **Handler**：负责处理特定的事件，执行具体的业务逻辑。
- **Event**：表示网络事件，如连接建立、数据读取、数据写入、连接关闭等。
- **Event Loop**：事件循环，不断监听和处理事件。

#### 3种实现方式

##### 单Reactor单线程

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202410261845139.png)

1、Select是前面I/O复用模型介绍的标准网络编程API，可以实现应用程序通过一个阻塞对象监听多路连接请求；

2、Reactor对象通过Select监听客户端请求事件，收到事件后通过Dispatch进行分发；

3、如果是建立连接请求事件，则由Acceptor通过accept处理连接请求，然后创建一个Handler对象处理连接完成后的后续业务处理；

4、如果不是建立连接事件，则Reactor会分发调用连接对应的Handler来响应；

5、Handler会完成Read -> 业务处理 -> Send 的完整业务流程。

##### 单Reactor多线程

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202410261846971.png)

1、Reactor对象通过select监控客户端请求事件，收到事件后，通过dispatch进行分发；

2、如果是建立连接请求，则由Acceptor通过accept处理连接请求，然后创建一个Handler对象处理完成连接后的各种事件；

3、如果不是连接请求，则由reactor分发调用连接对应的handler来处理；

4、handler只负责下响应事件，不做具体的业务处理，通过read读取数据后，会分发给后面的worker线程池的某个线程处理业务；

5、worker线程池会分配独立线程完成真正的业务，并将结果返回给handler；

6、handler收到响应后，通过send将结果返回给client。

##### 主从Reactor多线程

![ ](https://www.ddkk.com/images/2023/9/11/1715/1694423711294.png)

1、Reactor主线程MainReactor对象通过select监听连接事件，收到事件后，通过Acceptor处理连接事件；

2、当Acceptor处理连接事件后，MainReactor将连接分配给SubReactor；

3、SubReactor将连接加入到连接队列进行监听，并创建handler进行各种事件处理；

4、当有新事件发生时，SubReactor就会调用对应的handler处理；

5、handler通过read读取数据，分发给后面的worker线程处理；

6、worker线程池会分配独立的 worker 线程进行业务处理，并返回结果；

7、handler收到响应的结果后，再通过send方法将结果返回给client；

8、Reactor主线程可以对应多个Reactor子线程，即MainReactor可以关联多个SubReactor。

**方案优缺点说明：**

1、**优点：**父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。

2、**优点：**父线程与子线程的数据交互简单，Reactor主线程只需要把新连接传给子线程，子线程无需返回数据。

3、**缺点：**编程复杂度较高。

**结合实例：**这种模型在许多项目中广泛使用，包括Nginx主从Reactor多线程模型，Memcached主从多线程，Netty主从多线程模型的支持。

#### 总结

| 特性/模型      | 单Reactor单线程模型                                          | 单Reactor多线程模型                                          | 主从Reactor多线程模型                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **线程模型**   | 一个Reactor和一个线程处理所有事件                            | 一个Reactor和多个线程处理事件                                | 主Reactor负责监听连接请求，从Reactor负责处理连接上的I/O事件  |
| **优点**       | 1. **简单易用** ：模型简单，易于理解和实现。<br>2. **资源消耗低** ：只有一个线程，资源消耗低。 | 1. **性能提升** ：使用多个线程处理事件，提高了并发处理能力。<br>2. **资源利用率高** ：使用较少的线程处理多个连接，减少了线程切换和管理的开销。 | 1. **高性能** ：主Reactor负责监听连接请求，从Reactor负责处理I/O事件，提高了并发处理能力。<br>2. **扩展性好** ：适合高并发场景，能够处理大量并发连接。 |
| **缺点**       | 1. **性能瓶颈** ：所有事件都在一个线程中处理，容易成为性能瓶颈。<br>2. **扩展性差** ：不适合高并发场景，连接数较多时性能会下降。 | 1. **复杂性增加** ：相比单线程模型，多线程模型增加了复杂性，需要处理线程同步和竞争问题。<br>2. **资源消耗** ：虽然使用了多个线程，但相比传统阻塞I/O模型，资源消耗仍然较低。 | 1. **复杂性高** ：相比单Reactor模型，主从Reactor模型增加了复杂性，需要处理主从Reactor之间的协调问题。<br>2. **资源消耗** ：虽然使用了多个线程，但相比传统阻塞I/O模型，资源消耗仍然较低。 |
| **适用场景**   | 适用于连接数较少的场景，如小型服务器或客户端。               | 适用于连接数较多的场景，如中型服务器或客户端。               | 适用于高并发场景，如大型服务器或客户端。                     |
| **资源利用率** | 低，所有事件都在一个线程中处理。                             | 高，使用多个线程处理事件。                                   | 高，主Reactor负责监听连接请求，从Reactor负责处理I/O事件。    |
| **扩展性**     | 差，不适合高并发场景。                                       | 较好，适合连接数较多的场景。                                 | 好，适合高并发场景。                                         |
| **复杂性**     | 简单，易于理解和实现。                                       | 较复杂，需要处理线程同步和竞争问题。                         | 复杂，需要处理主从Reactor之间的协调问题。                    |
| **性能**       | 低，所有事件都在一个线程中处理，容易成为性能瓶颈。           | 高，使用多个线程处理事件，提高了并发处理能力。               | 高，主Reactor负责监听连接请求，从Reactor负责处理I/O事件，提高了并发处理能力。 |



## Netty模型

Netty主要基于主从Reactor多线程模型做了一定的改进，其中主从Reactor多线程模型有多个Reactor。

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202410261846858.png)

1、BossGroup线程维护Selector，只关注Accept事件；

2、当接收到Accept事件，获取到对应的SocketChannel，封装成NIOSocketChannel并注册到Worker线程（事件循环），并进行维护；

3、当Worker线程监听到selector中通道发生自己感兴趣的事件后，就进行处理（就由handler完成），注意handler已经加入到通道中。

#### Netty模型工作原理示意图-详细版

![ ](https://www.ddkk.com/images/2023/9/11/1715/1694423714526.png)

#### BossGroup 和 WorkerGroup

1. **BossGroup**:
    - **职责** : `BossGroup` 主要负责监听和接受客户端的连接请求。当一个新的连接请求到达时，`BossGroup` 中的线程会处理这个请求，并将其注册到一个 `Channel` 上。
    - **线程数** : `BossGroup` 通常只需要一个或几个线程，因为每个线程可以处理多个连接请求。线程的数量取决于服务器的配置和预期的并发连接数。
2. **WorkerGroup**:
    - **职责** : `WorkerGroup` 负责处理已经建立的连接上的数据读写操作。一旦 `BossGroup` 接受了一个新的连接并将其注册到一个 `Channel` 上，`WorkerGroup` 中的线程就会接管这个 `Channel`，负责处理后续的数据传输。
    - **线程数** : `WorkerGroup` 通常需要更多的线程，因为每个线程需要处理多个连接的数据读写操作。线程的数量可以根据服务器的处理能力和预期的并发连接数进行调整。

#### 各自的工作

- **BossGroup**:
    - **监听端口** : `BossGroup` 中的线程会监听指定的端口，等待客户端的连接请求。
    - **接受连接** : 当有新的连接请求到达时，`BossGroup` 中的线程会接受这个连接，并将其注册到一个 `Channel` 上。
    - **传递连接** : 接受连接后，`BossGroup` 会将这个连接传递给 `WorkerGroup`，由 `WorkerGroup` 负责后续的数据处理。
- **WorkerGroup**:
    - **处理数据** : `WorkerGroup` 中的线程会处理已经建立的连接上的数据读写操作。这包括从客户端读取数据、处理数据、以及将数据发送回客户端。
    - **事件驱动** : `WorkerGroup` 中的线程是事件驱动的，它们会根据 `Channel` 上的事件（如数据到达、连接关闭等）来执行相应的操作。
    - **线程池** : `WorkerGroup` 通常是一个线程池，可以并发处理多个连接的数据，从而提高系统的吞吐量和响应速度。

#### 总结

- **BossGroup** 负责监听和接受连接，将连接传递给 `WorkerGroup`。
- **WorkerGroup** 负责处理已经建立的连接上的数据读写操作。
- 这种分工明确的线程模型使得 Netty 能够高效地处理大量的并发连接，同时保持较低的资源消耗。

#### 服务端和客户端

```java
public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        // 创建 BossGroup 和 WorkerGroup
        // 说明
        // 1.创建两个线程组 BossGroup 和 WorkerGroup
        // 2. BossGroup 只是处理连接请求，真正的和客户端业务处理，会交给 WorkerGroup 完成
        // 3. 两个都是无线循环
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            // 创建服务器端的启动对象，配置启动参数
            ServerBootstrap bootstrap = new ServerBootstrap();
            // 使用链式编程进行设置
            bootstrap.group(bossGroup, workerGroup) // 设置两个线程组
                    .channel(NioServerSocketChannel.class) // 使用 ioServerSocketChannel 作为服务器通道实现
                    .option(ChannelOption.SO_BACKLOG, 128) // 设置线程队列等待连接个数
                    .childOption(ChannelOption.SO_KEEPALIVE, true)  // 设置保持活动连接状态
                    .childHandler(new ChannelInitializer<SocketChannel>() {// 创建一个通道初始化对象（匿名对象）
                        // 给 pipeline 设置处理器
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new NettyServerHandler());
                        }
                    });  // 给我们的WorkerGroup 的 EventLoop 对应的管道设置处理器
            System.out.println(".....服务器 is ready.....");

            // 绑定一个端口，并且同步，生成一个ChannelFuture对象
            // 启动服务器
            ChannelFuture cf = bootstrap.bind(6668).sync();

            // 对关闭通道进行监听
            cf.channel().closeFuture().sync();
        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

```java
/**
 * 说明：
 * 1.自定义一个 Handler 需要继承 netty 规定好的某个 handlerAdapter
 * 2.这是我们自定义的 Handler，才能称之为一个 handler
 */
public class NettyServerHandler extends ChannelInboundHandlerAdapter {
    // 读取数据实现（这里我们可以读取客户端发送的消息）
    /**
     * 1. ChannelHandlerContext ctx：上下文对象，含有管道 pipeline，通道channel，地址
     * 2. Object msg：就是客户端发送的数据 默认Object
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("server ctx = " + ctx);
        // 将 msg 转成一个 ByteBuf
        // ByteBuf 是 Netty 提供的，不是 NIO 的 ByteBuffer
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("客户端发送消息是：" + buf.toString(CharsetUtil.UTF_8));
        System.out.println("客户端地址：" + ctx.channel().remoteAddress());
    }

    // 数据读取完毕
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        // writeAndFlush 是 write + flush
        // 将数据写入到缓存，并刷新
        // 一般讲，我们对这个发送的数据进行编码
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello，客户端~", CharsetUtil.UTF_8));
    }

    // 处理异常，一般是需要关闭通道
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        // 客户端需要一个事件循环组
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            // 创建客户端启动对象
            // 注意客户端使用的不是 ServerBootStrap 而是 BootStrap
            Bootstrap bootstrap = new Bootstrap();

            // 设置相关参数
            bootstrap.group(group) // 设置线程组
                    .channel(NioSocketChannel.class) // 设置客户端通道的实现类（反射）
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            // 添加编码器
                            ch.pipeline().addLast(new StringEncoder());
                            ch.pipeline().addLast(new NettyClientHandler());  // 加入自己的处理器
                        }
                    });

            System.out.println("客户端 ok....");

            // 启动客户端去连接服务端
            // 关于 ChannelFuture 要分析，涉及到 netty 的异步模型
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6668).sync();
            Channel channel = channelFuture.channel();
            // 监听控制台的输入
            Scanner scanner = new Scanner(System.in);
            while (true) {
                String input = scanner.nextLine();
                if ("exit".equalsIgnoreCase(input)) {
                    break;
                }
                channel.writeAndFlush(input);
            }

            // 给关闭通道进行监听
            channelFuture.channel().closeFuture().sync();
        }finally {
            group.shutdownGracefully();
        }
    }
}
```

```java
public class NettyClientHandler extends ChannelInboundHandlerAdapter {

    // 当通道就绪，就会触发该方法
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("client " + ctx);
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello，server： 喵", CharsetUtil.UTF_8));
    }

    // 当通道有读取事件时，会触发
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("服务器回复的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
        System.out.println("服务器的地址： " + ctx.channel().remoteAddress());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

![recording](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202410261842455.gif)