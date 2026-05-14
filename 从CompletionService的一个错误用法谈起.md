# 从CompletionService的一个错误用法谈起

[TOC]

## 前言

ExecutorCompletionService 很容易让人产生一个错觉：既然它名字里带着 Service，又是基于 Executor 工作的，那么它是不是也可以像线程池一样做成全局对象复用？答案是否定的。原因在于，线程池保存的是执行资源，而 ExecutorCompletionService 内部保存的是完成结果。资源可以复用，结果不能混用。如果把它做成全局对象，不同请求提交的任务会共用同一个 completionQueue，一个请求中已经取消的 Future，可能会被另一个请求取出来，然后在 get() 时抛出 CancellationException。

这个问题看起来像是偶发的并发异常，但从源码看，它其实是 ExecutorCompletionService 的正常行为。接下来我们先从 Future 的顺序等待问题说起，再看一个看似合理的 CompletionService 写法为什么会出错，最后回到源码，看看这个 CancellationException 究竟是怎么来的。

##  从 Future 顺序等待的问题说起

这个问题是去年的，然后去年一直没有机会去写，今天终于有机会来写。话不多说，我们正文开始，之前我们的业务是这样的，我们在控制层要同时触发五十个任务到线程池里面去执行，这几十个任务都是请求第三方接口，对方的接口会向我们返回一个结果，这四五十个任务只有一个能执行成功，我们还要拿这些任务的执行结果。所以相关的开发同学第一件事想到的就是用Future，代码如下所示:

```java
// 源码都基于JDK 21 编写, 但基本在JDK 8 都能用, 没有用到高版本的API
public class CompletionServiceDemo {

    private static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(10);

    private static AtomicBoolean ATOMIC_RESULT = new AtomicBoolean(false);
    
    public static void main(String[] args) {
        List<Future<Boolean>> futures = new ArrayList<>();
        // 将任务提交到线程池里面
        for (int i = 0; i < 40; i++) {
            final int taskId = i;
            Future<Boolean> httpResult = THREAD_POOL.submit(() -> mockHTTP(taskId));
            futures.add(httpResult);
        }
        
        for (Future<Boolean> future : futures) {
            try {
                // 获取线程的执行结果
                Boolean taskResult = future.get();
                if (taskResult) {
                    
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } catch (ExecutionException e) {
                throw new RuntimeException(e);
            }
        }
    }
    
   // 原子类只会有一个线程会更新成功
    private static Boolean mockHTTP(int taskId) {
        if (taskId % 2 == 0){
            return ATOMIC_RESULT.compareAndSet(false, true);
        }
        return false;
    }
}
```

这段代码从正确性上看没有明显问题，但它的等待方式并不适合当前场景。我们提交了四十个任务，然后按照提交顺序依次调用 Future.get() 获取结果。问题在于，Future.get() 会阻塞当前线程。如果第一个任务迟迟没有完成，即使第四个任务已经先返回了成功结果，调用方也只能继续卡在第一个 Future 上，无法及时处理第四个任务的结果。

对于“多个任务并发执行，只要其中一个成功就可以结束”的场景来说，这种按提交顺序等待的方式会带来两个问题：一是已经完成的有效结果不能被及时消费；二是目标结果出现之后，其他任务仍然可能继续执行，造成不必要的资源浪费。

所以我真正需要的不是“按照提交顺序获取结果”，而是“谁先完成，就先处理谁”。

顺着这个思路，自然就会想到 CompletionService。它可以把已经完成的任务放入一个完成队列中，调用方通过 take() 获取最先完成的 Future。这样一来，我们就可以尽早检查已经完成的任务结果；如果发现目标结果已经出现，再尝试取消其他还没有必要继续执行的任务。

于是我写出了下面这段代码：

```java
public class CompletionServiceDemo {


    private static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(10);
	
	// 为了模拟“多个任务里只有一个会成功”的业务条件，这里用 AtomicBoolean 控制只有一个任务能够返回 true。
    private static AtomicBoolean ATOMIC_RESULT = new AtomicBoolean(false);

    private static final ExecutorCompletionService<Boolean> EXECUTOR_COMPLETION_SERVICE = new ExecutorCompletionService<>(THREAD_POOL);

    public static void main(String[] args) {
        List<Future<Boolean>> futures = new ArrayList<>();
        for (int i = 0; i < 40; i++) {
            final int taskId = i;
            Future<Boolean> httpResult = EXECUTOR_COMPLETION_SERVICE.submit(() -> mockHTTP(taskId));
            futures.add(httpResult);
        }
        try {
            boolean isTaskComplete = false;
            for (Future<Boolean> future : futures) {
                // 如果目标任务已经完成,则尝试取消任务
                if (isTaskComplete){
                    // cancel方法是尝试取消任务
                    // 如果任务已经完成、已经被取消，或者由于其他原因无法取消,则此方法无效
                    // 如果任务没有被启动的时候，调用cancel,则该任务将不会被开始
                    // 如果任务已经被启动，则mayInterruptIfRunning将决定参数执行任务的线程是否被中断
                    // 如果能够确定执行任务的是哪个线程，以尝试停止任务
                    future.cancel(true);
                }
                // take 方法会阻塞在这里,直到有任务完成
                Future<Boolean> result = EXECUTOR_COMPLETION_SERVICE.take();
                // 我们这里可以姑且认为Future里面的结果为true就认为任务已经完成
                // 如果计算被取消了，调用future.get会抛出CancellationException
                // 如果计算抛出了异常,调用future.get会抛出ExecutionException 
                // 如果当前线程在等待过程中被中断,调用future.get会抛出InterruptedException
                if (result.get()){
                    isTaskComplete = true;
                }
            }
        } catch (InterruptedException | ExecutionException e) {
           
            throw new RuntimeException(e);
        }
    }

    private static void asyncTask1() {
        List<Future<Boolean>> futures = new ArrayList<>();
        for (int i = 0; i < 40; i++) {
            final int taskId = i;
            Future<Boolean> httpResult = THREAD_POOL.submit(() -> mockHTTP(taskId));
            futures.add(httpResult);
        }
        for (Future<Boolean> future : futures) {
            try {
                Boolean taskResult = future.get();
                if (taskResult) {

                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } catch (ExecutionException e) {
                throw new RuntimeException(e);
            }
        }
    }

    private static Boolean mockHTTP(int taskId) {
        if (taskId % 2 == 0){
            return ATOMIC_RESULT.compareAndSet(false, true);
        }
        return false;
    }
}
```

需要注意的是，CancellationException 是运行时异常，不在当前 catch 块的捕获范围内。所以一旦 take() 返回的是已经取消的 Future，后续 result.get() 就会直接把 CancellationException 抛出去。

从 Demo 的角度看，这段代码似乎已经用上了 CompletionService 的核心能力：任务提交之后，不再按照提交顺序等待，而是通过 take() 获取最先完成的 Future。但问题也藏在这里。

我们一边从 CompletionService 中获取已经完成的任务，一边又在遍历 futures 列表，尝试取消后续任务。这里有一个很容易忽略的点：futures 列表是按照提交顺序保存的，而 completionService.take() 返回的是按照完成顺序排列的结果。

提交顺序和完成顺序并没有一一对应关系。也就是说，for 循环中当前遍历到的 future，和 take() 返回的 result，很可能不是同一个任务。当前面的某个任务返回 true 之后，isTaskComplete 会被设置为 true。接下来循环继续往后走，我们会开始调用 future.cancel(true)，尝试取消剩余任务。

而被取消的 Future，后面如果再调用 get()，就会抛出 CancellationException。

所以这段代码的问题并不在于 CompletionService 不能用，而在于我们混用了两套顺序：一套是提交顺序，一套是完成顺序。我们按提交顺序取消任务，却按完成顺序消费结果，这就给后面的异常埋下了伏笔。

不过，并发问题还有一个麻烦的地方：它不一定每次都稳定复现。有时候这段代码跑起来并不会抛出 CancellationException。原因也不难理解：任务执行得太快了。等我们开始 cancel 后续任务时，很多任务可能已经执行完成了。既然任务已经完成，cancel 就不会真正改变它的状态，后续调用 get() 自然也能正常拿到结果。所以如果只是把这段 Demo 丢给诸君，让大家“多运行几次看看”，显然不是一个好的办法。

我们需要构造一个更稳定的例子，让这个问题可以被明确观察到。稳定复现的 bug，才是适合拿出来分析的 bug。

##  用 JCStress 让问题稳定复现

接下来，我们用 JCStress 构造一个更稳定的观察环境，让这个问题更容易暴露出来：

```java
@JCStressTest
@Outcome(id = "0, 0", expect = Expect.ACCEPTABLE_INTERESTING, desc = "没有观察到 CancellationException")
@Outcome(id = "0, 1", expect = Expect.ACCEPTABLE_INTERESTING, desc = "观察到了 CancellationException")
@State
public class API_07_Future {
	
    
    private final AtomicBoolean atomicResult = new AtomicBoolean(false);

    @Actor
    public void actor1(II_Result r) {
        ExecutorService threadPool = Executors.newFixedThreadPool(10);
        ExecutorCompletionService<Boolean> completionService = new ExecutorCompletionService<>(threadPool);
        List<Future<Boolean>> futures = new ArrayList<>();
        try {
            for (int i = 0; i < 40; i++) {
                final int taskId = i;
                Future<Boolean> httpResult = completionService.submit(() -> mockHTTP(taskId));
                futures.add(httpResult);
            }

            boolean isTaskComplete = false;
            for (Future<Boolean> future : futures) {
                if (isTaskComplete) {
                    future.cancel(true);
                }
                Future<Boolean> result = completionService.take();
                try {
                    if (result.get()) {
                        isTaskComplete = true;
                    }
                } catch (CancellationException e) {
                    r.r2 = 1;
                }
            }
        } catch (InterruptedException | ExecutionException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        } finally {
            threadPool.shutdownNow();
        }
    }

    private Boolean mockHTTP(int taskId) {
        if (taskId % 2 == 0) {
            return atomicResult.compareAndSet(false, true);
        }
        return false;
    }
}
```

最后我这边跑出来的结果如下所示:

```java
 RESULT  SAMPLES     FREQ       EXPECT  DESCRIPTION
    0, 0      969    0.24%  Interesting  任务已取消，但 get 没有观察到已取消的 Future。
    0, 1  401,223   99.76%  Interesting  任务已取消，并且 get 观察到了 CancellationException。
```

从结果看，大多数运行都观察到了 CancellationException。少数情况下没有观察到异常，并不代表代码没有问题，只是说明那些任务可能在 cancel 生效之前就已经完成了。并发问题难就难在，它依赖具体的执行时机。同一段代码，在不同的线程调度下，可能表现出不同结果。与其依赖“多跑几次”碰运气，不如用 JCStress 这样的工具，把我们关心的执行结果稳定地观测出来。

## 另一种错误的用法

现在让我们将这个例子移植到Controller层, 代码如下所示:

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

@RestController
public class SlowController {

    private static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(10);

    private static final AtomicBoolean UPDATE_RESULT = new AtomicBoolean(false);

    private static final ExecutorCompletionService<Boolean> EXECUTOR_COMPLETION_SERVICE = new ExecutorCompletionService<>(THREAD_POOL);

    private static final HttpClient HTTP_CLIENT = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_1_1)
            .followRedirects(HttpClient.Redirect.NORMAL)
            .connectTimeout(Duration.ofSeconds(20))
            .build();

    @GetMapping("/slow")
    public Boolean test() {
        Random random = new Random();
        int i = random.nextInt(1, 3);
        try {
            TimeUnit.SECONDS.sleep(i);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return UPDATE_RESULT.compareAndSet(false, true);
    }

    @GetMapping("/slowTest")
    public String slowTest() {
        List<Future<Boolean>> futures = new ArrayList<>();
        for (int i = 0; i < 40; i++) {
            Future<Boolean> future = EXECUTOR_COMPLETION_SERVICE.submit(this::remoteCall);
            futures.add(future);
        }
        try {
            boolean isTaskComplete = false;
            for (Future<Boolean> future : futures) {
                Future<Boolean> takeResult = EXECUTOR_COMPLETION_SERVICE.take();
                if (isTaskComplete) {
                    future.cancel(true);
                }
                if (takeResult.get()){
                    isTaskComplete =  true;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
        UPDATE_RESULT.compareAndSet(true,false);
        return "hello world";
    }
	//  注意这个HttpClient，在JDK 11以上的版本才有，在JDK8 你可以用其他HTTP Client代替。
    public boolean remoteCall() {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("http://localhost:8090/slow"))
                .timeout(Duration.ofMinutes(2))
                .header("Content-Type", "application/json")
                .GET()
                .build();
        try {
            HttpResponse<String> response = HTTP_CLIENT.send(request, HttpResponse.BodyHandlers.ofString());
            return Boolean.valueOf(response.body());
        } catch (IOException e) {
            throw new RuntimeException(e);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

这里的 UPDATE_RESULT 只是为了模拟“多个远程调用里只有一个返回成功”。真实业务中，一次请求内的状态不应该放在 static 字段里。本文关注的重点是另一个 static 对象：EXECUTOR_COMPLETION_SERVICE。它内部的 completionQueue 一旦被多个请求共享，就会导致不同请求的完成结果混在一起。

在Spring Boot 中写下如此的测试代码，然后我们进行测试，会发现前面几次会正常返回hello world，后面就返回服务器异常了。观察Spring Boot 这边的日志，会发现抛出了CancellationException，异常堆栈如下所示:

```java
java.util.concurrent.CancellationException
	at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:121) ~[na:na]
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191) ~[na:na]
	at com.example.webstudty.controller.SlowController.slowTest(SlowController.java:67) ~[classes/:na]
	at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:103) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:580) ~[na:na]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:258) ~[spring-web-6.2.18.jar:6.2.18]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:191) ~[spring-web-6.2.18.jar:6.2.18]
	at 
```

对于CancellationException，我们知道，如果我们对一个Future执行了cancel操作，然后再get，就会抛出这个异常。而对于slowTest是如何被执行的，我们也了然于胸。在《Tomcat 源码阅读笔记 · 01 | 拆包/粘包的处理机制》一文，我们已经探讨了Tomcat的运作流程，Acceptor线程负责监听TCP连接，Poller线程负责监听该Socket上面的读事件，Worker线程池中的线程负责读数据，解析报文，执行Servelet的逻辑，返回响应。所以我们在不断的并发发起对slowTest的请求时，是不同的线程在执行slowTest这个方法。

但是令我不解的是为什么会抛出CancellationException，这意味着我们从ExecutorCompletionService获取到了已经处于取消状态的Future。那也就是说我们将Future取消之后，没有从ExecutorCompletionService移除掉。 导致其他线程执行这段代码，从ExecutorCompletionService获取已经不再处于执行中的Future时，获取到了cancel状态的Future，于是执行Future.get就抛出了CancellationException。

当我们将现象完整的描述出来，我们就可以构造出来合理的解释，但没有在源码中验证之前，我们只是对现象进行了总结，我们的认为是合理的。那ExecutorCompletionService究竟是如何处理的呢？ 让我们来看源码

## 浅析ExecutorCompletionService的源码

打开ExecutorCompletionService的源码，映入眼帘的是三个成员变量:

```java
private final Executor executor;
private final AbstractExecutorService aes;
private final BlockingQueue<Future<V>> completionQueue;
```

take方法的实现就是从这个队列里面获取完成Future:

```java
public Future<V> take() throws InterruptedException {
    return completionQueue.take();
}
```

也就是说线程池里面完成的Future，会被放入这个阻塞队列里。那这一切究竟是如何发生的呢？ 我们接着来追踪源码，

```java
public Future<V> submit(Callable<V> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<V> f = newTaskFor(task);
    executor.execute(new QueueingFuture<V>(f, completionQueue));
    return f;
}
```

也就是我们的提交进入的任务，通过newTaskFor这个方法的加工，被变成了RunnableFuture。newTaskFor的实现就很简单， 粗略的说就是将task变成FutureTask返回出去:

```java
private RunnableFuture<V> newTaskFor(Callable<V> task) {
    if (aes == null)
        return new FutureTask<V>(task);
    else
        return aes.newTaskFor(task);
}
```

Future 只是一个结果接口，表示任务可以取消、可以查询状态、可以获取结果。FutureTask 则是一个具体实现，它实现了 RunnableFuture。RunnableFuture 同时继承 Runnable 和 Future，所以 FutureTask 既可以作为任务交给线程池执行，也可以作为 Future 返回给调用方获取结果。

更关键的是，FutureTask 提供了一个 protected done() 方法。任务进入完成态之后，会回调这个方法。然后接着看QueueingFuture的实现，QueueingFuture是一个静态内部类:

```java
private static class QueueingFuture<V> extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task,
                   BlockingQueue<Future<V>> completionQueue) {
        super(task, null);
        this.task = task;
        this.completionQueue = completionQueue;
    }
    private final Future<V> task;
    private final BlockingQueue<Future<V>> completionQueue;
    protected void done() { completionQueue.add(task); }
}
```

这里我们就可以看出ExecutorCompletionService的思路，将任务包装成FutureTask，然后扩展FutureTask的能力，在FutureTask任务完成之后，将任务添加进入阻塞队列里面。注意这里没有判断 task 是正常完成、异常完成，还是被取消。只要 QueueingFuture 进入完成态，done() 就会被触发，然后把原始 task 放入 completionQueue。所以 CompletionService 收集的不是“成功完成的任务”，而是“已经完成的任务”。这里的完成，包括正常返回、抛出异常，也包括被取消。实现的相当精巧。

![](https://a.a2k6.com/gerald/i/2026/05/14/hkt.png)

到这里其实我们的疑问就已经解决了，我们发起第一个请求的时候，由于我们将所有的任务都取消了，但这些取消的任务也会进入到这个队列里面。于是我们接着调用take，然后获取Future的执行结果，于是抛出CancellationException。其实解决这个问题也很简单：线程池可以继续共享，但 ExecutorCompletionService 不能作为全局对象共享。它应该绑定到一次批量任务上。也就是说，每次 slowTest 调用时，都创建一个新的 ExecutorCompletionService。所以我们将代码可以改成下面这个样子:

```java
@GetMapping("/slowTest")
public String slowTest() {
    ExecutorCompletionService<Boolean> completionService =
            new ExecutorCompletionService<>(THREAD_POOL);

    List<Future<Boolean>> futures = new ArrayList<>();

    try {
        for (int i = 0; i < 40; i++) {
            futures.add(completionService.submit(this::remoteCall));
        }

        for (int i = 0; i < futures.size(); i++) {
            Future<Boolean> completed = completionService.take();

            try {
                if (Boolean.TRUE.equals(completed.get())) {
                    return "hello world";
                }
            } catch (CancellationException ignore) {
                // 被取消的任务也可能进入 completionQueue
            }
        }

        return "no result";
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new RuntimeException(e);
    } catch (ExecutionException e) {
        throw new RuntimeException(e);
    } finally {
        futures.forEach(future -> future.cancel(true));
        UPDATE_RESULT.compareAndSet(true, false);
    }
}
```

一旦遇到并发问题，我的疑心病就相当严重，我总是会担心我们这个例子，还会跑出来别的结果。为了消除我的疑虑，我们还是要构造JCStress的测试，来看看我们会跑出来多少结果集，验证我们的猜想。所以我们构造出来的JCStress测试如下:  

```java
@JCStressTest
@Outcome(id = "0, 0", expect = Expect.ACCEPTABLE, desc = "没有观察到 CancellationException")
@Outcome(id = "0, 1", expect = Expect.FORBIDDEN, desc = "观察到了 CancellationException")
@State
public class API_07_Future_Fixed {

    private final AtomicBoolean atomicResult = new AtomicBoolean(false);

    @Actor
    public void actor1(II_Result r) {
        ExecutorService threadPool = Executors.newFixedThreadPool(10);
        ExecutorCompletionService<Boolean> completionService =
                new ExecutorCompletionService<>(threadPool);

        List<Future<Boolean>> futures = new ArrayList<>();

        try {
            for (int i = 0; i < 40; i++) {
                final int taskId = i;
                Future<Boolean> future =
                        completionService.submit(() -> mockHTTP(taskId));
                futures.add(future);
            }

            for (int i = 0; i < futures.size(); i++) {
                Future<Boolean> completed = completionService.take();

                try {
                    if (Boolean.TRUE.equals(completed.get())) {
                        // 已经拿到目标结果，结束消费
                        break;
                    }
                } catch (CancellationException e) {
                    // 正常情况下，不应该在消费阶段观察到 CancellationException。
                    r.r2 = 1;
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        } finally {
            // 统一取消剩余任务
            futures.forEach(future -> future.cancel(true));
            threadPool.shutdownNow();
        }
    }

    private Boolean mockHTTP(int taskId) {
        if (taskId % 2 == 0) {
            return atomicResult.compareAndSet(false, true);
        }
        return false;
    }
}
```

这次结果符合预期，没有再观察到 CancellationException：

```java
 RESULT  SAMPLES     FREQ       EXPECT  DESCRIPTION
    0, 0  284,722  100.00%  Interesting  没有观察到 CancellationException
    0, 1        0    0.00%  Interesting  观察到了 CancellationException
```

## 总结一下

这几个错误例子给我的启示是有时候你知道一些工具的用法，并且这些工具也确实好用，但是如果你不仔细看这些工具的说明书，那么你将会收到一个让你印象深刻的教训。不过高情商的说法，应该是我们都在错误中成长。在ExecutorCompletionService提供了示例调用， 只是我看代码的时候略过了:

```java
void solve(Executor e,
           Collection<Callable<Result>> solvers)
    throws InterruptedException {
  CompletionService<Result> cs
      = new ExecutorCompletionService<>(e);
  int n = solvers.size();
  List<Future<Result>> futures = new ArrayList<>(n);
  Result result = null;
  try {
    solvers.forEach(solver -> futures.add(cs.submit(solver)));
    for (int i = n; i > 0; i--) {
      try {
        Result r = cs.take().get();
        if (r != null) {
          result = r;
          break;
        }
      } catch (ExecutionException ignore) {}
    }
  } finally {
    futures.forEach(future -> future.cancel(true));
  }

  if (result != null)
    use(result);
}
```

 这次问题的根源，不在于 CompletionService 不好用，而在于我们把它当成了线程池一样的全局资源。线程池保存的是执行资源，可以被多个请求复用。
CompletionService 保存的是完成结果，它内部的 completionQueue 属于某一次任务批次，不能跨请求、跨批次混用。使用 ExecutorCompletionService 时，我觉得至少要记住三点：

1. 它返回的是按完成顺序排列的 Future，而不是按提交顺序排列的 Future。
2. completionQueue 里保存的是完成态 Future，正常完成、异常完成、取消完成都可能进入队列。
3. 线程池可以共享，但 ExecutorCompletionService 应该绑定一次任务批次，用完即弃。

