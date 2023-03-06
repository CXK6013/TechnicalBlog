# 异步Servlet学习笔记(一)

> 两周没更新了，感觉不写点什么，有点不舒服的感觉。

## 前言

回忆一下学Java的历程，当时是看JavaSE(基本语法、线程、泛型)，然后是JavaEE，JavaEE也基本就是围绕着Servlet的使用、JSP、JDBC来学习，当时看的是B站up主颜群的教学视频:

- JavaWeb视频教程（JSP/Servlet/上传/下载/分页/MVC/三层架构/Ajax）https://www.bilibili.com/video/BV18s411u7EH?p=6&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799

现在一看这个播放量破百万了，当初我看的时候应该播放量很少，现在这么多倒是有点昨舌。学完了这个之后，开始学习框架：Spring、SpringMVC、MyBatis、SpringBoot。虽然Spring MVC本质上也是基于Servlet做封装，但后面基本就转型成Spring 工程师了，最近碰到一些问题，又看了一篇文章，觉得一些问题之前自己还是没考虑到，颇有种离了Spring家族，不会写后端一样。本来今天的行文最初是什么是异步Servlet，异步Servlet该如何使用。但是想想没有切入本质，所以将其换成了对话体。

## 正文

我们接着有请之前的实习生小陈，每当我们需要用到对话体、故事体这样的行文。实习生小陈就会出场。今天的小陈呢觉得行情有些不好，但是还是觉得想出去看看，毕竟金三银四，于是下午就向领导请假去面试了。进到面试的地方，一番自我介绍，面试官首先问了这样一个问题:

> 一个请求是怎么被Tomcat所处理的呢？

小陈回答到:

> 我目前用的都是Spring Boot工程，我看都是启动都是在main函数里面启动整个项目的，而main函数又被main线程执行，所以我想应该是请求过来之后，被main线程所处理，给出响应的。

面试官:

> ╮(╯▽╰)╭，main函数的确是被main线程执行，但都是被main线程处理的？ 这不合理吧，假设某个请求占用了main线程三秒，那这三秒内，系统都无法再回应请求了。你要不再想想？

小陈挠了挠头，接着答到:

> 确实是，浏览器和Tomcat通讯用的是HTTP协议，我也学过网络编程，所以我觉得应该是一个线程一个请求吧。像下面这样:

```java
public class ServerSocketDemo {

    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(4);

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        while (true){
            // 一个socket对象代表一个连接
            // 等待TCP连接请求的建立,在TCP连接请求建立完成之前,会陷入阻塞
            Socket socket = serverSocket.accept();
            System.out.println("当前连接建立:"+ socket.getInetAddress().getHostName()+socket);
            EXECUTOR_SERVICE.submit(()->{
                try {
                    // 从输入流中读取客户端发送的内容
                    InputStream inputStream = socket.getInputStream();
                    // 从输出流里向客户端写入数据
                    OutputStream outPutStream = socket.getOutputStream();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
    }
}
```

> serverSocket的accept在连接建立起来会陷入阻塞。

面试官点了点头, 接着问到:

> 你这个main线程负责检测连接是否建立，然后建立之后将后续的业务处理放入线程池，这个是NIO吧。

小陈笑了笑说道:

> 虽然我对NIO了解不多，但这应该也不是NIO，因为后面的线程在等待数据可读可写的过程中会陷入阻塞。在操作系统中，线程是一个相当昂贵的资源，我们一般使用线程池，可以让线程的创建和回收成本相对较低，在活动连接数不是特别高的情况下(单机小于1000)，这种，模型是比较不错的，可以让每一个连接专注于自己的I/O并且编程模型简单。但要命的就是在连接上来之后，这种模型出现了问题。我们来分析一下我们上面的BIO模型存在的问题，主线程在接受连接之后返回一个Socket对象，将Socket对象提交给线程池处理。由这个线程池的线程来执行读写操作，那事实上这个线程承担的工作有判断数据可读、判断数据可写，对可读数据进行业务操作之后，将需要写入的数据进行写入。 那陷入阻塞的就是在等待数据可写、等待数据可读的过程，在NIO模型下对原本一个线程的任务进行了拆分，将判断可读可写任务进行了分离或者对原先的模型进行了改造，原先的业务处理就只做业务处理，将判断可读还是可写、以及写入这个任务专门进行分离。

> 我们将判断可读、可写、有新连接建立的线程姑且就称之为I/O主线程吧，这个主线程在不断轮询这三个事件是否发生，如果发生了就将其就给对应的处理器。这也就是最简单的Reactor模式: 注册所有感兴趣的事件处理器，单线程轮询选择就绪事件，执行事件处理器。

> 现在我们就可以大致总结出来NIO是怎么解决掉线程的瓶颈并处理海量连接的: 由原来的阻塞读写变成了单线程轮询事件，找到可以进行读写的网络描述符进行读写。出了事件的轮询是阻塞的(没有满足的事件就必须要要阻塞)，剩余的I/O操作都是纯CPU操作，没有必要开启多线程。

面试官点了点头，说道:

> 还可以嘛，小伙子，刚刚问怎么卡(qia)壳了？

小陈不好意思的挠挠头, 笑道:

> 其实之前看过这部分内容，只不过可能知识不用就想不起来，您提示了一下，我才想起来。

面试官笑了一下，接着问:

> 那现在的服务器，一般都是多核处理，如果能够利用多核心进行I/O， 无疑对效率会有更大的提高。 能否对上面的模型进行持续优化呢？

小陈想了想答道：

> 仔细分一下我们需要的线程，其实主要包括以下几种:
>
> 1. 事件分发器，单线程选择就绪的事件。
> 2. I/O处理器，包括connect、read、writer等，这种纯CPU操作，一般开启CPU核心个线程就可以了
> 3. 业务线程，在处理完I/O后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等。只要有阻塞，就需要单独的线程。

面试官点了点头，接着问道: 

> 不错，不错。那Java的NIO知道嘛。

小陈点了点头说道：

> 知道，Java引入了Selector、Channel 、Buffer，来实现我们新建立的模型，Selector字面意思是选择器，负责感应事件，也就是我们上面提到的事件分发器。Channel是一种对I/O操作的抽象，可以用于读取和写入数据。Buffer则是一种用于存储数据的缓冲区，提供统一存取的操作。

面试官又问道: 

> 有了解过Java的Selector在Linux系统下的限制嘛？ 

小陈答道:

> Java的Selector对于Linux系统来说，有一个致命的限制: 同一个channel的select不能被并发的调用。因此，如果有多个I/O线程，必须保证: 一个socket只能属于一个IO线程，而一个IO线程可以管理多个socket。

面试官点了点头:

> 不错，不错。Tomcat有常用的默认配置参数有: acceptorThreadCount 、 maxConnections、maxThreads、 。解释一下这几个参数的意义，并且给出一个请求在到达Tomcat之后是怎么被处理的，要求结合Servlet来进行说明。

小陈沉思了一下道:

> acceptorThreadCount  用来控制接收连接的线程数，如果服务器是多核心，可以调大一点。但是Tomcat的官方文档建议不要超过2个。控制接收连接这部分的代码在Acceptor这个类里，你可以看到这个类是Runnable的实现类。在Tomcat的8.0版本，你还能查到这个参数的说明，但是在8.5这个版本就查不到，我没找到对应的说明，但是在Tomcat 9.0源码的AbstractProtocol类中的setAcceptorThreadCount方法可以看到，这个参数被废弃，上面还有说明，说这个参数将在Tomcat的10.0被移除。

面试官道：

> 

小陈思索了一下回答到: 

> 再引出 异步Servlet 请求线程。然后如何使用

## 参考资料

- Java NIO浅析 https://zhuanlan.zhihu.com/p/23488863
- 深度解读 Tomcat 中的 NIO 模型 https://klose911.github.io/html/nio/tomcat.html



