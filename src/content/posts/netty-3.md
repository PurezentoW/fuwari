---
title: netty-3
published: 2024-11-26
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---
# Netty-应用

## 架构

### Selector 模型

`Selector` 模型解决了传统的阻塞 I/O 编程一个客户端一个线程的问题。Selector 提供了一种机制，用于监视一个或多个 NIO 通道，并识别何时可以使用一个或多个 NIO 通道进行数据传输。这样，一个线程就可以管理多个通道，从而管理多个网络连接。

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556286.png)

`Selector` 提供了选择执行已经就绪的任务的能力。从底层来看，Selector 会轮询 Channel 是否已经准备好执行每个 I/O 操作。Selector 允许单线程处理多个 Channel 。Selector 是一种多路复用的技术。



### 事件驱动

> **Netty**是一款异步的事件驱动的网络应用程序框架。在 Netty 中，事件是指对某些操作感兴趣的事。例如，在某个Channel注册了 OP_READ，说明该 Channel 对读感兴趣，当 Channel 中有可读的数据时，它会得到一个事件的通知。

在`Netty` 事件驱动模型中包括以下核心组件。

#### Channel

> Channel（管道）是 Java NIO 的一个基本抽象，代表了一个连接到如硬件设备、文件、网络 socket 等实体的开放连接，或者是一个能够完成一种或多种不同的`I/O` 操作的程序。

#### 回调

> 回调 就是一个方法，一个指向已经被提供给另外一个方法的方法的引用。这使得后者可以在适当的时候调用前者，Netty 在内部使用了回调来处理事件；当一个回调被触发时，相关的事件可以被一个`ChannelHandler`接口处理。

例如：在上一篇文章中，Netty 开发的服务端的管道处理器代码中，当`Channel`中有可读的消息时，`NettyServerHandler`的回调方法`channelRead`就会被调用。

```java
public class NettyServerHandler extends ChannelInboundHandlerAdapter {
    //读取数据实际(这里我们可以读取客户端发送的消息)
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("server ctx =" + ctx);
        Channel channel = ctx.channel();
        //将 msg 转成一个 ByteBuf
        //ByteBuf 是 Netty 提供的，不是 NIO 的 ByteBuffer.
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("客户端发送消息是:" + buf.toString(CharsetUtil.UTF_8));
        System.out.println("客户端地址:" + channel.remoteAddress());
    }
    //处理异常, 一般是需要关闭通道
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

#### Future

> Future 可以看作是一个异步操作的结果的占位符；它将在未来的某个时刻完成，并提供对其结果的访问，Netty 提供了 `ChannelFuture` 用于在异步操作的时候使用，每个 Netty 的出站 I/O 操作都将返回一个 `ChannelFuture`(完全是异步和事件驱动的)。

以下是一个 `ChannelFutureListener`使用的示例。

```java
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ChannelFuture future = ctx.channel().close();
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture) throws Exception {   
                //..
            }
        });
    }
```

#### 事件及处理器

在Netty 中事件按照出/入站数据流进行分类：

**入站数据或相关状态更改触发的事件包括：**

- 连接已被激活或者失活。
- 数据读取。
- 用户事件。
- 错误事件。

**出站事件是未来将会出发的某个动作的操作结果：**

- 打开或者关闭到远程节点的连接。
- 将数据写或者冲刷到套接字。

每个事件都可以被分发给`ChannelHandler`类中的某个用户实现的方法。如下图展示了一个事件是如何被一个这样的`ChannelHandler`链所处理的。

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556997.png)

`ChannelHandler` 为处理器提供了基本的抽象，可理解为一种为了响应特定事件而被执行的回调。

### 责任链模式

> 责任链模式(Chain of Responsibility Pattern)是一种行为型设计模式，它为请求创建了一个处理对象的链。其链中每一个节点都看作是一个对象，每个节点处理的请求均不同，且内部自动维护一个下一节点对象。当一个请求从链式的首端发出时，会沿着链的路径依次传递给每一个节点对象，直至有对象处理这个请求为止。

责任链模式的重点在这个 "链"上，由一条链去处理相似的请求，在链中决定谁来处理这个请求，并返回相应的结果。在Netty中，定义了`ChannelPipeline`接口用于对责任链的抽象。

责任链模式会定义一个抽象处理器（Handler）角色，该角色对请求进行抽象，并定义一个方法来设定和返回对下一个处理器的引用。在Netty中，定义了`ChannelHandler`接口承担该角色。

#### 责任链模式的优缺点

优点：

- 发送者不需要知道自己发送的这个请求到底会被哪个对象处理掉，实现了发送者和接受者的解耦。
- 简化了发送者对象的设计。
- 可以动态的添加节点和删除节点。

缺点：

- 所有的请求都从链的头部开始遍历，对性能有损耗。
- 不方便调试。由于该模式采用了类似递归的方式，调试的时候逻辑比较复杂。

使用场景：

- 一个请求需要一系列的处理工作。
- 业务流的处理，例如文件审批。
- 对系统进行扩展补充。

#### ChannelPipeline

Netty 的`ChannelPipeline`设计，就采用了责任链设计模式， 底层采用双向链表的数据结构,，将链上的各个处理器串联起来。

客户端每一个请求的到来，Netty都认为，`ChannelPipeline`中的所有的处理器都有机会处理它，因此，对于入栈的请求，全部从头节点开始往后传播，一直传播到尾节点(来到尾节点的msg会被释放掉)。

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556253.png)

**入站事件**：通常指 IO 线程生成了入站数据（通俗理解：从 socket 底层自己往上冒上来的事件都是入站）。
比如`EventLoop`收到`selector`的`OP_READ`事件，入站处理器调用`socketChannel.read(ByteBuffer)`接受到数据后，这将导致通道的`ChannelPipeline`中包含的下一个中的`channelRead`方法被调用。

**出站事件**：通常指 IO 线程执行实际的输出操作（通俗理解：想主动往 socket 底层操作的事件的都是出站）。
比如`bind`方法用意时请求`server socket`绑定到给定的`SocketAddress`，这将导致通道的`ChannelPipeline`中包含的下一个出站处理器中的`bind`方法被调用。

#### 将事件传递给下一个处理器

处理器必须调用`ChannelHandlerContext`中的事件传播方法，将事件传递给下一个处理器。

入站事件和出站事件的传播方法如下图所示：

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556360.png)

以下示例说明了事件传播通常是如何完成的：

```java
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("Connected!");
        ctx.fireChannelActive();
    }
}

public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        System.out.println("Closing...");
        ctx.close(promise);
    }
}
```





## Netty 之 任务队列taskQueue

### 任务队列

**1、** 用户程序自定义的普通任务；

**2、** 用户自定义定时任务；

**3、** 非当前Reactor线程调用Channel的各种方法；

例如在推送系统的业务线程里面，根据用户的标识，找到对应的Channel引用，然后调用Write类方法向该用户推送消息，就会进入到这种场景。最终的Write会提交到任务队列中后被异步消费。

### 使用场景

**1、** 比如在服务器端channelRead中有一个非常耗费时间的业务，我们要异步执行，把它提交到channel对应的NioEventLoopGroup的taskQueue中；

**2、** 每个NioEventLoop是一个单线程线程池，提交任务相当于还是它自己来做，只不过是它会根据你设定的ioradio参数来分配io事件和普通任务的时间；

### 方案1：用户程序自定义的普通任务

#### 服务端Handler

```java
/**
 * 说明
 * 1. 我们自定义一个Handler，需要继承netty规定好的某个HandlerAdapter（规范）
 * 2. 这时我们自定义一个Handler，才能称之为Handler
 *
 */
public class NettyChannelHandler2 extends ChannelInboundHandlerAdapter {

	//读取数据的事件（这里我们可以读取客户端发送的消息）
	/*
	 * 1. ChannelHandlerContext ctx：上下文对象，含有管道pipeline，通道channel，地址
	 * 2. Object msg：就是客户端发送的数据，默认是Object
	 * 3. 通道读写数据，管道处理数据
	 */
	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

		//方案1：用户程序自定义的普通任务
		//会提交到当前channel关联的NioEventLoop里面的taskQueue执行
		//任务一
		ctx.channel().eventLoop().execute(() -> {
            try {
                Thread.sleep(5 * 1000);
                ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端1~ "+ LocalDateTime.now(), CharsetUtil.UTF_8));
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }

        });

		//任务二
		ctx.channel().eventLoop().execute(() -> {
            System.out.println("任务二...");
            ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端3~ "+ LocalDateTime.now(), CharsetUtil.UTF_8));

        });

		ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端2~ "+ LocalDateTime.now(), CharsetUtil.UTF_8));

		//客户端
		//会收到：hello，客户端2~
		//再收到：hello，客户端~
		//再收到：hello，客户端3~

	}

	//数据读取完毕
	//这个方法会在channelRead读完后触发
	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		//把数据写到缓冲区，并且刷新缓冲区，是write + flush
		//一般来讲，我们对这个发送的数据进行编码
		//ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端~", CharsetUtil.UTF_8));

	}

	//处理异常，一般是需要关闭通道
	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
		ctx.channel().close();
	}
}


```

#### 客户端

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

#### 客户端执行结果

```java
服务器回复的消息：hello，客户端2~ 2024-11-24T22:52:40.528
服务器的地址： /127.0.0.1:6668
服务器回复的消息：hello，客户端1~ 2024-11-24T22:52:45.534 //间隔5秒
服务器的地址： /127.0.0.1:6668
服务器回复的消息：hello，客户端3~ 2024-11-24T22:52:45.534 //间隔0秒
服务器的地址： /127.0.0.1:6668
```

服务器端nioEventLoop还是一个线程执行，taskQueue里是按照添加的顺序依次执行

### 方案2：用户自定义定时任务

#### 服务端Handler

```JAVA
/**
 * 说明
 * 1. 我们自定义一个Handler，需要继承netty规定好的某个HandlerAdapter（规范）
 * 2. 这时我们自定义一个Handler，才能称之为Handler
 */
public class NettyChannelHandler3 extends ChannelInboundHandlerAdapter {

    //读取数据的事件（这里我们可以读取客户端发送的消息）
    /*
     * 1. ChannelHandlerContext ctx：上下文对象，含有管道pipeline，通道channel，地址
     * 2. Object msg：就是客户端发送的数据，默认是Object
     * 3. 通道读写数据，管道处理数据
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

       //方案2：用户自定义定时任务
       ctx.channel().eventLoop().schedule(() -> {
            try {
                Thread.sleep(2 * 1000);
                ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端~ "+LocalDateTime.now(), CharsetUtil.UTF_8));
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }

        }, 1, TimeUnit.SECONDS); //延迟5秒，然后执行
        //方案2：用户自定义定时任务
       ctx.channel().eventLoop().schedule(() -> {
            try {
                Thread.sleep(3 * 1000);
                ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端3~ "+LocalDateTime.now(), CharsetUtil.UTF_8));
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }

        }, 5, TimeUnit.SECONDS); //延迟5秒，然后执行
       ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端2~ "+ LocalDateTime.now(), CharsetUtil.UTF_8));

    }

    //数据读取完毕
    //这个方法会在channelRead读完后触发
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
       //把数据写到缓冲区，并且刷新缓冲区，是write + flush
       //一般来讲，我们对这个发送的数据进行编码
       //ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端~", CharsetUtil.UTF_8));

    }

    //处理异常，一般是需要关闭通道
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
       ctx.channel().close();
    }
}
```

#### 客户端执行结果

```JAVA
服务器回复的消息：hello，客户端2~ 2024-11-24T23:01:37.313
服务器的地址： /127.0.0.1:6668
服务器回复的消息：hello，客户端~ 2024-11-24T23:01:40.307 //间隔1+2=3秒
服务器的地址： /127.0.0.1:6668
服务器回复的消息：hello，客户端3~ 2024-11-24T23:01:45.308 //间隔5+3=8秒
服务器的地址： /127.0.0.1:6668
```

该任务是提交到scheduleTaskQueue中，并行

### 方案3：服务器端要推送多个管道

#### 服务端Server

```java
/**
 * 可以传递一个集合保存SocketChannel的引用
 * @author user
 *
 */
public class NettyServer2 {
	public static void main(String[] args) throws Exception {
		
		//创建BossGroup和WorkerGroup
		//说明
		//1. 创建两个线程组bossGroup和workerGroup
		//2. bossGroup它只是处理连接请求，真正的与客户端业务处理会交给workerGroup去完成
		//3. 两个都是无限循环
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup(8);
		
		try {
			//创建服务器端的启动对象，配置启动参数
			ServerBootstrap bootstrap = new ServerBootstrap();
			
			//集合保存所有SocketChannel引用
			Map<Integer, SocketChannel> map = new ConcurrentHashMap<>();
			
			//使用链式编程来进行设置
			bootstrap.group(bossGroup, workerGroup) //设置两个线程组
				.channel(NioServerSocketChannel.class) //使用NioServerSocketChannel作为服务器的通道实现
				.childHandler(new ChannelInitializer<SocketChannel>() { //创建一个通道初始化对象
					//给pipeline设置处理器
					@Override
					protected void initChannel(SocketChannel ch) throws Exception {
						
						//将channel引用放入map
						map.put(ch.hashCode(), ch);
						
						ChannelPipeline pipeline = ch.pipeline();
						pipeline.addLast(new NettyChannelHandler4(map)); //向管道的最后增加一个处理器
						
					};
				}); //给我们的workerGroup的EventLoop对应的管道设置处理器
			
			//bossGroup参数
			bootstrap.option(ChannelOption.SO_BACKLOG, 1024); //设置线程队列等待连接的个数
			
			//workerGroup参数
			bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true); //设置保持活动连接状态
			
			System.out.println("...服务器 is ready...");
			
			//绑定一个端口并且同步，生成了一个ChannelFuture对象
			//启动服务器并绑定端口
			ChannelFuture cf = bootstrap.bind(6668).sync();
			
			//对关闭通道进行监听
			cf.channel().closeFuture().sync();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			System.out.println("Shutdown Netty Server...");
			//优雅的关闭
			workerGroup.shutdownGracefully();
			bossGroup.shutdownGracefully();
			System.out.println("Shutdown Netty Server Success!");
		}
		
	}
}
```

#### 服务端Handler

```java
/**
 * 说明
 * 1. 我们自定义一个Handler，需要继承netty规定好的某个HandlerAdapter（规范）
 * 2. 这时我们自定义一个Handler，才能称之为Handler
 *
 */
public class NettyChannelHandler4 extends ChannelInboundHandlerAdapter {

    private Map<Integer, SocketChannel> map;

    public NettyChannelHandler4(Map<Integer, SocketChannel> map) {
       this.map = map;
    }

    //读取数据的事件（这里我们可以读取客户端发送的消息）
    /*
     * 1. ChannelHandlerContext ctx：上下文对象，含有管道pipeline，通道channel，地址
     * 2. Object msg：就是客户端发送的数据，默认是Object
     * 3. 通道读写数据，管道处理数据
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

       System.err.println("channel的数量：" + map.size());

       //方案3：服务器端要推送到管道A、管道B、管道C。。。
       map.forEach((key, value) -> {
            value.writeAndFlush(Unpooled.copiedBuffer("server向"+key+"发送消息", CharsetUtil.UTF_8));
        });
    }

    //数据读取完毕
    //这个方法会在channelRead读完后触发
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
       //把数据写到缓冲区，并且刷新缓冲区，是write + flush
       //一般来讲，我们对这个发送的数据进行编码
       //ctx.channel().writeAndFlush(Unpooled.copiedBuffer("hello，客户端~", CharsetUtil.UTF_8));

    }

    //处理异常，一般是需要关闭通道
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
       //客户端主动断开连接时会进到这里

       //移除集合
       map.remove(ctx.channel().hashCode());
       ctx.channel().close();
    }
}
```

#### 客户端执行结果

启动2个客户端

```java
channel的数量：1
channel的数量：2

服务器回复的消息：server向-1696812958发送消息
服务器的地址： /127.0.0.1:6668
服务器回复的消息：server向-1696812958发送消息
服务器的地址： /127.0.0.1:6668
```

### netty模型方案再说明

**1、** netty抽象出两组线程池，BossGroup专门负责接收客户端连接，WorkerGroup专门负责网络读写操作；

**2、** NioEventLoop表示一个不断循环执行处理任务的线程，每个NioEventLoop都有一个selector，用于监听绑定在其上的socket网络通道；

**3、** NioEventLoop内部采用串行化设计，`从消息的读取->解码->处理->编码->发送`，始终由IO线程NioEventLoop负责；

**4、** NioEventLoopGroup下包含多个NioEventLoop；

- 每个NioEventLoop中包含有一个Selector，一个taskQueue。

- 每个NioEventLoop的Selector上可以注册监听多个NioChannel。

- 每个NioChannel只会绑定在唯一的NioEventLoop上。

- 每个NioChannel都绑定有一个自己的ChannelPipeline。




## Netty 之 异步模型

### Netty异步模型介绍

**1、** 异步的概念和同步相对当一个异步过程调用发出后，调用者不能立刻得到结果实际处理这个调用的组件在完成后，通过状态、通知和回调来通知调用者；

**2、** Netty中的I**/O操作**是异步的，包括**Bind、Write、Connect**等操作会简单的返回一个**ChannelFuture**；

**3、** 调用者并不能立刻获得结果，而是通过**Future-Listener机制**，用户可以方便的**主动获取**或者通过**通知机制**获得IO操作结果；

**4、** Netty的异步模型是建立在**future**和**callback**的之上的；

- callback就是回调。
- 重点说Future，它的核心思想是：假设一个方法 fun，计算过程可能非常耗时，等待 fun 返回显然不合适。那么可以在调用 fun 的时候，立马返回一个 Future，后续可以通过 Future去监控方法 fun 的处理过程（即：Future-Listener机制）

### Future说明

**1、** 表示异步的执行结果，可以通过它提供的方法来检测执行是否完成，比如检索计算等等；

**2、** ChannelFuture是一个接口：publicinterfaceChannelFutureextendsFuture，我们可以添加监听器，当监听的事件发生时，就会通知到监听器；

工作原理示意图，如下所示。

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556266.png)

![ ](https://gcore.jsdelivr.net/gh/PurezentoW/PicGo/img/202411271556913.png)

**说明：**

1、在使用Netty进行编程时，拦截操作和转换出入站数据只需要您提供 callback 或利用 future 即可。这使得**链式操作**简单、高效，并有利于编写可重用的、通用的代码。

2、Netty框架的目标就是让你的业务逻辑从网络基础应用编码中分离出来、解脱出来。

### Future-Listener机制

**1、** 当Future对象刚刚创建时，处于非完成状态，调用者可以通过返回的ChannelFuture来获取操作执行的状态，注册监听函数来执行完成后的操作；

**2、** 常见有如下操作：

- 通过 isDone 方法来判断当前操作是否完成；
- 通过 isSuccess 方法来判断已完成的当前操作是否完成；
- 通过 getCause 方法来获取已完成的当前操作失败的原因；
- 通过 isCancelled 方法来判断已完成的当前操作是否被取消；
- 通过 addListener 方法来注册监听器，当操作已完成（isDone 方法返回完成），将会通知指定的监听器；如果 Future 对象已完成，则通知指定的监听器。

举例说明

演示：绑定端口是异步操作，当绑定操作处理完，将会调用相应的监听器处理逻辑

```java
// 绑定一个端口，并且同步，生成一个ChannelFuture对象
// 启动服务器
ChannelFuture cf = bootstrap.bind(6668).sync();

// 给 cf 注册监听器，监控我们关心的事件
cf.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        if (cf.isSuccess()) {
            System.out.println("监听端口 6668 成功");
        } else {
            System.out.println("监听端口 6668 失败");
        }
    }
});
```

**小结：**相比传统阻塞 I/O，执行 I/O 操作后线程会被阻塞住，直到操作完成；异步处理的好处是不会造成线程阻塞，线程在 I/O 操作期间可以执行别的程序，在高并发情形下会更稳定和更高的吞吐量。



## Netty 之 HTTP服务

- 步骤一：创建两个线程组 BossGroup 和 WorkerGroup，他们的类型都是 NioEventLoopGroup。bossGroup 只处理连接请求，workerGroup 处理客户端业务。
- 步骤二：创建服务器端启动对象 ServerBootstrap ，并进行参数配置：
- 设置 BossGroup 和 WorkerGroup
- 设置使用 NioSocketChannel 作为服务器的通道实现
- 设置保持活动连接
- 创建一个 通道(pipline) 测试对象（匿名对象），并为 pipline设置一个 Handler
- 步骤三：绑定端口并且同步，启动服务器，生成并返回一个 ChannelFuture 对象
- 步骤四：设置对关闭通道事件进行监听

### Http服务端

```java
/**
 * HttpServer类用于启动一个基于Netty的HTTP服务器
 */
public class HttpServer {
    /**
     * 程序的入口点
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 创建一个EventLoopGroup用于接收客户端连接
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // 创建一个EventLoopGroup用于处理网络通信
        EventLoopGroup workerGroup = new NioEventLoopGroup(8);

        try {
            // 初始化服务器配置
            ServerBootstrap bootstrap = new ServerBootstrap();
            // 配置服务器的EventLoopGroup、通道和处理器
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new HttpServerInitializer());

            // 绑定端口并启动服务器
            ChannelFuture channelFuture = bootstrap.bind(7000).sync();
            // 等待服务器端口关闭
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            // 打印异常信息
            e.printStackTrace();
        } finally {
            // 关闭所有EventLoopGroup
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

### 启动类

```java
public class HttpServerInitializer extends ChannelInitializer<SocketChannel>{

	@Override
	protected void initChannel(SocketChannel ch) throws Exception {
		
		//向管道加入处理器
		
		//得到管道
		ChannelPipeline pipeline = ch.pipeline();
		
		//加入一个netty提供的httpServerCodec（编解码器）
		//HttpServerCodec的说明
		//1. HttpServerCodec是netty提供的处理http的编解码器
		pipeline.addLast("MyHttpServerCodec", new HttpServerCodec());
		
		//增加一个自定义的Handler
		pipeline.addLast("MyHttpServerHandler", new HttpServerHandler());
		
	}

}
```

### 服务端Handler

```java
public class HttpServerHandler extends SimpleChannelInboundHandler<HttpObject> {

    //channelRead0：读取客户端数据
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {

       //判断msg是不是HttpRequest请求
       if (msg instanceof HttpRequest) {
          System.out.println("msg 类型 = " + msg.getClass());
          System.out.println("客户端地址 = " + ctx.channel().remoteAddress());

            //过滤信息
          HttpRequest httpRequest = (HttpRequest) msg;
            //获取uri
          URI uri = new URI(httpRequest.uri());
          if ("/favicon.ico".equals(uri.getPath())) {
             System.out.println("请求了favicon.ico，不做响应");
             return;
          }


          //回复信息给浏览器 [http协议]
          ByteBuf content = Unpooled.copiedBuffer("hello，我是服务器！", CharsetUtil.UTF_8);
          //构造一个http的响应，即httpResponse
          FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK, content);

          response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain;charset=utf-8");
          response.headers().set(HttpHeaderNames.CONTENT_LENGTH, content.readableBytes());

          //将构建好的response返回
          ctx.writeAndFlush(response);
       }
    }

}
```



### 测试结果问题分析

- 首先启动程序
- 浏览器发送请求 http://localhost:7000/
- 查看控制台打印结果，我们发现服务端接收到了两次请求

```java
msg 类型 = class io.netty.handler.codec.http.DefaultHttpRequest
客户端地址 = /0:0:0:0:0:0:0:1:50957
msg 类型 = class io.netty.handler.codec.http.DefaultHttpRequest
客户端地址 = /0:0:0:0:0:0:0:1:50957
请求了favicon.ico，不做响应
```




