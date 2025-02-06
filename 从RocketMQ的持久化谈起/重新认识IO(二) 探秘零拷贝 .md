# 重新认识IO(二) 探秘零拷贝之tranferto

[TOC]

> 在上篇中，我们介绍了Java IO的基本原理和文件读写机制。本篇将在此基础上，探讨一种特定场景下的高效数据复制方式——零拷贝技术。需要注意的是，这种技术并非在所有场景下都是最优选择。
>
> 本篇的副线是，为什么kafka选择了sendfile，而RocketMQ选择了mmap。我们尝试从两者的设计架构去回答。

## 前言

回忆一下我们上面讲的《深入Java IO：文件读写原理（一）》里面讲的内容，我们主要介绍了字节流FileOutputStream和FileOutputStream, 缓冲流BufferedOutputStream和BufferedInputStream。最基础的还是字节流FileOutputStream和FileOutputStream，缓冲流还是在字节流之上做了一层封装:

![img](https://a.a2k6.com/gerald/i/2025/01/16/m7g6.png)

而字节流读写数据本质上都是借助操作系统的能力，也就是调用操作系统的api:

```java
// FileOutputStream
public void write(byte b[]) throws IOException {
        writeBytes(b, 0, b.length, append);
}
private native void writeBytes(byte b[], int off, int len, boolean append) throws IOException;
// FileInputStream
public int read() throws IOException {
     return read0();
}
private native int read0() throws IOException;
```

在Linux上调用的是：

```java
//read()函数尝试从文件描述符fd中读取最多count个字节的数据，并将其存储到以buf为起始地址的缓冲区中。
size_t read(int fd, void buf[.count], size_t count);
// write() 函数尝试将最多count个字节从buf指向的缓冲区写入到文件描述符fd所引用的文件中
// 成功调用并不承诺数据到达磁盘
size_t write(int fd, const void buf[.count], size_t count);
```

这其实是一个系统调用，所谓系统调用是指应用程序调用操作系统的函数，这些操作系统提供的函数是进程在用户态没有特权和能力完成的操作，当应用进程发起系统调用的时候，会进入到内核态，由操作系统来操作系统资源，完成之后再返回到进程。我们在《深入Java IO：文件读写原理（一）》提到write调用其实是将数据刷新到了page cache中去了，并把对应页面标记为脏页（dirty page）, 然后添加到链表(dirty list), 内核会定期把这个链表里面的内容刷到磁盘上，保证磁盘和缓存的一致性。

![img](https://a.a2k6.com/gerald/i/2025/01/16/3a6.png)

这么设计的好处是提升了写数据的性能，但是劣势是磁盘和缓存的一致性降低了。放松一致性的要求来提升性能，这种例子并不少见。为了提升读数据的性能，Linux在page cache里面引入了两个链表：

- active list：活跃链表
- inactive list：不活跃链表



![img](https://a.a2k6.com/gerald/i/2024/12/21/5kvf2.jpg)



当内核需要分配内存或者加载磁盘上的文件时，会触发缺页中断(page fault)。新加入的页面会被放入Inactive链表。在内存页面里面有两个标志位：PG_referenced、PG_active。PG_active为1则当前页面在活跃链表里面，PG_active = 0 , 则该页面在非活跃链表里面。如果inactive List上的页面被访问两次，则该页面会被晋升到活跃链表中。

## read/write调用的流程

现在让我们分析一下程序中常见的文件复制流程：本机文件传输、网络文件传输。

### 本机的文件复制

```java
try(FileInputStream fileInputStream  = new FileInputStream("");
  FileOutputStream fileOutputStream = new FileOutputStream("")
) {
    byte[] readData = new byte[4096];
    int bytesRead = -1;
    while( (bytesRead = fileInputStream.read(readData)) != -1){
        fileOutputStream.write(readData, 0, bytesRead);
    }
}
```



![img](https://a.a2k6.com/gerald/i/2025/01/17/ypv6.png)



我们解释一下DMA复制这个词，DMA全称是Direct Memory Access，意为直接存储器读取，是一种用来提供在外设和存储器之间或者存储器和存储器之间的高速数据传输。整个过程无需CPU参与，数据直接通过DMA控制器进行快速地进行移动拷贝，节省CPU的资源去做其他工作。与之相对的是CPU复制，限于篇幅，本篇我们不介绍过多，我们只需要知道借助DMA机制，计算机的I/O过程就能更加高效。

在上面的数据复制过程中，我们首先发起read系统调用然后进入内核态，然后数据从磁盘到达内核缓冲区，所谓内核缓冲区在Linux的语境下也就是page cache,  然后数据由内核缓冲区被复制到用户进程的内核缓冲区。也就是我们填入的字节数组中。由磁盘到内核缓冲区是DMA复制，由内核缓冲区到用户进程缓冲区是CPU复制。 接着数据由用户进程传送到内核缓冲区，如果你显示效用了刷新操作或者流被自动关闭了，数据将会由内核缓冲区通过DMA复制到磁盘的另外一个外置上。

### 网络文件复制

接下来我们来看在一般的web应用服务器中一般都有数据复制场景，我们一般是从磁盘上读取数据然后通过网络将数据传送到另一台计算机上。比如我们常见的Nginx我们就用来部署静态资源。在Java中我们通过网络传输数据其实可以这么写:

```java
try (ServerSocket serverSocket = new ServerSocket(8080);) {
    // 因为不知道连接会在什么时候简历所以是不断
    while (true) {
        // 这行代码会阻塞直到有新连接进来
        Socket socket = serverSocket.accept(); // 语句二
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write("hello world".getBytes()); // 语句1
    }
} catch (Exception e) {
    // 只是演示代码
}
```

上面代码的缺点是这段代码只有一个线程在监听连接写数据, 如果

![img](https://a.a2k6.com/gerald/i/2025/01/21/5lgf4.png)

所以一般的做法是我们会在主线程接收到一个连接，将处理这个连接的任务丢到线程池里面去处理，来获得更高的吞吐量。在上面的示例中数据直接是在内存中这会让传送数据相对于从磁盘上加载数据上快一些, 现在让我们从磁盘上读数据通过网络将数据传送出去:

```java
try (ServerSocket serverSocket = new ServerSocket(8080);) {
    // 因为不知道连接会在什么时候简历所以是不断
    while (true) {
        // 这行代码会阻塞直到有新连接进来
        Socket socket = serverSocket.accept();
        CompletableFuture.runAsync(() -> {
            byte[] bytes = new byte[4096];
            try (FileInputStream fis = new FileInputStream("")) {
                int readByteTotal = -1;
                OutputStream outputStream = socket.getOutputStream();
                while ( (readByteTotal  =   fis.read(bytes) ) != -1 ){
                    outputStream.write(bytes,0, readByteTotal);    
                    
                }
            }catch (Exception e){
            }
        });
    }
} catch (Exception e) {
    // 只是演示代码
}
```

但在上面的复制中似乎同样也面临了文件复制一样的问题, 进程充当了数据复制的无效媒介:

![img](https://a.a2k6.com/gerald/i/2025/01/21/15ino1.png)

1. 进程发起read调用由用户态转入内核态  
2. 数据从磁盘通过DMA到达内核缓冲区
3. 然后通过CPU复制的方式到达用户进程，内核态转为用户态。
4. 进程发起write调用，由用户态转入内核态
5. 数据从用户进程通过CPU复制到达Socket缓冲区，用内核态转入用户态
6. 然后Socket缓冲区通过DMA复制到网卡。   

### 小小总结一下

在上面我们提供的代码中，无论是本机文件复制还是网络文件复制，我们可以看到数据流到用户进程都是多余的，能否省略这个步骤呢，减少内核实现从内核缓冲区直接到Socket缓冲区呢?  当然可以，由此就引出了零拷贝，所谓零拷贝并不是一次复制都没有，而是相对于上面的数据传输方式，减少了上下文切换和数据复制的次数。

## 更高效的数据复制方式(一)  sendFile

### FileChannel的transferTo

 我们先不介绍理论，先看代码该怎么写, 然后介绍底层原理:

```java
public static void main(String[] args) throws Exception{
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.bind(new InetSocketAddress(8080));
    while (true){
        SocketChannel clientChannel = serverSocketChannel.accept();
        CompletableFuture.runAsync(()-> handleRequest(clientChannel));
    }
}
private static void handleRequest(SocketChannel clientChannel) {
    try(FileInputStream fileInputStream = new FileInputStream("")){
        FileChannel fileChannel = fileInputStream.getChannel();
        fileChannel.transferTo(0,fileChannel.size(),clientChannel);
    } catch (FileNotFoundException e) {
        throw new RuntimeException(e);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

JVM是跨平台的，执行到transferTo的时候，在Linux平台(版本大于2.1)调用的是sendFile：

```
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

在windows平台下类似的api是TransmitFile.

所以上面的代码中文件的复制流程在Linux上的文件复制就变成了下面这样:

![img](https://a.a2k6.com/gerald/i/2025/02/04/t95q.jpg)

1. 用户进程调用sendFile(), 从用户态陷入内核态。
2. DMA控制器将数据从磁盘复制到内核缓冲区
3. CPU 将内核缓冲区中的数据拷贝到套接字缓冲区
4. DMA控制器将数据从Socket缓冲区复制到网卡

### sendfile + DMA scatter 

在Linux上基于sendFile调用，只需要一次上下文切换，两次必要的DMA复制，一次CPU复制。相对于我们上面介绍的文件传输方式，减少了一次上下文切换，一次CPU复制。这看起来速度更快了一些，那能不能把这一次CPU复制也给省略掉呢？ 当然可以，但是也需要硬件的支持。Linux在2.4版本里面引入了DMA的 scatter/gather -- 分散/收集功能，并修改了sendfile的代码来适配。

所谓DMA scatter 就是使得DMA拷贝不需要把数据都存储在一片连续的内存空间上，而是允许离散存储，gather则能够让DMA控制器根据少量的元信息：一个包含了内存地址和数据大小的缓冲描述符，收集存储在各处的数据，最终还原成一个完整的网络包，直接拷贝到网卡而非socket缓冲区，避免了最后一次的CPU拷贝。

![img](https://a.a2k6.com/gerald/i/2025/02/04/1622uz.jpg)

基于sendfile + DMA gather的数据传输过程如下:

1. 用户发起sendfile调用进入，由用户态进入内核态
2. DMA将数据从磁盘复制到内核缓冲区(更为细节的描述是使用scatter功能把数据从硬盘拷贝到内核缓冲区进行离散存储)
3. DMA将数据从内核缓冲区复制到网卡(更为细节的描述是CPU把包含内存地址和数据长度的描述符拷贝掉Socket缓冲区，DMA控制器能够根据这些信息生成网络包数据分组的报头和报尾，DMA控制器根据缓冲区描述符里面的内存地址和数据大小，使用scatter-gather功能开始从内核缓冲区收集离散的数据并组包，最后直接把网络包数据复制到网卡完成数据传输)
4. sendfile返回，上下文从内核态转入用户态

基于这种方案，我们就可以把这仅剩的唯一一次CPU复制也给去除了(严格意义上来说还是会有一次，但是这次CPU拷贝的只是那些微乎其微的元信息，开销几乎可以忽略不计)。理论上，数据传输过程就再也没有CPU的参与了，也因此CPU的高速缓存再也不会被污染了，CPU可以去执行其他的业务，同时DMA的I/O任务并行，此举极大地提升了系统性能。

### sendfile与Kafka

这里我们简单的介绍了sendfile之后，我们选择学以致用，观察两个软件的设计取舍，在实践中体会sendfile的适用点在哪里。 既然sendfile这么好，那么是不是一般的应用服务器软件都应该使用这种技术来向客户端传送数据呢？  答案当然是否定的，从Kafka的设计思路，可以一窥Kafka采取sendfile的原因(见参考文档[22])，Kafka指出一般的看法是磁盘很慢，这可能是一个误区，磁盘可能既比人们预期的慢，又比人们预期的快，这取决于使用方式。

在一个配备六个7200转SATA RAID-5阵列的JBOD配置中，线性写入的性能约为600MB/秒，但随机写入的性能仅约为100KB/秒——相差超过6000倍。所谓线性就是在文件的末尾追加，所谓随机写入就是指的是在文件的任意位置写入数据。在参考文档[23] 中我们可以看到，在某些情况下顺序磁盘访问甚至可以比随机内存访问更快。如果我们存储的数据具备顺序访问的特性，也就是写入顺序即为访问顺序，那么就能有效到利用到操作系统的缓存，	因为Linux会对顺序读进行预读，所谓预读就是预先将数据加载进入内存。而消息队列刚好是顺序访问因此就能有效的使用到文件缓存。除此之外操作系统提供的延迟写入也就是将小的逻辑写入合并为大的物理写入。 现代操作系统通常会更积极的使用内存，空闲内存会被自动用于磁盘缓存，回收时性能损失极小。如果在进程内部维护一份缓存，数据仍然会在操作系统缓存中重复存储

除此之外，kafka构建在JVM之上，"Java的对象并不是一个好东西"(见参考文档[24]), 原因就在Java的对象头上，有时候Java的对象头比真实的对象数据还要打，在JEP 450: Compact Object Headers (Experimental) 我们可以看到，JDK的开发人员已经开始着手对Java的对象头进行压缩，将Java的对象头大小从96位到128位压缩到64位，未来会更小压缩至32位。除此之外随着堆内数据的增多，Java的垃圾回收会变得越来越难以处理且越来越慢(随着高版本GC的加强，也许这一点会有所改观) 。所以基于顺序读和JVM的特性，我们使用文件系统并依赖页面缓存比维护内存缓存或其他结构性能更好。

这也就是kafka选择sendfile的原因，消息队列天然就是顺序读的。对比关系型数据库，我们则无法做这样的保证，一般的关系型数据随机读更多一些，所以关系型数据库更习惯自己做缓存，性能更加可控。所以关系型数据库更青睐B树，某种意义上来说B树的确更为通用，比如对随机读写支持的比较好，范围查询等。操作的复杂度是O(log N)。

但对于kafka来说，并不需要随机读写、范围查询，对于消息队列来说天生就是顺序读写，顺序读写更好的能够利用操作系统的延迟写特性。于是我们考虑追加，这种结构的优势在于对文件的简单读取和追加操作都是O(1)的。所以kafka选择了LSM树。关于LSM，我们这里不做过多的介绍。

### FileChannel的transferFrom实现

现在我们解决了将数据传送出去的问题，也就是读，现在让我们考虑写数据, 也就是追加文件，我们其实可以这么写:

```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.bind(new InetSocketAddress(8080));
    while (true){
        SocketChannel clientChannel = serverSocketChannel.accept();
        CompletableFuture.runAsync(()-> handleRequest(clientChannel));
    }
}

private static void handleRequest(SocketChannel clientChannel) {
    try(FileOutputStream fileInputStream = new FileOutputStream("",true)){
        FileChannel fileChannel = fileInputStream.getChannel();
        fileChannel.transferFrom(clientChannel,0,fileChannel.size());
    } catch (FileNotFoundException e) {
        throw new RuntimeException(e);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

理解了transferTo的实现原理后，一个自然的问题是：作为其对应的操作，transferFrom是否也使用了类似的零拷贝机制？让我们通过源码分析来回答这个问题：

```java
// 代码来自JDK 17
public long transferFrom(ReadableByteChannel src,
                         long position, long count)
    throws IOException
{
    ensureOpen();
    if (!src.isOpen())
        throw new ClosedChannelException();
    if (!writable)
        throw new NonWritableChannelException();
    if ((position < 0) || (count < 0))
        throw new IllegalArgumentException();
    if (position > size())
        return 0;
	// SocketChannel 不是FileChannelImpl的实例
    if (src instanceof FileChannelImpl fci) {
        long n = transferFromFileChannel(fci, position, count);
        if (n >= 0)
            return n;
    }
    return transferFromArbitraryChannel(src, position, count); // 语句一
}
```



```java
private long transferFromArbitraryChannel(ReadableByteChannel src,
                                          long position, long count)
    throws IOException
{
    // Untrusted target: Use a newly-erased buffer
    int c = (int)Math.min(count, TRANSFER_SIZE);
    ByteBuffer bb = ByteBuffer.allocate(c);
    long tw = 0;                    // Total bytes written
    long pos = position;
    try {
        while (tw < count) {
            bb.limit((int)Math.min((count - tw), (long)TRANSFER_SIZE));
            // ## Bug: Will block reading src if this channel
            // ##  is asynchronously closed
            int nr = src.read(bb);
            if (nr <= 0)
                break;
            bb.flip();
            int nw = write(bb, pos);
            tw += nw;
            if (nw != nr)
                break;
            pos += nw;
            bb.clear();
        }
        return tw;
    } catch (IOException x) {
        if (tw > 0)
            return tw;
        throw x;
    }
}
```

这里看起来是从SocketChannel中读取数据，然后写入到FileChannel对应的文件里面。数据还是流进了用户进程，但是仔细一想网络中的数据是不可控的，不确定下一个数据什么时候到，所以这里实现对应的sendfile还是比较困难。但是我们将数据传送出去，由于文件在本地，sendfile就刚刚好。

那如果我想随机读呢？随机读的话我们观察FileChannel的transferTo方法:

```java
public abstract long transferTo(long position, long count, WritableByteChannel target)
```

第一个参数position，我们就可以指定传送的文件位置，也可以实现随机读。那我们如果要合并读取的内容呢，也就是将数据读取出来再进行合并，我们就要退回普通的read调用嘛？由此就引入了mmap + write这对组合。

##  由此引出 mmap

我们先上代码:

```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.bind(new InetSocketAddress(8080));
    while (true) {
        SocketChannel clientChannel = serverSocketChannel.accept();
        CompletableFuture.runAsync(() -> handleRequest(clientChannel)).exceptionally(e->{
            e.printStackTrace();
            return null;
        });
    }
}
private static void handleRequest(SocketChannel socketChannel) {
    try (FileInputStream fileInputStream = new FileInputStream("1.txt");) {
        MappedByteBuffer mappedByteBuffer = fileInputStream.getChannel().
                map(FileChannel.MapMode.READ_ONLY, 0, fileInputStream.getChannel().size());
        // 这是直接传送数据                 
        // 刷盘
        mappedByteBuffer.force();
        // 清理
        Cleaner cleaner = ((sun.nio.ch.DirectBuffer) mappedByteBuffer).cleaner();
        if (cleaner != null) {
             cleaner.clean();
         }
        socketChannel.write(mappedByteBuffer);
        socketChannel.close();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

然后数据的传送流程如下图所示:

![](https://a.a2k6.com/gerald/i/2025/02/05/o56q.png)

利用mmap替换read(), 配合write() 调用的整个流程如下:

1. 用户进程调用mmap() , 从用户态陷入到内核态，将内核缓冲区映射到用户缓冲区
2. DMA 控制器将数据从硬盘复制到内核缓冲区
3. mmap返回，上下文从内核态切换回用户态
4. 用户进程调用write(), 长时把文件数据写到内核里面的套接字缓冲区，再次陷入内核态;
5. CPU将内核缓冲区中的数据复制到Socket缓冲区
6. DMA控制器将数据从Socket缓冲区拷贝到网卡完成数据传输
7. write返回，上下文从内核态切换回用户态。

相比传统的Linux I/O读写，数据不需要经过用户进程进行转发了，而是直接在内核里完成了拷贝。所以使用mmap()之后的复制次数是2次DMA拷贝，1次CPU复制，加起来一共3次拷贝操作，比起传统的I/O方式节省了一次CPU拷贝以及一倍的内存，不过因为mmap也是一个系统调用，因此用户态和内核态的切换还是4次。

我看到这个流程的时候，第一个疑问就是这个用户缓冲区在哪个进程的哪个位置？ 一般来说我们知道JVM启动之后，我们将内存划分为:

![](https://a.a2k6.com/gerald/i/2025/02/05/4jsi.png)



1. 虚拟机栈(JVM Stack): 每次方法调用都会触发栈上创建一个新帧，用于存储方法的局部变量和返回地址。
2. 本地方法栈: 同虚拟栈类似，由于本地方法不会被编译为字节码，所以需要单独一块区域进行存储。
3. 堆: 运行时存储对象，当我们new 一个实例或者创建一个数组，JVM都会从堆里面分配内存。上面的图片里面堆里面分两块: Young Generation 、Old Generation。 这其实是CMS和 Parallel Old的划分，到G1里面分代被弱化，引入了分区。
4. 堆外: 除去元空间占据的堆外部分
5. 元空间(堆外)
6. 程序计数器

那么 mmap出来的这块虚拟内存在哪个位置呢？ 上面是JVM视角的内存划分，但是从Linux上来说，并不是为JVM而生，从Linux视角，一般的进程内存划分如下图所示:

![img](https://a.a2k6.com/gerald/i/2024/09/08/6gys5.jpg)

上面讲的用户进程缓冲区其实就在文件映射与匿名映射区这个位置。那看到这里可能有同学会问了，你上面说的随机访问数据，然后合并传送在哪里体现？要实现随机访问数据然后合并传送，我们先讲基本思路， 首先定义一个对象里面是偏移量和读取的数据长度。然后通过文件的位置获取总的 MappedByteBuffer。然后将这个List转成数组，传递给SocketChannel，SocketChannel就会执行合并。代码如下图所示:

```java
 static class FileSegment {
       // 定义偏移量
        final long offset;
     	// 长度
        final long length;
        public FileSegment(long offset, long length) {
            this.offset = offset;
            this.length = length;
        }
    }
    
    public static void main(String[] args) throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 绑定端口
        serverSocketChannel.bind(new InetSocketAddress(8080));
        while (true) {
            // accept 会阻塞到直到有连接建立
            SocketChannel clientChannel = serverSocketChannel.accept();
            // 然后交给一个线程去处理
            CompletableFuture.runAsync(() -> handleRequest02(clientChannel)).exceptionally(e -> {
                e.printStackTrace();
                try {
                    clientChannel.close();
                } catch (IOException ex) {
                    throw new RuntimeException(ex);
                }
                return null;
            });
        }
    }
    private static void handleRequest02(SocketChannel socketChannel) {
        // 模拟随机读取
        Path commitLogPath = Path.of("/data/commitlog/commit-1.log");
        FileSegment fileSegment01 = new FileSegment(0, 1024);
        FileSegment fileSegment02 = new FileSegment(1056, 1065);
        FileSegment fileSegment03 = new FileSegment(2089, 3000);
        List<FileSegment> segmentList = new ArrayList<>();
        segmentList.add(fileSegment01);
        segmentList.add(fileSegment02);
        segmentList.add(fileSegment03);
        // 获得聚合之后的mmapfile
        List<MappedByteBuffer> mappedBuffers = mapFileSegements(commitLogPath, segmentList);
        // 获得需要写入的字节
        long totalBytes = segmentList.stream().mapToLong(e -> e.length).sum();
        // 转成ByteBuffer
        ByteBuffer[] byteBuffers = mappedBuffers.toArray(new ByteBuffer[0]);
        try {
            // 开始循环写入
            long totalWritten = 0;
            while (totalWritten < totalBytes){
               long written  = socketChannel.write(byteBuffers);
               if (written < 0){
                   // 写入出现了异常
                    break;
               }
                totalWritten += totalWritten + written;
            }
            // 注意清理MappedByteBuffer
            // 释放资源
            socketChannel.close();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
private static List<MappedByteBuffer> mapFileSegements(Path commitLogPath, List<FileSegment> segmentList) {
     try (FileChannel fileChannel = FileChannel.open(commitLogPath, StandardOpenOption.READ)) {
            return segmentList.stream().map(seg -> {
                try {
                    return fileChannel.
                            map(FileChannel.MapMode.READ_ONLY,seg.offset,seg.length);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }).collect(Collectors.toList());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
}
```



### mmap与RocketMQ

现在我们也许就能够回答为什么RocketMQ选择了mmap而不是sendfile了，其实在上面的文章中我们已经有了暗示，我们希望随机读取数据，那为什么RocketMQ需要随机读取数据呢？ 让我们从RocketMQ的架构入手: 

![](https://a.a2k6.com/gerald/i/2025/02/06/ofoj.png)

RocketMQ采用的是混合型的存储结构，Broker单个实例下所有的队列共用一个数据文件(commitlog)来存储。consumequeue消费文件，引入的目的主要是提高消息消费的性能，indexfile是索引文件，提供了一种可以通过key或时间区间来查询消息的功能。生产者发送消息到Broker端，然后Broker使用同步或者异步的方式对消息刷盘持久化，在Java里面也就是调用MappedByteBuffer的force方法，只要消息被刷盘持久化至磁盘文件commitlog中，那么生产者发送的消息就不会丢失。Broker端的后台服务线程会不停地分发请求并异步构建消费文件和索引文件。

答案就在RocketMQ的混合型存储结构中，也就是说在一个文件上会有多个topic，除此之外一个topic下面还会有不同的tag，服务端需要查找过滤合并消息。所以RocketMQ选择了mmap。而kakfa的存储结构如下图所示：

![](https://a.a2k6.com/gerald/i/2025/02/06/3zy8.png)

在Kakfa中Topic代表一类信息，Partition是topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition由多个segment组成。所以不需要处理直接通过sendfile返回给消费者就行。

### 彩蛋

用户进程发起mmap请求建立起虚拟内存和物理内存之间的映射时，操作系统非常狡猾，并没有立刻将磁盘上的文件加载进入操作系统中的缓存中，这也就是延迟加载。 当你真正开始读写，操作系统就会产生缺页中断(page fault)，来分配物理内存 , 同时将磁盘上的数据加载进入物理内存。那么我们能否避免这一点呢，当然也是可以的，能否提前就让操作系统为我们分配好内存，避免缺页中断造成的性能损伤呢？  在Linux  5.14 引入了预读接口:

```C
int madvise(void addr[.size], size_t size, int advice);
```

如果你还是担心，在内存紧张的情况下页面被换出，也可以使用m_lock来锁定页面:

```C
int mlock(const void addr[.size], size_t size);
```

RocketMQ中就用到了mlock。

## 总结一下

本篇尝试从两个问题入手，分析普通的read/write传输数据问题的流程在哪里，但这并不代表在所有场景下都存在问题，我们指出如果随机访问磁盘上的数据比较多，那么自己做缓存可能更为明智。如果你的程序顺序访问比较多，就可以充分的利用操作系统提供的零拷贝系统调用，这也就是sendfile和mmap，sendfile在第一个版本里面相对于普通的read/write调用，只需要只需要一次上下文切换，两次必要的DMA复制，一次CPU复制就能完成数据复制。而sendfile后面也引入了加强也就是DMA scatter ，这次加强之后CPU复制这层开销也被省去，获得了更好的性能。 而mmap则只减少了一次CPU复制，上下文切换还是没变，但是如果能够有效利用内存，那么mmap的速度也会非常快，因为都在内存里面。我们可以根据对应的需要来选择对应的技术，比如传送数据的时候sendfile，追加数据的时候选择mmap。

| 特性     | Kafka               | RocketMQ      |
| -------- | ------------------- | ------------- |
| 存储方式 | Topic独立存储       | 混合存储      |
| 文件组织 | 分区(Partition)方式 | 统一commitlog |
| 读取特点 | 顺序读取为主        | 随机读取为主  |
| 技术选择 | sendfile            | mmap          |

## 参考资料

[1] What they don’t tell you about demand paging in school  https://offlinemark.com/demand-paging/

[2]   malloc vs mmap in C https://stackoverflow.com/questions/1739296/malloc-vs-mmap-in-c

[3] What are void pointers for in C++? https://stackoverflow.com/questions/2860626/what-are-void-pointers-for-in-c [4] Is there any difference between mmap vs mmap64? https://stackoverflow.com/questions/59453555/is-there-any-difference-between-mmap-vs-mmap64

[5] CreateFileMappingW https://learn.microsoft.com/zh-cn/windows/win32/api/memoryapi/nf-memoryapi-createfilemappingw?redirectedfrom=MSDN

[6]  Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[7] Is there any difference between mmap vs mmap64?  https://stackoverflow.com/questions/59453555/is-there-any-difference-between-mmap-vs-mmap64 

[8] Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[9] How to use mmap to allocate a memory in heap?  https://stackoverflow.com/questions/4779188/how-to-use-mmap-to-allocate-a-memory-in-heap

[10]  mmap 内存映射，是越过了操作系统，直接通过内存访问文件吗？  https://www.zhihu.com/question/522132580/answer/3241695059

[11] 一步一图带你深入理解 Linux 虚拟内存管理  https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486732&idx=1&sn=435d5e834e9751036c96384f6965b328&chksm=ce77cb4bf900425d33d2adfa632a4684cf7a63beece166c1ffedc4fdacb807c9413e8c73f298&scene=178&cur_album_id=2559805446807928833#rd

[12] 大量类加载器创建导致诡异FullGC  https://heapdump.cn/article/1924890

[13]  JDK-8268893  https://bugs.openjdk.org/browse/JDK-8268893

[14] Temurin™ Supported Platforms  https://adoptium.net/zh-CN/supported-platforms/

[15] glibc  https://sourceware.org/glibc/wiki/MallocInternals

[16] Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[17] Chapter 16: The Page Cache and Page Writeback https://github.com/firmianay/Life-long-Learner/blob/master/linux-kernel-development/chapter-16.md

[18] Why does not Redis use linux zero-copy syscall api ? https://github.com/redis/redis/issues/12682

[19]   一步一图带你深入理解 Linux 物理内存管理    https://www.cnblogs.com/binlovetech/p/16914715.html

[20] 文件IO原理及Kafka高效读写原因分析 [https://www.eula.club/blogs/%E6%96%87%E4%BB%B6IO%E5%8E%9F%E7%90%86%E5%8F%8AKafka%E9%AB%98%E6%95%88%E8%AF%BB%E5%86%99%E5%8E%9F%E5%9B%A0%E5%88%86%E6%9E%90.html#_1-%E5%89%8D%E8%A8%80](https://www.eula.club/blogs/文件IO原理及Kafka高效读写原因分析.html#_1-前言)

[21] Linux I/O 原理和 Zero-copy 技术全面揭秘  https://zhuanlan.zhihu.com/p/308054212 

[22]   Kafka Design   https://docs.confluent.io/platform/6.2/kafka/design.html#don-t-fear-the-filesystem 

[23] https://queue.acm.org/detail.cfm?id=1563874

[24] 深度解读｜Spark 中 CodeGen 与向量化技术的研究 https://cn.kyligence.io/blog/spark-codegen-vectorization-technology/

[25] B-Tree vs LSM-Tree  

对于工程点来说，在指定的架构下面选取指定的技术，而不是因为指定的技术来选取指定的架构

[26] 终于弄明白了 RocketMQ 的存储模型  https://www.51cto.com/article/743692.html

[27]  一文聊透 Linux 缺页异常的处理 —— 图解 Page Faults   https://www.cnblogs.com/binlovetech/p/17918733.html 
