# HTTP Client 学习笔记 (一) 初遇篇

> 因为经常调用第三方接口，HttpClient是我经常使用的框架之一，这里打算系统的学习一下，同时呼应《HTTP协议学习笔记(一) 初遇篇》，一边是理论，一边是实践。同时也是在JDK8停留很久了，打算学习一下新版本的JDK特性，我注意到JDK 11也有一个HTTP Client，本篇我们的关注点构建HTTP请求，发出请求，然后解析响应。这篇文章也换一种风格。

[TOC]

## 从一个任务开始说起

我们故事的主人公叫小陈，目前还是一个实习生, 刚进公司安排的第一个需求是定时任务调第三方的接口去拉取数据到指定的表里，小陈想到之前的博客有教Apache HttpClient示例的, 于是写出了如下拉数据的代码: 

```java
  @Override
 public void collectData() {
        StringEntity stringEntity = new StringEntity("", ContentType.APPLICATION_JSON);
        HttpUriRequest request = RequestBuilder.post("").addHeader("请求头", "请求尾").addParameter("key", "value").setEntity(stringEntity).build();
        CloseableHttpClient httpClient = HttpClients.createDefault();
        try {
            CloseableHttpResponse response = httpClient.execute(request);
            // 拿到响应
            String responseString = EntityUtils.toString(response.getEntity());
            // 假设拿到了需要调用对面接口的次数
            for (int i = 0; i < Integer.parseInt(responseString) ; i++) {
                 httpClient = HttpClients.createDefault();
                 response = httpClient.execute(request);
                 // 做对应的业务处理
            }
        } catch (IOException e) {
			// 日志暂时省略。
        }
   }
```

小陈觉得自己完成了这个需求，就去找领导看看代码，毕竟是实习生嘛，公司还是要把控一下代码质量的, 领导看了之后，产生了如下对话：

领导: 小陈啊, HTTP协议是基于应用层的哪个协议啊？

小陈心想这个我熟，这个我面试的背过，我狠下了一番功夫在三次握手和四次握手上, 于是答到: HTTP 是基于TCP的。

领导接着问: 那么这个HttpClient是怎么使用TCP协议的呀？

小陈心里在偷偷的乐，还好我关注了公众号爱喝汤的技术少年，看了公众号的爱喝汤的技术少年《TCP学习笔记(一) 初遇篇》、《计算机网络引论》，于是

答到:  TCP/IP协议已经驻留在现代操作系统中，在Java中，这个Apache HttpClient主要通过调用Java提供Socket相关的类，来实现调用操作系统的TCP/IP协议族，

需要使用网络通信的时候，最终的调用到操作系统，操作系统会为该进程创建一个Socket(套接字)，操作系统会为该套接字分配相关的资源(存储空间，带宽等)

领导，说到这里，我必须画个图来彰显我对网络相关的水平: 

![HttpClient 调用](http://tva4.sinaimg.cn/large/006e5UvNly1h2fvshseajj30go0k4gmx.jpg)

领导笑了笑说道: 那你在循环里产生HttpClient，有没有这种可能，循环次数过多的时候会大量占用操作系统的Socket资源呢？你看看不断的创建CloseableHttpClient对象，会发生些什么？

小陈想了想：是的，那我在循环外面用？ 

领导接着又说: 那有看过Apache HttpClient的文档吗？ 这个CloseableHttpClient有没有可能是线程安全的呢？

小陈恍然大悟说道：那我把它加进IOC容器里，这样我们整个系统就都可以使用这个HttpClient了，节省资源。

领导接着说: 那还有别的问题没有了啊，好好想想哦。

小陈说: 我想不到了诶。

领导说道：通信过程有没有可能失败啊，假设某个时刻，网络比较拥堵，那HttpClient有没有可能失败啊？

小陈想了想，好像会，又问道： 那我们加重试？ 但是该怎么加比较优雅啊？ 我目前想到的就是调用的时候用try catch，如果catch到异常了，然后for循环重试次数。

领导笑了笑说道： 有没有这样一种可能，这个Apache HttpClient带有重试器啊，你下去查一下，然后改一下我们再看看？

小陈说：好的。

首先小陈打开了CloseableHttpClient的源码:

![CloseableHttpClient](http://tva1.sinaimg.cn/large/006e5UvNly1h2fwics45vj30q40gkdnq.jpg)

看来领导说的没毛病，这个类看来是线程安全的。想到领导说到了这个Apache的HttpClient文档,  于是打开了百度，在搜索框上输入了Apache HttpClient:

![Apache HttpClient Document](http://tva4.sinaimg.cn/large/006e5UvNgy1h2fwpsb5byj312i0hyn9i.jpg)



![教程](http://tvax1.sinaimg.cn/large/006e5UvNgy1h2fwr2r8raj31580mdas1.jpg)

![retry的重试器](http://tva2.sinaimg.cn/large/006e5UvNgy1h2fwt38lvij315w0p9165.jpg)



![retry示例](http://tvax3.sinaimg.cn/large/006e5UvNgy1h2fwu66j8aj31cv0jyagi.jpg)

小陈觉得都被满足了, 于是写出了如下的代码: 

```java
@Configuration
public class HttpClientConfiguration {
    @Bean
    public CloseableHttpClient closeableHttpClient(){
        HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {
            @Override
            public boolean retryRequest(
                    IOException exception,
                    int executionCount,
                    HttpContext context) {
                if (executionCount >= 5) {
                    // Do not retry if over max retry count
                    return false;
                }
                if (exception instanceof InterruptedIOException) {
                    // Timeout
                    return false;
                }
                if (exception instanceof UnknownHostException) {
                    // Unknown host
                    return false;
                }
                if (exception instanceof ConnectTimeoutException) {
                    // Connection refused
                    return false;
                }
                if (exception instanceof SSLException) {
                    // SSL handshake exception
                    return false;
                }
                HttpClientContext clientContext = HttpClientContext.adapt(context);
                HttpRequest request = clientContext.getRequest();
                // idempotent 幂等
                boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
                if (idempotent) {
                    // Retry if the request is considered idempotent
                    return true;
                }
                return false;
            }

        };
        CloseableHttpClient httpClient = HttpClients.custom().setRetryHandler(myRetryHandler).build();
        return httpClient;
    }
}
 @Override
 public void collectData() {
        StringEntity stringEntity = new StringEntity("", ContentType.APPLICATION_JSON);
        HttpUriRequest request = RequestBuilder.post("").addHeader("请求头key", "请求value").addParameter("key", "value").setEntity(stringEntity).build();
        try {
            CloseableHttpResponse response = closeableHttpClient.execute(request);
            // 拿到响应
            String responseString = EntityUtils.toString(response.getEntity());
            // 假设拿到了需要调用对面接口的次数
            for (int i = 0; i < Integer.parseInt(responseString); i++) {
                closeableHttpClient = HttpClients.createDefault();
                response = closeableHttpClient.execute(request);
                // 做对应的业务处理
            }
        } catch (IOException e) {

        }
   }
```

然后把领导叫了过来，问道：大佬，再审一下我的代码呗。

领导点了点头，说道： 重试，都做好了，还可以。那下一个问题，调用次数过多的情况下，这个CloseableHttpClient会为每一个HttpClient连接开辟一个TCP连接吗?  如果是的话是不是有点奢侈了啊，要不再看看源码。

小陈想了想说道: 是HTTP1.1的keep-alive吗？ 

领导笑道: 是的，其实还有一个问题，如果请求了接口，请求了四五千次，那这个for循环是不是有点费时间了啊？

小陈恍然大悟: 那这里我开个线程池来做一下，HttpClient做一个连接池，做到像数据库连接池那样复用？

领导点了点头，说道：那你再改一下吧。

于是小陈再次来到了Apache Httpclient 官方网站: 

![keep-alive策略](http://tva2.sinaimg.cn/large/006e5UvNly1h2fx5kgc79j31040nik33.jpg)

![调用示例](http://tvax1.sinaimg.cn/large/006e5UvNly1h2fx6oql37j31f60ekq7e.jpg)

看到了这里长连接的问题算是解决了，那连接池呢，天呐这不会让我自己写一个连接池吧，小陈想Apache HttpClient 里面肯定也有于是就接着翻文档：

![连接管理](http://tvax3.sinaimg.cn/large/006e5UvNly1h2fx9i52emj313l0khk19.jpg)

![连接池池化](http://tva3.sinaimg.cn/large/006e5UvNgy1h2fxepz8z1j31g50fm4gj.jpg)

然后最终的程序就被改成了这个样子:

```java
@Configuration
public class HttpClientConfiguration {
    @Bean
    public CloseableHttpClient closeableHttpClient(){
        HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {

            @Override
            public boolean retryRequest(
                    IOException exception,
                    int executionCount,
                    HttpContext context) {
                // 返回true 就代表重试
                if (executionCount >= 5) {
                    // Do not retry if over max retry count
                    return false;
                }
                if (exception instanceof InterruptedIOException) {
                    // Timeout
                    return false;
                }
                if (exception instanceof UnknownHostException) {
                    // Unknown host
                    return false;
                }
                if (exception instanceof ConnectTimeoutException) {
                    // Connection refused
                    return false;
                }
                if (exception instanceof SSLException) {
                    // SSL handshake exception
                    return false;
                }
                HttpClientContext clientContext = HttpClientContext.adapt(context);
                HttpRequest request = clientContext.getRequest();
                boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
                if (idempotent) {
                    // Retry if the request is considered idempotent
                    return true;
                }
                return false;
            }

        };
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        // Increase max total connection to 200
        cm.setMaxTotal(200);
        // Increase default max connection per route to 20
        cm.setDefaultMaxPerRoute(20);
        // Increase max connections for localhost:80 to 50
        HttpHost localhost = new HttpHost("locahost", 80);
        cm.setMaxPerRoute(new HttpRoute(localhost), 50);
        CloseableHttpClient httpClient = HttpClients.custom().setRetryHandler(myRetryHandler).setConnectionManager(cm).build();
        return httpClient;
    }
}
```

关于多次请求，小陈打算做个线程池来并发处理请求, 于是程序最终就变成了下面这个样子：

```java
@Override
 public void collectData() {
        StringEntity stringEntity = new StringEntity("", ContentType.APPLICATION_JSON);
        HttpUriRequest request = RequestBuilder.post("").addHeader("请求头", "请求尾").addParameter("key", "value").setEntity(stringEntity).build();
        try {
            CloseableHttpResponse response = closeableHttpClient.execute(request);
            // 拿到响应
            String responseString = EntityUtils.toString(response.getEntity());
            // 假设拿到了需要调用对面接口的次数
            for (int i = 0; i < Integer.parseInt(responseString); i++) {
                POOL_EXECUTOR.submit(()->{
                    try {
                        CloseableHttpResponse threadResponseString = closeableHttpClient.execute(request);
                    } catch (IOException e) {
                        
                    }
                });
               
                // 做对应的业务处理
            }
        } catch (IOException e) {

        }
  }
```

于是小陈再次找到了领导，请他再次审阅自己的代码，领导看了看，点了点头：还可以，勉强通过了，这次的任务算你完成，那再去了解一下JDK 11中的HttpClient吧，这次你的任务是对JDK 11中新的HttpClient有一个大致了解，基本会用即可，主要的目的是为了让你看下不同HttpClient的实现。

### JDK 11的HTTP Client

小陈得到领导分配的任务，首先在百度搜索open jdk, open jdk会有对JDK 11新特性的说明: 

![open jdk](http://tva2.sinaimg.cn/large/006e5UvNgy1h2fxsshdipj312y0gv47r.jpg)





![JDK11的说明](http://tva1.sinaimg.cn/large/006e5UvNgy1h2fxtwx129j30rd0me445.jpg)



![JDK 11 ](http://tva1.sinaimg.cn/large/006e5UvNgy1h2fxv1jn1gj30v90gkwmc.jpg)



![HTTP Client的说明](http://tvax1.sinaimg.cn/large/006e5UvNgy1h2fxvqms19j30p30h1jzy.jpg)



下面是对JDK 11 对这个特性的说明

The existing `HttpURLConnection` API and its implementation have numerous problems:

- The base `URLConnection` API was designed with multiple protocols in mind, nearly all of which are now defunct (`ftp`, `gopher`, etc.).

- The API predates HTTP/1.1 and is too abstract.
- It is hard to use, with many undocumented behaviors.
- It works in blocking mode only (i.e., one thread per request/response).
- It is very hard to maintain.

已有的HttpURLConnection API实现上存在许多问题:

- URLConnection 为多种协议所设计，但是当初的那些协议大多都不存在了
- 这个接口早于HTTP Client， 但是太抽象了。
- 很难用，并且有些行为没有没被注释到
- 只有阻塞模式(为每对请求和响应一个线程)

小陈看完了这个说明，心中的第一个想法就是，我该怎么用JDK 11的HttpClient.

![HttpClient基本示例](https://tva3.sinaimg.cn/large/006e5UvNgy1h2i759slihj30tl0lph3f.jpg)

JDK 11是这么介绍新的HttpClient的:

The HTTP Client was added in Java 11. It can be used to request HTTP resources over the network. It supports *HTTP/1.1* and *HTTP/2*, both synchronous and asynchronous programming models, handles request and response bodies as reactive-streams, and follows the familiar builder pattern.

这个新实现的HTTP Client在Java 11被引入，可以在网络中用作请求HTTP 资源，支持HTTP1.1、HTTP/2, 同步和异步模式。处理请求和响应流支持响应流模式，也能用熟悉方式构建。

- 示例一解读：

```java
 public static void main(String[] args) {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("http://openjdk.java.net/"))
                .build();
        // 这是一个异步请求
        // 处理响应将响应当作一个字符串,
        // 返回结果CompletableFuture对象,不知道CompletableFuture
     	// 可以去翻一下我之前写的《Java多线程学习笔记(六) 长乐未央篇》
     	// thenApply 收到线程的时候 消费HttpResponse的数据,最终被
     	// thenAccept所处理,join同Thread.join方法一样,
     	//CompletableFuture链式处理数据完毕才会走到下一行
        client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body)
                .thenAccept(System.out::println).join();
    }
}
```

上面不是说支持HTTP/1.1 和 HTTP/2吗，那我该如何使用, 请求参数该如何添加呢? 

```java
// 通过Version字段指定
HttpClient client = HttpClient.newBuilder()
      .version(Version.HTTP_2)
```

Http请求的构建主要借助于 HttpRequest.newBuilder()来构建,newBuilder最终指向了HttpRequestBuilderImpl，我们来看HttpRequestBuilderImpl这个类能帮助我们构建什么参数:

```java
HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("http://openjdk.java.net/")) // URI 是地址
      .timeout(Duration.ofMinutes(1)) // 超时时间
      .header("Content-Type", "application/json") // 请求头
      .POST(BodyPublishers.ofFile(Paths.get("file.json"))) // 参数主要通过BodyPublishers来构建
      .build()
```

看到这里小陈在想，JDK 11的HttpClient为什么没有Apache HttpClient的构建请求时的addParameter吗？查阅诸多资料都没找到paramsPushlisher这个操作，但看起来是需要我们自己去拼接在URL上。构建请求体HttpClient给我们提供了BodyPublishers来进行构建:

```java
HttpRequest.BodyPublishers::ofByteArray(byte[]) // 向服务端发送字节数组
HttpRequest.BodyPublishers::ofByteArrays(Iterable)
HttpRequest.BodyPublishers::ofFile(Path) // 发送文件
HttpRequest.BodyPublishers::ofString(String) // 发送String 
HttpRequest.BodyPublishers::ofInputStream(Supplier<InputStream>) // 发送流
```

发送POST请求：

```java
// HttpResponse.BodyHandlers 用来处理响应体中的数据,ofString,将响应体中的数据处理成String
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
.thenApply(HttpResponse::body)
thenAccept(System.out::println).join();
```

到这里还没有发现JDK 11的HttpClient是怎么管理连接的,  在StackOverFlow上也有人问这个问题, 这是JDK的默认策略。 但是有的请求如果不需要keep-alive呢，这似乎就得通过改整个JVM的属性来实现，还有该如何实现重试，在这里也没找到。重试在某些场景下还是很重要的，如果用JDK 11自带的还需要自己再包装一下。在这个JEP下面也谈到了Apache HttpClient:

```
A number of existing HTTP client APIs and implementations exist, e.g., Jetty and the Apache HttpClient. Both of these are both rather heavy-weight in terms of the numbers of packages and classes, and they don't take advantage of newer language features such as lambda expressions.
已经存在了一些HTTP Client库，Jetty和Apache HttpClient，就代码量来说这两个都相当的庞大，他们也没有用到JDK的新特性比如Lamda表达式。
```

## 写在最后

本篇基本介绍了Apache HttpClient 和 JDK 11 的Httpclient中的基本使用，目前大致来看似乎Apache HttpClient的完整度更高一些，但是JDK 11的实现也有亮点，像是对响应的封装也十分让人心动。

## 参考资料

- How to keep connection alive in Java 11 http client  https://stackoverflow.com/questions/53617574/how-to-keep-connection-alive-in-java-11-http-client
- Introduction to the Java HTTP Client  https://openjdk.java.net/groups/net/httpclient/intro.html
