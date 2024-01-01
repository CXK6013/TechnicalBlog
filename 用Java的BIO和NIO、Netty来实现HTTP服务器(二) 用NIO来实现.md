# 用Java的BIO和NIO、Netty来实现HTTP服务器(三)  用Netty实现



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

在Netty的世界里面是一个又一个多处理器，在启动的时候Netty会感知哪些处理器，在对应的事件触发之后就会触发对应的处理，像是一道链条一样:

![](https://a.a2k6.com/gerald/i/2024/01/01/m0il.jpg)

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

pipeline 是一个流水线，可以在这个上面不断的添加处理器，消息流经过后会被这些处理器处理，HttpServerCode实现对HTTP请求的解码和响应的编码，HttpContentCompressor实现对响应内容的压缩。 在HTTP中有一个独特的功能叫做，100 (Continue) Status，就是说client在不确定server端是否会接收请求的时候，可以先发送一个请求头，并在这个头上加一个"100-continue"字段，但是暂时还不发送请求body。直到接收到服务器端的响应之后再发送请求body。HttpServerExpectContinueHandler用于处理这个请求，消息经过HttpServerCodec、HttpContentCompressor、HttpServerExpectContinueHandler之后到达HttpHelloWorldServerHandler, 也就是我们的HttpHelloWorldServerHandler。

```java
public class HttpHelloWorldServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    
    private static final byte[] CONTENT = { 'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd' };

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            boolean keepAlive = HttpUtil.isKeepAlive(req);
            FullHttpResponse response = new DefaultFullHttpResponse(req.protocolVersion(), OK,Unpooled.wrappedBuffer(CONTENT));
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

## 参考资料

[1] 《Netty in Action笔记(二)》 https://fangjian0423.github.io/2016/08/29/netty-in-action-note2/
