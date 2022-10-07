# Dubbo学习笔记(一)基本概念与简单使用

> 其实上周是打算写Dubbo的，但是发现Dubbo需要一个注册中心，因为也有学习Dubbo的计划，所以将Zookeeper和Dubbo放在一起介绍。

[TOC]

## 是啥？

我记得上一次看Dubbo的官网，Dubbo将自己定义为一款RPC 框架，到Dubbo3就变成了:

> Apache Dubbo 是一款微服务开发框架，它提供了 RPC通信 与 微服务治理 两大关键能力。这意味着，使用 Dubbo 开发的微服务，将具备相互之间的远程发现与通信能力， 同时利用 Dubbo 提供的丰富服务治理能力，可以实现诸如服务发现、负载均衡、流量调度等服务治理诉求。同时 Dubbo 是高度可扩展的，用户几乎可以在任意功能点去定制自己的实现，以改变框架的默认行为来满足自己的业务需求。

这里再讨论一下什么是RPC(这一点我在RPC学习笔记初遇篇(一) 讨论的已经很完备了)，不少介绍RPC的文章都会先从一个应用想要调用另一个应用的函数入手，但这不如维基百科直观：

> 分布式计算中，远程过程调用（英语：Remote Procedure Call，RPC）是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一个地址空间(通常为一个开放网络的一台计算机）的子程序，而程序员就像调用本地程序一样，无需额外地为这个交互作用编程（无需关注细节).

那为什么都从函数上入手，这是一种抽象和封装，两个进程需要进行通信，需要在TCP之上制定标准，也就是制定应用层的协议，可以选择HTTP(跨语言)，也可以基于TCP，自定义应用层的协议。我们可以在Dubbo3概念架构一节的协议印证我们的观点:

> Dubbo3 提供了 Triple(Dubbo3)、Dubbo2 协议，这是 Dubbo 框架的原生协议。除此之外，Dubbo3 也对众多第三方协议进行了集成，并将它们纳入 Dubbo 的编程与服务治理体系， 包括 gRPC、Thrift、JsonRPC、Hessian2、REST 等。以下重点介绍 Triple 与 Dubbo2 协议。
>
> 最终我们选择了兼容 gRPC ，以 HTTP2 作为传输层构建新的协议，也就是 Triple。

也就是我们可以认为HTTP协议是RPC的一种。至于微服务治理，这里不再重复的进行的讨论，参考我掘金的文章: 《写给小白看的Spring Cloud入门教程》。那既然你说HTTP协议是RPC的一种，那Dubbo的意义又何在呢，我个人认为是对HTTP协议进行改造吧，HTTP 2.0之前都是文本形式的，采取二进制字节流在网络中传输更快，除此之外使用HTTP协议传送数据，还需要自己动手将数据进行序列化，如果需要跨语言通信，定义的规则就更多了，Dubbo帮我们做好了这一切，甚至做的更多。这也就是我们学习Dubbo的意义所在。

梳理一下，RPC是一个计算机通信协议，那为什么都构造成了函数调用这一形式，这是因为从抽象来说是最合理的，我们可以大致推断演进一下:

- 首先是两个进程需要进行交换信息, 选择了TCP作为传输层的协议, 有的人选择了HTTP协议，因为这更简单一些, 当交换的信息比较简单，各个高级语言的Socket API 是可以满足其需求的。
- 如果我们期待这种交换的信息要更复杂一点呢，如果说我们选择TCP或HTTP作为应用间通信的形式，那么就有一些重复性的编码工作，比如取值，序列化为对象，如果是TCP协议还要考虑拆包等等，这无疑加重了程序员们编码的负担，那么能不能简化这个过程呢，屏蔽掉网络编程中涉及的复杂细节，抽象出来一个简单的模型呢，高级语言都内置有函数，那不如就从函数入手，让进程间交换信息就像是调用两个应用一样，这也就是很多RPC教程都从函数入手的原因，我觉得是由进程通信的过程中，为了屏蔽掉网络编程的复杂细节，选择从函数入手，这样让人容易理解一些，而不是一开始就是函数调用的形式。换句话说，多数程序员可能没了解过Socket 编程中的拆包之类的概念，但是一定理解函数这个概念，这是一种封装。

而Dubbo虽然在官网将自己声明为是一款微服务开发框架，但是在实际应用场景中，Apache Dubbo 一般会作为后端系统间RPC调用的实现框架，我们可以将其类比为HTTP协议对应的诸多HTTP Client。Dubbo提供了多语言支持，目前只支持Java、Go、Erlang这三种语言，那么我们自然提出一个问题，不同语言内置的数据类型、方法形式是不一样的，那作为RPC的实现者，它是如何做到跨语言的。

## 当然是引入一个中间层-IDL

为了和计算机进行通信，我们引入了编程语言，编程语言就是一个中间层，那么为了让不同的高级语言进行通信，Dubbo引入了IDL，Dubbo中推荐使用IDL定义跨语言服务，那什么是IDL，Dubbo官方并没有解释，于是我去了维基百科:

> An **interface description language** or **interface definition language** (**IDL**), is a generic term for a language that lets a program or object written in one language communicate with another program written in an unknown language. IDLs describe an interface in a language-independent way, enabling communication between software components that do not share one language, for example, between those written in C++ and those written in Java.
>
> 接口定义语言或者接口描述语言，是一种两种不同的语言进行通信的一种语言，IDL以独立于语言的任何形式描述接口，支持不同的高级语言进行通信。例如C++写的应用和Java写的应用
>
> IDLs are commonly used in   remote procedure call  software. In these cases the machines at either end of the *link* may be using different  operating systems  and computer languages. IDLs offer a bridge between the two different systems.
>
> IDL 通常在应用RPC，在RPC中通信的双方，链路的两端通常是不同的操作系统和编程语言。IDL为两个不同的系统提供了桥梁。

为什么又把英文贴出来了，维基百科不是也有中文吗？下面是维基百科中IDL的中文解释：

> **接口描述语言**（Interface description language，缩写**IDL**），是用来描述软件组件介面的一种计算机语言。IDL通过一种独立于编程语言的方式来描述接口，使得在不同平台上运行的对象和用不同语言编写的程序可以相互通信交流；比如，一个组件用C++写成，另一个组件用写成。

看到这个介面我懵了一下，我估计是对interface的翻译，interface的中文有界面的意思。那既然是一种计算机语言，我们合情推理，那就有语法，在Dubbo中提供了IDL的示例：

```IDL
syntax = "proto3";

option java_multiple_files = true;
option java_package = "org.apache.dubbo.demo";
option java_outer_classname = "DemoServiceProto";
option objc_class_prefix = "DEMOSRV";

package demoservice;

// The demo service definition.
service DemoService {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

Dubbo是如是描述的:

> 以上是使用 IDL 定义服务的一个简单示例，我们可以把它命名为 `DemoService.proto`，proto 文件中定义了 RPC 服务名称 `DemoService` 与方法签名 `SayHello (HelloRequest) returns (HelloReply) {}`，同时还定义了方法的入参结构体、出参结构体 `HelloRequest` 与 `HelloReply`。 IDL 格式的服务依赖 Protobuf 编译器，用来生成可以被用户调用的客户端与服务端编程 API，Dubbo 在原生 Protobuf Compiler 的基础上提供了适配多种语言的特有插件，用于适配 Dubbo 框架特有的 API 与编程模型。

 又出现了一个新名词: Protobuf,  Protobuf是啥？Apache Dubbo没有解释，我只好再诉诸于搜索引擎:

> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. --《Protocol Buffers》官网
>
> Protocol Buffers 是Google 为序列化结构数据设计的一种独立于语言、平台的一种可扩展机制, 类似于XML, 但是更小、更简单、更快。你只需要定义数据如何被结构化，然后用生成的源码，就能够在不同的语言中读取和写入你的结构化数据。

XML是一种描述数据的一种形式，那既然是类似于XML，那又是一种描述数据的一种形式，结合上面的跨语言语境，也就是说我们借助对应的Protobuf编译器用proto来生成调用客户端与服务端编程API。

![IDL调用图](http://tvax3.sinaimg.cn/large/006e5UvNly1h3sdhwdmfgj31130ghgo8.jpg)

### Protobuf 简单入门

既然是描述数据，那么就会有数据类型,Protobuf为了跨语言，声明了一些数据类型, 与各个语言的数据类型有对应的映射关系 , 这里简单列出一一下和java数据类型的映射关系:

- double ==>  java double
- float ==>     java float
- int64 ==>     java long
- uint32 ==>  java int
- bool ==>   java bool
- String  ==> java String

上面的示例中HelloRequest、HelloReply的字段每个都进行了赋值，但这并不是默认值, 而是字段编号，这些字段编号用于标识二进制形式的字段。到目前为止我们就只剩上面的几个optional看不懂了:

- java_multiple_files   

> 如果为true, 每个message 和 service 都会被生成为一个类。如果是false，则所有的message和service都会被生成到一个类中。

- java_package

> 生产的代码所处的位置，如果没有则会产生在package 后面声明的包。

- java_outer_classname

> 生产服务的名称。

- objc_class_prefix

> 很奇怪官方的示例为什么会把这个放进去，我查了很多资料，这个语法为objective-c所提供，用于为指定的类生成前缀。

Dubbo 还说:

> 使用 Dubbo3 IDL 定义的服务只允许一个入参与出参，这种形式的服务签名有两个优势，一是对多语言实现更友好，二是可以保证服务的向后兼容性，依赖于 Protobuf 序列化的兼容性，我们可以很容易的调整传输的数据结构如增、删字段等，完全不用担心接口的兼容性

到现在为止我们已经看懂了官方的示例，现在我们就要用起来。

## 基本使用示例

Dubbo官方推荐使用IDL，那我们还是使用官方的示例，来定义服务。官方提供了示例:

![Dubbo 官方提供的示例](http://tvax2.sinaimg.cn/large/006e5UvNly1h3sgjodizxj31ey0ndqb3.jpg)

我这里贴下指令:

```shell
git clone -b master https://github.com/apache/dubbo-samples.git
cd dubbo-samples/dubbo-samples-protobuf
# 要求配置maven的环境变量
mvn clean package
# 运行 Provider
java -jar ./protobuf-provider/target/protobuf-provider-1.0-SNAPSHOT.jar 
# 运行 consumer
java -jar ./protobuf-consumer/target/protobuf-consumer-1.0-SNAPSHOT.jar 
```

  然后你会发现跑不起来, 我跑是这样:

![官方的示例-Zookeeper](http://tva2.sinaimg.cn/large/006e5UvNly1h3sgmm5b8zj315m0m9hdt.jpg)

Zookeeper连接不上，这里批评一下Apache Dubbo的官方示例文档，完全跑不起来，真的是在用心写文档吗！ 这个Zookeeper我们在《Zookeeper学习笔记(一)基本概念和简单使用》已经介绍过了，一个分布式协调服务，提供命名服务。那Dubbo这个示例中为什么要求连接Zookeeper呢，为了解耦合，我们如果直接通过IP+端口的方式去调服务提供者的服务的话，这样就耦合在一起了，假设生产上换台机器我们还得改代码，再有就是集群的情况下，我知道服务名就好，不需要知道特定ip的，这也就是注册中心的概念，服务提供者将服务注册到注册中心，消费者提供服务名和腰消费的服务即可。Dubbo服务常见的架构:

![Dubbo的架构](http://tva2.sinaimg.cn/large/006e5UvNly1h3sgubynzbj30q10iagok.jpg)

Monitor是监控，监控服务调用，这里我们不做介绍。其实在Dubbo提供的源码中也默认连接了Zookeeper这个注册中心：

![Dubbo源码](http://tvax1.sinaimg.cn/large/006e5UvNly1h3sgy2fq70j30xd0ddn5m.jpg)

还好我们已经装过了Zookeeper，我们将地址改掉就行。注意哈，高版本的JDK改动了很多东西，Dubbo 官方提供的示例，在JDK 17下可能跑不起来，如果到时候编译报错，将环境调整到JDK8就行，我自己测试的话，JDK 11也可以的,但是有的时候会报Zookeeper连接不上的错误。我用IDEA启动一下:

![Dubbo Service 成功的被启动](http://tvax2.sinaimg.cn/large/006e5UvNly1h3sidkj2b5j317a0a0e17.jpg)

然后启动消费者:

![消费者消费成功](http://tvax2.sinaimg.cn/large/006e5UvNly1h3sif6ga65j31dn09yx4f.jpg)

发现是没问题的，我分析了一下为啥在我的windows power shell 中出现Zookeeper 连接不上的原因可能是我配置的JDK环境变量是 JDK 11的，在IDEA中能够成功跑起来的原因是IDEA用的是JDK8.

颇有种你发任你发，我接着用JDK8的感觉。

### 从示例中分析

![proto文件](http://tva1.sinaimg.cn/large/006e5UvNly1h3sj036wbwj30nd0bqwif.jpg)



 pom里面有Protobuf插件:

```xml
<plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.5.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.7.1:exe:${os.detected.classifier}</protocArtifact>
                     <!--将protobuf文件输出到这个目录-->
                    <outputDirectory>build/generated/source/proto/main/java</outputDirectory>
                    <clearOutputDirectory>false</clearOutputDirectory>
                    <protocPlugins>
                        <protocPlugin>
                            <id>dubbo</id>
                            <groupId>org.apache.dubbo</groupId>
                            <artifactId>dubbo-compiler</artifactId>
                            <version>${compiler.version}</version>
                            <mainClass>org.apache.dubbo.gen.dubbo.Dubbo3Generator</mainClass>
                        </protocPlugin>
                    </protocPlugins>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>test-compile</goal>
                        </goals>
                    </execution>
                </executions>
  </plugin>
```

```
public class ConsumerApplication {
    public static void main(String[] args) throws Exception {
        // 加载Spring的上下文文件
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-consumer.xml");
        context.start();
        // 从容器中获取demoService
        DemoService demoService = context.getBean("demoService", DemoService.class);
        // 构建入参
        HelloRequest request = HelloRequest.newBuilder().setName("Hello").build();
        // 实现RPC
        HelloReply reply = demoService.sayHello(request);
        System.out.println("result: " + reply.getMessage());
        System.in.read();
    }
}
public class Application {
    public static void main(String[] args) throws Exception {
        // 加载bean
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-provider.xml");
        context.start();
        System.out.println("dubbo service started");
        // 避免应用关闭
        new CountDownLatch(1).await();
    }
}
/**
 * 真正的实现类
 */
public class DemoServiceImpl implements DemoService {
    private static final Logger logger = LoggerFactory.getLogger(DemoServiceImpl.class);

    @Override
    public HelloReply sayHello(HelloRequest request) {
        logger.info("Hello " + request.getName() + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return HelloReply.newBuilder()
                .setMessage("Hello " + request.getName() + ", response from provider: "
                        + RpcContext.getContext().getLocalAddress())
                .build();
    }

    @Override
    public CompletableFuture<HelloReply> sayHelloAsync(HelloRequest request) {
        return CompletableFuture.completedFuture(sayHello(request));
    }
}
```

##  总结一下

进程间的通信可以直接使用应用层的协议如HTTP、也可以基于TCP自定义应用层的协议，但是对于面向对象的高级语言来说，数据接过来说还要有一个序列化过程，如果说是基于TCP的话，我们还要考虑拆包的问题，我们都喜欢的东西，我们能否屏蔽中间的通信细节呢，两个进程的通信就像是调用各自的函数一样，这也就是RPC，但是如果两个进程是用不同的语言编写的呢，为了语言中立，我们引入IDL，跨语言，但是通信还是要选择应用层的协议，要么自己基于TCP，要么基于已有的应用层协议，比如说HTTP，但是现在已经有高度成熟的RPC框架了，你不需要关心那么多HTTP协议的通信细节、以及序列过程，在一定的配置下，你可以实现像调本地函数一样，调另一个进程的函数，高度的封装。RPC是在演进的过程中选择了函数作为载体，这是为了屏蔽掉通信和序列化的细节，而不是一开始就是就是以函数的形式出现，本质上是一种通信协议。

## 参考资料

- 从原理到操作，让你在 Apache APISIX 中代理 Dubbo 服务更便捷  https://dubbo.apache.org/zh/blog/2022/01/18/%E4%BB%8E%E5%8E%9F%E7%90%86%E5%88%B0%E6%93%8D%E4%BD%9C%E8%AE%A9%E4%BD%A0%E5%9C%A8-apache-apisix-%E4%B8%AD%E4%BB%A3%E7%90%86-dubbo-%E6%9C%8D%E5%8A%A1%E6%9B%B4%E4%BE%BF%E6%8D%B7/

- gRPC 官方文档中文版 https://doc.oschina.net/grpc?t=60140
