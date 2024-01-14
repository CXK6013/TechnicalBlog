# 用Java的BIO和NIO、Netty来实现HTTP服务器(三)  用Netty实现

[TOC]

##  用Netty来重构

《Netty学习笔记(一)初遇篇》已经基本讲过Netty了，这里我们再讲一遍，首先Netty是啥?  

> Netty is an NIO client server framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.
>
> Netty是一个 NIO的客户端、服务端框架，能够让你快速、简单的开发网络应用。它极大的简化了网络编程，如TCP、UDP服务器
>
> 'Quick and easy' doesn't mean that a resulting application will suffer from a maintainability or a performance issue. Netty has been designed carefully with the experiences earned from the implementation of a lot of protocols such as FTP, SMTP, HTTP, and various binary and text-based legacy protocols. As a result, Netty has succeeded to find a way to achieve ease of development, performance, stability, and flexibility without a compromise.
>
> 快速和简单不意味着开发出来的应用程序会有可维护性和性能问题，Netty在设计的时候从一些网络协议中吸取了大量的经验。最终，Netty成功的找到了一种方法，在开发简易性、性能、灵活性都不打折扣。

我提取到关键词简单、易用、性能强，好那现在怎么用，在用Netty的时候我们需要引入依赖:

```java
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.104.Final</version>
</dependency>
```

上代码:

```java
public class NettyHttpServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", SSL? "8443" : "8080"));

    public static void main(String[] args) throws Exception {
        // boosGroup 只处理连接,所以这里我们给了一个
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // 这个是真正处理读写请求的线程组
        EventLoopGroup workerGroup = new NioEventLoopGroup(Runtime.getRuntime().availableProcessors() * 2);
        try {         
            ServerBootstrap b = new ServerBootstrap();
            // 设置全连接队列数量
            b.option(ChannelOption.SO_BACKLOG, 1024);
            // 模板代码,设置日志级别,处理器
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)               
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new HttpHelloWorldServerInitializer());
            // 绑定端口
            Channel ch = b.bind(PORT).sync().channel();
            System.err.println("Open your web browser and navigate to " +
                    (SSL? "https" : "http") + "://127.0.0.1:" + PORT + '/');	
            ch.closeFuture().sync();
        } finally {
            // 关闭线程池
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

```

```java
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    public void initChannel(SocketChannel socketChannel) {
        ChannelPipeline p = socketChannel.pipeline();
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpContentCompressor((CompressionOptions[]) null));
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }
} 
```

```java
public class HttpHelloWorldServerHandler  extends SimpleChannelInboundHandler<HttpObject> {
    private static final byte[] CONTENT = { 'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd' };

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        super.channelRegistered(ctx);
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            boolean keepAlive = HttpUtil.isKeepAlive(req);
            FullHttpResponse response = new DefaultFullHttpResponse(req.protocolVersion(), OK,
                    Unpooled.wrappedBuffer(CONTENT));
            response.headers()
                    .set(CONTENT_TYPE, TEXT_PLAIN)
                    .setInt(CONTENT_LENGTH, response.content().readableBytes());

            if (keepAlive) {
                if (!req.protocolVersion().isKeepAliveDefault()) {
                    response.headers().set(CONNECTION, KEEP_ALIVE);
                }
            } else {
                // Tell the client we're going to close the connection.
                response.headers().set(CONNECTION, CLOSE);
            }
            ChannelFuture f = ctx.write(response);
            if (!keepAlive) {
                f.addListener(ChannelFutureListener.CLOSE);
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

在Netty的世界里面是一个又一个多处理器，在启动的时候Netty会感知哪些处理器，在对应的事件触发之后就会触发对应的处理，像是一道链条一样:

![](https://a.a2k6.com/gerald/i/2024/01/01/m0il.jpg)

pipeline 是一个流水线，可以在这个上面不断的添加处理器，消息流经过后会被这些处理器处理，HttpServerCode实现对HTTP请求的解码和响应的编码，HttpContentCompressor实现对响应内容的压缩。 在HTTP中有一个独特的功能叫做，100 (Continue) Status，就是说client在不确定server端是否会接收请求的时候，可以先发送一个请求头，并在这个头上加一个"100-continue"字段，但是暂时还不发送请求body。直到接收到服务器端的响应之后再发送请求body。HttpServerExpectContinueHandler用于处理这个请求，消息经过HttpServerCodec、HttpContentCompressor、HttpServerExpectContinueHandler之后到达HttpHelloWorldServerHandler, 也就是我们的HttpHelloWorldServerHandler。

上面我们轻飘飘的说了HttpServerCodec的作用是对请求进行解码，对响应进行编码，那我们添加进去的处理器究竟是如何被发挥作用的呢?  首先看我们通过流水线添加进去的处理器，都被添加到了哪里?

## 问题一 initChannel如何被触发？

我们的HttpHelloWorldServerInitializer的initChannel来自于ChannelInitializer，那这个方法我理解就应该是扩展方法，被父类的某个方法所调用 ，在ChannelInitializer我们重写的initChannel被ChannelInitializer的initChannel所调用:

```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
        }
        return true;
    }
    return false;
}
```

而initChannel的的调用有两处: 

![](https://a.a2k6.com/gerald/i/2024/01/07/246p.jpg)

这个handlerAdded从名字推测，感觉是我们在ChannelPipeline添加处理器的时候，每添加一次就触发一次, 我们做个测试:

```java
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpContentCompressor((CompressionOptions[]) null));
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("---------------"+ ctx.name());
        super.handlerAdded(ctx);
    }
}
```

只输出了一次，想想觉得也是合理的，我们的HttpHelloWorldServerInitializer是每个TCP连接建立之后，为每个连接加上一个处理链条, 但是想想又觉得不对，handlerAdded只触发了一次，不是每次TCP连接建立，都会走这个链条嘛？

![](https://a.a2k6.com/gerald/i/2024/01/07/2ls3.jpg)



原因在于，我们在HttpHelloWorldServerHandler回应了浏览器的请求，也就是keep-alive，所谓的keep-alive是HTTP-1.1引入的特性，重用上一次的TCP连接，不必每一次都发起新的昂贵的TCP连接请求。我们拒绝keep-alive试试看:

```java
if (keepAlive) {
    response.headers().set(CONNECTION, CLOSE);
  /*  if (!req.protocolVersion().isKeepAliveDefault()) {
        response.headers().set(CONNECTION, KEEP_ALIVE);
    }*/
} else {
    // Tell the client we're going to close the connection.
    response.headers().set(CONNECTION, CLOSE);
}
```

这也印证了我们的推测，每次连接建立触发HttpHelloWorldServerInitializer,经过HttpHelloWorldServerInitializer添加的每个处理器，像是一个过滤器链条一样，handlerAdded方法是处理器添加的时候所调用，那么初始化连接设置处理器那也就是channelRegistered调用initChannel了。我们简单看一下ChannelInitializer的channelRegistered源码:

```java
@Override
@SuppressWarnings("unchecked")
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    if (initChannel(ctx)) {     
        ctx.pipeline().fireChannelRegistered();
        removeState(ctx);
    } else {
        // Called initChannel(...) before which is the expected behavior, so just forward the event.
        ctx.fireChannelRegistered();
    }
}
```

channelRegistered来自于ChannelHandler，然后我们可以看到用ChannelHandlerContext调用initChannel添加完处理之后，接着从ChannelHandlerContext获取pipeline调用fireChannelRegistered，那么这里就可以推测这是一个链表，我们看下ChannelPipeline的addLast的实现:

```java
@Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
    return addLast(null, handlers);
}
```

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    ObjectUtil.checkNotNull(handlers, "handlers");   
    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h);
    }
    return this;
}
```

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx); // 语句一

        // If the registered is false it means that the channel was not registered on an eventLoop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx); // 语句二
    return this;
}
```

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

![](https://a.a2k6.com/gerald/i/2024/01/07/jbv8.jpg)



注意到语句二这里有一个callHandlerAdded0后面其实调用的就是我们重写的handlerAdded方法。所以到这里已经有Netty有了一个大致的感觉，Netty抽象了一组方法: handlerAdded、channelRegistered等，然后初始化的时候添加对应的处理器，处理器里面通过ChannelPipeline来添加处理器，然后根据事件的先后顺序来回调这些处理器中被重写的方法。那么Netty是如何处理TCP连接的呢?

##  问题二  Netty是如何处理TCP连接的呢？

我们本次考察的还是TCP协议，那么在前面的文章里面我们知道Java的标准库提供的NIO API是以ServerSocketChannel为核心的，ServerSocketChannel的accept方法接受连接返回SocketChannel，我们就可以用SocketChannel来读写TCP连接上的数据了，Netty 是对Java NIO API的封装，那么我们还是找ServerSocketChannel的accept方法在哪里被调用:

![](https://a.a2k6.com/gerald/i/2024/01/06/7od5.jpg)

点进去看发现果然是:

```java
public static SocketChannel accept(final ServerSocketChannel serverSocketChannel) throws IOException {
    try {
        return AccessController.doPrivileged(new PrivilegedExceptionAction<SocketChannel>() {
            @Override
            public SocketChannel run() throws IOException {
                return serverSocketChannel.accept();
            }
        });
    } catch (PrivilegedActionException e) {
        throw (IOException) e.getCause();
    }
}
```

这里将Java标准库提供的SocketChannel进行了包装，变成了Netty自己的SocketChannel, 我们接着看这个accept方法被谁调用，这个accept方法被NioServerSocketChannel所调用:

```java
@Override
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }
    return 0;
}
```

这里我们可以很明显的看到，accept的SocketChannel，变成了Netty自定义的SocketChannel，然后又变成了NioSocketChannel加入方法传入的List中:

![](https://a.a2k6.com/gerald/i/2024/01/06/6xa.jpg)

我们接着看doReadMessages被谁调用, doReadMessages的调用方只有一个, 也就是AbstractNioMessageChannel的内部类NioMessageUnsafe中的read方法:

![](https://a.a2k6.com/gerald/i/2024/01/06/bdd1.jpg)

我们还是先看这个read方法被谁调用, 这个read方法的调用方也只有一处，也就是我们前面的NioEventLoop的processSelectedKey方法:

```java
// 获取就绪了哪些操作
int readyOps = k.readyOps();

if ((readyOps & SelectionKey.OP_WRITE) != 0) { // 语句一
    // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
    ch.unsafe().forceFlush();
}

// Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
// to a spin loop 
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) { //语句二
    unsafe.read();
}
```

这个readyOps，我一瞬间以为是readOps，以为是拿读事件呢，又仔细读了读，是readyOps获取选择器上有哪些就绪事件,  这里复习一下且运算，一个数字与自己做运算，还是原数字，所以语句一是判断是否可写了。 语句二表达了什么呢？  我们看下SelectionKey的OP_READ、OP_ACCEPT：

```java
public static final int OP_READ =  1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
```

<< 被称为左移运算符，举个例子:

```java
int number = 1;     // 二进制表示为 0001
int shifted = number << 2;  // 将number的二进制位向左移动2位得到 0100，即十进制的4
// 也就是1 * 2^2 = 4
```

那么OP_READ = 1;   OP_WRITE = 4, OP_CONNECT = 8,OP_ACCEPT = 16。 刚好是2的次幂数，也就是说最高位为1，其他位为0: 

![](https://a.a2k6.com/gerald/i/2024/01/06/4vu1zg.jpg)

![](https://a.a2k6.com/gerald/i/2024/01/06/3mbap.jpg)

又还原回来了，这样的写法有点高级，一个二的次幂数和1做或运算，设为x，x | 1，得到y，然后y & x = x。我们来简单来证明一下，一个2的次幂数除了1然后最高位一定是1，然后和1做运算拿到的二进制数一定是最开头那一位是1，最末尾那一位也是1，然后再跟你自己做运算，末位变成0还原为原来的数。又学了一招。然后语句二上面的注释也吸引了我的注意:

>  Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead  to a spin loop 
>
> 也要检查 `readOps` 是否为 0，以解决可能的 JDK bug，否则可能会导致无限循环。

翻了Github没找到这个bug是啥，这个不影响我们继续看, 我们接着看processSelectedKey被哪里调用:

![](https://a.a2k6.com/gerald/i/2024/01/06/e8m.jpg)

这两处调用又被processSelectedKeys调用；

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

processSelectedKeys又被NioEventLoop的run方法所调用, run方法来自于SingleThreadEventExecutor的run，这个run又被doStartThread所调用, doStartThread又被startThread调用，startThread又被SingleThreadEventExecutor的execute所调用,  这个execute来自JDK的Executor，这个是一个线程池。那这个SingleThreadEventExecutor是什么时候被调用的, 我们在SingleThreadEventExecutor的execute这个方法上打个断点观察下这个调用链是怎么样的:

![](https://a.a2k6.com/gerald/i/2024/01/06/knvl.jpg)

 到现在我们已经大致明白Netty已经怎么处理请求了，在初始化的时候启动了一个线程不断的accept连接，将连接不断的变成NioSocketChannel，我们接着看Channel是给下面的Handler的，也就是NioMessageUnsafe的read方法:

![](https://a.a2k6.com/gerald/i/2024/01/06/l1us.jpg)

```java
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}
```

到现在为止Netty大致的启动流程和处理请求流程我们已经有了大致的了解: 

- ServerBootstrap先是调用bind，然后是doBind，然后doBind里面调用了initAndRegister
- 在initAndRegister里面调用了MultithreadEventLoopGroup的register方法，然后调用SingleThreadEventLoopregister方法
- 然后走到AbstractChannel的register方法，最后走到SingleThreadEventExecutor的execute方法。
- 这个方法里面开启了一个线程无限循环调用Selector来轮询就绪的事件，然后通过流水线来不断的触发fireChannelRegistered、fireChannelRead。

先是连接建立，再是可以读取了触发需要读的处理器，我们只需要重写对应的方法，Netty就会在对应的事件触发之后来回调我们的方法，这也许就是事件驱动吧。                                                                                                                                                                                                                                                                                   

## 问题三  HttpServerCodec 的作用是什么

### 拆包器简介

在《一》和《二》里面我们在读数据的时候都做了判断，判断报文什么时候结束，原因在于对于TCP协议来说，应用层交付的数据包会被合并为一个数据包，比如我们发送了两个数据报文合并在一起了，就好像两个人交流的时候我们会停顿，停顿就是分隔符便于我们理解上下文，但是对于网络应用来说，如果事先没有约定，那么两个数据报文合并在一起，对于应用层来说就会有解析问题，所以HTTP的做法是用连续的特殊字符来判断请求体是否结束，也就是\r\n\r\n:

```java
public  static boolean isComplete(ByteBuffer byteBuffer){
    int position = byteBuffer.position() - 4;
    if (position < 0){
        return false;
    }
    return byteBuffer.get(position + 0) == '\r'
            && byteBuffer.get(position + 1) == '\n'
            && byteBuffer.get(position + 2) == '\r'
            && byteBuffer.get(position + 3) == '\n';
}
```

这也引出了TCP沾包问题，每次通信需要界定边界，该如何界定，HTTP协议解决这种问题的手段是在报文后面插入特殊字符，其实可以通过消息定长这个手段来进行解码，即固定消息的长度，不够就补特殊字符，对应的类也就是FixedLengthFrameDecoder，明确消息边界的分隔符拆包器为: DelimiterBasedFrameDecoder。 还有一种思路是变长协议，也就是规定前几个字节为长度，后面跟实际的消息内容:

![](https://a.a2k6.com/gerald/i/2024/01/07/l83q.jpg)

对应的的类是LengthFieldBasedFrameDecoder，在Netty的这个类里面有注释，我们简单选择几个来解读一下，对于上面形式的报文来说，我们可以在LengthFieldBasedFrameDecoder填入的参数为: 

- lengthFieldOffset   = 0   从哪里开始读取长度字段
- lengthFieldLength  = 2   长度字段占几个字节
- lengthAdjustment  = 0
- initialBytesToStrip  = 0

如果我们想在收到的报文里面移除头部，那么可以将lengthAdjustment赋值为-2，这样解码之后移除的头部就会只包括内容本身。那如果我们的报文如下所示呢:

![](https://a.a2k6.com/gerald/i/2024/01/07/4l5ys.jpg)



现在让我们对长度字段进行一些调整，但是长度字段表示的是整个消息的长度，而不仅仅是消息内容的长度。那么应该配置的参数为:

- lengthFieldOffset = 1
- lengthFieldLength = 2
- lengthAdjustment = -3：由于长度字段包含了整个消息长度，我们需要从解码后的长度中减去头部HDR1和长度字段本身的长度（共3字节），从而得到正确的内容长度。
- 解码时，同样跳过头部HDR1和长度字段。

###  关于HttpServerCodec我有两个问题:

- 在哪里分割报文的
- 原本提取的是字节，在哪里变成的HttpRequest。

在观察一下HttpServerCodec:

![](https://a.a2k6.com/gerald/i/2024/01/07/n4ic.jpg)

然后我们接着看HttpServerCodec填入的泛型 , 一个是HttpRequestDecoder:

![](https://a.a2k6.com/gerald/i/2024/01/07/3c6h.jpg)

ByteToMessageDecoder是一个抽象类，留的扩展方法是:

```java
protected abstract void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception;
```

我们直到Java 标准库的 NIO API提取到的数据都是ByteBuffer，这里我想ByteToMessageDecoder就是将提取到的数据进行转码。

![](https://a.a2k6.com/gerald/i/2024/01/07/s1b7.jpg)

![](https://a.a2k6.com/gerald/i/2024/01/07/kk9.jpg)

我们还是选择调试代码来看看Netty的HttpObjectDecoder的decode是如何解析HTTP报文的: 

![](https://a.a2k6.com/gerald/i/2024/01/07/xoeor.jpg)

![](https://a.a2k6.com/gerald/i/2024/01/07/4tqxt.jpg)

![](https://a.a2k6.com/gerald/i/2024/01/07/i7u.jpg)

跟我想的不一样，不是通过判断\r\n来判断报文结束，而是报文里面就填入了content-length，直接获取长度，在这一步提取ByteBuffer里面的信息，将ByteBuffer里面的信息变成HttpRequest。

##  问题四  如果我想用私有协议怎么办

终于走到重头戏了，有时候我们需要对接硬件，或者我们需要自定义协议，那么怎么办呢，从上面的HttpServerCodec我们可以看出我们可以自定义一个编码、解码器，如果协议简单我们也可以直接使用Netty自定义的拆包器，我们举一个例子，我们想使用特殊字符分割器来做拆包, 也就是DelimiterBasedFrameDecoder，我们看下这个方法是如何实现的:

```java
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null) {
        out.add(decoded);
    }
}
```

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
        if (lineBasedDecoder != null) {
            return lineBasedDecoder.decode(ctx, buffer);
        }
        // Try all delimiters and choose the delimiter which yields the shortest frame.
        int minFrameLength = Integer.MAX_VALUE;  
        if (minDelim != null) {
            int minDelimLength = minDelim.capacity();
            ByteBuf frame;      
            if (stripDelimiter) {
                frame = buffer.readRetainedSlice(minFrameLength);
                buffer.skipBytes(minDelimLength);
            } else {
                frame = buffer.readRetainedSlice(minFrameLength + minDelimLength);
            }
            return frame;
        } 
}
```

主体逻辑还是从ByteBuf里面读到值，然后返回给我们frame，我们可以仿照Http解码的思路, 也对读取到的数据进行解码，最后到我们的处理器上。

```java
public class CustomerDelimiterBasedFrameDecoder extends DelimiterBasedFrameDecoder {
    public CustomerDelimiterBasedFrameDecoder(int maxFrameLength, ByteBuf delimiter) {
        super(maxFrameLength, delimiter);
    }
    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
        ByteBuf readByteBuffer = (ByteBuf)super.decode(ctx, buffer);
        byte[] bytes = new byte[readByteBuffer.readableBytes()];
        readByteBuffer.readBytes(bytes);
        Student student = new Student(new String(bytes));
        return student;
    }
}
public class FrameDemoServerHandler extends SimpleChannelInboundHandler<Student> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Student msg) throws Exception {
        System.out.println(msg);
    }
}
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {
    private static final String DELIMITER = "\n";

    @Override
    public void initChannel(SocketChannel ch) {
        ByteBuf delimiter = Unpooled.wrappedBuffer(DELIMITER.getBytes()); // 将分隔符字符串转换为ByteBuf
        ChannelPipeline p = ch.pipeline();
		// 最大长度200
        p.addLast(new CustomerDelimiterBasedFrameDecoder(200,delimiter));
        p.addLast(new FrameDemoServerHandler());
    }  
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        super.handlerAdded(ctx);
    }
}
public class ClientDemo {
    public static void main(String[] args)throws  Exception {
        try (Socket socket = new Socket()){
            socket.connect(new InetSocketAddress("127.0.0.1",8080));
            try(OutputStream outputStream = socket.getOutputStream()){
                outputStream.write("helloworld\n".getBytes());
            }
        }
    }
}
```

其实Netty有内置的StringDecoder和StringEncoder，Decoder是解码，将读到的数据转成String，编码器也就是将我们写入的内容转换成ByteBuf。我们观察一下StringEncoder的实现看看与我们自定义的有什么不同:

![](https://a.a2k6.com/gerald/i/2024/01/07/5oih5.jpg)

```java
@Sharable
public class StringDecoder extends MessageToMessageDecoder<ByteBuf> {

    // TODO Use CharsetDecoder instead.
    private final Charset charset;

    /**
     * Creates a new instance with the current system character set.
     */
    public StringDecoder() {
        this(Charset.defaultCharset());
    }

    /**
     * Creates a new instance with the specified character set.
     */
    public StringDecoder(Charset charset) {
        this.charset = ObjectUtil.checkNotNull(charset, "charset");
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
        out.add(msg.toString(charset));
    }
}
```

按照处理器的顺序是先DelimiterBasedFrameDecoder、再StringDecoder，经过DelimiterBasedFrameDecoder处理之后消息变成ByteBuf，然后再经过StringDecoder转成String，这样一看貌似MessageToMessageDecoder更方便一点。现在我们讲完了读该如何读，下面就需要写了，写相对自由一点，因为在本机，纯内存操作会很快，一般来说写的时候也要约定格式，所以我们可以做一个写的编码器，我们只用写内容，写的时候自动帮我们补全格式。写的话一般有两个api， writeAndFlush和write，write将写入内容放入缓冲区，而flush则将缓冲区的内容发送出去，writeAndFlush是两个动作的合并。

## 其他常用的事件

这些方法都在SimpleChannelInboundHandler，可以根据自己的需要重写.

1. channelRegistered 连接被注册到事件循环组
2. channelActive  连接可用可读可写
3. channelInactive  连接关闭
4. channelUnregistered    连接脱离事件循环组 

5. userEventTrigger  这个单独说一下，目前常用的就是配合心跳检测，所谓心跳检测就是服务端判断设备是否存活每隔一段时间读写的数据包，在指定时间读没成功，或者写没成功，触发此事件。

## 写在最后

这其实也算是看源码的过程，看源码的时候，也问了chatGPT，然后想到一句话:

> 我使用 ChatGPT 的感觉，就像在使用某个电话自动应答系统，到了某些时候我不得不大声尖叫，要求与人类交谈。

我感到他不明白我的意思，后面只让他给了一些简单示例。这次是重新看Netty，跟之前看有别样的感觉，这次学习之后对Netty更加得心应手，这次不怎么看文档了，也就是纯看源码，猜想验证自己的猜想，然后写示例。

## 参考资料

[1] 《Netty in Action笔记(二)》 https://fangjian0423.github.io/2016/08/29/netty-in-action-note2/

[2] 《TCP 协议简介》  https://www.ruanyifeng.com/blog/2017/06/tcp-protocol.html 

[3] Netty-11-channelHandler的生命周期  https://zhouze-java.github.io/2019/07/02/Netty-11-channelHandler%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/

[4] Netty in Action笔记(二)  https://fangjian0423.github.io/2016/08/29/netty-in-action-note2/ 