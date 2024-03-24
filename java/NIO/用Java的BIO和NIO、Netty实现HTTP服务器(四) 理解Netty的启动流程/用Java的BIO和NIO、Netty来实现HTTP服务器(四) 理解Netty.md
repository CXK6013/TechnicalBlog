# 用Java的BIO和NIO、Netty来实现HTTP服务器(四) 理解Netty

[TOC]

## 前言

让我们来捋一捋前面讲了什么，前面我们试图用Java 标准API构建BIO、NIO模型的HTTP服务器，用Java标准API构建的时候，我们操心的事情也就是管理连接，读写报文，BIO就简单些，连接到了，我们就在那里读写，读到结束符，交给下层的去解析，然后回写报文。到NIO的时候编码又复杂了一些，连接到了，未必是可读可写，我们有一个选择器，选择器上面有我们感兴趣的就绪事件，就绪事件触发了，有对应的处理者去处理。然后用Netty来重构，相对于我们用Java标准API来写的，代码量更小更简单，那么我想起一句话，如有人岁月静好，那么一定有人负重前行。因此我们在用Java的BIO和NIO、Netty来实现HTTP服务器(三)里面盘了盘Netty是如何接受连接的，处理连接的，我们的处理器如何被触发的， 我们是如何看的，我的思路是什么呢，我认为Netty还是基于Java提供的API，那么核心还在Java 提供的NIO API上，也就是ServerSocketChannel，我们知道不管是BIO模型、还是NIO模型，都是通过accept方法来接受连接的，我们就以这个点来观察这个方法在Netty是如何被调用的, 我是怎么做的呢，

![](https://a.a2k6.com/gerald/i/2024/01/22/5223k.png)

那也就是看SocketUtils这个调用，点进去看发现这个类果然在Netty的包里面：

![](https://a.a2k6.com/gerald/i/2024/01/22/zstkj.png)

![](https://a.a2k6.com/gerald/i/2024/01/22/qdye.png)

然后来到NioServerSocketChannel的doReadMessages方法:

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

上面这段代码的逻辑比较简单，就是拿到SocketChannel之后，将其加入到传入的List里面。doReadMessages方法的调用处也只有一处来自AbstractNioMessageChannel的内部类NioMessageUnsafe的read方法:

![](https://a.a2k6.com/gerald/i/2024/01/22/57tim.png)

我们接着看这个read方法被谁所调用，这个方法的调用方也只有一处，也就是NioEventLoop中的processSelectedKey方法，这个processSelectedKey我们在用《Java的BIO和NIO、Netty来实现HTTP服务器(三) 理解Netty》里面我们已经细致的讲过了，processSelectedKey的调用方有两处:

- processSelectedKeysOptimized
- processSelectedKeysPlain

这两个方法的调用逻辑都是从Selector中获取SelectionKey，然后触发对应的逻辑处理，也就是我们上面的NioMessageUnsafe的read方法，触发pipeline的fireChannelRead方法。

```java
private final class NioMessageUnsafe extends AbstractNioUnsafe {

    private final List<Object> readBuf = new ArrayList<Object>();
	
    @Override
    public void read() {
        assert eventLoop().inEventLoop();
        final ChannelConfig config = config();
        final ChannelPipeline pipeline = pipeline();
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.reset(config);

        boolean closed = false;
        Throwable exception = null;
        try {
            try {
                do {
                    int localRead = doReadMessages(readBuf);
                    if (localRead == 0) {
                        break;
                    }
                    if (localRead < 0) {
                        closed = true;
                        break;
                    }
					
                    allocHandle.incMessagesRead(localRead);
                } while (continueReading(allocHandle));
            } catch (Throwable t) {
                exception = t;
            }

            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                readPending = false;              
                pipeline.fireChannelRead(readBuf.get(i)); // 语句一触发pipeline的fireChannelRead
            }
            readBuf.clear();
            allocHandle.readComplete(); 
            pipeline.fireChannelReadComplete(); // 语句二 触发pipeline的fireChannelReadComplete

            if (exception != null) {
                closed = closeOnReadError(exception);	
                pipeline.fireExceptionCaught(exception); //语句三触发pipeline的fireExceptionCaught
            }
            if (closed) {
                inputShutdown = true;
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        } finally {          
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```

回想HttpHelloWorldServerHandler，这个类继承了SimpleChannelInboundHandler，我们重写了channelRead0方法，在里面解析HTTP请求报文，并回写HTTP响应。在SimpleChannelInboundHandler里面channelRead0又被SimpleChannelInboundHandler的channelRead方法所调用，这也就跟上面代码块的，语句一中的  pipeline.fireChannelRead和我们重写的channelReadComplete对应上了，fireExceptionCaught和我们重写的exceptionCaught对应上了。

```java
public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
    System.out.println("数据被完全处理");
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
        ChannelFuture f = ctx.write(response).addListener(e->{
            System.out.println("hello world");
        });
        if (!keepAlive) {
            f.addListener(ChannelFutureListener.CLOSE);
        }
    }
}
```

### 总结一下

我们总结一下我们看源码的收获，还是以JDK中最基础的API为核心，Netty帮我们做了封装，将处理网络链接，读写消息变成了一个又一个的事件, 一个又一个方法，我们可以重写这些方法，对应事件触发之后会回调我们的方法。 从最基础的ServerSocketChannel的accept方法，Netty里面使用SocketUtils的accept方法，将JDK的SocketChannel变成Netty自己的SocketChannel，而SocketUtils的accept方法的调用方又只有一处，也就是NioServerSocketChannel的doReadMessages，doReadMessages方法会将拿到的SocketChannel实例放到传入的List里面，而doReadMessages方法则又被NioMessageUnsafe的read方法所调用，在这个方法里面我们看到了pipeline.fireChannelRead、pipeline.fireChannelReadComplete()、pipeline.fireExceptionCaught。而NioMessageUnsafe的read方法又被NioEventLoop的processSelectedKey所调用，NioEventLoop的processSelectedKey又有两处调用: processSelectedKeysOptimized、processSelectedKeysPlain，这两个方法又只有一处调用也就是NioEventLoop的processSelectedKeys，NioEventLoop的processSelectedKeys又直接被NioEventLoop的run方法直接调用，run方法里面是一个无限循环。

![](https://s1.wzznft.com/i/2024/02/17/h9st9s.jpg)

## 接着捋一捋Netty的启动流程

而NioMessageUnsafe的read方法则是被NioEventLoop的run方法所调用，这个方法的主体逻辑是一个无限循环，这也许就是NioEventLoop由来，事件循环组。这个run方法在什么时候被启动呢，我们还是按着ctrl去找调用方，然后来到SingleThreadEventExecutor的doStartThread方法，SingleThreadEventExecutor是一个抽象类，NioEventLoop是它的子类，在SingleThreadEventExecutor的doStartThread方法中的逻辑如下:

![](https://a.a2k6.com/gerald/i/2024/01/22/5m2sa.png)

  那这个doStartThread方法是如何被调用的呢，该如何和我们的启动方法联系起来的呢?   我们在这里打上断点，观察执行链，看看这个方法是如何被调用的。

![](https://a.a2k6.com/gerald/i/2024/01/31/uv4b.png)

  调用链是从AbstractBootstrap的bind方法，到AbstractBootstrap的doBind方法，看到了这里想到了什么呢？ 我们们再把启动代码再拎出来看一下：

```java
public class HttpHelloWorldServer {
static final boolean SSL = System.getProperty("ssl") != null;
static final int PORT = Integer.parseInt(System.getProperty("port", SSL? "8443" : "8080"));

public static void main(String[] args) throws Exception {
	// 线程池
    EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 语句一
    // 线程池
    EventLoopGroup workerGroup = new NioEventLoopGroup(Runtime.getRuntime().availableProcessors()); // 语句二
    try {
        ServerBootstrap b = new ServerBootstrap(); // 语句三
        b.option(ChannelOption.SO_BACKLOG, 1024); // 语句四
        b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new HttpHelloWorldServerInitializer()); // 语句五
        Channel ch = b.bind(PORT).sync().channel(); // 语句6     
        ch.closeFuture().sync(); // 语句七
    } finally {
        bossGroup.shutdownGracefully(); //语句八
        workerGroup.shutdownGracefully(); // 语句九
    }
  }
}   
```

ServerBootstrap继承AbstractBootstrap，语句6调用bind方法就来自AbstractBootstrap的bind方法， 然后一路就把事件循环组跑起来了。那sync是在干嘛呢?  单看ServerBootStrap的bind方法，这个方法来自AbstractBootstrap的bind方法:

```java
public ChannelFuture bind() {
    validate();
    SocketAddress localAddress = this.localAddress;
    if (localAddress == null) {
        throw new IllegalStateException("localAddress not set");
    }
    return doBind(localAddress);
}
```

ChannelFuture是一个接口，我们看看doBind返回的是哪个实现类，我们还是打断点，看实现类是哪一个:

![](https://a.a2k6.com/gerald/i/2024/01/31/ohwg.png)

DefaultChannelPromise，这让我想到了JavaScript的Promise:

```java
new Promise(function(resolve, reject) {
              setTimeout(()=>{
                    console.log(0);
                    resolve("1");    
               },3000)
}).then((result)=> console.log(result)); 
```

上面的代码中，我们声明了一个Promise对象，并向其传递了一个函数，函数的行为是延迟秒执行，并将结果传递给下一个过程，也就是then里面使用的result，所以控制台会输出0,1。这个Promise在Java里面对应的类叫CompletableFuture，上面的代码用CompletableFuture来写就是:

```java
CompletableFuture.supplyAsync(()->{
    try {
        TimeUnit.SECONDS.sleep(3);
         System.out.println(0);
         return 1;
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 1;
}).thenAccept(result->{
    System.out.println(1);
});
```

Future代表计算执行结果，Promise代表执行链条。我们看下这个DefaultChannelPromise的sync方法:

```java
@Override
public ChannelPromise sync() throws InterruptedException {
    super.sync();
    return this;
}
```

DefaultChannelPromise继承DefaultPromise，在DefaultPromise的sync实现是:

```java
private volatile Object result; // 语句一

public Promise<V> sync() throws InterruptedException {
    await(); 
    rethrowIfFailed();
    return this;
}
public boolean isDone() {
  return isDone0(result);
}
private static boolean isDone0(Object result) {
  return result != null && result != UNCANCELLABLE;
}
public Promise<V> await() throws InterruptedException {
        if (isDone()) {
            return this;
        }
        if (Thread.interrupted()) {
            throw new InterruptedException(toString());
        }
        checkDeadLock();
        synchronized (this) {
            // 语句二
            while (!isDone()) {
                incWaiters();
                try {
                    wait(); // 语句三
                } finally {
                    decWaiters(); // 语句四
                }
            }
        }
        return this;
}
```

我打断点在语句三，照我的想法，应该卡在这里不动，毕竟调用了wait方法之后, 线程的处于WAITING状态，应该等待其他线程唤醒才能接下去执行，也就是调用notify或者notifyAll方法，但是main线程在执行到语句三之后，没有阻塞在这里，而是接着往下执行，执行到了语句四，那就说明main线程处于WAITING状态没多久就被唤醒了，但究竟是在哪里唤醒的呢，我们翻一翻DefaultPromise，在checkNotifyWaiters找到了notifyAll方法:

```java
private synchronized boolean checkNotifyWaiters() {
    if (waiters > 0) {
        notifyAll();
    }
    return listener != null || listeners != null;
}
```

我们在这个方法上打上断点，看一下这个方法被谁调用:

![](https://a.a2k6.com/gerald/i/2024/02/16/4dyk.jpg)

### 浅浅总结一下

我们启动类里面的ServerBootstrap在bind的时候事实上调用的是ServerBootstrap父类的AbstractBootstrap的bind方法，然后接着调用AbstractBootstrap的doBind方法，然后doBind方法调用initAndRegister方法，最后调用到MultithreadEventLoopGroup的register方法，然后走到SingleThreadEventLoop的register方法，接着走到AbstractUnsafe的register方法上，然后走到SingleThreadEventExecutor的execute方法上，接着走到了SingleThreadEventExecutor的startThread方法上，整个事件循环组就开起来了。

```java
 Channel ch = b.bind(PORT).sync().channel(); 
```

 bind返回的是一个DefaultChannelPromise对象，调用sync方法会让执行这段代码的线程进入WAITING状态，进入WAITING状态是为了等待其他线程初始化任务完成，粗略的说也就是将事件循环组跑起来，跑起来之后会调用DefaultChannelPromise的trySuccess方法，来唤醒进入WAITING状态的线程(要求是一把锁上的wait和notify哦)。

```
 ch.closeFuture().sync();
```

最后拿到的Channel事实上是NioServerSocketChannel，调用closeFuture方法拿到的是AbstractChannel的CloseFuture实例，CloseFuture继承DefaultChannelPromise类，所以调用sync方法还是让主线程在这里进入WAITING状态，毕竟是一个HTTP服务器需要不断的回应请求，如果没有这一行最后就会走到finally里面的代码，我们开启的事件循环组会被关闭。

## 再理解启动流程

首先让我们再拎出来用《Java的BIO和NIO、Netty来实现HTTP服务器(三)》的HttpHelloWorldServer:

```java
public class HttpHelloWorldServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", SSL? "8443" : "8080"));

    public static void main(String[] args) throws Exception {
    	// 线程池
        EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 语句一
        // 线程池
        EventLoopGroup workerGroup = new NioEventLoopGroup(Runtime.getRuntime().availableProcessors()); // 语句二
        try {
            ServerBootstrap b = new ServerBootstrap(); // 语句三
            b.option(ChannelOption.SO_BACKLOG, 1024); // 语句四
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new HttpHelloWorldServerInitializer()); // 语句五
            Channel ch = b.bind(PORT).sync().channel(); // 语句6
            System.err.println("Open your web browser and navigate to " +
                    (SSL? "https" : "http") + "://127.0.0.1:" + PORT + '/');
            ch.closeFuture().sync(); // 语句七
        } finally {
            bossGroup.shutdownGracefully(); //语句八
            workerGroup.shutdownGracefully(); // 语句九
        }
    }
}
```

语句一和语句二我们声明了两个EventLoopGroup，可以当做线程池来使用，EventLoopGroup实现了ScheduledExecutorService，NioEventLoopGroup是一个多线程的事件循环，用于处理I/O操作(连接建立，读写数据)，语句一我们声明了一个叫bossGroup的EventLoopGroup对象，bossGroup负责接受进来的连接，语句二我们声明了一个叫workerGroup的EventLoopGroup对象，一旦bossGroup接受到了连接就会接受的连接注册给workerGroup，之后就是workerGroup负责处理该连接的流量，我们可以通过EventLoopGroup的构造函数来指定线程数量。

语句四我们声明了一个ServerBootstrap，这个类辅助我们设置一些参数，语句四的我们声明了SO_BACKLOG的数量，三次握手之后，连接会进入到一个队列里面，应用程序调用accept之后，连接从队列里面移除，这个参数指定队列里面最多能够承受多少TCP连接。语句五里面我们组合bossGroup，workerGroup，ServerBootstrap中NioServerSocketChannel的作用是表示当新连接建立之后，我们用NioServerSocketChannel这个来表示，NioServerSocketChannel用于TCP协议，NioDatagramChannel用于UDP协议。

##  理解流水线的触发逻辑

### 为什么handlerAdded方法被触发了两次?

我们还在语句五中声明了childHandler，也就是HttpHelloWorldServerInitializer，每次连接建立之后都会被触发，我在HttpHelloWorldServerInitializer加了一行输出代码:

```java
public class HttpHelloWorldServerInitializer extends 
    <SocketChannel> {
    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        System.out.println("被触发一次");
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpContentCompressor((CompressionOptions[]) null));
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }
}
```

发现控制台输出了两次"被触发一次"，这让我比较好奇initChannel的调用逻辑，initChannel来自ChannelInitializer，在ChannelInitializer中调用方有两处，一处是channelRegistered，一处是handlerAdded，所以输出两次也不是不能说的通，

```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
            if (!ctx.isRemoved()) {
                ctx.pipeline().remove(this);
            }
        }
        return true;
    }
    return false;
}
```

```java
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    if (initChannel(ctx)) {    
        ctx.pipeline().fireChannelRegistered();
        removeState(ctx);
    } else {    
        ctx.fireChannelRegistered();
    }
}
```

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        if (initChannel(ctx)) {
            removeState(ctx);
        }
    }
}
```

在《用Java的BIO和NIO、Netty来实现HTTP服务器(三)》我们提到了handlerAdded方法，这个方法是添加一个处理器触发一次，我们重写一下这个方法看一下输出结果:

```java
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {

    private static final AtomicInteger ORDER = new AtomicInteger(1);

    @Override
    public void initChannel(SocketChannel ch) {
        System.out.println(Thread.currentThread().getName()+ " initChannel:" + ORDER.incrementAndGet());
        ChannelPipeline p = ch.pipeline();
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpContentCompressor((CompressionOptions[]) null));
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("ip address port"+ ctx.channel().remoteAddress());
        System.out.println(Thread.currentThread().getName() + " Channel ID: " + ctx.channel().id() + " handlerAdded:" + ORDER.incrementAndGet());
        super.handlerAdded(ctx);
    }
}
```

为什么上面用原子类呢，只是我随手写的，但是却意外的试出来发现打印出来的线程名称是不一样的，输出结果为:

```java
nioEventLoopGroup-3-1   handlerAdded:2
nioEventLoopGroup-3-1 initChannel:3
nioEventLoopGroup-3-2   handlerAdded:4
nioEventLoopGroup-3-2 initChannel:5
```

奇怪这个handlerAdded 为什么会被调用两次呢?   打了一下断点发现调用链是一样的： 

![](https://a.a2k6.com/gerald/i/2024/02/17/v01d.jpg)

![](https://a.a2k6.com/gerald/i/2024/02/17/vcpx.jpg)

并且channel的ID都是不一样的，所以是不同的TCP连接，但是这是为什么，我猜想是为了兼容HTTP 1.0, 在HTTP 1.0每次HTTP请求都对应一个TCP连接请求，为了提高速度，客户端可能会为同一个请求打开一个新的连接，然后关闭它，再打开另一个新的连接用于后续的请求。这通常发生在客户端和服务器之间的 keep-alive 选项不匹配时。我猜想到这个之后感到很兴奋，但是我该怎么验证我的猜想呢，如果是按照我的猜想来说，那么假设第一次输出的channelId是1,2，那么第二次输出的应该是2,3，也就是第二次请求复用第一次的预先创建的TCP连接，但是观察控制台发现，我在chrome、postman浏览器里面发起HTTP请求，控制台输出的channelId都是不同的。但是放到火狐浏览器下面handlerAdded就只触发了一次，更深层次的原因我们放在后面考究吧，目前也就是说，当前处理器被添加的时候会触发handlerAdded一次。

下午跑完步回来跟别人讨论了一下，看了一下浏览器的network:

![](https://a.a2k6.com/gerald/i/2024/02/17/1x0.jpg)

所以原因就是浏览器会多余请求一次favicon一次，才导致handlerAdd会被触发两次，那么为什么火狐只触发了一次呢，原因在于火狐浏览器会在首次请求的时候默认对favicon进行缓存，那Postman为什么会请求两次呢，原因是Postman的bug，见参考链接[1]。

### 理解流水线

启动流程我们梳理了一遍，现在也就剩ChannelInitializer、ChannelPipeline、SimpleChannelInboundHandler没有仔细的瞧上一瞧了，我们通过ChannelPipeline来添加处理器，我们添加了HttpServerCodec、HttpContentCompressor、HttpServerExpectContinueHandler、HttpHelloWorldServerHandler，这些处理器会被依次触发。

![](https://a.a2k6.com/gerald/i/2024/02/17/6h3zi.jpg)

我们这里以HttpServerCodec为例大致讲解一下处理器的运作流程，我们照例还是审视一下HttpServerCodec:

```java
public final class HttpServerCodec extends CombinedChannelDuplexHandler<HttpRequestDecoder, HttpResponseEncoder>
```

里面有两个内部类HttpServerRequestDecoder、HttpServerResponseEncoder，从名字上我们可以推断出来一个是解析HTTP请求，一个将响应编码成HTTP响应。我们在HttpServerRequestDecoder上打上断点观察一下调用链:

![](https://a.a2k6.com/gerald/i/2024/02/17/4tna.jpg)

 也就是从AbstractChannelHandlerContext的invokeChannelRead触发，走到CombinedChannelDuplexHandler的channelRead方法，然后是ByteToMessageDecoder的channelRead方法，在ByteToMessageDecoder的channelRead方法中调用callDecode方法，最终调用到decodeRemovalReentryProtection方法，然后调用HttpServerRequestDecoder的decode方法。这些都是我们打断点调试得出的结论，那么对于DefaultChannelPipeline来说，想再了解的更详细一点该怎么做呢，答案是看ChannelPipeline的注释，我之前翻的是DefaultChannelPipeline的注释，但是没翻到，后面无意间就翻到了ChannelPipeline上有比较详尽的注释，所以启示我们如果在当前类里面找不到注释，我们不妨就去找他的父类上去找找。

![](https://a.a2k6.com/gerald/i/2024/02/17/y0wm.jpg)

 这个注释还是比较详细的，这里我就不贴原文了，直接放我的理解了，有兴致的同学可以自己去翻注释去看一下。

每个Channel都有自己的Pipeline，一个Channel被创建的时候Pipeline 被自动创建。Pipeline如何处理I/O事件呢：

![](https://a.a2k6.com/gerald/i/2024/02/17/6q3kx.jpg)

   也就是说读到数据之后会经过一系列InBoundHandler，我们可以通过ChannelHandlerContext来将事件传播给最近的InBoundHandler，而ChannelHandlerContext的write方法经历过一系列OutBoundHandler来传播，最后到达Socket的Write方法。让我们假设有下面的处理器链:

```java
 ChannelPipeline p = ...;
 p.addLast("1", new InboundHandlerA());
 p.addLast("2", new InboundHandlerB());
 p.addLast("3", new OutboundHandlerA());
 p.addLast("4", new OutboundHandlerB());
 p.addLast("5", new InboundOutboundHandlerX());
```

Inbound开头的代表是入站程序，以Outbound开头的代表是出站程序，所为入站也就是读取数据经过的链路，出站也就是回写数据经过的链路。ChannelPipeline会根据处理器类型和事件类型来决定是否要经过这个处理器，处理器3和4没有实现ChannelInboundHandler，所以读取到数据之后，不会经过这两个处理器，读取数据事实上只会经过1、2、5。对于出站事件，也就是回写数据事件，由于1,2没有实现ChannelOutboundHandler，所以回写数据的时候不会经过1,2，只会经过3、4、5。在回写数据的时候是反向的，因为出站事件通常涉及到从应用程序向网络发送数据，所以数据会先通过最后添加的处理器。我们假定InboundOutboundHandlerX同时实现了ChannelInboundHandler和ChannelOutboundHandler，那么该处理器就能同时处理入站(read)和出站事件(write):

- 对于入站事件，处理器5会在最后被调用，因此入站事件的处理顺序将是1、2、5。
- 对于出站事件，处理器5会被最先调用，因此出站事件的处理顺序将是5、4、3。

如图所示，处理程序必须调用 ChannelHandlerContext 中的事件传播方法才能将事件转发给下一个处理程序。这些方法包括
入站事件的传播方法

- ChannelHandlerContext.fireChannelRegistered()
- ChannelHandlerContext.fireChannelActive()
- ChannelHandlerContext.fireChannelRead(Object)
- ChannelHandlerContext.fireChannelReadComplete()
- ChannelHandlerContext.fireExceptionCaught(Throwable)
- ChannelHandlerContext.fireUserEventTriggered(Object)
- ChannelHandlerContext.fireChannelWritabilityChanged()
- ChannelHandlerContext.fireChannelInactive()
- ChannelHandlerContext.fireChannelUnregistered()

 出站事件的传播方法

- ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
- ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
- ChannelHandlerContext.write(Object, ChannelPromise)
- ChannelHandlerContext.flush()
- ChannelHandlerContext.read()
- ChannelHandlerContext.disconnect(ChannelPromise)
- ChannelHandlerContext.close(ChannelPromise)
- ChannelHandlerContext.deregister(ChannelPromise)

所谓传播也就是调用下一个处理器中对应的方法，关于出站事件我比较好奇，出站事件不是和写数据有关嘛，为什么里面会有bind、connect、read这三个方法，于我的直觉相违背，然后我发现有人跟我一样抱有同样的疑问, 有人在StackOverFlow上问:  In Netty4,why read and write both in OutboundHandler。Netty的作者的回复是:

> Inbound handlers are supposed to handle inbound events. Events are triggered by external stimuli such as data received from a socket.
>
> 入站处理器由入站事件触发。事件由外部触发比如从Socket中接收到数据。
>
> Outbound handlers are supposed to intercept the operations issued by your application.
>
> 出站应用程序应当拦截
>
> Re: Q1) `read()` is an operation you can issue to tell Netty to continue reading the inbound data from the socket, and that's why it's in an outbound handler.
>
> read是一个可以让发出的操作，用于告诉Netty继续从Socket中读取入站数据，这也就是它被放在入站处理器的原因。
>
> Re: Q2) You don't usually issue a `read()` operation because Netty does that for you automatically if `autoRead` property is set to `true`. Typical flow when `autoRead` is on:
>
> 如果autoRead属性被设置为true，Netty通常情况下会自动执行读取操作。自动读取打开时的一般流程为:
>
> 1. Netty triggers an inbound event `channelActive` when socket is connected, and then issues a `read()` request to itself (see `DefaultChannelPipeline.fireChannelActive()`)
>
>    当Socket被连接上，Netty触发入站事件ChannelActive，然后发起read请求，参看DefaultChannelPipeline的fireChannelActive。
>
> 2. Netty reads something from the socket in response to the `read()` request.
>
>     Netty响应这个请求从Socket中读取响应。
>
> 3. If something was read, Netty triggers `channelRead()`.
>
>    如果有内容被读取，Netty触发channelRead。
>
> 4. If there's nothing left to read, Netty triggers `channelReadComplete()`
>
>    如果读取完毕，会触发channelReadComplete
>
> 5. Netty issues another `read()` request to continue reading from the socket.
>
>     Netty发起另一个read请求，继续从Socket中读取。
>
> If `autoRead` is off, you have to issue a `read()` request manually. It's sometimes useful to turn `autoRead` off. For example, you might want to implement a backpressure mechanism by keeping the received data in the kernel space
>
> 如果自动读取关闭，你必须手动的发起读取。有的时候关闭自动读取是有用的，例如，你可能想要实现背压将接收到的数据保存在内核里面。

这段话我们分成三段解读，第一段解释了Inbound Handler和Outbound Handler的设计意图，被触发被Inbound Handler来处理，被程序主动触发由Outbound Handler来处理，这样一说其实好像也能说的通，但是这里我的想法是为什么出站操作这里要放read方法，这看起来很割裂，在我看来入站和出站应当根据数据的流向来定义，读取是流入，出站是写出，Netty的这个设计在我看来有些不符合直觉。

第二段我们聊一聊autoRead属性，也就是自动读取，默认是打开的，如果想要关闭可以通过下面的方式去关闭

```java
ServerBootstrap b = new ServerBootstrap();
b.option(ChannelOption.AUTO_READ, Boolean.);
```

我们看下这个属性是如何发挥作用的，那么已知的是ChannelOutboundHandler的read方法能够拦截读操作，那么真正的读操作一定是在这个方法之后才开启读的，我猜想应当是在这段逻辑之后将数据读取进来，ChannelOutboundHandler是一个接口，实现类有又比较多，那么应该怎么看这个ChannelOutboundHandler在哪里发挥作用的呢，这里我主要有两个思路，一个是我们写这个HTTP服务器的HttpHelloWorldServerInitializer里面，用到了HttpServerCodec，而HttpServerCodec的继承图如下所示:

![](https://a.a2k6.com/gerald/i/2024/03/02/67187.jpg)

刚好有我们看到的ChannelOutboundHandler这个接口，而ChannelDuplexHandler继承ChannelInboundHandlerAdapter，实现了ChannelOutboundHandler，所以我们可以选择在ChannelDuplexHandler的read方法上打断点来观察是如何读取数据的：

```java
@Skip
@Override
public void read(ChannelHandlerContext ctx) throws Exception {
    ctx.read();
}
```

然后我们就可以借助IDEA的Step Into来观察读数据的具体操作，在IDEA的Debugger中可以看到调用链:

![](https://a.a2k6.com/gerald/i/2024/03/02/17niga.jpg)

我猜想应当是在这个方法里面获取自动读取属性是否打开的:

```java
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        channel.read();
    }
}
```

然后我们就可以在ServerBootstrap关闭和打开自动读取，验证这里的值是否跟我们设置的一致。经过验证这里的值确实是跟我们设置的一致，那么自动读取应当就是在这里获取的。那么下一个问题Netty是如何读数据的，我们还是点下一步，然后会走到DefaultChannelPipeline的HeadContext的read方法上:

```java
@Override
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```

然后我们看下这个beginRead的逻辑，然后走到AbstractChannel的beginRead方法:

```java
public final void beginRead() {
    assertEventLoop();
    try {
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```

然后最后走到AbstractNioChannel的doBeginRead过程，咦，为什么没有从SocketChannel里面捞数据的过程呢。

```
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

我们在HttpHelloWorldServerHandler的channelRead0上打上断点然后观察调用链，看能不能查到Netty是怎么捞的数据:

![](https://a.a2k6.com/gerald/i/2024/03/02/4po9.jpg)

我们简单的看下实现: 

```java
try {
    do {
        byteBuf = allocHandle.allocate(allocator);
        allocHandle.lastBytesRead(doReadBytes(byteBuf));
        if (allocHandle.lastBytesRead() <= 0) {
            // nothing was read. release the buffer.
            byteBuf.release();
            byteBuf = null;
            close = allocHandle.lastBytesRead() < 0;
            if (close) {
                // There is nothing left to read as we received an EOF.
                readPending = false;
            }
            break;
        }

        allocHandle.incMessagesRead(1);
        readPending = false;
        pipeline.fireChannelRead(byteBuf);
        byteBuf = null;
 } while (allocHandle.continueReading());
```

在这里将读取到的数据放入ByteBuffer里面，等等那SocketChannel去哪里了，JDK的标准API不是从SocketChannel里面read数据取哪里了，我们在SocketChannel的read方法打上断点观察一下调用链:

![](https://a.a2k6.com/gerald/i/2024/03/02/6ipsi.jpg)

第三段我们聊背压的实现，其实我们在《JDK 9新特性之Flow API 初探》里面已经聊到了背压了，在这一篇文章里面我们提到了响应流规范:

> 在异步系统中处理,处理数据流,尤其是数据量未预先确定的实施数据要特别小心。最为突出而又常见的问题是资源消费控制的问题，以便防止大量数据快速到来淹没目的地。

通俗的说法也就是说数据在由生产者流向消费者的过程中，上游的生产速度大于下游的消费速度，将下游压垮。这里我们重新聊一下响应式编程、背压这两个概念: 



## 总结一下

这离我们聊了

Netty的流水线处理流程，然后引入到了ChannelInboundHandler、ChannelOutboundHandler，最后引出到了ChannelOutboundHandler的设计问题，为什么

## 题外话如何学习Netty

continuations 是一种上下文，有助于恢复线程之间调用的上下文，

所以本质上，当你在协程之间调用之间来回切换的时候，

它能够记住调用的状态并恢复状态继续前进、

continuations 是一种数据结构

从别人告诉我什么，到我想知道什么

没有开销不会给你任何解决方案，挂载和卸载同样需要一些时间

执行上下文切换比花费等待的世界要好的多的时候，那么这将非常有益、线程在任务之间切换而不是被阻塞。

线程数 <=  核心数/ 1 -阻塞系数



##  参考资料

[1]  Client connects twice when using Postman to send requests? https://stackoverflow.com/questions/71088167/client-connects-twice-when-using-postman-to-send-requests

[2] JDK 9新特性之Flow API 初探 https://juejin.cn/post/7104961299670368264