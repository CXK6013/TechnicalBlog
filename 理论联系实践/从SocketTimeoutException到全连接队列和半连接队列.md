# 从SocketTimeoutException到全连接队列和半连接队列



## 前因

大概在一年半之前的时候，我们的应用的某个业务开始间歇报SocketTimeoutException, 不是前端调用我们发生SocketTimeoutException，而是我们用 HTTP Client中台拉取数据的时候，会偶尔报SocketTimeException, 这个偶尔可能是一个月报一次，也可能是两个月报一次，可能一个星期报两次，频率不固定，次数也不固定，当我第一次看到这个异常的时候，我的第一个反应就是用这个异常信息去搜索引擎上搜索解决方案，我并不理解这个异常说明了什么，但是按照我以往的经验来说，一般都有解决方案，对搜索引擎的方案一般都是延长超时时间，于是我延长了超时时间，但这并没有根本上解决问题，还是会出问题。我开始害怕，因为这个异常在测试环境上不管如何都复现出来，我参加工作一来，首次碰见这么吊诡的异常，无法复现，能够复现的bug都是好bug，刚开始的时候一般出了问题，一般我的经验是看日志都能很快的解决，或者也可以通过其他方法来验证自己的猜想，但是唯独这个异常，我不知所措，我之前面试的时候也背过三次握手，也学过Java 的原生Socket 编程，Netty，我背过Tomcat的acceptCount参数，但是碰到这个问题，这些知识仍然没有帮我解决问题，原因当时我网络的知识没有连接起来，他们孤零零的，向孤零零的神经元一样，没建立起来连接，最后这个问题开始让这些知识开始建立连接，成体系的发展。连接才是有价值的，

## 抽丝剥茧

我们这里尝试将上面的业务问题进行简化，首先我们借助Spring Boot 搭建项目，里面只选web的starter，然后我们简单写一个Controller:

```java
@RestController
public class SocketController {
    @GetMapping("hello-world")
    public String test(){
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return "hello-world";
    }
}
```

如你所见，这很简单，这里的逻辑也就是让处理请求的线程沉睡10s，返回hello-world, 我们现在用Apache HTTP Client 尝试调用这个接口:

```java
// 设置Socket读写时间
RequestConfig requestConfig = RequestConfig.custom().setSocketTimeout(2000).build();
// 构建请求
HttpUriRequest request = RequestBuilder.create("GET").setConfig(requestConfig).setUri("http://localhost:8080/socket/hello-world").build();
// 获取http client
CloseableHttpClient  httpClient = HttpClients.custom().build();
// 执行请求
CloseableHttpResponse response = httpClient.execute(request);
```

然后客户端就报这个错: 

```java
Exception in thread "main" java.net.SocketTimeoutException: Read timed out
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at org.apache.http.impl.conn.LoggingInputStream.read(LoggingInputStream.java:84)
	at org.apache.http.impl.io.SessionInputBufferImpl.streamRead(SessionInputBufferImpl.java:137)
	at org.apache.http.impl.io.SessionInputBufferImpl.fillBuffer(SessionInputBufferImpl.java:153)
	at org.apache.http.impl.io.SessionInputBufferImpl.readLine(SessionInputBufferImpl.java:280)
	at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:138)
	at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:56)
	at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:259)
	at org.apache.http.impl.DefaultBHttpClientConnection.receiveResponseHeader(DefaultBHttpClientConnection.java:163)
	at org.apache.http.impl.conn.CPoolProxy.receiveResponseHeader(CPoolProxy.java:157)
	at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:273)
	at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:125)
	at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:272)
	at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:186)
	at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:89)
	at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
	at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:83)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:108)
	at org.example.HttpClientDemo.main(HttpClientDemo.java:47)
```

所以直接一点的原因就是接口长时间没给到我们响应？我的解决方式也很简单，直接延长超时时间，心满意足关闭了bug，然后没好多久，问题依然发生了。那我们接着分析这个问题为什么会出现，我们知道JVM在执行GC的时候，会STW，所以有没有可能是调用的时候，对方请求刚好在GC呢，导致接口没在指定时间给到响应，观察监控发现调用失败发生的时候，并没有发生GC动作， 接着往下排除, 我们来分析当时的请求被处理的流程。被调用方的架构是:

![](https://a.a2k6.com/gerald/i/2023/07/23/xp3s.jpg) 

请求首先到达Nginx，然后负载均衡到Tomcat中，那么Tomcat是如何处理请求的呢，当时的Tomcat版本肯定是大于8的，那么想来Tomcat就是在NIO模式下面来处理请求的，那么一个请求在到达Tomcat之后是如何被Tomcat处理的呢？ 这点我们来结合Tomcat的源码来进行说明, 首先启动Tomcat之后，会有几个核心线程:

- Acceptor  线程
- 









## 参考资料

- Tomcat源码篇之HTTP请求处理流程  https://www.wormholestack.com/archives/649/
