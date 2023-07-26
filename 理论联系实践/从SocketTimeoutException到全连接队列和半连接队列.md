# 从SocketTimeoutException到全连接队列和半连接队列

[TOC]

## 前因

大概在一年半之前的时候，我们的应用的某个业务开始间歇报SocketTimeoutException, 不是前端调用我们发生SocketTimeoutException，而是我们用 HTTP Client中台拉取数据的时候，会偶尔报SocketTimeException, 这个偶尔可能是一个月报一次，也可能是两个月报一次，可能一个星期报两次，频率不固定，次数也不固定，当我第一次看到这个异常的时候，我的第一个反应就是用这个异常信息去搜索引擎上搜索解决方案，我并不理解这个异常说明了什么，但是按照我以往的经验来说，一般都有解决方案，对搜索引擎的方案一般都是延长超时时间，于是我延长了超时时间，但这并没有根本上解决问题，还是会出问题。延长超时时间不管用之后，我就扩容，但是扩容依然也不管用，我当时在尝试复现这个异常的时候，也忽略了一些东西，然后导致我在测试无法复现，能够复现的问题都是好问题，我之前面试的时候也背过三次握手，也学过Java 的原生Socket 编程，Netty，我背过Tomcat的acceptCount参数，但是碰到这个问题，这些知识仍然没有帮我解决问题，原因当时我网络的知识没有连接起来，他们孤零零的，向孤零零的神经元一样，没建立起来连接，最后这个问题开始让这些知识开始建立连接，成体系的发展。连接才是有价值的。

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

所以直接一点的原因就是接口长时间没给到我们响应？我的解决方式也很简单，直接延长超时时间，心满意足关闭了bug，然后没好多久，问题依然发生了，我想可能是不是对面处理不过来呢，于是我让被调用方从3实例扩容到8实例，但是好景不长，问题还是存在，这个头就有点大了。那我们接着分析这个问题为什么会出现，我们知道JVM在执行GC的时候，会STW，所以有没有可能是调用的时候，对方请求刚好在GC呢，导致接口没在指定时间给到响应，观察监控发现调用失败发生的时候，并没有发生GC动作， 接着往下排除。 我其实还是怀疑对面的承受能力不足，但是扩了一倍多还没解决问题，我得找点证据说服对面，让对面接着扩容， 那么要找证据就要分析当时的请求被处理的流程。被调用方的架构是:

![](https://a.a2k6.com/gerald/i/2023/07/23/xp3s.jpg) 

请求首先到达Nginx，然后负载均衡到Tomcat中，那么Tomcat是如何处理请求的呢，当时的Tomcat版本肯定是大于8的，那么想来Tomcat就是在NIO模式下面来处理请求的，那么一个请求在到达Tomcat之后是如何被Tomcat处理的呢？ 这点我们来结合Tomcat的源码来进行说明, 首先启动Tomcat之后，会有几个核心线程:

- Acceptor  线程
- Poller守护线程
- Worker批量工作线程

注意我看的Tomcat版本是9.0.50，所以我解读源码也是基于Tomcat 9.0.50，其他版本大差不差, 首先我们看Acceptor线程的执行逻辑, 这里我们省略掉其他逻辑Acceptor实现了Runnable接口，我们主要看重写的run方法的逻辑:

```java
package org.apache.tomcat.util.net;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.apache.juli.logging.Log;
import org.apache.juli.logging.LogFactory;
import org.apache.tomcat.util.ExceptionUtils;
import org.apache.tomcat.util.res.StringManager;

public class Acceptor<U> implements Runnable {

    @SuppressWarnings("deprecation")
    @Override
    public void run() {

        int errorDelay = 0;
        long pauseStart = 0;

        try {
            // Loop until we receive a shutdown command
            // 循环直到接收到shutdown指令
            while (!stopCalled) {
                // 如果endpoint处于暂停就自旋
                while (endpoint.isPaused() && !stopCalled) {
                    if (state != AcceptorState.PAUSED) {
                        pauseStart = System.nanoTime();
                        // Entered pause state
                        state = AcceptorState.PAUSED;
                    }
                    if ((System.nanoTime() - pauseStart) > 1_000_000) {
                        // Paused for more than 1ms
                        try {
                            if ((System.nanoTime() - pauseStart) > 10_000_000) {
                                Thread.sleep(10);
                            } else {
                                Thread.sleep(1);
                            }
                        } catch (InterruptedException e) {
                            // Ignore
                        }
                    }
                }
				//如果Endpoint终止了,跳出循环
                if (stopCalled) {
                    break;
                }
                // 修改Acceptor的状态
                state = AcceptorState.RUNNING;

                try {
                    //if we have reached max connections, wait
                    // 判断endpoint是否达到最大连接数,如果达到了等待
                    endpoint.countUpOrAwaitConnection();

                    // Endpoint might have been paused while waiting for latch
                    // If that is the case, don't accept new connections
                    if (endpoint.isPaused()) {
                        continue;
                    }

                    U socket = null;
                    try {
                        // Accept the next incoming connection from the server                    
                        socket = endpoint.serverSocketAccept();
                    } catch (Exception ioe) {
                        // We didn't get a socket
                        endpoint.countDownConnection();
                        if (endpoint.isRunning()) {
                            // Introduce delay if necessary
                            errorDelay = handleExceptionWithDelay(errorDelay);
                            // re-throw
                            throw ioe;
                        } else {
                            break;
                        }
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // Configure the socket
                    if (!stopCalled && !endpoint.isPaused()) {
                        // setSocketOptions() will hand the socket off to
                        // an appropriate processor if successful
                        // 为拿到的Socket设置参数
                        if (!endpoint.setSocketOptions(socket)) {
                            endpoint.closeSocket(socket);
                        }
                    } else {
                        endpoint.destroySocket(socket);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    String msg = sm.getString("endpoint.accept.fail");
                    // APR specific.
                    // Could push this down but not sure it is worth the trouble.
                    if (t instanceof org.apache.tomcat.jni.Error) {
                        org.apache.tomcat.jni.Error e = (org.apache.tomcat.jni.Error) t;
                        if (e.getError() == 233) {
                            // Not an error on HP-UX so log as a warning
                            // so it can be filtered out on that platform
                            // See bug 50273
                            log.warn(msg, t);
                        } else {
                            log.error(msg, t);
                        }
                    } else {
                            log.error(msg, t);
                    }
                }
            }
        } finally {
            stopLatch.countDown();
        }
        state = AcceptorState.ENDED;
    }
}

```

这里处理的逻辑相对简单，判断endpoint状态，接收连接，如果连接达到最大值就等待，正确获得连接之后对获得的Socket对象进行参数设置。核心代码也就是

```java
endpoint.countUpOrAwaitConnection();
```

我们简单解读一下endpoint的countUpOrAwaitConnection方法: 

```java
protected void countUpOrAwaitConnection() throws InterruptedException {
    if (maxConnections==-1) {
        return;
    }
    LimitLatch latch = connectionLimitLatch;
    if (latch!=null) {
        latch.countUpOrAwait();
    }
}
```

这里的LimitLatch都是基于AQS实现，我们简单看一下:

```java
public class LimitLatch {

    private static final Log log = LogFactory.getLog(LimitLatch.class);

    private class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1L;

        public Sync() {
        }
		
       
        @Override
        protected int tryAcquireShared(int ignored) {
            long newCount = count.incrementAndGet();
            if (!released && newCount > limit) {
                // Limit exceeded
                count.decrementAndGet();
                return -1;
            } else {
                return 1;
            }
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            count.decrementAndGet();
            return true;
        }
    }

    private final Sync sync;
    private final AtomicLong count;
    private volatile long limit;
    private volatile boolean released = false;

    /**
     * Instantiates a LimitLatch object with an initial limit.
     * @param limit - maximum number of concurrent acquisitions of this latch
     */
    public LimitLatch(long limit) {
        this.limit = limit;
        this.count = new AtomicLong(0);
        this.sync = new Sync();
    }
    public void countUpOrAwait() throws InterruptedException {
        if (log.isDebugEnabled()) {
            log.debug("Counting up["+Thread.currentThread().getName()+"] latch="+getCount());
        }
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * Releases a shared latch, making it available for another thread to use.
     * @return the previous counter value
     */
    public long countDown() {
        sync.releaseShared(0);
        long result = getCount();
        if (log.isDebugEnabled()) {
            log.debug("Counting down["+Thread.currentThread().getName()+"] latch="+result);
        }
        return result;
    }
}
```

tryAcquireShared方法定义获取共享变量的规则，先自增判断是否到达连接上限，如果到达连接上限，就进入AQS队列。那这个连接上限，由Tomcat的哪个参数控制呢? 初始化LimitLatch这个方法在AbstractEndpoint的initializeConnectionLatch方法来控制:

```java
protected LimitLatch initializeConnectionLatch() {
        if (maxConnections==-1) {
            return null;
        }
        if (connectionLimitLatch==null) {
            connectionLimitLatch = new LimitLatch(getMaxConnections());
        }
        return connectionLimitLatch;
}
```

这个方法获取的是AbstractEndpoint的maxConnections成员变量的值:

```java
private int maxConnections = 8*1024;
```

也就是说默认是8182个连接，这个参数我们可以在IDEA中靠提示感应一下，看看我们是否可以手动改这个连接，在Spring Boot 中通过server.tomcat.max-connection来控制，那看起来这个参数肯定是够用的，我们接着往下分析，接着看setSocketOptions方法:

```java
 protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            // Allocate channel and wrapper
            NioChannel channel = null;
            if (nioChannels != null) {
                channel = nioChannels.pop();
            }
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(bufhandler, this);
                } else {
                    channel = new NioChannel(bufhandler);
                }
            }
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            // Set socket properties
            // Disable blocking, polling will be used
            socket.configureBlocking(false);
            if (getUnixDomainSocketPath() == null) {
                socketProperties.setProperties(socket.socket());
            }

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error(sm.getString("endpoint.socketOptionsError"), t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            if (socketWrapper == null) {
                destroySocket(socket);
            }
        }
        // Tell to close the socket if needed
        return false;
    
```

这里将原生的SocketChannel包装为NioSocketWrapper，同时为NioSocketWrapper设置超时时间，同时在将打开包装好的NioSocketWrapper交给Poller线程来处理，我们接着来看Poller是如何处理SocketChannel的，我们首先看Poller的register方法:

```java
public void register(final NioSocketWrapper socketWrapper) {
    // 对读事件感兴趣
    socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    // 将SocketWrapper包装成事件
    PollerEvent pollerEvent = createPollerEvent(socketWrapper, OP_REGISTER);
    // 加入到队列中
    addEvent(pollerEvent);
}
```

我们接着来看addEvent方法：

```java
private final SynchronizedQueue<PollerEvent> events = new SynchronizedQueue<>();

private void addEvent(PollerEvent event) {
    // 入队操作
    events.offer(event);
    if (wakeupCounter.incrementAndGet() == 0) {
        selector.wakeup();
    }
}
```

也就是说Acceptor任务(现成)负责侦听Socket，连接建立好之后，对SocketChannel进行参数设置然后将其交给Poller线程，这两个线程的启动是在EndPoint的startInternal方法来启动的。

```java
@Override
public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        if (socketProperties.getProcessorCache() != 0) {
            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getProcessorCache());
        }
        if (socketProperties.getEventCache() != 0) {
            eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getEventCache());
        }
        if (socketProperties.getBufferPool() != 0) {
            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getBufferPool());
        }
		
        // Create worker collection
        if (getExecutor() == null) {
            createExecutor();
        }

        initializeConnectionLatch();

        // Start poller thread
        poller = new Poller();
        Thread pollerThread = new Thread(poller, getName() + "-Poller");
        pollerThread.setPriority(threadPriority);
        pollerThread.setDaemon(true);
        pollerThread.start();

        startAcceptorThread();
    }
}
```

这里出现了@Override，代表这是重写父类的方法，这个方法来自于AbstractEndpoint: 

![](https://a.a2k6.com/gerald/i/2023/07/25/52el.png)

我们本次看的也就是NioEndpoint, Nio2Endpoint和NioEndpoint的不同:

```java
public class NioEndpoint extends AbstractJsseEndpoint<NioChannel,SocketChannel> 
public class Nio2Endpoint extends AbstractJsseEndpoint<Nio2Channel,AsynchronousSocketChannel>    
```

 也就是Nio2Endpoint用了AIO，我们这里还是想弄清楚一个请求从操作系统来到Tomcat之后，是如何被Tomcat处理的，HTTP/2包括HTTP/2之前用的都是TCP请求，所以客户端要跟服务端建立连接，Acceptor线程承担此任务, 建立连接完成之后，将对应的Socket进行参数设置之后交给Poller线程处理，现在我们观察一下Poller线程是如何处理SocketChannel的: 

```java
// 省略无关代码
public class Poller implements Runnable {

    private Selector selector;
    private final SynchronizedQueue<PollerEvent> events =
            new SynchronizedQueue<>();

    private volatile boolean close = false;
    // Optimize expiration handling
    private long nextExpiration = 0;

    private AtomicLong wakeupCounter = new AtomicLong(0);

    private volatile int keyCount = 0;
	
    // 每个Poller线程都是单独的selector
    // 控制Poller线程数量的参数将会被宰Tomcat 10.0移除
    public Poller() throws IOException {
        this.selector = Selector.open();
    }

    public int getKeyCount() { return keyCount; }

    public Selector getSelector() { return selector; }

    @Override
    public void run() {
        // Loop until destroy() is called
        while (true) {

            boolean hasEvents = false;

            try {
                if (!close) {
                    hasEvents = events();
                    if (wakeupCounter.getAndSet(-1) > 0) {
                        // If we are here, means we have other stuff to do
                        // Do a non blocking select
                        keyCount = selector.selectNow();
                    } else {
                        keyCount = selector.select(selectorTimeout);
                    }
                    wakeupCounter.set(0);
                }
                if (close) {
                    events();
                    timeout(0, false);
                    try {
                        selector.close();
                    } catch (IOException ioe) {
                        log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                    }
                    break;
                }
                // Either we timed out or we woke up, process events first
                if (keyCount == 0) {
                    hasEvents = (hasEvents | events());
                }
            } catch (Throwable x) {
                ExceptionUtils.handleThrowable(x);
                log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
                continue;
            }
			// 获取选择器中已就绪的事件
            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;
            // Walk through the collection of ready keys and dispatch
            // any active event.
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                iterator.remove();
                NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                // Attachment may be null if another thread has called
                // cancelledKey()
                if (socketWrapper != null) {
                    // 处理对应的事件
                    processKey(sk, socketWrapper);
                }
            }

            // Process timeouts
            timeout(keyCount,hasEvents);
        }

        getStopLatch().countDown();
    }
}
```

我们接着看processKey方法:

```java
protected void processKey(SelectionKey sk, NioSocketWrapper socketWrapper) {
            try {
                if (close) {
                    cancelledKey(sk, socketWrapper);
                } else if (sk.isValid()) {
                    // 判断可读 还是可写
                    if (sk.isReadable() || sk.isWritable()) {
                        if (socketWrapper.getSendfileData() != null) {
                            processSendfile(sk, socketWrapper, false);
                        } else {
                            unreg(sk, socketWrapper, sk.readyOps());
                            boolean closeSocket = false;
                            // Read goes before write
                            // 处理读事件,在processSocket处理读事件生成Request对象
                            if (sk.isReadable()) {
                                if (socketWrapper.readOperation != null) {
                                    if (!socketWrapper.readOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.readBlocking) {
                                    synchronized (socketWrapper.readLock) {
                                        socketWrapper.readBlocking = false;
                                        socketWrapper.readLock.notify();
                                    }                                
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (!closeSocket && sk.isWritable()) {
                                if (socketWrapper.writeOperation != null) {
                                    if (!socketWrapper.writeOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.writeBlocking) {
                                    synchronized (socketWrapper.writeLock) {
                                        socketWrapper.writeBlocking = false;
                                        socketWrapper.writeLock.notify();
                                    }
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_WRITE, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (closeSocket) {
                                cancelledKey(sk, socketWrapper);
                            }
                        }
                    }
                } else {
                    // Invalid key
                    cancelledKey(sk, socketWrapper);
                }
            } catch (CancelledKeyException ckx) {
                cancelledKey(sk, socketWrapper);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("endpoint.nio.keyProcessingError"), t);
            }
}
```

我们接着看processSocket方法:

```java
 public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            // SocketProcessorBase 也实现了Runnable方法
            SocketProcessorBase<S> sc = null;
            if (processorCache != null) {
                sc = processorCache.pop();
            }
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            // 获取上面初始化的线程池,
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                // 在这里将SocketProcessorBase交给线程池处理              
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
 }
```

这里创建的SocketProcessorBase是SocketProcessor ，后面看的话就是由getExecutor的线程池的线程来处理任务，所以在Tomcat处理HTTP请求的流程如下:

![](https://a.a2k6.com/gerald/i/2023/07/25/765zp.png)

那这个Worker线程池的线程数量在Spring Boot web starter 是由server.tomcat.threads.max来控制，默认为200，这个线程池我们在《异步Servlet学习笔记(一)》已经解析了他的特性，这个线程池是Tomcat改写的线程池，原先JDK的线程池的工作逻辑为接收到任务的时候首先看活跃线程数是否小于核心线程数，如果小于添加线程执行接收的任务，如果不小于则将任务加入到阻塞队列中，任务队列满了，判断当前活跃的线程数目是否小于最大线程数目，如果小于接着创建线程执行提交的任务，那Tomcat改写的线程池呢？我们看下Tomcat创建的线程池有何不同:

```java
 public void createExecutor() {
     internalExecutor = true;
     TaskQueue taskqueue = new TaskQueue();
     TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
     executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
     taskqueue.setParent( (ThreadPoolExecutor) executor);
 }
```

这里我们唯一感到陌生的恐怕就是TaskQueue，我们看下这个类的入队逻辑:

```java
@Override
public boolean offer(Runnable o) {
      //we can't do any checks
        if (parent==null) {
            return super.offer(o);
        }
        //we are maxed out on threads, simply queue the object
    	// 如果线程池的工作线程等于线程池的最大线程数目,这个值很大,入队
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) {
            return super.offer(o);
        }
        //we have idle threads, just add it to the queue
    	// getSubmittedCount 获取的是当前已经提交但是还未完成的任务的数量，其值是队列中的数量加上正在运行的任务的数量。
    	// 小于工作线程的数量，那么入队
        if (parent.getSubmittedCount()<=(parent.getPoolSize())) {
            return super.offer(o);
        }
        //if we have less threads than maximum force creation of a new thread
    	// 如果工作线程的数量小于最大值接着创建线程
        if (parent.getPoolSize()<parent.getMaximumPoolSize()) {
            return false;
        }
        //if we reached here, we need to add it to the queue
        return super.offer(o);
 }
```

所以超时的时候在排队? 所以还是被调用方没忙过来？得到了结论，于是喜滋滋的找被调用方让他们再度扩容，然后被拒绝，调用方检查了发生超时发生的时间点，Tomcat工作线程能忙得过来，也没有活跃很多线程，那问题出在哪里呢? 头秃呦，记得我们在《用Java的BIO和NIO、Netty实现HTTP服务器(一) BIO与绪论》中写服务端的时候，指定了一个backlog参数，当时我们对这个参数的解释是:

> TCP是面向连接的，但是服务端处理客户端请求建立的连接也需要时间，ServerSocket会维护一个队列，还没来得及处理的连接就会放到这个队列里面，如果队列已经满了，就会抛出连接被拒绝的异常 《用Java的BIO和NIO、Netty实现HTTP服务器(一) BIO与绪论》

这里给人的暗示是Java在自行维护这个队列，事实上这个参数最终是被操作系统所控制，在Linux中这个参数控制的是全连接队列的大小，那什么是全连接队列。

## 全连接队列与半连接队列

这里我们再来回忆一下TCP连接队列三次握手的过程:

- 第一步: 客户端发送syn到server发起握手
- 第二步: 服务端收到syn之后，回复syn+ack给客户端。
- 第三步: 客户端收到syn+ack之后, 回复server一个ack表示收到了server的syn + ack(此时客户端的tcp连接状态已经是established, 在客户端看来连接已经成功建立 )

站在服务端的角度来说，一个到来的连接变成established之前，需要经过一个中间状态SYNRECEIVED; 进入established状态之后，服务端调用accept操作，即可返回socket。这意味着，tcp/ip协议栈要实现backlog队列，有两种选择:

1. 使用一个单独的队列，队列的长度由listen调用的backlog参数决定，当收到一个syn包时，给客户端返回SYN/ACK，并将此链接加入到队列。对应的ACK到达之后，连接状态改变为ESTABLISHED，即可移交给应用程序处理。这意味着，对了可以包含两状态连接: SYN RECEIVED 和 ESTABLISHED。只有处于SYN RECEIVED状态的连接，才能返回给应用程序发起的accept调用。
2. 使用两个队列，一个SYN对了(或者叫半连接队列)和一个accept队列(或者叫完全连接队列)。处于 SYN RECEIVED状态的连接将被加入到SYN队列，后续当状态变为ESTABLISHED状态时(也就是说三次握手的最后一次ACK到达时)，被迁移到accept对了。就像accept函数的名字所表示的那样，实现accept调用，只要简单低从accept队列中获取连接时，只需要简单地从accept队列中获取连接即可。在这种方式下back参数决定了accept队列的长度。

历史上，BSD系统的TCP实现使用的是一种方式。在这种方式下，当队列到达backlog指定的最大值时，系统将不再给客户端发来的SYN返回SYN/ACK。通常，TCP实现会简单的丢弃SYN包(甚至不会返回RST包)，因此客户端会触发重试。这个在W.Richard Steven的TCP/IP卷三种的14.5节有讲。值得注意的事，Stevens解释了，BSD实际上使用了两个单独的队列，但是它们表现的的一个单独的，具有backlog参数指定长度的对了没什么差别。BSD逻辑上表现得和下面表述一致:

> 对了的大小是半连接队列的长度和全连接队列的长度之和(sum = 半连接队列长度 + 全连接队列长度)

但是在Linux上，事情不太一样，Linux上选了第二种方案: 一个SYN队列，大小由系统级别的参数指定; 一个accept队列大小由应用程序指定:

> The behavior of the `backlog` argument on TCP sockets changed with Linux 2.2. Now it specifies the queue length for*completely* established sockets waiting to be accepted,
>
> instead of the number of incomplete connection requests. The maximum length of the queue for incomplete sockets can be set using `/proc/sys/net/ipv4/tcp_max_syn_backlog`.

从Linux 2.2 版本之后backlog参数的行为被修改了，这个参数指定了已完成三次握手的 accept 队列的长度，而不是半连接队列的长度。半连接队列的长度可以通过 /proc/sys/net/ipv4/tcp_max_syn_backlog来设置。这两个参数也并不是你给多少，Linux就设置多少。全连接队列的大小取决于**min(backlog, somaxconn)**，somaxconn是操作系统内核级参数，由/proc/sys/net/core/somaxconn来控制的，这个参数默认是128。所以就算Tomcat给了200，Linux也就认128，这让本不富裕的吞吐量进一步下降，半连接队列由**max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)**来控制，默认也是128。





![img](https://img2018.cnblogs.com/blog/519126/201906/519126-20190611134827358-915643494.png)



图片来自《**Linux中，Tomcat 怎么承载高并发（深入Tcp参数 backlog）**》，那么backlog在Tomcat中是由哪些参数控制的呢？ 注意在Tomcat中的初始化参数都在AbstractEndPoint中进行，我们在这个类里面全局搜这个参数就行: 

```java
/**
* Allows the server developer to specify the acceptCount (backlog) that
* should be used for server sockets. By default, this value
* is 100.
* 允许开发者自由指定backlog参数，被用于服务端的ServerSocket,默认值是100
*/
private int acceptCount = 100;
```

才100？ 这么小，话说我一秒一百个TCP请求，两秒200个TCP请求过来，那这个TCP全连接队列是不是就满了，那在队列已经被占满的情况下，一个连接又需要从SYN队列移动到accept队列时(收到了三次握手中的第三个包，客户端发来的ack)，那么Linux会如何处理呢? 对应的处理代码在net/ipv4/tcp_minisocks.c中的:tcp_check_req

```c
child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb, req, NULL);
        if (child == NULL)
                goto listen_overflow;
```

对于 ipv4， 代码中第一行会最终调用net/ipv4/tcp_ipv4.c 中的 tcp_v4_syn_recv_sock：

```c
## 看起来像我们的监听者模式。。。listen_overflow: 
if (!sysctl_tcp_abort_on_overflow) {
      inet_rsk(req)->acked = 1;
      return NULL;
}
```

这个什么意思呢？ 意思是，除非 /proc/sys/net/ipv4/tcp_abort_on_overflow 设为 1 ,这种情况下，会发送 RST 包。那么如果ack包被无视，你可以认为操作系统将客户端发送的ack扔掉了，也就是说在Server端认为连接还没建立起来。那注意在SYN RECEIVED状态下的socket还有一个定时器，该定时器的机制是:  如果ack数据包没被收到(或者被无视，就像我们上面描述的情况)，TCP协议栈会重发SYN/ACK包(重发次数由/proc/sys/net/ipv4/tcp_synack_retries指定)，如果服务端监听socket 的 backlog 值降低了 （比如，从 accept 队列消费了一个连接，因此队列变成未满），而且， SYN/ACK 重试次数没有达到最大值的情况下，那么， tcp 协议栈就可以最终处理 客户端发来的 ack 包， 将连接状态从 SYN RECEIVED 改为 ESTABLISHED， 并将其加入到 accept 队列中。 否则， 客户端最终将会拿到一个 RST 包。 这对应我们的日志中出现了: java.net.socketexception connection reset 。那么我们还知道TCP协议是全双工的，在收到服务端发来的SYN/ACK之后，一直就处于ESTABLISHED状态，如果它向服务端写入数据，那么数据就会被重传， TCP 慢开始算法，会限制发出的包的数量。 （这里意思是， 慢开始算法下，一开始不会传很多包，可能只传一个，收到服务端的响应后，下一次传2个，再一次传4个，这样指数级增长，直到达到一个值后，进入线性增长阶段，因为服务端一直没响应，就不会增大发送的包的个数，避免浪费网络流量）。 另一方面，如果客户端一直等待服务端发送数据，但是服务端的backlog一直没有降低，客户端没有连接上服务端，那么最终的结果就是，客户端连接为ESTABLISHED 状态，在服务端，该连接状态CLOSED。

还有一个问题就是，在半连接队列没满之前，服务端收到的SYN包会被添加到SYN队列，但这个并不完全准备，在Linux的tcp_v4_conn_request 函数中，该函数负责SYN包的处理, 我们可以看到处理逻辑是如果accept队列满了， 那么内核会隐式限制 SYN 包接收的速度。 如果收到了太多的 SYN 包， 部分会被丢弃。 在这种情况下， 由客户端决定 进行重发，然后我们最终表现就和在 BSD 下的实现一样。

##  更强的验证

到现在为止答案已经复现到了我们的眼前，也就是请求太过密集，导致TCP连接队列被打满，事实上Tomcat的工作线程还有在空闲的，到这里我们的逻辑已经很严密了，那我们该如何验证发生了队列溢出了呢:

```shell
netstat -s | egrep "listen|LISTEN" 
```

这个命令用于查看队列溢出的次数，次数如果一直在增加，代表队列在一直溢出。我们还可以修改队列溢出之后的处理策略, 也就是说将/proc/sys/net/ipv4/tcp_abort_on_overflow的丢弃策略改为1，也就是队列满了，直接让服务端发送一个rst给客户端，如果观察到客户端的异常由connection rest 、read time out 变成了connection reset by peer，那么就说明我们的理论模型是正确的，是有的放矢的。于是再联合被用方将这个参数改为1，果然如我们所料。但是我们也同时也检测到调用方的keep-alive没有进行合理设置，因此我们对这个不分进行重新设置，也修改了内核参数，问题遂被解决。总结一下这里的排查错误参数，首先出现read time out的时候有两种情况，第一种就是服务端收到了客户端的数据，但是在指定时间没有给到回应，第二种就是服务端的全连接对了被打满，对于客户端来说，连接建立完毕，传送数据给服务端，但是服务端压给就没处理这个数据，客户端一直在等，导致超时。这个时候我们就需要观察服务端是否收到了数据，如果收到了客户端发送的数据，那代表全连接队列没被打满，只是接口超时而已，如果服务端没有收到数据，那么就是服务端的全连接队列被打满，这个时候我们可以检查溢出次数、全连接队列、半连接队列大小，如果溢出次数上升，即说明发生了队列溢出，同时我们也需要检查应用程序的繁忙度，如果非常繁忙，那么也可能是超出了应用程序的处理能力。说到这里想起了Netty如果没做好对应的调优，那么处理的请求能力也不能被完全打开，在Netty的3.6.3版本，默认是50，《netty新建连接并发数很小的case》这篇文章分享了排查经验，修改默认backlog之前，每秒处理的请求在40个就出现了问题，强制修改了backlog之后，轻松每秒破200。 这篇文章总的参考了:

- Tomcat源码篇之HTTP请求处理流程
- Linux中，Tomcat 怎么承载高并发（深入Tcp参数 backlog）

之所以声明就是因为参考的内容还是蛮多的，我用一个问题连接了这两篇文章。

## 写在最后

这篇文章让我感慨良多，我想起了一句话:

> 显然，我们曾经的「成功」或是知识，阻碍了我们「看见」更多的东西。
>
> 但世界不管你的成见，它只会按照自己的规律展开向前。

如果我的测试就只限于HTTP调用的超时，那么我恐怕会认为是对面的应用能力处理不足，这也就是成见。佛陀讲:

> 我们所称之为"知识"的东西，只是我们目前对事物的理解，而这个理解随着条件的变化，可能会改变。这就是说，我们的"知识"并不是绝对的真理，而是相对的，依赖于特定的视角和环境。

这次也尝试将我所学的网络相关的知识融合起来，以前学习的一些知识点建立了连接，我以前都觉得这个被这个三次连接是没有用的，我平时只是为了面试而却背，散落的知识不能发挥作用，一旦它们建立连接，又觉得很有意思。

## 参考资料

- Tomcat源码篇之HTTP请求处理流程  https://www.wormholestack.com/archives/649/
- How TCP backlog works in Linux http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html
- TCP 连接状态 https://getiot.tech/zh/tcpip/tcp-connection-states
- java.net.SocketException：Connection reset 原因分析与故障重现 https://ld246.com/article/1572277642298#%E5%AE%A2%E6%88%B7%E7%AB%AFread%E6%97%B6%E9%81%87%E5%88%B0%E5%8D%8A%E5%BC%80%E8%BF%9E%E6%8E%A5%E6%83%85%E5%86%B5%E7%9A%84%E5%88%86%E6%9E%90
- What is the difference between tcp_max_syn_backlog and somaxconn? https://stackoverflow.com/questions/62641621/what-is-the-difference-between-tcp-max-syn-backlog-and-somaxconn

- 再聊 TCP backlog https://heapdump.cn/article/2351759
- netty新建连接并发数很小的case https://blog.51cto.com/iteyer/3236945
- 应用频繁报出cause java.net.SocketTimeoutException: Read timed out怎么办  
- 就是要你懂TCP--半连接队列和全连接队列 https://plantegg.github.io/2020/04/07/%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82TCP--%E5%8D%8A%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97%E5%92%8C%E5%85%A8%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97--%E9%98%BF%E9%87%8C%E6%8A%80%E6%9C%AF%E5%85%AC%E4%BC%97%E5%8F%B7%E7%89%88%E6%9C%AC/
- TCP三次握手中的全连接与半连接队列 https://blog.fintopia.tech/60d6ea482078082a378ec5ea/
- **Linux中，Tomcat 怎么承载高并发（深入Tcp参数 backlog）** https://www.cnblogs.com/grey-wolf/p/10999342.html
