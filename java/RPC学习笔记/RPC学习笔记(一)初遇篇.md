# RPC学习笔记初遇篇(一)

> 最近似乎一直在写数据库方面的文章，这周打算换个口味写写RPC，老实说我是在学习Spring Cloud 系列的时候碰到RPC这个名词了, 也查了一些文章，
>
> 但是感觉总是差点什么，今天就彻底的学习一下RPC。

[TOC]

##  什么是RPC？

> 分布式计算中，远程过程调用（英语：Remote Procedure Call，RPC）是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一个地址空间(通常为一个开放网络的一台计算机）的子程序，而程序员就像调用本地程序一样，无需额外地为这个交互作用编程（无需关注细节）。RPC是一种服务器-客户端（Client/Server）模式，经典实现是一个通过发送请求-接受回应进行信息交互的系统。 
>
> RPC是一种进程间通信的模式，程序分布在不同的地址空间里。如果在同一主机里，RPC可以通过不同的虚拟地址空间（即便使用相同的物理地址）进行通讯，而在不同的主机间，则通过不同的物理地址进行交互。许多技术（通常是不兼容）都是基于这种概念而实现的。
>
> ---《维基百科》

从维基百科对RPC的阐释来看，我们可以提炼出来以下几个关键词:

- 分布式计算中的一个计算机通信协议
- 进程间通信方式

进程间通信我们可以联想到HTTP协议,  或者使用Socket编程，这也能实现进程间的通信。但是如果你看过RPC相关的文章的话，会发现大多时候多数文章大多都从函数讲起，即A进程希望能直接调用B进程的一个函数，然后接着往下引出RPC，RPC就是要像调用本地的函数一样去调远程函数，这是一种相当直观的描述。 但这是为什么呢?    我看了这些文章缠绕在心头的一个疑问是，你要调用另一个进程中的函数，通过HTTP不行吗？为什么好像又自己定义了一种网络通信协议一样。

### 为什么都选择了函数？ 

A进程希望直接调用B进程的一个函数，让我想到了WebService(不懂什么是WebService的，参看WebService学习笔记(一) 初遇篇)，也是服务端发布接口, 客户端访问WebService的时候直观上看好像是直接调用服务端的方法一样，但WebService是单向的，一方发布服务，一方访问，双方是不对等的。那我们上面的问题? 

HTTP不行吗？可以Spring Cloud Feign就是基于HTTP协议，也是RPC的一种形式。 但有人说Feign是一种伪RPC框架, 我其实并不明白这个“伪”在哪里，从定义上来说，Feign实现了进程间通信。也许是因为基于HTTP？ 但是在Dubbo官网在论述RPC上，也是承认基于HTTP也是RPC。

![基于HTTP的RPC.png](http://tva1.sinaimg.cn/large/006e5UvNly1gyebmzmr8hj30wt0bhjy9.jpg)

到此我们得出的结论是RPC并不一定从函数开始，两个进程之间进行HTTP进行通信也是RPC的一种形式，这也可以从维基百科对RPC的阐释可以看出来: 

> 分布式计算中，远程过程调用（英语：Remote Procedure Call，RPC）是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一个地址空间(通常为一个开放网络的一台计算机）的子程序，而程序员就像调用本地程序一样，无需额外地为这个交互作用编程（无需关注细节）

主要的点在于“一台计算机的程序调用另一个程序的子程序”，所以不一定要从函数开始。只是我当时看RPC相关的文章，大多都是从本地函数调用延伸到远程函数调用。本地函数调用可以简单理解为，这些函数位于的文件属于一个进程，在被加载到内存中变成进程的时候，位于相同的地址空间(粗略的说就是在一个房子里)，这样函数A调用函数B的时候，或者将函数B引入(你可以理解为两者位于同一个地址空间)就可以了。

但是如果函数A和函数B如果位于不同的进程，也就是说在不同的房子之内，两者进行通信就需要借助一些外力了，可以通过电话，也可以通过微信、QQ等多种方式。

那为什么其他形式的RPC协议都选择了以函数作为远程调用的基本单位，换句话说，就是为什么其他形式的RPC好像是一个服务端提供一个函数，客户端在调用的时候好像是直接调用这个函数一样。 我个人认为首先这是一种封装，A B两个进程之间需要进行交换，于是我们想到了网路编程，两个进程借助Socket来进行通信，但只使用比较简单的形式的通信，一旦我们交互的内容变的复杂，我们考虑的东西就变得多起来，比如拆包，分包之类的，如果你直接使用TCP协议的话，事实上我们只是想通信，交换信息，并不想太过关注于通信细节。于是我们开始制定协议，将函数参数和调用的函数名称作为协议的一部分，逐层抽奖，将复杂的协议编码和数据传输封装到一个函数中,尽可能的让开发者编写两个进程间进行通信的时候，尽可能的少关注通信的细节， 这也是RPC的其他形式像是一个进程直接调用一个进程的函数的原因。

### RPC协议简述

协议是RPC的核心，它规范了数据在网络中的传输内容和格式。除了必须的请求、响应数据外，通常还会包含额外空值数据，如单次请求的序列化方式、超时时间、压缩方式和鉴权等。

协议的内容包含三部分: 

- 数据交换格式: 定义RPC的请求和响应对象在网络传输中的字节流内容，也叫序列化方式
- 协议机构:  定义包含字段列表和各字段语义以及不同字段的排列方式。
- 协议通过定义规则、格式和语义来约定数据如何在网络间传输，一次成功的RPC需要通信的两端都能按照协议约定进行网络字节流的读写和对象转换。如果两端使用的协议不能达成一致，就会出现鸡同鸭讲，无法满足远程通信的需求。

关于序列化，我们在《浅谈序列化与反序列化》一文中已经简单的讨论过了, 序列化是（serialization）在计算机科学的数据处理中，是指将数据结构或对象状态转换成可取用格式（例如存成文件，存于缓冲，或经由网络中发送)。在Java的世界中一切皆对象，对应上了对象状态，在其他高级语言中，还存在结构体这样的数据结构, 结构体算是对象吗？ 某种程度上算是轻量级的对象, 对应上了数据结构。

反序列化就是将可取用格式再换为对应进程的数据结构或者对象状态，因为两个进程未必是使用一种高级语言来编写的。

### 小小的总结一下

RPC: Remote Procedure Call    远程过程调用是一个计算机通信协议，该协议允许运行于一台计算机的程序调用另一个地址空间(通常为一个开放网络的一台计算机）的子程序，而程序员就像调用本地程序一样，无需额外地为这个交互作用编程（无需关注细节）。有不同的外在表达形式: 

- 基于HTTP协议的RPC。
- 基于TCP的RPC，这种形式对外的表现好像就是一个进程直接调用另一个进程的方法一样。

## 简单的自定义RPC示例 Java版

首先我们先捋一下RPC过程，我们目前将调用方称之为Client，服务提供方称之为Server，Client想要调用Server中的方法，那么Client应该直接和服务产生联系吗？客户端直接记录所有的Server，因为服务提供方可能会有多个，在我们可以搭建集群的情况下，让客户端做负载均衡选择调用哪一个服务？这可不是一个好设计，我们应当设立一个注册中心来分发客户端的请求。我们借助网络来传输结果，所以还需要Socket，客户端传递全类名，方法执行参数。产生对应全类名的对象，这里就需要用到动态代理，我们需要产生服务端方法所在类的对象，客户端又没有实现类，只能用动态代理做一下。我们这里使用JDK基于接口的动态代理，客户端持有提供服务类的接口，服务端回传结果，动态代理造出对象，就能实现RPC的效果。

服务端接口代码: 

```java
public interface HelloService {
    String sayHi(String string);
}
public class HelloServiceImpl implements  HelloService{
    @Override
    public String sayHi(String string) {
        return string;
    }
}
```

RPC调用示意图:

![RPC示例.png](http://tva1.sinaimg.cn/large/006e5UvNly1gyeje5p67ej310h0h1q5v.jpg)

不懂动态代理参看文末最后的参考资料《代理模式-AOP绪论》, 客户端可能有多个发送请求，注册中心应当是并发的处理客户端的请求，一个线程处理一个连接。

注册中心: 

```java
// 注册中心
public interface ServerCenter {
    void start() throws IOException;
    void  register(Class serverName,Class serverImpl);
    void stop();
}
package org.example.rpc;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author xingke
 * @date 2022-01-15 17:08
 */
public class ServerCenterImpl implements  ServerCenter{

    /**
     * 注册过来的服务存放到多个Map.
     * 这里不考虑服务端集群的场景。
     */
    private static Map<String,Class> serviceRegister = new ConcurrentHashMap<>();

    /**
     * 这里采用JDK内置的线程池。
     * 实际中推荐自定义线程池
     */
    private static ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors()) ;

    /**
     * 启动标志位
     */
    private static boolean isRunning = false;


    @Override
    public void start() throws IOException {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress(8080));
        isRunning = true;
        while (true){
            System.out.println("server start....");
            Socket socket = serverSocket.accept();
            executor.execute(new RemoteCallTask(socket));
        }
    }

    @Override
    public void register(Class serverName, Class serverImpl) {
        serviceRegister.put(serverName.getName(),serverImpl);
    }

    @Override
    public void stop() {
        isRunning = false;
        executor.shutdown();
    }

    public static class RemoteCallTask implements Runnable{
        private Socket socket;

        public RemoteCallTask(Socket socket) {
            this.socket = socket;
        }
        @Override
        public void run() {
            try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                 ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream())
            ){
                String serviceName = objectInputStream.readUTF();
                String methodName = objectInputStream.readUTF();
                Class[] parameterTypes = (Class[]) objectInputStream.readObject();
                Object[] args = (Object[]) objectInputStream.readObject();
                Class serviceClass = serviceRegister.get(serviceName);
                Method method = serviceClass.getMethod(methodName, parameterTypes);
                Object result = method.invoke(serviceClass.newInstance(), args);
                objectOutputStream.writeObject(result);
            } catch (IOException | ClassNotFoundException | NoSuchMethodException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }
    }
}
public class RpcClient{
    public static <T> T getRemoteProxyObj(Class serverInterface, InetSocketAddress address) {
        return (T) Proxy.newProxyInstance(serverInterface.getClassLoader(), new Class[]{serverInterface}, (proxy, method, args) -> {
                    Socket socket = new Socket();
                    ObjectInputStream objectInputStream = null;
                    ObjectOutputStream objectOutputStream = null;
                    socket.connect(address);
                    objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                    objectOutputStream.writeUTF(serverInterface.getName());
                    objectOutputStream.writeUTF(method.getName());
                    objectOutputStream.writeObject(method.getParameterTypes());
                    objectOutputStream.writeObject(args);
                    objectInputStream = new ObjectInputStream(socket.getInputStream());
                    return objectInputStream.readObject();
                }
        );
    }
}
// 先启动服务端,要不然客户端会连接不上报异常。
public class RpcServerTest{
    public static void main(String[] args) {
        new Thread(()->{
            ServerCenter serverCenter = new ServerCenterImpl();
            serverCenter.register(HelloService.class, HelloServiceImpl.class);
            try {
                serverCenter.start();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }).start();
    }
}
public class RpcClientTest {
    public static void main(String[] args) throws ClassNotFoundException {
        HelloService httpService = RpcClient.getRemoteProxyObj(Class.forName("org.example.rpc.server.HelloService"), new InetSocketAddress(8080));
        // 方法在执行的时候开始远程调用
        httpService.sayHi("hello world");
    }
}
```

## 由此引出RPC框架

我们自定义的RPC还十分的原始, 还没有处理服务集群和跨语言调用等场景，除此之外在调用情况频繁的情况，线程占用会比较高，这块我们可以用Netty来进行优化，Netty是Java领域内一款网络编程框架，简化了网络编程，同时性能良好。那么Java领域内有没有已经做好的RPC框架呢，已经做好了完备的封装，且性能良好。当然是有的，而且不少:

- Dubbo: 阿里开源，基于Java开发，目前支持多种语言(Java Erlang Golang)。Dubbo的架构主要包含四个角色，如下图所示：

  ![Dubbo架构.png](http://tva1.sinaimg.cn/large/006e5UvNly1gyel41ge63j30qa0ibn08.jpg)

   	

​    Consumer是服务消费者，Provider是服务提供者，Registry是注册中心，Monitor是监控系统。

- Motan 新浪开源，基本架构同Dubbo类似。

​     ![Motan基本架构.png](http://tva1.sinaimg.cn/large/006e5UvNly1gyeld04ve0j30tn0cwwkc.jpg)

- Tars

​	Tars是基于名字服务使用Tars协议的高性能RPC开发框架，同时配套一体化的服务治理平台，帮助个人或者企业快速的以微服务的方式构建自己稳定可靠的分布式应用。

- Thrift

​    最初由Facebook于2007年开发，2008年进入Apache开源项目。Thrift通过一个中间语言(IDL, 接口定义语言)来定义RPC的接口和数据类型，然后通过一个编译器生成不同语言的代码（目前支持C++,Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, Smalltalk和OCaml等）,并由生成的代码负责RPC协议层和传输层的实，RPC是C-S模式的。

对于接口语言的理解，因为Thrift是支持多语言的，客户端和服务端能使用不同的语言开发，那么一定就要有一种中间语言来关联客户端和服务端的语言，那么这种语言就是IDL（Interface Description Language）语言，当我们定义了统一的IDL语言之后，在生成不同的语言之后，照样实现互相正确的通信。

- grpc

 grpc 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计,使用Protocol Buffers 作为接口描述语言。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持.

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。

## 参考资料

- 维基百科 https://zh.wikipedia.org/wiki/%E9%81%A0%E7%A8%8B%E9%81%8E%E7%A8%8B%E8%AA%BF%E7%94%A8
- 谁能用通俗的语言解释一下什么是 RPC 框架？https://www.zhihu.com/question/25536695/answer/1846152026
- Dubbo 官方文档 RPC 协议的选择
- JAVA常见应用-整理版（反射、RPC、SOCKET、文件、JSON、二维码、MAIL、加密等）https://www.bilibili.com/video/BV1k4411W7xq?p=7
- Thrift入门 | RPC基础&&Thrift概念   https://zhuanlan.zhihu.com/p/85033562
- 代理模式-AOP绪论  https://juejin.cn/post/6933947703923572743
- gRPC 官方文档中文版 V1.0  https://doc.oschina.net/grpc?t=56831