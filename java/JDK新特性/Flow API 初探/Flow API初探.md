# JDK 9新特性之Flow API 初探

> 本身我是只打算介绍JDK 11的 新的Http Client的，但是又碰见Flow API 响应式流，只好将这部分东西独立出来，简单介绍一下。

[TOC]

## 响应式流的引入 

Reactive Stream 反应式流或响应式流，这个词我是在介绍JDK 11中的HttpClient中碰到的：

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
.uri(URI.create("http://openjdk.java.net/"))
.POST(HttpRequest.BodyPublishers.ofString("aaaa"))
                .build();
// BodyHandlers.fromLineSubscriber要求的参数是Subscriber类实例
// 然后我点了点发现Subscriber位于Flow类内是一个静态接口
client.sendAsync(request, HttpResponse.BodyHandlers.fromLineSubscriber())
```

![JDk](http://tva4.sinaimg.cn/large/006e5UvNgy1h2uzschuh3j30no0j7drq.jpg)

往上翻了一下发现这个Flow出自Doug Lea大佬之手，上面还写了Since 9，也就是说这个类是在JDK 9之后才进入到JDK里面的。

![Doug Lea](http://tvax4.sinaimg.cn/large/006e5UvNgy1h2uzu8jq2aj30pu0fxwo2.jpg)

Doug Lea的注释一向是注释比代码多，我们先看注释看，看看引入这个Flow 类是为了什么？

>  Interrelated interfaces and static methods for establishing flow-controlled components in which {@link Publisher Publishers} produce items consumed by one or more {@link Subscriber Subscribers}, each managed by a {@link Subscription Subscription}.
> 这些接口和静态方法都是为了建立一起发布-订阅者模式(Publisher发布者发布 一个或多个Subscriber订阅者消费,每个订阅者被Subscription管理)的流式控制组件。
>
>  <p>These interfaces correspond to the reactive-streams
>  specification.  They apply in both concurrent and distributed
> asynchronous settings: All (seven) methods are defined in {@code
> void} "one-way" message style. Communication relies on a simple form
>  of flow control (method {@link Subscription#request}) that can be
>  used to avoid resource management problems that may otherwise occur
> in "push" based systems.
> 这些接口遵循响应式流的规范，他们被应用于并发和分布式异步设置: 所有七个方法都被定义为返回值为void的单向消息风格。
> 消息的交流依赖于简单的流式控制(Subscription的request方法)可以用来避免基于推送系统的一些资源管理问题。    

这个响应流规范是啥? 我打开了href的这个链接进行查看。

### 为什么要引入响应流规范

Reactive Streams is an initiative to provide a standard for asynchronous stream processing with non-blocking back pressure. This encompasses efforts aimed at runtime environments (JVM and JavaScript) as well as network protocols.
响应流式一种倡议，旨在为具有非阻塞背压的异步流处理提供标准，这包括针对JVM运行时环境、javaScript、网络协议的工作。

Handling streams of data—especially “live” data whose volume is not predetermined—requires special care in an asynchronous system. The most prominent issue is that resource consumption needs to be controlled such that a fast data source does not overwhelm the stream destination. Asynchrony is needed in order to enable the parallel use of computing resources, on collaborating network hosts or multiple CPU cores within a single machine.

在异步系统中处理,处理数据流,尤其是数据量未预先确定的实施数据要特别小心。最为突出而又常见的问题是资源消费控制的问题，以便防止大量数据快速到来淹没目的地。
为了让让一片网络的计算机或者一台计算机内的多核CPU在执行计算任务的时候使用并行模式，我们需要异步。

The main goal of Reactive Streams is to govern the exchange of stream data across an asynchronous boundary—think passing elements on to another thread or thread-pool—while ensuring that the receiving side is not forced to buffer arbitrary amounts of data. In other words, back pressure is an integral part of this model in order to allow the queues which mediate between threads to be bounded. The benefits of asynchronous processing would be negated if the communication of back pressure were synchronous (see also the Reactive Manifesto), therefore care has to be taken to mandate fully non-blocking and asynchronous behavior of all aspects of a Reactive Streams implementation.

响应流的主要目标是控制跨越异步边界的数据交换,即将一个元素传递到令外一个线程或线程中要确保接收方不会被迫缓冲任意数量的数据。换句话说，背压是该模型的重要组成，该模型可以让线程之间的队列是有界的。如果采取背压方式的通信是同步的，那么异步处理的方式将会被否定的(详见响应式宣言)。因此必须要求所有的反应式流实现都是异步和非阻塞的。
It is the intention of this specification to allow the creation of many conforming implementations, which by virtue of abiding by the rules will be able to interoperate smoothly, preserving the aforementioned benefits and characteristics across the whole processing graph of a stream application.
遵守本规范的实现可以实现交互操作，从而在整个流应用的处理过程中受益。

 JDK 9 的正式发布时间是2017年9月, 如果你点搜索Reactive Manifesto，会发现这个宣言于14年9月16日发布，这是一种编程理念，对应的有响应式编程，同面向对象编程、函数式编程一样，是一种理念。推出规范就是为了约束其实现，避免每个库有有自己的一套响应式实现，这对于开发者来说是一件很头痛的事情。响应式编程的提出如上文所示主要是为了解决异步数据处理的背压现象，那什么是背压。

### 背压的解释

背压并不是响应式编程独有的概念，背压的英文是BackPressure，不是一种机制，也不是一种策略，而是一种现象: 在数据流从上游生产者向下游消费者传输的过程中，上游生产速度大于下游消费速度，导致下游的Buffer溢出，这种现象我们称之为Backpressure出现。背压的重点在于上游的生产速度大于下游消费速度，而在于Buffer溢出。

举一个例子就是在Java中，我们向线程池中提交任务，队列满了触发拒绝策略(拒绝接受新任务还是丢弃旧的处理新的)。写到这里可能有同学会说，那你用无界队列不行吗？那如果提交的任务不断膨胀，导致你整个系统崩溃掉了怎么办？ 如果上游系统生产速度快到可以把系统搞崩溃，那么就需要设置Buffer上限。

### 梳理一下

首先出现响应式编程理念，然后出现响应式编程实现，再然后出现响应式规范，响应流主要解决处理元素流的问题—如何将元素流从发布者传递到订阅者，不而不需要发布者阻塞，或者要求订阅者有无限的缓冲区，有限的缓冲区在到达缓冲上界的时候，对到达的元素进行丢弃或者拒绝，订阅者可以异步通知发布者降低或提升数据生产发布的速率，它是响应式编程实现效果的核心特点。

![发展历程](http://tva3.sinaimg.cn/large/006e5UvNgy1h2v4vqv39aj313i09l0un.jpg)





而响应式规范则是一种倡议，遵循此倡议的系统可以让数据在各个响应式系统中都实现响应式的处理数据，规范在Java中的形式就是接口，也就是我们本篇的主题Flow 类，对于一项标准而言，它的目的自然是用更少的协议来描述交互。而响应流模型也非常简单:

- 订阅者异步的向发布者请求N个元素
- 发布者异步的向订阅者发送( 0 < M <= N)个元素。

写到这里可能有同学会问了，为啥不是订阅者要多少元素，发布者给多少啊？ 这其实上是一种协调机制， 在消费数据中有以下两种情况值得我们注意:

- 订阅者消费过快(在响应式模型中, 处理这种情况是通知发布者产生元素快一点，类似于去包子店吃包子,  饭量比较大的顾客来，包子店生产不及，就会告诉包子店做的快一点，说完还接着吃包子)

![push模型](http://tvax4.sinaimg.cn/large/006e5UvNgy1h2v56a9gz9j310z0ia0uk.jpg)

- 发布者发布过快(在响应式模型中，处理这种情况是通知生产者降低生产速率，还是去包子店吃包子，虽然顾客饭量比较大，但是吃的比较慢，很快摆不下了，就会告诉包子店做的慢一些)

​	![pull模型](http://tva2.sinaimg.cn/large/006e5UvNly1h2v5bdo510j30yx0hl408.jpg)



### Flow的大致介绍

Flow是一个被final关键字修饰的类，里面是几组public static接口和buffer变量长度：

- Publisher 发布者
- Subscriber 订阅者
- Subscription 订阅信件(或订阅令牌), 通过此实例, 用于订阅者和发布者之间协调请求元素数量和请求订阅元素数量
- Processor 继承Publisher 和 Subscriber，用于连接Publisher和Subscriber, 也可以连接其他处理器

![响应式流程](http://tvax2.sinaimg.cn/large/006e5UvNgy1h2v6cw3ln4j310d0h50w7.jpg)

###  简单示例

```java
public class FlowDemo {
    static class SampleSubscriber<T> implements Flow.Subscriber<T> {
        final Consumer<? super T> consumer;
        Flow.Subscription subscription;
        SampleSubscriber(Consumer<? super T> consumer) {
            this.bufferSize = bufferSize;
            this.consumer = consumer;
        }
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            System.out.println("建立订阅关系");
            this.subscription = subscription; // 赋值
            subscription.request(2);
        }
        public void onNext(T item) {
            System.out.println("收到发送者的消息"+ item);
            consumer.accept(item);
            // 可调用 subscription.request 接着请求发布者发消息
          //  subscription.request(1);
        }
        public void onError(Throwable ex) { ex.printStackTrace(); }
        public void onComplete() {}
    }

    public static void main(String[] args) {
        SampleSubscriber subscriber = new SampleSubscriber<>(200L,o->{
            System.out.println("hello ....." + o);
        });
        ExecutorService executor = Executors.newFixedThreadPool(1);
        SubmissionPublisher<Boolean> submissionPublisher = new SubmissionPublisher(executor,Flow.defaultBufferSize());
        submissionPublisher.subscribe(subscriber);
        submissionPublisher.submit(true);
        submissionPublisher.submit(true);
        submissionPublisher.submit(true);
        executor.shutdown();
    }
}
```

输出结果:

![结果示例](http://tvax2.sinaimg.cn/large/006e5UvNly1h2v709gvmnj30l504tgn4.jpg)

为啥发布者发布了三条消息，你订阅者只处理了两条啊，因为在建立订阅关系的时候订阅者就跟发布者说明了, 我只要两条消息, 当前消费能力不足, 在消费之后, 还可以再请求发布者发送。

下面我们来演示一下背压效果, 我们现在设定缓冲池大小的任务是Flow定义的默认值, 256。 我们现在尝试提交1000个任务试试看: 

```java
public class FlowDemo {
    static class SampleSubscriber<T> implements Flow.Subscriber<T> {
        final Consumer<? super T> consumer;
        Flow.Subscription subscription;
        SampleSubscriber(Consumer<? super T> consumer) {
            this.consumer = consumer;
        }
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            System.out.println("建立订阅关系");
            this.subscription = subscription; // 赋值
            subscription.request(1);
        }
        public void onNext(T item) {
            try {
                System.out.println("thread name 0"+Thread.currentThread().getName());
                TimeUnit.SECONDS.sleep(30);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("收到发送者的消息"+ item);
            consumer.accept(item);
            // 可调用 subscription.request 接着请求发布者发消息
            subscription.request(1);
        }
        public void onError(Throwable ex) { ex.printStackTrace(); }
        public void onComplete() {}
    }

    public static void main(String[] args) {
        SampleSubscriber subscriber = new SampleSubscriber<>(o->{
            System.out.println("hello ....." + o);
        });
        ExecutorService executor = Executors.newFixedThreadPool(1);
        SubmissionPublisher<Boolean> submissionPublisher = new SubmissionPublisher(executor,Flow.defaultBufferSize());
        submissionPublisher.subscribe(subscriber);
        for (int i = 0; i < 1000; i++) {
            System.out.println("开始发布第"+i+"条消息");
            submissionPublisher.submit(true);
            System.out.println("开始发布第"+i+"条消息发布完毕");
        }
        executor.shutdown();
    }
}
```

![缓冲区大小示例](http://tva4.sinaimg.cn/large/006e5UvNgy1h2v7qfmhn5j30lk05eabd.jpg)

为什么到第257条被阻塞住了, 那是因为缓冲区满了, 缓冲区出现空闲才会被允许接着生产。

```java
public class MyProcessor extends SubmissionPublisher<Boolean> implements Flow.Processor<Boolean, Boolean> {
    private Flow.Subscription subscription;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        this.subscription.request(1);
    }

    @Override
    public void onNext(Boolean item) {
        if (item){
            item = false;
            // 处理器将此条信息转发
            this.submit(item);
            System.out.println("将true 转换为false");
        }
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
        this.subscription.cancel();
    }

    @Override
    public void onComplete() {
        System.out.println("处理器处理完毕");
        this.close();
    }
}
public class FlowDemo {
    static class SampleSubscriber<T> implements Flow.Subscriber<T> {
        final Consumer<? super T> consumer;
        Flow.Subscription subscription;
        SampleSubscriber(Consumer<? super T> consumer) {
            this.consumer = consumer;
        }
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            System.out.println("建立订阅关系");
            this.subscription = subscription; // 赋值
            subscription.request(1);
        }
        public void onNext(T item) {
            System.out.println("收到发送者的消息"+ item);
            consumer.accept(item);
            // 可调用 subscription.request 接着请求发布者发消息
            subscription.request(1);
        }
        public void onError(Throwable ex) { ex.printStackTrace(); }
        public void onComplete() {}
    }

    public static void main(String[] args) throws Exception{
        SampleSubscriber subscriber = new SampleSubscriber<>(o->{
            System.out.println("hello ....." + o);
        });
        ExecutorService executor = Executors.newFixedThreadPool(1);
        SubmissionPublisher<Boolean> submissionPublisher = new SubmissionPublisher(executor,Flow.defaultBufferSize());
        MyProcessor myProcessor = new MyProcessor();
        // 做信息转发
        submissionPublisher.subscribe(myProcessor);
        myProcessor.subscribe(subscriber);
        for (int i = 0; i < 2; i++) {
            System.out.println("开始发布第"+i+"条消息");
            submissionPublisher.submit(true);
            System.out.println("开始发布第"+i+"条消息发布完毕");
        }
        TimeUnit.SECONDS.sleep(2);
        executor.shutdown();
    }
}
```

输出结果:

![转换成功](http://tvax2.sinaimg.cn/large/006e5UvNly1h2v80zki1xj30m306uq40.jpg)



##  总结一下

我们由JDK 11的HTTP Client的请求参数看到了Flow API, 在Flow类中的注释中看到了Reactive Stream, 由Reactive Stream看到了响应式规范, 由规范引出响应流解决的问题, 即协调发布者和订阅者，发布者发布太快, 订阅者请求发布者减缓生产速度，生产太慢，订阅者请求发布者加快速度。在Java领域已经有了响应式的一些实现: 

- RXJava 是ReactiveX项目中的Java实现，Rxjava早于Reactive Streams规范, RXJava 2.0+确实实现了Reactive Streams API规范。
- Reactor是Pivotal提供的Java实现，它作为Spring Framework 5的重要组成部分，是WebFlux采用的默认反应式框架
- Akka Streams完全实现了Reactive Streams规范，但Akka Streams API与Reactive Streams API完全分离。

为了统一规范，JDK 9引入了Flow，Flow可以类似于JDBC, 属于API规范，实际使用时需要使用API对应的具体实现，Reactive Streams为我们提供了一个我们可以代码的接口，而无需关心其实现。



## 参考资料

- 反应式流 Reactive Streams 入门介绍  https://zhuanlan.zhihu.com/p/95966853
- Reactive Streams  http://www.reactive-streams.org/
- 如何形象的描述反应式编程中的背压(Backpressure)机制？ https://www.zhihu.com/question/49618581/answer/237078934

- Java9特性-响应式流(Reactive Stream)   https://zhuanlan.zhihu.com/p/463579630
- 反应式编程探索与总结  https://developer.aliyun.com/article/728068#slide-13
- 反应式宣言  https://www.reactivemanifesto.org/zh-CN
- Java9-Reactive Stream API响应式编程  https://zhuanlan.zhihu.com/p/266407815



