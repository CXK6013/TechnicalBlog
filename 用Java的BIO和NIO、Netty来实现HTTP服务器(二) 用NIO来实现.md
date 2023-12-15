# 用Java来实现BIO和NIO模型的HTTP服务器(二) 

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
// 监听连接,该方法会阻塞到这里直到有连接建立完成
// 发起系统调用
Socket socket = serverSocket.accept();
while (true){
    InputStream socketInputStream = socket.getInputStream();
    byte[] readByte = new byte[4096];
    // 这里其实数据不见得立马可以读, 因为数据不代表立马可以读
    // 发起系统调用
    int readTotalNumber  = socketInputStream.read(readByte);
    String s = new String(readByte,0,readTotalNumber);
    System.out.println(s);
}
```

这里我们详细的解释一下为什么数据为什么不可读，我们知道现代高级语言程序想要使用网络服务，必须调用操作系统提供的接口，这种调用也被称为系统调用，发生系统调用的时候发生了什么？

> 系统调用指运行在用户空间(User space)向操作系统内核请求需要更高权限运行的服务。



![](https://a.a2k6.com/gerald/i/2023/12/11/pcej.png)

让我们回忆一下操作系统的内核空间和用户空间，计算机的内存被切割为两个部分:

用户空间:  正如同的它的名字一样，处内核以外所有的用户进程运行在这个空间上。内核的作用是管理在该空间内运行的应用程序，防止他们互相干扰，避免机器出现混乱。

内核空间:  内核的代码和数据存放在这个位置上，内核也是一个进程，内核运行在这块内存之上。

与之相对的两个概念是内核模式(Kernel mode，有资料也称为System Model 系统模式)，是Linux中CPU运行模式之一。另一种是用户模式(user model)，是用户程序的非特权模式，也就是内核以外的所有操作模式。当CPU运行在内核模式下面，默认运行的是受信任的程序，因此它可以执行任何指令和访问任何内存位置。内核(操作系统的核心，对系统中发生的一切拥有完全的控制权)是被信任的软件，其他程序不受信任，因此所用的用户进程都必须使用系统调用来请求内核执行特权指令，比如创建进程、I/O操作。

术语System Call 和 System Exit是实际汇编语言指令的占位符，分别用于将CPU从用户模式切换到内核模式，从内核模式切换到用户模式。当用户进程发起一个调用，Linux会为这个调用分配一个系统调用编号，Linux使用系统调用表(System Call Dispatch Table)存储调用编号和实际执行系统调用对应功能的函数。

实际运行上面的程序会发现 , 会出现下面的异常: 

```java
Exception in thread "main" java.net.SocketException: Connection reset
	at java.net.SocketInputStream.read(SocketInputStream.java:210)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.net.SocketInputStream.read(SocketInputStream.java:127)
	at com.example.quicktest.ServerSocketDemo.oldAPI(ServerSocketDemo.java:42)
	at com.example.quicktest.ServerSocketDemo.main(ServerSocketDemo.java:20)
```

原因在于我们上面写的程序是是在不断的处理连接的，收到数据之后，再读收到了RST包，那什么是RST包，让我们回忆一下关于TCP的经典面试题三次握手和四次挥手:

![](https://a.a2k6.com/gerald/i/2023/12/14/4ktl.png)

声明图片来自于参考文档[16]

我们一边写这个一边将我们的所学联系起来，在TCP协议中客户端主动关闭连接，发起系统调用之后，内核发一个TCP数据包，这个TCP数据包的终止位FIN置成1，序号seq = u，它等于前面已传送的数据的最后一个字节的序号加1，这时Client进入FIN-WAIT-1(终止等待1)状态，等待服务端的确认。

服务端收到FIN之后，向Client发送确认包ack = u +1，这个u等于前面已经传送的数据的最后一个序号加1 ， Client收到ack之后进入到FIN-WAIT-2，服务端如果没有数据要发送了就会向客户端发送TCP数据包，数据包中的FIN置为1, 服务端还必须重复上次已发送过的确认好ack = u + 1.这时B就进入LAST-ACK(最后确认)状态，等待A的确认。

客户端再次收到服务端的连接释放报文段后，必须对此发出确认。在确认报文段中把ACK置为1、确认号ack = w + 1 ，而自己的序号是seq = u + 1(根据TCP标准, 前面发送给的FIN报文段要消耗一个序号)。然后进入到TIME-WAIT(时间等待)的状态。请注意，现在TCP连接还没有释放掉。必须经过时间等待计时器(TIME-WAIT timer) 设置的时间2MSL后，A才进入到CLOSED状态

然后我们客户端退出之后，没有显式的调用close，也就是客户端没有走正常流程关闭TCP连接，但对于操作系统来说还是要回收对应的资源，所以进程退出的时候，内核会监测到这个变化，因为这个连接已经是异常了。 在传输控制协议（TCP）连接的数据包流中，每个数据包都包含一个TCP包头。这些包头中的每一个都包含一个称为“复位”（RST）标志的位。在大多数数据包中，该位设置为0，并且无效；但是，如果此位设置为1，则向接收计算机指示该计算机应立即停止使用TCP连接；它不应使用连接的标识号（端口）发送更多数据包，并丢弃接收到的带有包头的其他数据包，这些包头指示它们属于该连接。

![](https://a.a2k6.com/gerald/i/2023/12/14/onvi.png)

所以上面的代码客户端完善一点应当是这个样子: 

```java
// 借助try-with-resources 自动关闭释放资源
try (Socket socket = new Socket()){
    socket.connect(new InetSocketAddress("127.0.0.1",8080));
    try(OutputStream outputStream = socket.getOutputStream()){
        outputStream.write("hello world".getBytes());
    }
}
```

accept调用也应该放在whie(true)循环里面，所以代码应当改成下面这个样子：







在《用Java的BIO和NIO、Netty来实现HTTP服务器(一) 》里面我们用的是在1.4引入的新API，这套API的优势就是比较统一，可以通过ServerSockeChannel的configureBlocking来制定使用BIO还是NIO，所以上面服务端的写法可以等价转换为: 

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.bind(new InetSocketAddress(8080));
while (true){
	SocketChannel socketChannel = serverSocketChannel.accept();
    ByteBuffer byteBuffer = ByteBuffer.allocate(4096);
    socketChannel.read(byteBuffer);
    byteBuffer.flip();
    byte[] readDataArray = new byte[byteBuffer.limit()];
    byteBuffer.get(readDataArray);
    String readData = new String(readDataArray);
    System.out.println(readData);
}
```

ByteBuffer是个容器，







## 前面面临的问题

下面是前面设计的架构图





## NIO简介

上面模型也被称为BIO模型，也就是Blocking Input/Output, 那让我们分析一下阻塞在哪里





## 改造





## Reactor模式





## 总结一下





## 参考资料

[1] what happens after read is called for a Linux socket  https://stackoverflow.com/questions/10226294/what-happens-after-read-is-called-for-a-linux-socket

[2] What is the difference between the kernel space and the user space?  https://stackoverflow.com/questions/5957570/what-is-the-difference-between-the-kernel-space-and-the-user-space

[3] What is difference between User space and Kernel space?  https://unix.stackexchange.com/questions/87625/what-is-difference-between-user-space-and-kernel-space

[4] Linux网络数据包接受过程  https://simonzgx.github.io/2020/08/17/Linux%E7%BD%91%E7%BB%9C%E6%95%B0%E6%8D%AE%E5%8C%85%E6%8E%A5%E5%8F%97%E8%BF%87%E7%A8%8B/

[5] Kernel Mode Definition https://www.linfo.org/kernel_mode.html

[6] What are high memory and low memory on Linux? https://unix.stackexchange.com/questions/4929/what-are-high-memory-and-low-memory-on-linux/5151#5151 

[7] Implementing System Calls https://www.cs.swarthmore.edu/~kwebb/cs45/s19/labs/lab2.html

[8] LinuxSystemCalls.pdf http://comet.lehman.cuny.edu/jung/cmp426697/LinuxSystemCalls.pdf

[9] The Operating System https://www.cs.swarthmore.edu/~kwebb/cs31/s15/bucs/system_calls.html

[10] Difference between System call and System call service routines  https://stackoverflow.com/questions/70410917/difference-between-system-call-and-system-call-service-routines

[11] what is “java.net.SocketException: Connection reset” https://learn.redhat.com/t5/General/what-is-java-net-SocketException-Connection-reset/td-p/6757

[12] What does "connection reset by peer" mean?  https://stackoverflow.com/questions/1434451/what-does-connection-reset-by-peer-mean 

[13] TCP: Differences Between FIN and RST  https://www.baeldung.com/cs/tcp-fin-vs-rst

[14] FIN vs RST in TCP connections  https://stackoverflow.com/questions/13049828/fin-vs-rst-in-tcp-connections

[15] TCP学习笔记(二) 相识篇  https://juejin.cn/post/7103092974841511950#heading-2 

[16] TCP-4-times-close https://wiki.wireshark.org/TCP-4-times-close.md
