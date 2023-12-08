# 用Java的BIO和NIO、Netty来实现HTTP服务器(二) 

> 翻了一下(一)发现整体还是不大好, 这里重新再梳理一下

## 前言

这是一个系列的文章，按照规划是用Java标准库、Netty来实现一个非常简单的HTTP服务器，HTTP服务器我们可以使用Java标准库提供的api，实现BIO、NIO模型的HTTP服务器，然后再用Netty实现，前一篇我们写的类在这一篇还可以用到，让我们回忆一下上一篇我们讲了什么，我们回顾了通信的发展史，从最开始的点对点链路，到总线链路，再到mac地址，ip地址，最后引出两台计算机之间的通信事实上是两台计算机上面进程之间的通信，那么该数据包到达计算机之后该如何交给哪个进程呢，这也就是端口，运输层引入了端口的概念，ip+端口构成TCP连接的一端，那么要通信就首先要建立连接，也就是三次握手，连接建立之后就可以通过连接来传输数据了，那么该如何管理连接呢?  操作系统在连接建立的时候，会将这个消息通知给进程。

我们的程序通过Socket来和操作系统进行交互，这里的Socket指的是操作系统提供的服务，当一个进程向一个进程发起请求建立连接的请求，这个数据包首先经过操作系统提供的接口向下传递，然后通过互联网中层层设备转发来到另一个进程所在的计算机上，两台计算机完成连接建立之后，通知上层的应用程序。

![](https://a.a2k6.com/gerald/i/2023/12/03/l0hw.jpg)

当我们编写的应用程序需要使用网络服务的时候，在Java中我们首先要明确自己是客户端还是服务端，客户端是发起请求的一方, 我们客户端的代码可以这么写: 

```java
Socket socket = new Socket();
// 代表客户端请求连接ip地址为127.0.0.1,端口为8080的进程
socket.connect(new InetSocketAddress("127.0.0.1",8080));
// 连接之后获取输入流
OutputStream outputStream = socket.getOutputStream();
// 写入hello world
outputStream.write("hello world".getBytes());
```

 ![](https://a.a2k6.com/gerald/i/2023/12/03/4zmvg.jpg)

客户端在发起连接请求的时候，这个请求会首先到达操作系统，操作系统会为这次调用所需要的一些资源(CPU时间，网络带宽、存储器空间等)分配该应用进程。操作系统为这些资源总和创建一个套接字描述符的好嘛来表示，然后将这个套接字描述返回给应用进程。看到这里有一个疑问，客户端没声明自己在那个端口上，那服务端在给客户端发送消息的时候，这个消息到达操作系统应该给谁呢? 答案是操作系统会从可用的端口分配一个，但是如果你想绑定指定的端口其实也可以，Socket这个类里面提供了bind方法:

```java
public void bind(SocketAddress bindpoint) throws IOException 
```

客户端写完之后我们来写服务端， TCP协议中我们需要一个服务端，监听指定的端口。在Java里面写服务端的应用程序事实上有两套API，一套是JDK 1.0引入的以ServerSocket为中心的API，一套是JDK  1.4 引入的以ServerSocketChannel为核心的API。第一套写监听的方法如下:

```java
ServerSocket serverSocket = new ServerSocket();
// 绑定在8080端口
serverSocket.bind(new InetSocketAddress(8080));
// 监听连接,该方法会阻塞到这里直到
Socket socket = serverSocket.accept();
while (true){
    InputStream socketInputStream = socket.getInputStream();
    byte[] readByte = new byte[4096];
    int readTotalNumber  = socketInputStream.read(readByte);
    String s = new String(readByte,0,readTotalNumber);
    System.out.println(s);
}
```

这里只是简单做个回忆





在JDK 11面临的问题

连接建立操作系统发起系统调用，应用程序可以选择读或者写，但是连接建立未必就是可读，原因有两个，一是数据还未到来，连接建立的时候操作系统请求发起系统调用 ，二是数据还不可读，

## 前面面临的问题







## NIO简介

上面模型也被称为BIO模型，也就是Blocking Input/Output, 那让我们分析一下阻塞在哪里









## 开始梳理我们需要的组件











## 回答一下为什么没有AIO





## 总结一下



## 参考资料

[1] what happens after read is called for a Linux socket  https://stackoverflow.com/questions/10226294/what-happens-after-read-is-called-for-a-linux-socket
