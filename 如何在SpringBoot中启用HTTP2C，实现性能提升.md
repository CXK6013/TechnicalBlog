# 解锁网络性能优化利器HTTP2C(一) 绪论

> 我总要言说一些东西，因为我的心始终在喋喋不休。

[TOC]

## 前言

### HTTP的发展现状

最近脑海里面始终活跃着一些想法，一部分是对过去错误认知的纠正，比如HTTP/2。在《HTTP学习笔记(三) HTTP/2》，这里已经提过了，HTTP 1.0的性能缺点是每一个连接都对应一个TCP连接，到HTTP 1.1对这个问题进行了解决，也就是keep-alive和流水线，所谓keep-alive, 也就是说客户端和服务端请求维持这个TCP连接一段时间，这样有效的减少了频繁建立TCP连接的开销。

而流水线则是允许客户端在收到第一个响应之前，连续发送其他请求，这看起来是个不错的设计，有效的将请求报文传输并行化。但这一般是一个误解，英文原文是:

> HTTP pipelining is a way to send another request while waiting for the response to a previous request.

但其实表达的真实意思应该是将多个HTTP请求放到一个TCP连接中一一发送，而在发送过程中不需要等待服务器对前一个请求的响应。但是遗憾的是HTTP 1.1 要求，服务器必须严格按照接收到请求的相同顺序来回送HTTP响应。但就像在超市排队一样，如果队头的人买了很多东西，那么后续排队人都要在这里等待。

![](https://a.a2k6.com/gerald/i/2025/09/22/6e2fo.png)

当然你也可以和超市协商再起一个新队伍，即新建一个TCP连接。但不管怎么样，你总归得选择一个队伍，而且一旦选定之后，就不能更换队伍。但是新队伍也会导致资源耗费和性能损失。

我们分析一下HTTP 1.1 为什么要这么要求，原因在于如果不强制要求顺序，那接收响应的时候怎么知道对应的是哪个请求的呢？ 于是这些HTTP请求看起来还是串行处理，在一个TCP连接上。

### 管线化的问题

管线化的思路没什么问题，我们在RFC-2616，也就是参考文档[2]可以看到对管线化的论述:

>    Clients which assume persistent connections and pipeline immediately  after connection establishment SHOULD be prepared to retry their
>   connection if the first pipelined attempt fails. If a client does    such a retry, it MUST NOT pipeline before it knows the connection is   persistent.                  Clients MUST also be prepared to resend their requests if  the server closes the connection before sending all of the  corresponding responses

那些假定连接是持久的、并且在连接建立后立即使用流水线的客户端，应该准备好在第一次流水线尝试失败之后，重试他们与服务器之间的连接。如果客户端进行了这样的重试，那么在它确认该连接是持久的之前，客户端必须禁止再次使用流水线。如果客户端在发送完所有的响应之前就关闭了连接，客户端必须准备重发它们的请求。

注意这个持久连接，默认情况下，HTTP/1.0会在每次请求/响应交互关闭连接，这个连接是TCP连接，因此HTTP/1.0的持久连接必须经过明确协商。也就是请求头里面加入Connection: keep-alive来保持连接，连接的其他参数可以通过 keep-alive来指定，如果希望关闭连接，则是在请求标头里面加入Connection:close。这是http/1.0请求的默认值。如果在Http/1.1下面将会自动维持长连接，自动启用keep-alive。 

注意这里的话，我认为这个假设有点脆弱，原因在于没有经过假设，客户端只能通过猜测的方式来判断服务器是否支持这一特性，为什么这么说呢？ 原因在于我们考虑服务端早期对 http 1.0的支持，许多Http 1.1web服务器是从 1.0演变过来的，由于无法判断这一特性是否被这些服务器支持，客户端必须猜测这一特性是否被服务端支持。在参考资料可以看到，火狐浏览器为了支持这个特性做出的努力，通过尝试和维护黑名单(网站不支持加入黑名单，黑名单里面的网站默认不会开启这个特性)，最终还是发现风险大于收益。

举个例子，请求A、B、C依次到达代理服务器，假设代理服务器不支持这一特性，返回顺序是B、C、A，对于一些页面渲染就会出现问题。由此就印出来了HTTP/2的多路复用。

#### 多路复用解决了这个问题

##### 观测流水线

我们在这里再度明确一下，我们希望在发送请求的时候尽可能的降低延迟，但是遇到有依赖的网络资源的话，HTTP 1.1给出的方案是可以向服务端发送多个请求而不等待响应。一般我们用HttpClient发送请求的伪代码如下所示：

```java
Request request = new Request():
response = httpclient.send(request);
```

在Http 1.1下面我们可以写成下面这样:

```java
// 注意这里是伪代码
Request requestOne = new Request();
Request requestTwo = new Request():
List<Request> listRequest = new  ArrayList<>();
listRequest.add(requestOne);
listRequest.add(requestTwo);
List<Response> response = httpclient.send(request);
```

注意这里的核心问题在于，如何知道请求和响应之间的对应关系，HTTP /1.1的设计响应顺序即为发送请求的顺序。我们不妨看看一些Http Client是怎么实现你这个特性的，这里以Vertx为例我们来做个简单的分析, 首先我们需要引入Vertx:

```xml
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-web-client</artifactId>
    <version>5.0.4</version>
</dependency>
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-web</artifactId>
    <version>5.0.4</version>
</dependency>
```

注意如果你在Spring Boot 中写Vertx相关的代码，会有依赖冲突的问题，原因在于Spring Boot 锁定了Netty的版本，Vertx也依赖了Netty：

![](https://a.a2k6.com/gerald/i/2025/09/28/16x1v8.png)

而Vertx 依赖Netty的版本是4.2.5，所以这里要注意对齐Netty的版本, 所以这里要对齐Netty的版本，在maven里面声明一下:

```xml
<properties>
    <java.version>17</java.version>
    <netty.version>4.2.5.Final</netty.version>
</properties>
```

 首先我们用Vertx 写一个简单的WebServer:

```java
public class SimpleWebServer extends AbstractVerticle {
    @Override
    public void start(Promise<Void> startPromise) throws Exception {
        HttpServer server = vertx.createHttpServer();
        Router router = Router.router(vertx);
        router.get("/test-1").handler(ctx -> {
            try {
                // 注意这里的延时是为了测试队头阻塞问题,
                // 为了模拟队头阻塞问题,看响应是否按顺序返回
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            HttpServerResponse response = ctx.response();
            response.putHeader("content-type", "text/plain"); // 设置响应头
            response.end("你好，这里是 /test-1 的响应！");
        });

        router.get("/test-2").handler(ctx -> {
            HttpServerResponse response = ctx.response();
            response.putHeader("content-type", "application/json");
            JsonObject myJson = new JsonObject()
                    .put("message", "成功访问 /test-2")
                    .put("timestamp", System.currentTimeMillis());
            response.end(myJson.encodePrettily());
        });
        router.get("/test-3").handler(this::handleTest3);
        server.requestHandler(router);
        server.listen(8080,"localhost");
        super.start(startPromise);
    }

    private void handleTest3(RoutingContext ctx) {
        HttpServerResponse response = ctx.response();
        response.putHeader("content-type", "text/plain; charset=utf-8");
        response.end("你好, " + "! 欢迎来到 /test-3。");
    }

    public static void main(String[] args) {
        Vertx vertx = Vertx.vertx();
        vertx.deployVerticle(new SimpleWebServer());
    }
}
```

下面是客户端的代码:

```java
HttpClientOptions options = new HttpClientOptions()
        .setProtocolVersion(HttpVersion.HTTP_1_1)
        .setPipelining(true)
        .setPipeliningLimit(4);
Vertx vertx = Vertx.vertx(new VertxOptions().setWorkerPoolSize(40));
HttpClientAgent client = vertx.createHttpClient(options);
List<Future<HttpClientResponse>> futureList = new ArrayList<>();
RequestOptions requestOptionsOne = new RequestOptions()
        .setMethod(HttpMethod.GET)
        .setHost("localhost")
        .setPort(8080)
        .setURI("/test-1");

RequestOptions requestOptionsTwo = new RequestOptions()
        .setMethod(HttpMethod.GET)
        .setHost("localhost")
        .setPort(8080)
        .setURI("/test-2");
RequestOptions requestOptionsThree = new RequestOptions()
        .setMethod(HttpMethod.GET)
        .setHost("localhost")
        .setPort(8080)
        .setURI("/test-3");
List<RequestOptions> requestOptionsList = new ArrayList<>();
requestOptionsList.add(requestOptionsOne);
requestOptionsList.add(requestOptionsTwo);
requestOptionsList.add(requestOptionsThree);
for (int i = 0; i < 3; i++) {
    Future<HttpClientResponse> responseFuture = client.request(requestOptionsList.get(i)).compose(HttpClientRequest::send);
    futureList.add(responseFuture);
}
for (Future<HttpClientResponse> responseFuture : futureList) {
    HttpClientResponse clientResponse = responseFuture.await();
    Buffer bodyBuf = null;
    try {
        bodyBuf = clientResponse.body().toCompletionStage()
                .toCompletableFuture()
                .get();
        String body = bodyBuf.toString("UTF-8");
        System.out.println(body);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    } catch (ExecutionException e) {
        throw new RuntimeException(e);
    }
}
```

最终输出结果为:

```
你好，这里是 /test-1 的响应！
{
  "message" : "成功访问 /test-2",
  "timestamp" : 1759046207167
}
你好, ! 欢迎来到 /test-3。
```

可以看到Vertx的思路和我们想象的是一致的。 到现在我们总结一下，Http 1.0面临的问题，虽然在一个TCP连接上可以并发的发送报文，但是由于没有报文标识，报文请求和响应没办法形成对应关系，所以就只能要求请求报文排队被处理，然后按请求顺序返回。

那Http/2的解药就是改造Http 1.1的报文，为了解决请求和响应之间的映射关系，Http/2 为报文引入了标识符的概念:

>  Streams are identified with an unsigned 31-bit integer. 

 流由一个**无符号的31位整数**来标识

那什么是流？  在RFC 7540 我们可以看到对应的描述:

> A "stream" is an independent, bidirectional sequence of frames exchanged between the client and server within an HTTP/2 connection. 

一个“流”（Stream）是存在于一个HTTP/2连接内部的，客户端与服务器之间交换的**一个独立的、双向的帧（Frame）序列**

因为是服务器和客户端之间交换，所以是双向的，那帧是什么?  帧是HTTP/2的基本单元，**HEADERS（头部）帧**和**DATA（数据）帧**构成了HTTP请求和响应的基础，其他类型的帧，如**SETTINGS**、**WINDOW_UPDATE\**和\**PUSH_PROMISE**，则用于支持HTTP/2的其他各项功能。这也就是HTTP/2的另一个重要特性: 基于二进制的协议。

![](https://a.a2k6.com/gerald/i/2025/09/28/euc.png)

### 二进制的协议

熟悉Http协议的同学可能会有点印象，HTTP/1.1是基于文本的，那这个基于文本的是什么意思？ 底层不都是二进制嘛？ 在Java里面，我们可以通过String，将字符串转成字节数组。本质上就是二进制式的。那HTTP/2的二进制式是什么意思？我们在RFC-2626里面可以看待，想通的报文在1.1和2.2格式之间的区别:









### 那该如何兼容从前

现在我们是客户端要和服务端通信，由于HTTP/2和HTTP/1.1的报文格式都发生了改变，所以就需要确定通信的时候使用哪个版本进行通信。这在日常生活中也很常见，假设我们有一个支持120W的充电头，那给手机充电的时候，充电头会直接上120w的充电功率嘛？





#### Tomcat对流水线的支持

按道理测试应该结束了，但是我还想测试一下Tomcat对流水线的支持是怎么样的，Tomcat对流水线的支持见参考链接[5]关于maxKeepAliveRequests的说明:

The maximum number of HTTP requests which can be pipelined until the connection is closed by the server.

直到连接被关闭之前，这个连接可以被管线化发送HTTP请求的最大数量。

Setting this attribute to 1 will disable HTTP/1.0 keep-alive, as well as HTTP/1.1 keep-alive and pipelining. 

将此属性设置为1将会禁用HTTP/1.0的keep-alive功能，以及Http/1.1的keep-alive和流水线功能。

Setting this to -1 will allow an unlimited amount of pipelined or keep-alive HTTP requests. If not specified, this attribute is set to 100.

将这个值设置

## 如何在Spring Boot 中启用HTTP/2c

### 如何观测和验证



### JDK 1.8 以上





### JDK 1.8



## 现在大型网站的架构









## 参考资料

[1] HTTP的现状 https://http2-explained.haxx.se/zh/part2

[2] https://datatracker.ietf.org/doc/html/rfc2616#section-19.2  

[3] https://bugzilla.mozilla.org/show_bug.cgi?id=264354

[4] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Keep-Alive

[5]  https://tomcat.apache.org/tomcat-8.0-doc/config/http.html#Proxy_Support
