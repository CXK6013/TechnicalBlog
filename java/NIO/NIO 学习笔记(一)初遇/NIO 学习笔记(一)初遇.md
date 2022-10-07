## 前言

NIO是什么?  这个我还是老习惯先去翻翻官方写的指导书[《The Java™ Tutorials》](https://docs.oracle.com/javase/tutorial/essential/io/summary.html)
然后《The Java™ Tutorials》只是介绍了基本操作,想了解更多的话，去 [OpenJDK: NIO](http://openjdk.java.net/projects/nio/)。然后我就在这个页面找到了NIO的相关介绍。

## NIO的前世今生

NIO 意味 New I/O,主要来自于[JSR 203](https://jcp.org/en/jsr/detail?id=203)和[JSR 51](https://jcp.org/en/jsr/detail?id=51)。JSR: Java Specification Requests java  规范提案。什么意思呢？就是建议，就是对java的建议。然后JCP(Java Community Process)就会审议你的提案，如果你的提案获得通过，那么提案就会进入到JDK中。

这两个提案都是建议加强java IO的接口,分成两个方面: 定义新的接口和增强过去的接口。JSR 203是 JSR 51的延续。JSR 51被称为NIO.1, JSR则被称为NIO.2。主要包括三个部分:
-  [A new filesystem interface that supports bulk access to file attributes, change notification, escape to filesystem-specific APIs, and a service-provider interface for pluggable filesystem implementations;](https://jcp.org/en/jsr/detail?id=203)
能够访问更多文件属性和改变标记的和可以避免去使用特定文件系统接口的新文件系统接口。
-   [An API for asynchronous (as opposed to polled, non-blocking) I/O operations on both sockets and files;](https://jcp.org/en/jsr/detail?id=203#orig) 
支持异步IO操作的接口,与轮询相反,不阻塞。所以这部分的接口有时也被称为NIO。
-  [The completion of the socket-channel functionality defined in JSR-51, including the addition of support for binding, option configuration, and multicast datagrams](https://jcp.org/en/jsr/detail?id=203#orig)
完善定义在JSR-51中socket-channel的功能,包括添加对绑定,选项配置,多播数据报的支持。

本篇我们主要介绍的是异步IO，因为是非阻塞的，所以有时候也被称作NIO(non-blocking i/o)


## 为什么要引入异步IO呢?

让我们从CS架构说起,CS架构的一个典型应用就是聊天软件，大致的结构像下面这样。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1d90975cc70b~tplv-t2oaga2asx-image.image)

服务端用于传递客户端的信息。在没有异步IO之前，我们的代码是这样写的:

```java
 ServerSocket serverSocket = new ServerSocket(6666);
 Socket socket = serverSocket.accept();
 InputStream inputStream = socket.getInputStream();
```
请注意accept()方法是阻塞的,直到连接建立 。那么比较有局限的是，当前代码只能处理一个客户端，而且是出于一直等待状态。直到连接建立，我们才能处理客户端发送过来的信息。想一想，这么设计也有一定的合理之处，服务端和客户端的连接不建立，客户端怎么能发送信息呢？ 那么肯定有人要问了，你这个服务端只能处理一个客户端啊，因为你只有一个Socket啊。
那我就开多线程，一个线程处理一个连接。

那么在java中常规情况下，一个线程是要消耗1M内存的，这是一个比较大的消耗。这就是同步I/O的缺点，前面的事情不做完，后面的事情别想做。那么自然会有人想，能否不要一直监听啊，连接建立了，你在通知我，我再处理你发过来的信息。

这就是异步。这反应到聊天室这里就是，很多客户端跟服务端建立连接，但是总要有先后顺序的，你跟我建立连接完成，我再处理该客户发送过来的信息。这就是为什么要引入异步IO的原因,原本的IO不够灵活，消耗资源过多。

## Buffer Channel Selector


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1d95efeda2ac~tplv-t2oaga2asx-image.image)
Buffer(缓冲区)用来存数据，channel(通道)用来从Buffer中读和写数据，传统的流是单向的，像InputStream和OutputStream的子类。那么Selector(选择器)呢? 当选择器遇到它所感兴趣的事情之后(比如连接建立完成)， 就会激活通道，通道从缓冲区中取出客户端发过来的信息。

### Buffer
Buffer在java中就是一个抽象类,我们可以将其视作一个存储特定基本类型数据的容器。在Buffer类中只定义了基本属性和基本操作。我们主要看它的八个子类:

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1d9a9dbbde26~tplv-t2oaga2asx-image.image)
- ByteBuffer
- IntBuffer
- StringCharBuffer
- CharBuffer
- FloatBuffer
- LongBuffer
- ShortBuffer
- DoubleBuffer
是的没有BooleanBuffer。
这8个子类中,用来装数据的都是数组。但是共有的属性是在Buffer中被定义。


#### 主要属性

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1da418401a15~tplv-t2oaga2asx-image.image)

- capacity 容量
- limit  指向第一个不能写或者不能读
- position   指向下一个将要被写入或读取的元素
- mark 标记 reset 方法之后  position 指向mark
- address 堆外内存地址  这个我们稍候在讲


#### 主要操作
我们主要以ByteBuffer为例来介绍,Buffer的相关操作。对于容器来说我们比较关心的就是两个:
- 读  从容器中获取元素   读对应  get(byte[] dst)          将缓冲区中的元素放入传入的字节数组中
- 写  向容器中放入元素   写对应  put(byte[] src)方法   将src中的元素放入缓冲区中

我们来借助demo来说明Buffer的常规操作:

```java
		 // 创建一个长度为buffer
        ByteBuffer buffer = ByteBuffer.allocate(100); 
        // position = 0 
        System.out.println(buffer.position());
        // limit = 100 
        System.out.println(buffer.limit());
        // capacity = 100 
        System.out.println(buffer.capacity()); 
	    // 向buffer,也就是数组中放入元素。一个字符对应数组的一个位置
        buffer.put("hello".getBytes());
	    // position = 5
        System.out.println(buffer.position());
         // limit = 100 
        System.out.println(buffer.limit());
        // capacity = 100 
        System.out.println(buffer.capacity());	        		
```
我们来思考这么一个问题,我们取的时候，也是从缓冲区拿到东西，放入到一个新的容器中，直白点说就是get(byte[] dst)这个方法中的字节数组,我们给多大比较合适。理想的做法通常是你这里面有多少，我这个就取多少。position刚好就标识了此时元素的数量。这就是flip方法做的事情。
##### flip()和compact()这两个好搭档
```java
// 从写转为读可以用,此时limit为Buffer的实际容量。
public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```
```java
    buffer.flip();
    // postion = 0
    System.out.println(buffer.position());
    // limit是5
    System.out.println(buffer.limit());
    byte[] array = new byte[buffer.limit()];
    buffer.get(array);
    // 打印hello
    System.out.println(new String(array));
```
flip和compact常常配合在一起使用。compact()的实际作用为:
> 将buffer中position和limit之间的数据到Buffer的0位置和(limit-position-1),注意是[position,limit)->[0,limit-position-1]


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1db0a1721973~tplv-t2oaga2asx-image.image)
常应用于两个通道之间互相传输数据,也就是一个channel用来写,一个channel用来读。像下面这样:

```java
 buf.clear();          // Prepare buffer for use
 while (in.read(buf) >= 0 || buf.position != 0) {
      buf.flip();
      out.write(buf);
      buf.compact();    // In case of partial write   
 }
```
有人这里可能就会问了，上面你不是说position是下一个将要读或者写的元素的位置吗? 此时position不是还没值吗？那你为啥要复制呢？ 因为Channel是非阻塞的,write()并不会将buffer中的数据全部写入。像下面这样:

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1db63cac5469~tplv-t2oaga2asx-image.image)


##### rewind 和 reset、clear
其实这里也就是介绍API的使用而已,可以直接看Buffer类上的注释的就可以了。写的话,一般就是从0位置开始写了。
此时重置Buffer的limit、position、mark属性即可。这也就是Buffer的clear()方法做的事情。
- rewind()

rewind的源码:
```java
public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
 }
```
在写转向读的时候可以调用, 也有称为读操作到读操作的。像下面这样: 
```java
 out.write(buf);    // Write remaining data
 buf.rewind();      // Rewind buffer
 buf.get(array);
```
 前提是limit已经被合适的设置,那不是要调一下flip方法吗？这个可以应用于反复的从buffer中获取数据。
- reset 

reset的源码:
```java
public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
```
reset 有重置的意思，这里我们可以将其理解为归档。默认情况下: 调用之后position=mark。
mark方法用于标记position的位置。
- clear
clear的源码:
```java
public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```
一切从头开始,在向缓冲区中放入元素之前要调用此方法。此时Buffer的状态由读状态转入写状态。
### Channel  
Channel本身不负责存储数据，在读或者写的时候，都是从buffer中获取。
Channel是一个接口，那我们该如何使用呢，或者说如何获取呢。
主要是两种方式:
- FileChannel的open方法
-  SocketChannel的open方法
-  ServerSocket的getChannel方法
-  DatagramChannel的open方法
- 字节流的getChannel(缓冲流和字符流拿不到通道)，通过这种方式拿到的都是单向的，channel本身是双向的。
	-  FileInputStream的getChannel(); 
	-  FileOutputStream的getChannel();

#### 使用Channel和Buffer实现文件的复制

我们首先引入两个概念: 直接缓冲区和非直接缓冲区。

```java
 //直接缓冲区 address 指向这块地址
 ByteBuffer directByteBuffer = ByteBuffer.allocateDirect(1024);
 //非直接缓冲区
 ByteBuffer byteBuffer = ByteBuffer.allocate(1024); 
```
这两个有什么区别呢?我们知道内存从逻辑上可以视为一个字节数组。程序在成为进程的时候，操作系统会分配给进程对应的资源，比如说从字节数组划出一部分给进程，但并不是真实的内存，是虚拟内存。非直接缓冲区在还是在操作系统分配给JVM的内存中。还在JVM的管辖范围之内，而直接缓冲区则是在JVM之外，向操作系统申请内存，所以这个方法也是native方法。

```java
   private static void studyChannel() throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        FileInputStream fileInputStream = new FileInputStream("D:基础笔记PDF.zip");
        FileOutputStream fileOutStream = new FileOutputStream("D:基础笔记PDF3.zip");
        FileChannel inChannel = fileInputStream.getChannel();
        FileChannel outChannel = fileOutStream.getChannel();
        long start = System.currentTimeMillis();
       while (inChannel.read(byteBuffer) != -1){
            byteBuffer.flip();
            outChannel.write(byteBuffer);
            byteBuffer.compact();
       }
        fileInputStream.close();
        fileOutStream.close();
        inChannel.close();
        outChannel.close();
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }
```
事实上这跟用流没多大的差距, 是的你用直接缓冲区和非直接都是一样的。都没差多少，那说好的高效呢。其实NIO的确很高效。这么用把他用低效了。
有关这种拷贝方式为什么这么慢, 我在[《操作系统与通用计算机组成原理简论》](https://segmentfault.com/a/1190000022858315%29)
已经大致介绍过了，这里我们在复习一下，程序无法直接接触硬件，让程序直接接触硬件是系统不稳定的原因，程序如果想访问某个文件，那么只能调用对应的操作系统的函数。

访问硬件属于特权指令，在执行指令的时候，CPU出于内核态，CPU使用一种称为内存映射I/O(memory-mapped I/O)的技术向I/O设备发射命令，假设磁盘控制器被映射到某个端口，随后，CPU可能通过执行三个对地址0xa0的存储指令，发起磁盘读: 第一条指令是发送一个命令字，告诉磁盘发起一个读，同时还发送了其他的参数，例如当读完成时，是否中断CPU(如果你不懂什么是中断的话，没关系，等着我)。第二条指令指明应该读的逻辑块号。第三条指令指明应该存储磁盘山区内容的主存地址。


当CPU发出了请求之后，在磁盘执行读的时候，CPU出于等待状态，磁盘是很慢的，此时让CPU一直陷入等待是一种极大的浪费。操作系统的CPU调度器通常会让CPU去做其他事情。这是加载进内存的操作，从内存写磁盘，又是一阵等待，因为我们的磁盘相对于CPU来说实在是太慢了。这个时候就是DMA出场的时候，在磁盘控制器收到来自CPU的读命令之后，它将逻辑块号翻译成一个扇区地址，读该扇区的内容，随后将这些内容直接传送到主存，不需要CPU的干涉。设备可以自己执行读或者写总线事务而不需要CPU的过程，我们称为直接内存访问(Direct Memory Access,DMA)。这种数据传统称为DMA传送(DMA transfer)。


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1dc6c25b3e36~tplv-t2oaga2asx-image.image)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/14/172b1dca12160848~tplv-t2oaga2asx-image.image)
这个过程中，CPU发送指令的时候，CPU出于内核态，在执行用户程序的时候出于用户态，CPU态频繁的切换。每次读写都需要CPU的参与，但是你知道CPU太快了，出于等待的CPU会被分配到其他线程，在切换回来，上下文切换。这就是这种IO慢的原因。我们能否借助DMA技术呢，让CPU只参与一次，磁盘控制器受到CPU的读命令之后，磁盘控制器直接将逻辑块号翻译成一个扇区地址，读该扇区的内容，随后直接将内容传送至主存。在CPU发送一次写指令之后，磁盘控制器将主存的内容写入磁盘中。可以的。这也就是下面介绍的零拷贝。
```java
  long start = System.currentTimeMillis();
        FileChannel inChannel = FileChannel.open(Paths.get("D:基础笔记PDF.zip"));
	    // 输出的时候要指明一下,channel的状态。    
        FileChannel outChannel = FileChannel.open(Paths.get("D:基础笔记PDF3.zip"), StandardOpenOption.WRITE,StandardOpenOption.CREATE_NEW,StandardOpenOption.READ);
        MappedByteBuffer inMappedByteBuffer = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());
        MappedByteBuffer outMappedyteBuffer = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, inChannel.size());
        byte[] byteArray = new byte[inMappedByteBuffer.limit()];
        inMappedByteBuffer.get(byteArray);
        outMappedyteBuffer.put(byteArray);
        inChannel.close();
        outChannel.close();
        long end = System.currentTimeMillis();
        System.out.println(end - start);
```
基础笔记PDF.zip大概是500多M,用流的话大概在6到7秒。用这种拷贝就是不到一秒。确实快。
第二种写法:
```java
   long start = System.currentTimeMillis();
        FileChannel inChannel = FileChannel.open(Paths.get("D:基础笔记PDF.zip"));
        FileChannel outChannel = FileChannel.open(Paths.get("D:基础笔记PDF3.zip"), StandardOpenOption.WRITE,StandardOpenOption.CREATE_NEW,StandardOpenOption.READ);
        inChannel.transferTo(0,inChannel.size(),outChannel);
        // outChannel.transferFrom(inChannel,0,inChannel.size());
        long end = System.currentTimeMillis();
        System.out.println(end - start);
```
transferTo是将文件输出到哪个位置。
transferFrom是从哪个位置读，然后输出到当前通道代表的位置。
MappedByteBuffer 是内存映射。操纵MappedByteBuffer对象会自动同步.
例子:
```java
RandomAccessFile randomAccessFile = new RandomAccessFile("D:\\学习资料\\测试.txt","rw");
        FileChannel channel = randomAccessFile.getChannel();
        MappedByteBuffer map = channel.map(FileChannel.MapMode.READ_WRITE, 0, randomAccessFile.length());
        map.put(1,(byte)'q');
        channel.close();
```

## 参考资料

