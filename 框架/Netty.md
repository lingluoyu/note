## Netty

Netty 是一个利用 Java 的高级网络的能力，隐藏其背后的复杂性而提供一个易于使用的 API 的客户端/服务器框架。Netty 提供高性能和可扩展性，让你可以自由地专注于你真正感兴趣的东西，构建你的独特的应用！

### Netty核心模块

#### **ServerBootStrap和BootStrap**

ServerBootStrap是Netty创建服务器的辅助类，负责封装服务启动的一系列操作。和ServerBootStrap一样，Bootstrap也是封装客户端向服务端发送请求的一系列操作。

##### BootStrap常用方法

| 名称                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| group                  | 设置 EventLoopGroup 用于处理所有的 Channel 的事件            |
| channel channelFactory | channel() 指定 Channel 的实现类。如果类没有提供一个默认的构造函数,你可以调用 channelFactory() 来指定一个工厂类被 bind() 调用。 |
| localAddress           | 指定应该绑定到本地地址 Channel。如果不提供,将由操作系统创建一个随机的。或者,您可以使用 bind() 或 connect()指定localAddress |
| option                 | 设置 ChannelOption 应用于 新创建 Channel 的 ChannelConfig。这些选项将被 bind 或 connect 设置在通道,这取决于哪个被首先调用。这个方法在创建管道后没有影响。所支持 ChannelOption 取决于使用的管道类型。请参考9.6节和 ChannelConfig 的 API 文档 的 Channel 类型使用。 |
| attr                   | 这些选项将被 bind 或 connect 设置在通道,这取决于哪个被首先调用。这个方法在创建管道后没有影响。请参考9.6节。 |
| handler                | 设置添加到 ChannelPipeline 中的 ChannelHandler 接收事件通知。 |
| clone                  | 创建一个当前 Bootstrap的克隆拥有原来相同的设置。             |
| remoteAddress          | 设置远程地址。此外,您可以通过 connect() 指定                 |
| connect                | 连接到远端，返回一个 ChannelFuture, 用于通知连接操作完成     |
| bind                   | 将通道绑定并返回一个 ChannelFuture,用于通知绑定操作完成后,必须调用 Channel.connect() 来建立连接。 |

#### EventLoop

EventLoop定义了Netty的核心抽象，用于处理连接的生命周期中所发生的事件。EventLoop是协调设计的一部分，采用了两个基本的API：并发和网络编程。

#### Channel

[Channel](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/Channel.html) 是 NIO 基本的结构。它代表了一个用于连接到实体如硬件设备、文件、网络套接字或程序组件,能够执行一个或多个不同的 I/O 操作（例如读或写）的开放连接。

##### Channel生命周期

| 状态                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| channelUnregistered | channel创建但未注册到一个 EventLoop.                         |
| channelRegistered   | channel 注册到一个 EventLoop.                                |
| channelActive       | channel 的活动的(连接到了它的 remote peer（远程对等方）)，现在可以接收和发送数据了 |
| channelInactive     | channel 没有连接到 remote peer（远程对等方）                 |

#### Callback (回调)

callback (回调)是一个简单的方法,提供给另一种方法作为引用,这样后者就可以在某个合适的时间调用前者。这种技术被广泛使用在各种编程的情况下,最常见的方法之一通知给其他人操作已完成。

Netty 内部使用回调处理事件时。一旦这样的回调被触发，事件可以由接口 ChannelHandler 的实现来处理。

#### Future

Future 提供了另外一种通知应用操作已经完成的方式。这个对象作为一个异步操作结果的占位符,它将在将来的某个时候完成并提供结果。Netty中的所有的IO操作都是异步的，因为一个操作可能不会立即返回，所以我们需要一种用于在之后某个时间点确定其结果的方法。为此，Netty提供了ChannelFuture接口，其addListener()方法注册了一个ChannelFutureListener，以便在某个操作完成是得到通知。

#### ChannelHandler

从应用程序开发人员的角度来看，ChannelHandler是Netty的主要组件，它充当了所有处理入站和出站数据的应用程序逻辑的容器，因为ChannelHandler的方法是由事件来触发的。

##### ChannelHandler生命周期

| 类型            | 描述                                            |
| --------------- | ----------------------------------------------- |
| handlerAdded    | 当 ChannelHandler 添加到 ChannelPipeline 调用   |
| handlerRemoved  | 当 ChannelHandler 从 ChannelPipeline 移除时调用 |
| exceptionCaught | 当 ChannelPipeline 执行发生错误时调用           |

Netty 提供2个重要的 ChannelHandler 子接口：

- ChannelInboundHandler - 处理进站数据，并且所有状态都更改
- ChannelOutboundHandler - 处理出站数据，允许拦截各种操作

##### ChannelInboundHandler

| 类型                      | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| channelRegistered         | Invoked when a Channel is registered to its EventLoop and is able to handle I/O. |
| channelUnregistered       | Invoked when a Channel is deregistered from its EventLoop and cannot handle any I/O. |
| channelActive             | Invoked when a Channel is active; the Channel is connected/bound and ready. |
| channelInactive           | Invoked when a Channel leaves active state and is no longer connected to its remote peer. |
| channelReadComplete       | Invoked when a read operation on the Channel has completed.  |
| channelRead               | Invoked if data are read from the Channel.                   |
| channelWritabilityChanged | Invoked when the writability state of the Channel changes. The user can ensure writes are not done too fast (with risk of an OutOfMemoryError) or can resume writes when the Channel becomes writable again.Channel.isWritable() can be used to detect the actual writability of the channel. The threshold for writability can be set via Channel.config().setWriteHighWaterMark() and Channel.config().setWriteLowWaterMark(). |
| userEventTriggered(...)   | Invoked when a user calls Channel.fireUserEventTriggered(...) to pass a pojo through the ChannelPipeline. This can be used to pass user specific events through the ChannelPipeline and so allow handling those events. |

##### ChannelOutboundHandler

| 类型       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| bind       | Invoked on request to bind the Channel to a local address    |
| connect    | Invoked on request to connect the Channel to the remote peer |
| disconnect | Invoked on request to disconnect the Channel from the remote peer |
| close      | Invoked on request to close the Channel                      |
| deregister | Invoked on request to deregister the Channel from its EventLoop |
| read       | Invoked on request to read more data from the Channel        |
| flush      | Invoked on request to flush queued data to the remote peer through the Channel |
| write      | Invoked on request to write data through the Channel to the remote peer |

#### ChannelPipeline

ChannelPipeline为ChannelHandler链提供了容器，并定义了用于该链上传播入站和出站事件流的API。当Channel被创建时，它会被自动的分配到它所专属的ChannelPipeline。

| 名称                                | 描述                                           |
| ----------------------------------- | ---------------------------------------------- |
| addFirst addBefore addAfter addLast | 添加 ChannelHandler 到 ChannelPipeline.        |
| Remove                              | 从 ChannelPipeline 移除 ChannelHandler.        |
| Replace                             | 在 ChannelPipeline 替换另外一个 ChannelHandler |

#### ByteBuf

网络数据的基本单位是字节，NIO提供了ByteBuffer作为网络数据的字节容器，但是ByteBuffer本身设计并不优雅，使用繁琐，Netty使用ByteBuf来替代ByteBuffer，在Netty提供的内置编解码器StringDecoder/StringEncoder中，操作的对象就是ByteBuf。

#### Codec

通过Netty发送和接收一个消息的时候，就会发生一次数据转换，入站消息会被解码，也就是从字节转换为原本的形式，如果是出站消息，就会从一种形式变成字节，这个就是编码，编解码的根本原因就是因为网络数据就是一系列的字节。

### Echo服务器

- 一个服务器 handler：这个组件实现了服务器的业务逻辑，决定了连接创建后和接收到信息后该如何处理
- Bootstrapping： 这个是配置服务器的启动代码。最少需要设置服务器绑定的端口，用来监听连接请求。

通过覆盖`ChannelInboundHandlerAdapter`定义处理入站时间的方法

```java
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.write(msg);
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        //抛出异常时关闭连接
        cause.printStackTrace();
        ctx.close();
    }
}
```

创建服务器

```java
public class EchoServer {
    private int port;

    public EchoServer(int port) {
        this.port = port;
    }
    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        }

        new EchoServer(port).run();		// 1
    }
    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();		// 2
        EventLoopGroup workerGroup = new NioEventLoopGroup();	// 3
        try {
            ServerBootstrap b = new ServerBootstrap();		// 4
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)		// 5
                    .childHandler(new ChannelInitializer<SocketChannel>() { // 6
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new DiscardServerHandler());
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // 绑定并接收传入的连接
            ChannelFuture f = b.bind(port).sync();	// 7

            // 关闭服务
            f.channel().closeFuture().sync();	// 8
        } finally {
            workerGroup.shutdownGracefully();	// 9
            bossGroup.shutdownGracefully();
        }
    }
}
```

1. 调用服务器run()方法
2. 创建bossGroup，用于接收连接
3. 创建workerGroup，用于处理连接；一旦‘boss’接收到连接，就会把连接信息注册到‘worker’上
4. 创建ServerBootstrap，启动 NIO 服务的辅助启动类
5. 使用 NIO 的传输 Channel
6. 添加 EchoServerHandler 到 Channel 的 ChannelPipeline
7. 绑定服务器（调用 sync() 的原因是当前线程阻塞）
8. 等待关闭 channel
9. 关闭bossGroup和workerGroup

### 聊天功能

#### 服务端

##### SimpleChatServerHandler.java

```java
public class SimpleChatServerHandler extends SimpleChannelInboundHandler<String> {
    public static ChannelGroup channels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {  // (2)
        Channel incoming = ctx.channel();

        // Broadcast a message to multiple Channels
        channels.writeAndFlush("[SERVER] - " + incoming.remoteAddress() + " 加入\n");

        channels.add(ctx.channel());
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {  // (3)
        Channel incoming = ctx.channel();

        // Broadcast a message to multiple Channels
        channels.writeAndFlush("[SERVER] - " + incoming.remoteAddress() + " 离开\n");

        // A closed Channel is automatically removed from ChannelGroup,
        // so there is no need to do "channels.remove(ctx.channel());"
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String s) throws Exception { // (4)
        Channel incoming = ctx.channel();
        for (Channel channel : channels) {
            if (channel != incoming) {
                channel.writeAndFlush("[" + incoming.remoteAddress() + "]" + s + "\n");
            } else {
                channel.writeAndFlush("[you]" + s + "\n");
            }
        }
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception { // (5)
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:" + incoming.remoteAddress() + "在线");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception { // (6)
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:" + incoming.remoteAddress() + "掉线");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (7)
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:" + incoming.remoteAddress() + "异常");
        // 当出现异常就关闭连接
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. SimpleChatServerHandler 继承自 [SimpleChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/SimpleChannelInboundHandler.html)，这个类实现了 [ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html) 接口，ChannelInboundHandler 提供了许多事件处理的接口方法，然后你可以覆盖这些方法。现在仅仅只需要继承 SimpleChannelInboundHandler 类而不是你自己去实现接口方法。
2. 覆盖了 handlerAdded() 事件处理方法。每当从服务端收到新的客户端连接时，客户端的 Channel 存入 [ChannelGroup](http://netty.io/4.0/api/io/netty/channel/group/ChannelGroup.html) 列表中，并通知列表中的其他客户端 Channel
3. 覆盖了 handlerRemoved() 事件处理方法。每当从服务端收到客户端断开时，客户端的 Channel 自动从 ChannelGroup 列表中移除了，并通知列表中的其他客户端 Channel
4. 覆盖了 channelRead0() 事件处理方法。每当从服务端读到客户端写入信息时，将信息转发给其他客户端的 Channel。其中如果你使用的是 Netty 5.x 版本时，需要把 channelRead0() 重命名为messageReceived()
5. 覆盖了 channelActive() 事件处理方法。服务端监听到客户端活动
6. 覆盖了 channelInactive() 事件处理方法。服务端监听到客户端不活动
7. exceptionCaught() 事件处理方法是当出现 Throwable 对象才会被调用，即当 Netty 由于 IO 错误或者处理器在处理事件时抛出的异常时。在大部分情况下，捕获的异常应该被记录下来并且把关联的 channel 给关闭掉。然而这个方法的处理方式会在遇到不同异常的情况下有不同的实现，比如你可能想在关闭连接之前发送一个错误码的响应消息。

##### SimpleChatServerInitializer.java

SimpleChatServerInitializer 用来添加多个handler到 ChannelPipeline 上，包括编码、解码、SimpleChatServerHandler 等。

```java
public class SimpleChatServerInitializer extends
		ChannelInitializer<SocketChannel> {

	@Override
    public void initChannel(SocketChannel ch) throws Exception {
		 ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast("framer", new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));
        pipeline.addLast("decoder", new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());
        pipeline.addLast("handler", new SimpleChatServerHandler());
 
		System.out.println("SimpleChatClient:"+ch.remoteAddress() +"连接上");
    }
}
```

##### SimpleChatServer.java

编写一个 main() 方法来启动服务端。

```java
public class SimpleChatServer {
    private int port;

    public SimpleChatServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class) // (3)
                    .childHandler(new SimpleChatServerInitializer())  //(4)
                    .option(ChannelOption.SO_BACKLOG, 128)          // (5)
                    .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)

            System.out.println("SimpleChatServer 启动了");

            // 绑定端口，开始接收进来的连接
            ChannelFuture f = b.bind(port).sync(); // (7)

            // 等待服务器  socket 关闭 。
            // 在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
            f.channel().closeFuture().sync();

        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();

            System.out.println("SimpleChatServer 关闭了");
        }
    }

    public static void main(String[] args) throws Exception {
        int port;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        } else {
            port = 8080;
        }
        new SimpleChatServer(port).run();

    }
}
```

1. [NioEventLoopGroup](http://netty.io/4.0/api/io/netty/channel/nio/NioEventLoopGroup.html) 是用来处理I/O操作的多线程事件循环器，Netty 提供了许多不同的 [EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html) 的实现用来处理不同的传输。在这个例子中我们实现了一个服务端的应用，因此会有2个 NioEventLoopGroup 会被使用。第一个经常被叫做‘boss’，用来接收进来的连接。第二个经常被叫做‘worker’，用来处理已经被接收的连接，一旦‘boss’接收到连接，就会把连接信息注册到‘worker’上。如何知道多少个线程已经被使用，如何映射到已经创建的 [Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)上都需要依赖于 EventLoopGroup 的实现，并且可以通过构造函数来配置他们的关系。
2. [ServerBootstrap](http://netty.io/4.0/api/io/netty/bootstrap/ServerBootstrap.html) 是一个启动 NIO 服务的辅助启动类。你可以在这个服务中直接使用 Channel，但是这会是一个复杂的处理过程，在很多情况下你并不需要这样做。
3. 这里我们指定使用 [NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html) 类来举例说明一个新的 Channel 如何接收进来的连接。
4. 这里的事件处理类经常会被用来处理一个最近的已经接收的 Channel。SimpleChatServerInitializer 继承自[ChannelInitializer](http://netty.io/4.0/api/io/netty/channel/ChannelInitializer.html) 是一个特殊的处理类，他的目的是帮助使用者配置一个新的 Channel。也许你想通过增加一些处理类比如 SimpleChatServerHandler 来配置一个新的 Channel 或者其对应的[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) 来实现你的网络程序。当你的程序变的复杂时，可能你会增加更多的处理类到 pipline 上，然后提取这些匿名类到最顶层的类上。
5. 你可以设置这里指定的 Channel 实现的配置参数。我们正在写一个TCP/IP 的服务端，因此我们被允许设置 socket 的参数选项比如tcpNoDelay 和 keepAlive。请参考 [ChannelOption](http://netty.io/4.0/api/io/netty/channel/ChannelOption.html) 和详细的 [ChannelConfig](http://netty.io/4.0/api/io/netty/channel/ChannelConfig.html) 实现的接口文档以此可以对ChannelOption 的有一个大概的认识。
6. option() 是提供给[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html) 用来接收进来的连接。childOption() 是提供给由父管道 [ServerChannel](http://netty.io/4.0/api/io/netty/channel/ServerChannel.html) 接收到的连接，在这个例子中也是 NioServerSocketChannel。
7. 我们继续，剩下的就是绑定端口然后启动服务。这里我们在机器上绑定了机器所有网卡上的 8080 端口。当然现在你可以多次调用 bind() 方法(基于不同绑定地址)。

#### 客户端

##### SimpleChatClientHandler.java

将读到的信息打印出来

```java
public class SimpleChatClientHandler extends  SimpleChannelInboundHandler<String> {
	@Override
	protected void channelRead0(ChannelHandlerContext ctx, String s) throws Exception {
		System.out.println(s);
	}
}
```

##### SimpleChatClientInitializer.java

```java
public class SimpleChatClientInitializer extends ChannelInitializer<SocketChannel> {
 
	@Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        
        pipeline.addLast("framer", new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));
        pipeline.addLast("decoder", new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());
        pipeline.addLast("handler", new SimpleChatClientHandler());
    }
}
```

##### SimpleChatClient.java

编写一个 main() 方法来启动客户端。

```java
public class SimpleChatClient {
	public static void main(String[] args) throws Exception{
	        new SimpleChatClient("localhost", 8080).run();
	    }
	
	    private final String host;
	    private final int port;
	
	    public SimpleChatClient(String host, int port){
	        this.host = host;
	        this.port = port;
	    }
	
	    public void run() throws Exception{
	        EventLoopGroup group = new NioEventLoopGroup();
	        try {
	            Bootstrap bootstrap  = new Bootstrap()
	                    .group(group)
	                    .channel(NioSocketChannel.class)
	                    .handler(new SimpleChatClientInitializer());
	            Channel channel = bootstrap.connect(host, port).sync().channel();
	            BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
	            while(true){
	                channel.writeAndFlush(in.readLine() + "\r\n");
	            }
	        } catch (Exception e) {
	            e.printStackTrace();
	        } finally {
	            group.shutdownGracefully();
	        }
	
	    }
    }
}
```

