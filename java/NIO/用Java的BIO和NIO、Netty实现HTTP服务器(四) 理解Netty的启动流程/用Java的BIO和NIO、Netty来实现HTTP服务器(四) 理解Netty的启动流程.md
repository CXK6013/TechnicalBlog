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

![](https://a.a2k6.com/gerald/i/2024/03/10/5j8tr.jpg)

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

##  参考资料

[1]  Client connects twice when using Postman to send requests? https://stackoverflow.com/questions/71088167/client-connects-twice-when-using-postman-to-send-requests

[2] JDK 9新特性之Flow API 初探 https://juejin.cn/post/7104961299670368264