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

> 现在我们就可以大致总结出来NIO是怎么解决掉线程的瓶颈并处理海量连接的: 由原来的阻塞读写变成了单线程轮询事件，找到可以进行读写的网络描述符进行读写。除了事件的轮询是阻塞的(没有满足的事件就必须要要阻塞)，剩余的I/O操作都是纯CPU操作，没有必要开启多线程。

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

> 不错，不错。Tomcat有常用的默认配置参数有: acceptorThreadCount 、 maxConnections、maxThreads 。解释一下这几个参数的意义，并且给出一个请求在到达Tomcat之后是怎么被处理的，要求结合Servlet来进行说明。

小陈沉思了一下道:

> acceptorThreadCount  用来控制接收连接的线程数，如果服务器是多核心，可以调大一点。但是Tomcat的官方文档建议不要超过2个。控制接收连接这部分的代码在Acceptor这个类里，你可以看到这个类是Runnable的实现类。在Tomcat的8.0版本，你还能查到这个参数的说明，但是在8.5这个版本就查不到，我没找到对应的说明，但是在Tomcat 9.0源码的AbstractProtocol类中的setAcceptorThreadCount方法可以看到，这个参数被废弃，上面还有说明，说这个参数将在Tomcat的10.0被移除。maxConnections用于控制Tomcat能够承受的TCP连接数，当达到最大连接数时，操作系统会将请求的连接放入到队列里面，这个队列的数目由acceptCount这个参数控制，默认值为100，如果超过了操作系统能承受的连接数目，这个参数也会不起作用，TCP连接会被操作系统拒绝。maxConnections在NIO和NIO2下, 默认值是10000，在APR/native模式下，默认值是8192.  

> maxThreads控制最大线程数，一个HTTP请求默认会被一个线程处理，也就是一个Servlet一个线程，可以这么理解maxThreads的数目决定了Tomcat能够同时处理的HTTP请求数。默认为200。

面试官似乎很满意，点了点头，接着道：

> 小伙子，看的还挺多，NIO上面你已经讲了, NIO2和APR是什么，你有了解过嘛？

小陈思索了一下回答到: 

> 我先来介绍APR吧，APR是 Apache Portable Runtime的缩写，是一个为Tomcat提供扩展能力的库，之所以带上native的原因是APR不使用Java编写的连接器，而是选择直接调用操作系统，避免了JVM级别的开销，理论上性能会更好。NIO2增强了NIO，我们先在只讨论网络方面的增强，NIO上面我们是启用了轮询来判断对应的事件是否可以进行，NIO2则引入了异步IO，我们不用再轮询，只用接收操作系统给我们的通知。

面试官: 

> 现在我们将上面的问题连接在一起，向Tomcat应用服务器发出HTTP请求，在NIO模式下，这个请求是如何被Tomcat所处理的。

小陈道: 

> 请求会首先到达操作系统，建立TCP连接，这个过程由操作系统完成，我们暂时忽略，现在这个连接请求完成到达了Acceptor(连接器)，连接器在NIO模式下会借助NIO中的channel，将其设置为非阻塞模式，然后将NioChannel注册到轮询线程上，轮询工作由Poller这个类来完成，然后由Poller将就绪的事件生成SocketProcessor, 交给Excutor去执行，Excutor这是一个线程池，线程池的大小就是在Connector 节点配置的 maxThreads 的值，这个线程池处理的任务为:
>
> 1. 从socket中读取http request
> 2. 解析生成HttpServletRequest对象
> 3. 分派到相应的servlet并完成逻辑
> 4. 将response通过socket发回client。

面试官:

> 这个线程池，你有了解过嘛？

小陈道: 

> 这个线程池不是JDK的线程池，继承了JDK的ThreadPoolExecutor, 自身做了一些扩写，我看网上的一些博客是说的是这个ThreadPoolExecutor跟JDK的ThreadPoolExecutor行为不太一致，JDK里面的ThreadPoolExecutor在接收到任务的时候是，看当前线程池活跃的线程数目是否小于核心线程数，如果小于就创建一个线程来执行当前提交的任务，如果当前活跃的线程数目等于核心线程数，那么就将这个任务放到阻塞队列中，如果阻塞队列满了，判断当前活跃的线程数目是否到达最大线程数目，如果没达到，就创建新线程去执行提交的任务。当任务处理完毕，线程池中活跃的线程数超过核心线程池数，超出的在存活keepAliveTime和unit的时间，就会被回收。 简单的说，就是JDK的线程池是先核心线程，再队列，最后是最大线程数。我看到的一些博客说Tomcat是先核心线程，再最大线程数，最后是队列。但是我看了Tomcat的源码，在StandardThreadExecutor执行任务的时候还是调用父类的方法，这让我很不解，先核心线程，再最大线程数，最后是队列，这个结论是怎么得出来的。

面试官点了点头:

> 还不错，蛮有实证精神的，看了博客还会自己去验证。我还是蛮欣赏你的，你过来一下，我们看着源码看看能不能得出这个结论: 

```java
@Override
protected void startInternal() throws LifecycleException {

        taskqueue = new TaskQueue(maxQueueSize);
        TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());
    executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), maxIdleTime, TimeUnit.MILLISECONDS,taskqueue, tf);
        executor.setThreadRenewalDelay(threadRenewalDelay);
        if (prestartminSpareThreads) {
            executor.prestartAllCoreThreads();
        }
        taskqueue.setParent(executor);

        setState(LifecycleState.STARTING);
 }
```

> 你说的那个线程池在StandardThreadExecutor这个类的startInternal里面被初始化，我们看看有没有什么生面孔，恐怕唯一的生面孔就是这个TaskQueue,我们简单的看下这个队列。从源码里面我们可以看出来，这个类继承了LinkedBlockingQueue，我们重点看入队和出队的方法

```java
@Override
public boolean offer(Runnable o) {
  //we can't do any checks
    if (parent==null) {
        return super.offer(o);
    }
    //we are maxed out on threads, simply queue the object
    if (parent.getPoolSize() == parent.getMaximumPoolSize()) {
        return super.offer(o);
    }
    //we have idle threads, just add it to the queue
    if (parent.getSubmittedCount()<=(parent.getPoolSize())) {
        return super.offer(o);
    }
    //if we have less threads than maximum force creation of a new thread
    if (parent.getPoolSize()<parent.getMaximumPoolSize()) {
        return false;
    }
    //if we reached here, we need to add it to the queue
    return super.offer(o);
}
```

```java
  @Override
 public Runnable poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        Runnable runnable = super.poll(timeout, unit);
        if (runnable == null && parent != null) {
            // the poll timed out, it gives an opportunity to stop the current
            // thread if needed to avoid memory leaks.
            parent.stopCurrentThreadIfNeeded();
        }
        return runnable;
    }

    @Override
    public Runnable take() throws InterruptedException {
        if (parent != null && parent.currentThreadShouldBeStopped()) {
            return poll(parent.getKeepAliveTime(TimeUnit.MILLISECONDS),
                    TimeUnit.MILLISECONDS);
            // yes, this may return null (in case of timeout) which normally
            // does not occur with take()
            // but the ThreadPoolExecutor implementation allows this
        }
        return super.take();
    }

```

通过上文我们可以知道，如果在线程池的线程数量和最大线程数相等，才会入队。当前未完成的任务小于当前线程池的线程数目也会入队。如果当前线程池的线程数目小于最大线程数，入队失败返回false。Tomcat的ThreadPoolExecutor继承了JDK的线程池，但在执行任务的时候依然调用的是父类的方法，看下面的代码:

```java
  public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet();
        try {
            super.execute(command);
        } catch (RejectedExecutionException rx) {
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    if (!queue.force(command, timeout, unit)) {
                        submittedCount.decrementAndGet();
                        throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
                    }
                } catch (InterruptedException x) {
                    submittedCount.decrementAndGet();
                    throw new RejectedExecutionException(x);
                }
            } else {
                submittedCount.decrementAndGet();
                throw rx;
            }

        }
   }
```

所以我们还是要进JDK的线程池看这个execute方法是怎么执行的:

```java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

这个代码也比较直观，不如你提交了一个null值，抛空指针异常。然后判断当前线程池的线程数是否小于核心线程数，小于则添加线程。如果不小于核心线程数，判断当前线程池是否还在运行，如果还在运行，就尝试将任务添加进队列，走到这个判断说明当前线程池的线程已经达到核心线程数，但是还小于最大线程数，然后TaskQueue返回false，就接着向线程池添加线程。那么现在整个Tomcat处理请求的流程，我们心里就大致有数了，现在我想问一个问题，现在已知的是，我可以认为执行我们controller方法的是线程池的线程，但是如果方法里面执行时间比较长，那么线程池的线程就会一直被占用，我们的系统现在随着业务的增长刚好面临着这样的问题，一些文件上传碰上流量高峰期，就会一直占用这个线程，导致整个系统处于一种不可用的状态。请问该如何解决？

小陈道: 

> 通过异步可以解决嘛，就是将这类任务进行隔离，碰上这类任务先进行返回，等到执行完毕再给响应？我的意思是说使用线程池。

面试官道:

> 但用户怎么知道我上传的图片是否成功呢，你返回的结果是什么呢，是未知，然后让用户过几分钟再看看上传结果？ 这看起来有些不友好哦。 你能分析一下核心问题在哪里嘛？

小陈陷入了沉思，想了一会说道:

> 是的，您说的对，这确实有些不友好，我想核心问题还是释放执行controller层方法线程，同时保持TCP连接。

面试官点了点头:

> 还可以，其实这个可以通过异步Servlet来解决，Servlet 3.0 引入了异步Servlet，解决了我们上面的问题，我们可以将这种任务专门交付给一个线程池处理的同事，也保持着原本的HTTP连接。具体的使用如下：

```java
@WebServlet(urlPatterns = "/asyncServlet",asyncSupported = true)
public class AsynchronousServlet extends HttpServlet {

    private static final ExecutorService BIG_FILE_POOL = Executors.newFixedThreadPool(10);

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        AsyncContext asyncContext = req.startAsync(req,resp);
        BIG_FILE_POOL.submit(()->{
            try {
                TimeUnit.SECONDS.sleep(10);
                ServletOutputStream outputStream = resp.getOutputStream();
                outputStream.write("task complete".getBytes(StandardCharsets.UTF_8));
                outputStream.flush();
            } catch (Exception e) {
                e.printStackTrace();
            }
            asyncContext.complete();
        });
    }
}
```

在Spring  MVC下 该如何使用呢, Spring MVC对异步Servlet进行了封装，只需要返回DeferredResult，就能简便的使用异步Servlet：

```java
@RequestMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
  DeferredResult<String> deferredResult = new DeferredResult<String>();
  // Add deferredResult to a Queue or a Map...
  return deferredResult;
}
// In some other thread...
deferredResult.setResult(data);
// Remove deferredResult from the Queue or Map
```

哈哈哈哈，感觉不是面试，感觉我在给你上课一样。我对你的感觉还可以，等二面吧。

小陈: 

> 啊，好的。

## 写在最后

写本文的时候用到了 chatGPT来查资料，但是chatGPT给的资料存在很多错误，chatGPT出现了认知偏差，比如将Jetty处理请求流程当成了Tomcat处理请求的流程，更细一点感觉还是没办法回答出来。还是要自己去看的。

## 参考资料

- Java NIO浅析 https://zhuanlan.zhihu.com/p/23488863
- 深度解读 Tomcat 中的 NIO 模型 https://klose911.github.io/html/nio/tomcat.html
- Tomcat - maxThreads vs. maxConnections  https://stackoverflow.com/questions/24678661/tomcat-maxthreads-vs-maxconnections
- 从一次线上问题说起，详解 TCP 半连接队列、全连接队列 https://developer.aliyun.com/article/804896
- 就是要你懂TCP--半连接队列和全连接队列 https://plantegg.github.io/2017/06/07/%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82TCP--%E5%8D%8A%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97%E5%92%8C%E5%85%A8%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97/
- Tomcat 配置文档 https://tomcat.apache.org/tomcat-8.5-doc/config/http.html
- Java NIO 系列文章之 浅析Reactor模式 https://pjmike.github.io/2018/09/20/Java-NIO-%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E4%B9%8B-%E6%B5%85%E6%9E%90Reactor%E6%A8%A1%E5%BC%8F/
- Java NIO - non-blocking channels vs AsynchronousChannels  https://stackoverflow.com/questions/22177722/java-nio-non-blocking-channels-vs-asynchronouschannels
- asynchronous and non-blocking calls? also between blocking and synchronous  https://stackoverflow.com/questions/2625493/asynchronous-and-non-blocking-calls-also-between-blocking-and-synchronous
- Java AIO 源码解析  https://cdf.wiki/posts/2976168065/
- 每天都在用，但你知道 Tomcat 的线程池有多努力吗？ https://www.cnblogs.com/thisiswhy/p/12782548.html
- 异步Servlet在转转图片服务的实践  https://juejin.cn/post/7124116514382774286
