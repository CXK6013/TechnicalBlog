# RocketMQ漫谈之从消息队列到事务消息

[TOC]

> 当我书写的时候，我的痛苦从我的心里通过血液流淌到我书写的文字上。

## 引言

最近在零拷贝、分布式事务消息，架构的认知上有了新的理解，于是重新打算学习一下RocketMQ，关于RocketMQ的文章已经写了三篇:

-  《消息队列引论》

- 《RocketMQ学习笔记(一) 初遇篇》
- 《RocketMQ学习笔记(二) 相识篇》

关于零拷贝这里讲了三篇《译: 通过零拷贝实现高效数据传输》、《操作系统与通用计算机组成原理简论》、《NIO 学习笔记(一)初遇》，本篇尝试融合这几篇的知识点，做到理论与实践相融合。

## 概述

按照最初的设想是消息队列引论，总体论述消息队列，然后后面跟kafak、RabbitMQ、RocketMQ，到现在为止只学了RocketMQ，在《消息队列引论》里面我们首先讲消息队列的定位，首先消息队列是一个队列，队列是一种组织数据结构的形式，也就是数据结构，这种数据结构具备先入先出的特性，那RocketMQ既然是队列，也保持了这种顺序性，即RocketMQ消息按照进入队列的顺序写入存储，同一队列间的消息天然存在顺序关系，队列头部为最早写入的消息，队列尾部为最新写入的消息。消息在队列中的位置和消息之间的顺序通过位点(Offset)进行标记管理。

在RocketMQ下面队列是消息存储和传输的实际容器，以此来实现队列的数量的水平拆分和队列内部的流式存储，流式操作的语义为基于队列的存储模型可确保消息从任意位点读取任意数量的消息，以此来实现类似聚合读取、回溯读取等特性，这些是RabbitMQ、ActiveMQ等非队列模型不具备的。

消息队列表达的另一种语义是生产者消费者模型，也就是说在消息队列里面一般都会有生产者和消费者，从名字上就可以推断出来，生产者负责生产消息，消费者负责消费消息，还有一个存储消息的地方，在RocketMQ里面这个存储消息的地方叫Broker，那么问题来了， 这三个是如何联系起来的呢? 答案是NameServer:

![](https://a.a2k6.com/gerald/i/2024/08/14/37vi.png)



在我们的图片中生产者通过NameServer将消息存储到Broaker中，消费者通过NameServer对消息进行消费，在这种语境下消息好像是没有区分的一样，生产者生产消息，消息到达Broker之后被消费者消费，实际中的RocketMQ并不是这样，为了对消息进行区分RocketMQ引入topic(主题)和tag(标签)的概念，主题用于标识同一类业务的逻辑信息，主要作用有:

- 定义数据的分类隔离: 也就是将不同的业务类型数据拆分到不同的主题中管理。
- 定义数据的身份和权限:  RocketMQ的消息本身是匿名无身份的，同一分类的消息使用相同的主题来做身份识别和权限管理。

写到这里想起上海的垃圾分类，何尝不是一种分类隔离呢:

![](https://a.a2k6.com/gerald/i/2024/08/16/4hv60.png)

不同的垃圾桶放不同类型的垃圾，不同的主题放不同类型的消息:

![](https://a.a2k6.com/gerald/i/2024/08/16/p3ow.png)

topic容纳的是一种类型的信息，我们可以将其看做是一个分类标准，但有时候一个大分类还不够，我们需要对信息进一步分类，RocketMQ在主题的基础上为我们提供了tag，我们可以将Topic当做一级分类，Tag当做二级分类。

![](https://a.a2k6.com/gerald/i/2024/08/16/5jqq5.png)

这里说的分类是一种业务上的划分，订单状态流转我们可以将其放入到一个主题(topic)，而到达对应的状态则属于对应的标签(tag)。

## RocketMQ的消息类型

### 从普通消息到事务消息

而对于RocketMQ本身又可以根据传输特性将消费分为:普通消息、顺序消息、事务消息、定时/延时消息。 普通消息的普通是相对于顺序消息、事务消息、延时消息来说的，对于普通消息来说它的生命周期有四个阶段: 初始化、待消费、消费中、消费提交。

​	![](https://a.a2k6.com/gerald/i/2024/08/16/wd7z.png)

- 初始化: 消息被生产者构建并完成初始化，待发送到服务端状态

- 待消费: 消费被发送到服务端，对消费者可见，等待消费者消费的状态

- 消费中: 消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，RocketMQ会对消息进行重试。

  这也就是消费重试，消费重试指的是，消费者在消费某条消息失败后，超过一定时间没有收到回复，也会被认为是失败，RocketMQ服务端会根据重试策略重新消费该消息，超过一定次数后若还未消费成功，直接被发送到死信队列里面。

- 消息删除: RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。

然后普通的消息又可以同步发送或异步发送、单向发送，所谓同步发送也就是在调用发送消息的方法之后会阻塞到broker返回结果之后才会往下执行，异步发送不会阻塞，在接收到broker返回的结果之后会执行回调函数，单向发送不关心结果。像下面这样，需要引入的依赖如下:

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

```java
@Resource
private RocketMQTemplate rocketMQTemplate;

public  void sendMsg(){
    /**
     * 同步发送 会阻塞到broker返回发送结果之后,才会接着执行
     */
    SendResult sendResult = rocketMQTemplate.syncSend("testTopic", "hello world");
    // 异步发送,不会阻塞这里,在发送成功之后执行回调函数
    rocketMQTemplate.asyncSend("testTopic", "helloworld", new SendCallback() {
        @Override
        public void onSuccess(SendResult sendResult) {

        }

        @Override
        public void onException(Throwable throwable) {

        }
    });
}
// 不等待
rocketMQTemplate.sendOneWay("testTopic","hello world");
```

SendResult是一个枚举，有以下几个结果:

- SEND_OK 发送成功
- FLUSH_DISK_TIMEOUT  刷盘超时
- FLUSH_SLAVE_TIMEOUT 同步到从超时
- SLAVE_NOT_AVAILABLE  从不可用

同步发送的好处是可靠性高，但是因为阻塞等待broker返回的结果，异步的好处是不会阻塞代码，吞吐量高，但是一致性和可靠性低，这里我们来举个例子说明一下假设我们下单之后要触发一些动作，比如发短信，如果订单流程失败，消息发送出去了，这就造成了系统不一致。这里可能有同学会说，我将发消息这个动作放到事务提交之后触发不就可以了嘛，那我们不妨将例子举得的再通用一些，也就是两个系统之间进行通信，A、B之间两个系统之间需要保持一致，A系统的某条数据创建或者修改之后需要通知B系统，触发B系统中新数据的创建，状态流转。 如果是异步消息，有可能消息先发送出去，A系统的这里的流程还没走完，这也造成了不一致，如果我们将其放入到事务提交之后触发异步消息，一种可能的情况是消息由于网络抖动发送超时，我们回忆一下网络，一般我们所说的是TCP/IP协议族，而任何基于IP的协议都可能发生拥塞数据包丢失。如果两台计算机之间存在拥塞，路由器可以丢弃IP数据包，因为IP是尽力而为的协议。 

在ip数据包里面有一个TTL(time to live)字段，可以用于指定数据包可以经过的最大跳数，跳数表示数据包在网络中经过的路由器或网络设备的数量，经过一个路由器器，其TTL值会减一，TTL的主要作用防止数据包在网络中无限循环(永久滞留在网络中)，网络无法绝对可靠的原因就在于此，除此之外路由器还可能发生硬件故障，导致数据包丢失。

![](https://a.a2k6.com/gerald/i/2024/08/24/3tpxn.jpg)



虽然我们常用的TCP协议会重传，但也不是无限制的重传，这也就是TCP的拥塞控制，通常我们在网络编程中会指定超时时间, 如下面这样:

```java
// 这俩还是用BIO来写,我们只是用于说明
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
// 绑定端口
serverSocketChannel.bind(new InetSocketAddress(8080));
// 在bio模式下面会阻塞到有连接建立
SocketChannel socketChannel = serverSocketChannel.accept();
while (true){
    // 简单输出一下连接的基本信息
    System.out.println(socketChannel);
    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    byteBuffer.put("hello world".getBytes(Charset.defaultCharset()));
    byteBuffer.flip();
    // 这里是为了测试读超时
    TimeUnit.SECONDS.sleep(5);
    socketChannel.write(byteBuffer);
    break;
}
serverSocketChannel.close();
socketChannel.close();
```

```java
try (Socket socket = new Socket()) {
    // 设置连接时间3秒超时,
    // 注意这里的单位是毫秒
    socket.connect(new InetSocketAddress(8080), 3);
    
    // 设置超时时间,如果为0,表示无限期等待
    // 这里意味着两秒内没有数据读到,会抛出这样一个异常,SocketTimeoutException: Read timed out
    socket.setSoTimeout(2000);
    InputStream inputStream = socket.getInputStream();

    byte[] byteArray = new byte[1024];
    // bytesRead 返回读取了多少
    int bytesRead;
    // 这里我们没有设置报文格式, 只是给出一般的写法
    // 如果是一般的格式,我们这里就要判断报文结束了没有,
    // 举个例子HTTP报文,前后两个包可能会被送到一起,但是这属于两次不同的请求
    // 我们这里只读写一次,如果是持续发报文的
    // 没有数据会一直阻塞在这里
    while((bytesRead = inputStream.read(byteArray)) != -1){
        String readResult = new String(byteArray, 0, bytesRead, Charset.defaultCharset());
        System.out.println(readResult);
    }
}
```

最后输出的如下:

```java
Exception in thread "main" java.net.SocketTimeoutException: Read timed out
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.net.SocketInputStream.read(SocketInputStream.java:127)
	at com.example.demo.ClientSocketChannelDemo.main(ClientSocketChannelDemo.java:32)
```

注意在socketRead0里面会做超时控制，超过时间之后就会报超时异常。这里面看到了read time out，于是在想会不会write time out，就去翻SocketOutputStream的write方法，看看有没有哪个方法有超时字段，在socketWrite0这个方法上没翻到:

```java
private native void socketWrite0(FileDescriptor fd, byte[] b, int off, int len) throws IOException;
private native int socketRead0(FileDescriptor fd,byte b[], int off, int len,int timeout)
```

这里思考写，为什么不能控制超时呢，这代表写出去了一定能收到？  如果你熟悉网络编程，java.net.SocketException: Broken pipe 这个错一定不会陌生，这个错一般对应在另一端关闭连接的时候，还在写入数据。 我们一下socketWrite0在JDK 8的native实现，在windows下面的实现如下:

![](https://a.a2k6.com/gerald/i/2024/08/24/4u5v.jpg)

我们去微软的文档下去看下对这个函数的说明(见参考文档):

>  If no error occurs, **send** returns the total number of bytes sent, which can be less than the number requested to be sent in the *len* parameter. 
>
> 如果没发生错误，将会返回发送成功的字节数，可以小于请求发送的字节数。

> The successful completion of a **send** function does not indicate that the data was successfully delivered and received to the recipient. This function only indicates the data was successfully sent.
>
> send函数成功返回并不代表数据成功被发送，此函数仅表示数据成功被发送。

> If no buffer space is available within the transport system to hold the data to be transmitted, **send** will block unless the socket has been placed in nonblocking mode
>
> 如果传输系统中的缓冲区不够保存要传输的数据，则send将被阻塞，除非socket非阻塞模式。

也就是说在Java层面write调用只是将数据发送到操作系统的缓冲区，下面发送数据的重传、拥塞控制都由操作系统来接管，这部分相对不可控，如果传输的数据过多，传输过程中被打断，事实上一部分数据传了一半，这会造成数据的不完整，所以基于这种设计write调用没有给超时参数。我们接着思考TCP是全双工的，也就是说TCP连接建立的时候，数据传输过去，我们在大部分情况下可以信任这个结果，在RocketMQ里面发送消息，服务端会返回结果给客户端，消息的结果有:

- SEND_OK :  消息发送成功。要注意的是消息发送成功也不意味着它是可靠的。要确保不会丢失任何消息，还应启用同步Master服务器或同步刷盘，即SYNC_MASTER或SYNC_FLUSH。
- FLUSH_DISK_TIMEOUT: 消息发送成功但是服务器刷盘超时。此时消息已经进入服务器队列（内存），只有服务器宕机，消息才会丢失。消息存储配置参数中可以设置刷盘方式和同步刷盘时间长度
- FLUSH_SLAVE_TIMEOUT: 消息发送成功，但是服务器同步到Slave时超时。此时消息已经进入服务器队列，只有服务器宕机，消息才会丢失。
- SLAVE_NOT_AVAILABLE:  消息发送成功，但是此时Slave不可用。此时消息已经进入Master服务器队列，只有Master服务器宕机，消息才会丢失。

这么看在事务提交之后触发通知也不见得靠谱，原因在于我们无法判断我们的调用是成功了还是由于网络拥堵失败了，同步消息也面临同样的问题，发送的时候不会无限制的等待消息发送结果，在参考文档[4]里面我们可以看到默认超时时间为3s，所以往外丢消息面临的一个问题是在网络抖动的情况下，我们无法确认消息是否是成功丢出去的，所以在为了追求强一致，我们需要这样一类消息，在消息到达RocketMQ的时候对消费者不可见，在二次检查，如果检查之后发现业务成功提交了，就将这个消息标记为可见状态，如果失败就将这条消息删除，那么如果提交这个状态的过程中网络通信失败了怎么办，这里再加一个补丁RocketMQ回查。





![](https://a.a2k6.com/gerald/i/2024/08/17/17hzce.jpg)

以上也就是RocketMQ的基本设计思路，我们现在来看代码，在实际开发过程中，习惯性用rocketmq-spring-boot-starter比较多，代码如下所示:

```java
@Component
public class TranProducer {

    @Resource
    private RocketMQTemplate rocketMQTemplate;

  public void sendTransaction(){
 	  rocketMQTemplate.sendMessageInTransaction("transation-order",MessageBuilder.withPayload("hello world").build(),"test orgs");
  }
}
```

```java
@RocketMQTransactionListener
public class TransactionMsgListener implements RocketMQLocalTransactionListener {

    /**
     * 执行本地事务
     * @param msg
     * @param arg
     * @return
     */
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // 这里可以是提交订单检查,看看订单是否提交成功
        System.out.println("执行本地事务");
        // 如果查到订单了 这里commit,如果查不到rollBack,如果回复unknown会执行消息回查
        return RocketMQLocalTransactionState.COMMIT;
    }

    /**
     * 检查本地事务状态
     *在断网或者是生产者应用重启的特殊情况下，若服务端未收到发送者提交的二次确认结果，
     * 或服务端收到的二次确认结果为Unknown未知状态，
     * 经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。
     * @param msg
     * @return
     */
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        System.out.println("执行回查");
        return RocketMQLocalTransactionState.COMMIT;
    }
}
```

事务消息的生命周期相对于普通消息的生命周期多了一个事务周期，也就是待提交、回滚: 

![](https://a.a2k6.com/gerald/i/2024/08/25/4e0f.jpg)



- 初始化:  半事务消息被生产者构建并完成初始化，待发送到服务端的状态。

- 事务阶段

  - 待提交：半事务消息被发送到服务端，和普通消息不同，并不会直接被服务端持久化，而是会被单独存储到事务存储系统中，等待第二阶段本地事务返回执行结果后再提交。此时对下游消费者不可见。
  - 提交待消费:  若第二个阶段如果事务执行结果明确为提交，服务端会将半事务消息重新存储到普通的存储系统中，此时消息对下游消费可见，等待消费者获取并消费。

  - 消息回滚: 若第二个阶段如果事务执行结果明确为回滚，服务端会将半事务消息回滚，该事务消息流程终止。

- 消费中:  消费被消费者获取， 并按照消费者本地的业务逻辑进行处理的过程。此时服务端会等待消费者完成消费，如果一定时间后没有收到消费者的响应，RocketMQ会对消费进行重试处理。
- 消息提交: 消费者完成消息处理，并向服务端提交消费结果，服务端标记当前消费已被处理(包括消费成功和失败)。 消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。
- 消息删除: RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。

我们学习新事物或者新技术，总是先找特性最少得，与我们旧有的知识关联上，一种学习思维是在学习RocketMQ的时候，我们可以先学最普通的消息，然后再学习事务消息，延时消息、顺序消息，这其实也就是找不同，他们身上不同的点就是应用于不同的场景。

 经常会有些文章标题有了什么，为什么还要有，在我看来这就是在找另一种东西的诞生的动机，找不同点，找另一种技术的适应场景。另一种学习思维是我们走演绎式思维，从普通消息结合实际场景推导到我们想要什么样的消息的生产消费流程，最终得到RocketMQ中的事务消息，这两种思维可以互补，演绎式思维明确设计思路，明确设计思路的过程也就是明确使用场景的过程，找不同有时候也会疑惑为什么要这样设计，为什么要多一步回查这个过程。我们在《当数组遇上队列: Java线程安全实现详解(一)》提到这样一种思维:

> 当你养成一种分析问题、琢磨文章的习惯之后，日积月累；你便会感到复杂的东西也是由少数几个大的部分组成的。这些部分出现的原因和它们之间的相互关系也是可以理解的。与此同时，由于读的东西多了，运算的技巧也高了，你会发现，一些复杂的推演过程大部分是由某些必然的步骤所组成，就比较容易抓住新的关键性的部分 ---  越民义

我们分解RocketMQ事务消息的设计思路也就是为了解决网络不可靠，要引入回查，而网络不能保证绝对可靠，为了避免网络抖动，这里就需要定时回查，分解RocketMq组成也就是半消息+定时器。

回忆一下，我们在《RocketMQ学习笔记(二)相识篇》中讲事务消息的时候，讲的太过粗陋，说粗陋是因为虽然也考虑到了网络抖动，发送二次确认可能收不到，但是没有结合业务场景和普通消息的比较，也就是理论和实践结合的不够充分，就会认识不深刻。

###  延时消息

所谓延时消息也就是消息在到达RocketMQ之后，等待一段时间才能被消费者消费的消息，相当朴实无华，一个典型的场景就是订单下单之后未支付，在一定时间内关闭订单，我们就可以使用延迟消息来处理对应的场景，RocketMQ的场景的使用建议是避免大量相同定时时刻的消息，定时消息的实现逻辑需要先经过定时存储等待触发，定时时间到达后才会被投递给消费者。因此，如果将大量定时消息的定时时间设置为同一时刻，则到达该时刻后会有大量消息同时需要被处理，会造成系统压力过大，导致消息分发延迟，影响定时精度。RocketMQ 在5.0 之后支持任意精度的延迟消息，在5.0之前只支持18个等级的延迟投递:

![](https://a.a2k6.com/gerald/i/2024/08/25/67554.jpg)

5.0 之前发送延时消息:

```java
// 等级为3 延迟10s
rocketMQTemplate.syncSend("transation-order",MessageBuilder.withPayload("hello world").build(),3);
```

5.0之后发送延迟消息:

```java
// 延迟五秒
rocketMQTemplate.syncSendDelayTimeSeconds("transation-order",MessageBuilder.withPayload("hello world").build(),5);
```

### 顺序消息 

而顺序消息则是一种对消息发送和消费顺序有严格要求的消息，对于一个指定的Topic，消息严格按照先进先出(FIFO)的原则进行消息发布和消费，后发布的消息后消费。一个典型的场景就是订单的生命周期:  生成、付款、发货，这三个操作是顺序执行的，如果是普通消息，这三个操作对应的顺序可能会被放入到不同的队列中，不同队列的消息无法保证顺序，就算我们设置了一个队列，由于我们调用操作系统的函数成功，也不代表到达了RocketMQ的broker，如果是并发调用，就更加无法保证顺序了，所以为了保证消费顺序，我们需要保证生产顺序，这也就是生产者单线程，现在到broker，一个topic对应若干队列，对于订单来说，订单之间并不需要保持时间上的顺序，我们只需要保证相同订单关联的消息在一个队列就好，对此我们可以选择用订单号对队列数进行取余。

![](https://a.a2k6.com/gerald/i/2024/08/25/xhn2.jpg)





生产顺序保证了之后现在考虑消费顺序，在上面我们提到了消费位点的概念，RocketMQ消费成功的时候会将这个位点前移，如果消费的时候是互斥的，也就是等这个消费者在消费期间，其他消费者在等待这个消费者消费成功之后再消费，这无疑会大大的降低消费者的消费速度。由此就引出了RocketMQ的消费者类型:

- PushConsumer:    高度封装的消费者类型，消费消息仅通过消费监听器处理业务并返回消费结果。
- SimpleConsumer: 一种接口原子型的消费者类型，消息的获取、消费状态提交以及消费重试都是通过消费者业务逻辑主动发起调用完成。  可以一次性来取多条消息。
- PullConsumer:  Pull消费看起来和SimpleConsumer有些重合，来看文档像是在PushConsumer的基础上进行封装，一般的消费方式也就是服务端推和客户端主动去拉。

消费者类型为PushConsumer时， Apache RocketMQ 保证消息按照存储顺序一条一条投递给消费者，若消费者类型为SimpleConsumer，则消费者有可能一次拉取多条消息。此时，消息消费的顺序性需要由业务方自行保证

## 消费模式

在RocketMQ的领域模型中，同一个消息支持被多个消费者分组订阅，，每个消费者可消费到消费者分组内所有的消息，各消费者分组都订阅相同的消息，这在RocketMQ中被称为广播。

![](https://a.a2k6.com/gerald/i/2024/09/01/76fff.jpg)

同时，对于每个消费者分组可以初始化多个消费者，这些消费者共同分担消费者分组内的所有消息，实现消费者分组内流量的水平拆分和均衡负载。

![](https://a.a2k6.com/gerald/i/2024/09/01/78y7y.jpg)

## 总结一下

本文属于重学RocketMQ的第一篇，本来这篇还揉进了持久化的概念，但想来持久化的内容比我想象的要多，所幸就分拆出去了，我们本篇从消息队列这个名词讲起，队列具备先进先出这个特点，RocketMQ按照进入队列的顺序写入存储，消息队列表达的另一种语义是生产者消费模型，生产者将消息放入队列中，消费者消费消息，在RocketMQ中存储消息的角色被称为broker，而broker可以集群，为了实现生产者和broker解耦，RocketMQ引入了NameServer，生产者通过NameServer路由到broker中，消费者通过NameServer进行消费。这也就是RocketMQ的结构，所谓结构也就是组成和联系，RocketMQ由几个部分组成，这几个部分之间的联系，就像是水和面，做成了大饼，糅合烤制的过程就是将水和面之间建立联系。

然后我们从最普通的普通消息入手，讲了普通消息的周期，有讲了RocketMQ的发送方式，单向发送和异步发送，在这个基础上我们使用了演绎式思维，先是在网络无法保证绝对可靠的基础上进行推导，也就是说在发送的时候报超时异常，我们无法确认消息是否被成功投递，于是为了追求更高的可靠性，我们引入了反查机制，也就是说这种类型的消息在到达消息队列的时候，对消费者不可见，随后就会执行本地事务，如果这里提交了，那么这条消息对消费者就是可见的，如果回查这个过程也发生了异常，由于网络抖动发生的异常，那么RocketMQ会主动回查，这也就是事务消息的设计思路。

我们再借助演绎式思维推导出来事务消息之后，我们同时又分解事务消息的结构，发现事务消息也就是半消息+二次确认+回查。同时回忆了学习新技术的另一种方式，也就是跟旧有的建立连接，也就是经典问句，有了这个，为什么还要有，回答为什么还有这个过程也就是找使用场景的过程，不同点就是适用的场景。但这种思维并不见得是银弹，有时候出现另一种技术的动机可能不太强烈，我们需要用多种思维去看待问题。再接着我们讲解了延时消息，顺序消息。消费模式也就是广播消费和负载均衡。

## 参考资料

[1] How to fix java.net.SocketException: Broken pipe?  如何修复 java.net.SocketException：管道损坏？ https://stackoverflow.com/questions/2309561/how-to-fix-java-net-socketexception-broken-pipe

[2] Sending Large Data > 1 MB through Windows Sockets viz using the Send function  https://stackoverflow.com/questions/552612/sending-large-data-1-mb-through-windows-sockets-viz-using-the-send-function

 [3]  https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-send

[4] https://rocketmq.apache.org/zh/docs/introduction/03limits/ 

[5]  Java Write Operation io_append io_write   https://stackoverflow.com/questions/32923184/java-write-operation-io-append-io-write

[6] https://learn.microsoft.com/zh-cn/windows/win32/api/fileapi/nf-fileapi-writefile?redirectedfrom=MSDN

[7]   https://learn.microsoft.com/zh-cn/windows/win32/fileio/file-caching 

[8] https://docs.oracle.com/javase/tutorial/essential/io/buffers.html 

[9] CreateFileW 函数 （fileapi.h）https://learn.microsoft.com/zh-cn/windows/win32/api/fileapi/nf-fileapi-createfilew

[10] The Page Cache and Page Writeback https://github.com/firmianay/Life-long-Learner/blob/master/linux-kernel-development/chapter-16.md
