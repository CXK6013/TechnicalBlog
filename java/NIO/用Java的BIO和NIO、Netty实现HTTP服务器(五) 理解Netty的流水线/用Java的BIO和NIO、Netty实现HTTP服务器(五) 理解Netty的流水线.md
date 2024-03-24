# 用Java的BIO和NIO、Netty实现HTTP服务器(五) 理解Netty的流水线

[TOC]

##  理解流水线的触发逻辑

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
>    Netty响应这个请求从Socket中读取响应。
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
>    Netty发起另一个read请求，继续从Socket中读取。
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

通俗的说法也就是说数据在由生产者流向消费者的过程中，上游的生产速度大于下游的消费速度，将下游压垮。有两种常规的背压实现，我们日常使用的线程池会有一个任务队列，当我们向线程池提交任务的时候，如果线程池当前的线程数小于核心线程数，就会启用一个线程来执行我们提交的任务，如果线程池当前线程数大于线程池核心线程数，则会将将提交的任务放到阻塞队列中，如果阻塞队列满了，就会判断当前线程数已经大于最大线程数，就会触发拒绝策略:

```java
public class ThreadPoolDemo {
    public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                1,
                1,
                4L,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1)
                ,new ThreadPoolExecutor.CallerRunsPolicy());
        // CallerRunsPolicy 是用提交任务的线程去执行提交的任务
        // 我们的线程池只有一个活跃线程,我们提交了三个任务,前两个任务会把线程池占据
        // 因此触发拒绝策略,调用提交任务的线程去执行任务
        // 第四个任务将会阻塞
        for (int i = 0; i < 4 ; i++) {
            threadPoolExecutor.submit(ThreadPoolDemo::doWork);
        }
    }

    public static void doWork(){
        System.out.println("current thread : " + Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

CallerRunsPolicy这种拒绝策略是一种典型的实现，使用提交任务的线程去执行任务，后续的任务就会阻塞在这里，在TCP应用层也有同样的概念，是最经典的示例，协议本身就提供了背压，内核会保存一个有限的缓冲，当缓冲区满的时候，会阻塞send方法，也就是说阻塞是天然的流量控制。

这个让我们再回头看ChannelOutboundHandler的read方法，这个方法对read进行拦截，而且仅当autoRead属性关闭的时候，用于主动发起读取，这里可以做流量控制，我们回忆一下TCP的窗口，简单来说TCP接收窗口是TCP连接两端的一个缓冲区，用于暂时保存接收到的数据。该缓冲区中的数据被发送到应用程序，为接收数据腾出更多空间。如果缓冲区满了，数据接收方会提醒发送方，在缓冲区被清空之前，不能再接收更多数据。这只是基本功能，还涉及一些细节。设备会在TCP头信息中公布其TCP窗口的大小。

所以这个read方法我能想到的一个场景就是，我们在写的时候被卡在那里，然后还在不断的读导致内存不断上涨，但是Netty预留了写水位线，避免我们写入导致程序出现异常，让我们抛开Netty单纯来提水位线:

![](https://a.a2k6.com/gerald/i/2024/03/09/ob3k.png)

上面是一个杯子，再接着往里面倒水，就会溢出来，这也就是水位线的来源，那这跟Netty有什么关系，让我们从Netty是如何写数据讲起，也就是ChannelHandlerContext的write方法实现, 我们可以在AbstractChannel中可以看到，Netty是如何来写数据的，如果打断点调试一步一步跟的无所适从，我们还是可以通过观察调用链的方式来观察Netty是如何来写的，最终还是要落到SocketChannel的write方法上，我们在SocketChannel的write方法实现上打个断点就可以观察到Netty对于写的实现:

![](https://a.a2k6.com/gerald/i/2024/03/09/40v7.jpg)

 观察这里的调用链可以发现，最终是channelReadComplete之后的ChannelHandlerContext的flush方法开始真正的写入，那也就是说我们通过ChannelHandlerContext的write方法的时候并没有真正的写数据，那Netty做了什么呢? ， 那看来我们还需要跟一下这个断点看下这个write方法的实现是什么。最终逻辑还是在AbstractChannel的write方法中看到了实现:

![](https://a.a2k6.com/gerald/i/2024/03/09/tdhn.jpg)

```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

这个Entry是不是好面熟，有没有想到，HashMap的Entry，我们看到我们的消息被挂在了尾结点之后。我们接着看incrementPendingOutboundBytes的实现:

```java
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }
	// 计算总共要写入的数据大小
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size); 
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        // 如果无法写入,将通道标记为不可写状态
        setUnwritable(invokeLater);
    }
}
```

我们可以理解为高水位线意味着这个通道对应的阈值，颇有种积压数据太多，再写数据就溢出来的感觉。默认大小为64*1024也就是64k。另一种更加友好的背压实现是发送方和接收方进行协商的实现，这也就是我们在《 JDK 9新特性之Flow API 初探》提到的Reactive API 消费者根据自身的能力来定义要消费多少消息，生产者根据需要将消息提供给消费者，消费者实现按需获取，生产者可以按需生产，从而避免上游生产消息过快，下游消费不及压垮下游 , JDK 11 里面引入了一组API， 可以在《 JDK 9新特性之Flow API 初探》这里面看到我写的示例。

## 总结一下

本篇我们简单的理解了Pipeline和ChannelInboundHandler、ChannelOutboundHandler之间的联系，读事件会依次触发ChannelInboundHandler之上的方法，而入站事件则是最后到达write方法， 从数据流转的方向来说，read方法不应当放在ChannelOutboundHandler这个接口里面，但是从程序发起或者被发起的角度来说，放在这里似乎没有问题，至少Netty作者是这么解释的，Netty说可以用这个来做流量控制，但是我感到有些牵强，原因在于接收的缓冲区满了，自动就阻塞了，没有阻塞说明读取数据能力是够的，那么一种场景就是读了太多数据，向下游写的时候，下游能力不足，数据一直积压。

## 题外话如何学习Netty

有的时候学习Netty的过程中，也跟朋友交流了一些问题，有人会问如何学习Netty，我的想法是跟过去已有的知识结合在一起，Netty还是构建在Java 标准库的API之上的，在学习的过程中找到新的知识点和旧有知识点之间的联系是一个不错的选择，我们在学校学习的时候，是老师告诉我们什么，这也许是一种惯性思维，我觉得学习的路上另一种有效的思维方式是，我要知道什么，运用我已有的知识点去探索，我就是这么学习Netty的。

## 参考资料

[1]背压(Back Pressure)与流量控制  https://lotabout.me/2020/Back-Pressure/

[2] Understanding buffering in TCP https://stackoverflow.com/questions/42857662/understanding-buffering-in-tcp

[3] 解决注册中心性能瓶颈：从Netty高水位线告警到优化方案 https://juejin.cn/post/7281499704219598911

[2] JDK 9新特性之Flow API 初探  https://juejin.cn/post/7104961299670368264
