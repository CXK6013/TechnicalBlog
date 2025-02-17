# 从BIO到io_uring: IO模型的演进之路

[TOC]



## 先从用BIO构建一个小的服务器软件骨架

最近想重新梳理一下自己对BIO、NIO 、AIO、IO_Uring的理解，让我们来先说BIO，所BIO就是 Blocking IO，那究竟Blocking在什么地方呢？ 让我们先开始写代码，首先我们需要开启一个ServerSocketChannel，驻留在对应的端口上，监听连接: 

```java
// 代码基于openJDK17
private static void bio() throws Exception {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.bind(new InetSocketAddress(8080),200); // 语句二
}
```

语句二代表我们的服务端驻留在8080端口上，在这个端口上监听连接，第二个参数在JDK中的解释是:

> The backlog parameter is the maximum number of pending connections on the socket. Its exact semantics are implementation specific. In particular, an implementation may impose a maximum length or may choose to ignore the parameter altogether. If the backlog parameter has the value 0, or a negative value, then an implementation specific default is used.
>
> backlog参数是socket上待处理连接的最大数量。它的具体语义取决于具体实现。特别是，实现可能会强制限制最大长度或选择完全忽略该参数。如果backlog参数值为0或负值，则使用实现特定的默认值。

 让我们先回忆一下TCP协议建立连接的过程:

第一步: 客户端发送syn到server发起握手

第二步: 服务端收到syn之后，回复syn+ack给客户端。

第三步: 客户端收到syn+ack之后, 回复server一个ack表示收到了server的syn + ack(此时客户端的tcp连接状态已经是established, 在客户端看来连接已经成功建立)

对应到Linux上的实现如下图所示:

![](https://a.a2k6.com/gerald/i/2025/02/11/2d7.png)

第二次握手进入办连接队列，三次握手的最后一次ACK到达，连接就从半连接队列移动到全连接队列中。然后上层的程序发起accept调用，就可以拿到连接的抽象，从连接中读写数据。这个accept队列的大小就是由backlog指定。我们接着写程序，注意到上面我们建立连接的过程中出现了bind、listen，但是我们上面写的代码里面只有bind, 这是因为在调用bind的时候，自动调用了listen:

```java
@Override
public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
    synchronized (stateLock) {
        ensureOpen();
        if (localAddress != null)
            throw new AlreadyBoundException();
        if (isUnixSocket()) {
            localAddress = unixBind(local, backlog);
        } else {
            localAddress = netBind(local, backlog);
        }
    }
    return this;
}
```

```java
private SocketAddress unixBind(SocketAddress local, int backlog) throws IOException {
    UnixDomainSockets.checkPermission();
    if (local == null) {
        // Attempt up to 10 times to find an unused name in temp directory.
        // If local address supplied then bind called only once
        boolean bound = false;
        int attempts = 0;
        while (attempts < 10 && !bound) {
            try {
                Path path = UnixDomainSockets.generateTempName().getPath();
                UnixDomainSockets.bind(fd, path); // bind 
                bound = true;
            } catch (BindException e) { }
            attempts++;
        }
        if (!bound)
            throw new BindException("Could not bind to temporary name");
    } else {
        Path path = UnixDomainSockets.checkAddress(local).getPath();
        UnixDomainSockets.bind(fd, path);
    }
    Net.listen(fd, backlog < 1 ? 50 : backlog); // listen
    return UnixDomainSockets.localAddress(fd);
}
```

```java
private SocketAddress netBind(SocketAddress local, int backlog) throws IOException {
    InetSocketAddress isa;
    if (local == null) {
        isa = new InetSocketAddress(Net.anyLocalAddress(family), 0);
    } else {
        isa = Net.checkAddress(local, family);
    }
    @SuppressWarnings("removal")
    SecurityManager sm = System.getSecurityManager();
    if (sm != null)
        sm.checkListen(isa.getPort());
    NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
    Net.bind(family, fd, isa.getAddress(), isa.getPort()); // bind
    Net.listen(fd, backlog < 1 ? 50 : backlog); // listen
    return Net.localAddress(fd);
}
```

###  报文分割与定义

现在让我们考虑如何读数据，一般来说，服务端程序总是根据客户端的请求报文回应数据。由于TCP属于流式协议，所谓流式协议就是应用程序通过TCP协议发送给服务端的数据包，是没有边界的，假设客户端发送了两个hello world，服务端可能一次收到两个helloworld，也可能分多次收到。所以就需要应用层确定消息边界, 我们需要设计应用层的报文格式，那其实一般的应用层报文就有以下几种格式，一部分用于指示需要读取的数据长度，一部分是实际的报文内容。我们称之为自定义长度的报文，

#### 自定义长度

自定义长度: 规定前几个字节是表示长度，我们只需要向后读数据长度就行了。

​	![](https://a.a2k6.com/gerald/i/2025/02/12/3offm.png)

​	   我们规定byte数组的前两位代表报文的长度，下一个问题来了，我们知道Java的数据类型是有符号数，byte占一个字节，一个字节是八位, 表达无符号数，最小是0，最大是255, 但是在 Java中byte是有符号整数，数据范围是-128到127。那如果我想充分的利用这个字节，该如何将负数映射到对应的正数呢？ 这就要涉及Java中整数的二进制表示了，Java中的整数是用补码来表示的，在补码系统里面，最高位是符号位，0为正，1为负。对于负数，补码是其绝对值的二进制取反后加1，那么-127的绝对值是127,127对应的二进制是0111 1111，取反后是1000 0000，加1之后是1000 0001。

![](https://a.a2k6.com/gerald/i/2025/02/13/2zh7.png)

如果我们用超过byte的数字，然后将其强制转换为byte类型，像下面这样:

```java
byte s = (byte) 255;
System.out.println(s);
```

然后输出是-1，这是为什么呢？  255是一个正数对应的补码是1111 1111, -1的绝对值是1，二进制是0000 0001，取反是1111 1110，加1刚好是1111 1111。 我这里想拿到的结论是对于byte这种数据类型来说，超过byte这种数据类型范围的整数，对于如果x∈[128, 255] , 那么将其强制转换到有符号整数，那么x的补码刚好被转换成了对应的有符号数。

所以如果客户端填入的数字里面超过了255，转成byte就变成了-1。那我们如何将-1还原成255呢。观察到-1的补码是1111 1111，所以我们直接做强制转换？ 但是int也带符号的，所以byte一个字节，在byte里面的补码是 1111 1111 ，int占据4个字节，int的-1对应的补码 其实应该是  11111111 11111111   11111111 11111111。前面的符号位需要我们抹去，也就是我们需要三个字节也就是24位个0做与运算，后面的刚好八位我们保持不变，我们只需要再选取八个一即可。

 如果我们将255转换为byte类型，会被转换为-1,254转为-2。  -1的先取绝对值是1, 在二进制里面的表示是0000 0001，取反之后是1111 1110, 然后加1等于1111 1111， 这刚好是255的二进制表示，只不过一个byte占八个字节，最高位是1表示负数， 如果我们在前面补24个0，刚好是4个字节，这也就是int中的255。那么现在的问题来了，我们该怎么填0，也就是说对于byte中的负数，在更大的范围我们为其填24个0就行了，剩下的8位呢？  我们只需要让其和1111 1111做与运算就可以将其还原出来。由上面推导的命题可知，对于超过byte范围的数，但是还在一个字节的范围，我们只需要让其与上0xff(255的16进制表示)，我们就能将其还原出来。那如果255不够呢，我们就需要再借助一个字节，由此就引出了大端序和小端序。

### 大端序与小端序

现在我们拥有能够让一个字节表达0到255的能力，如果255还不够呢，我们再借助一个字节，现在有一个问题就是数组里面索引小的是高位，还是索引小的是高位。所谓高位举一个例子就是以十进制数123来说，1就处于高位，3就处于低位:

![](https://a.a2k6.com/gerald/i/2025/02/12/isdt.png)

高位在前被称为大端序，低位在前就被称为小端序。一般来说TCP/IP协议用大端序。X86系列CPU都是小端序。现在我们的数组前两位放的是数组长度，索引小的是高位，那么我们如何将这两个数字组合起来成为一个无符号整数呢? 

```java
// 将两个 byte 视为无符号整数
int high = byte1 & 0xFF;  // 消除符号位影响
int low = byte2 & 0xFF;
int value = (high << 8) | low;  // 组合为 16 位无符号整数
```

###  其他报文格式

我们谈论报文格式的时候，事实上就是在定义怎么读报文，除了我们在报文里面填入数据长度字段之外，常见的还有以特殊字符为消息结束符的报文格式，比如\n， 我们就可以将helloworld\n当做一个完整的报文。还有固定长度的报文，比如我们指定长度为10，我们一次就读10个字节，进行解析处理。

### 报文分割

有了对报文的基本了解，我们可以开始从数据里面提取报文了。我们目前为了讨论简单就使用\r\n\r\n，作为消息的结束符。这样其实面临以下几个问题:

1.  一次读了两个报文 1\r\n\r\n2\r\n\r\n。 
2.  1\r\n\r\n，可能读取了多次才读到。
3.  我们用ByteBuffer来读取数据，假设我们请求分配了4096个字节，但是从中读到的数据超过了容量，所以我们需要扩容。

针对第一个问题，每次读取完毕我们都需要从里面找消息的结尾，针对第二个问题，我们需要一个暂存，存储我们读到的还没有碰到结束符的报文。针对第三个问题，我们需要动态扩容。因此我们从中提取数据的大致程序结构应该是这样的，从SocketChannel里面提取数据，碰到消息结束符，提取消息，交给上层程序。

```java
public class SocketIOHandler {

    // 默认Buffer大小
    private static final int DEFAULT_BUFFER_SIZE = 1024;

    private static final byte[] DELIMITER = {'\r', '\n', '\r', '\n'};

    // 暂存读取到的数据
    private ByteBuffer carrierBuffer;

    public void processData(SocketChannel socketChannel) {
        try {
            for (; ; ) {
                ByteBuffer directByteBuffer = ByteBuffer.allocateDirect(DEFAULT_BUFFER_SIZE);
                int readData = socketChannel.read(directByteBuffer);
                mergeByteBuffer(directByteBuffer);
                // 代表连接正常关闭
                int readIndex = -1;
                if (readData < 0) {
                    // 判断是否有完整的报文
                    carrierBuffer.flip();
                    while ((readIndex = isHaveCompleteData(carrierBuffer)) != -1) {
                        // 清空数据
                        byte[] completeData = new byte[readIndex - carrierBuffer.position() + 1];
                        //
                        carrierBuffer.get(completeData);
                        carrierBuffer.position(readIndex);
                    }
                    carrierBuffer.clear();
                }
                // 连接正常
                carrierBuffer.flip();
                while ((readIndex = isHaveCompleteData(carrierBuffer)) != -1) {
                    byte[] completeData = new byte[readIndex - carrierBuffer.position() + 1];
                    carrierBuffer.get(completeData);
                    String readDataString  =  new String(completeData,0,completeData.length - 4);
                    System.out.println(readDataString);
                    carrierBuffer.position(readIndex + 1);
                }
            }
        } catch (IOException e) {
            // 这里只做代码演示
            throw new RuntimeException(e);
        }
    }

    private void doSomeThing(ByteBuffer carrierBuffer) {

    }

    private int isHaveCompleteData(ByteBuffer carrierBuffer) {
        // 转成读模式
        for (int i = carrierBuffer.position() ; i <= carrierBuffer.limit() -  4; i++) {
            if (carrierBuffer.get(i) == DELIMITER[0] &&
                    carrierBuffer.get(i + 1) == DELIMITER[1] &&
                    carrierBuffer.get(i + 2) == DELIMITER[2] &&
                    carrierBuffer.get(i + 3) == DELIMITER[3]
            ) {

                return i + 3;
            }
        }
        return -1;
    }

    private void mergeByteBuffer(ByteBuffer directByteBuffer) {
        if (this.carrierBuffer == null) {
            this.carrierBuffer = ByteBuffer.allocateDirect(DEFAULT_BUFFER_SIZE);
            // flip 之后转为读模式,remaining返回的是可读的元素数量
            directByteBuffer.flip();
            // carrierBuffer.remaining() 返回的是可写的元素数量
            this.carrierBuffer.put(directByteBuffer);
            // 剩余的小于最大的百分之五
            return;
        }
        // 判断暂存的容量还剩多少,够不够放
        if (this.carrierBuffer.remaining() < this.carrierBuffer.capacity() / 20) {
            resizeCarrierBuffer();
        }
        directByteBuffer.flip();
        this.carrierBuffer.put(directByteBuffer);
    }

    private void resizeCarrierBuffer() {
        ByteBuffer directBuffer = ByteBuffer.allocateDirect(DEFAULT_BUFFER_SIZE * 2);
        directBuffer.put(this.carrierBuffer);
        this.carrierBuffer = directBuffer;
    }
}

public class BIOServer {
    public static void main(String[] args) throws Exception {
        bio();

    }
    private static final SocketIOHandler SOCKET_IO_HANDLER = new SocketIOHandler();


    private static void bio() throws Exception {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080),200);
        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept();
            System.out.println("连接成功建立");
            SOCKET_IO_HANDLER.processData(socketChannel);
        }
    }
}
```

 然后我们来写客户端，我们先不写数据: 

```java
public class ClientDemo {
    public static void main(String[] args) throws Exception {
        Socket socket = new Socket();
        socket.connect(new InetSocketAddress(8080));       
        TimeUnit.SECONDS.sleep(40);
    }
}
```

![](https://a.a2k6.com/gerald/i/2025/02/14/5wmz3.png)

   ![](https://a.a2k6.com/gerald/i/2025/02/14/4cw2.png)

我们的main线程停留在SocketDispatcher.read0这一行，当前的线程状态是Runnable，在Java中处于这个状态的线程有以下三种情况:

1. 已经在JVM中运行
2. 可能在等待系统资源
3. 随时可被调度执行

如果再把read函数的细节展开，我们会发现其阻塞在了三个阶段，第一个阶段是数据到达网卡，在没有数据到达网卡之前，线程将一直被阻塞在这里。第二个阶段是从网卡到达内核缓冲区。第三个阶段是从内核缓冲区到达用户缓冲区。这个时候读才就绪，read函数就能接着向下执行了。

现在的情况属于第二种就是在等数据的到来，在数据到来之前，这个线程被等待在这里。 那为了能够处理多个连接，一种简单的方案是线程池，我们对每一个连接都弃用一个线程来处理。 就好像我们现在有个机场，这个机场效率很低，每一个航班都派了一个人在等飞机是否到来，在飞机到来之前这个人什么都不能干。这无疑造成了资源的浪费，一个连接一个线程。

那怎么提升效率呢？ 一种方案是引入塔台和飞机建立通讯，当飞机快要降落的时候，飞机主动上报，塔台通知对应的机场工作人员去准备。这也就是我们下面要讲的非阻塞IO。第二种方案是当线程阻塞在这里的时候，将这个线程切出去做别的事情，等到资源就绪的时候，再回复这个任务的执行，这对应协程，我们会在后面讲虚拟线程的实现。

## 由此引出NIO

在上面我们都是借助类比的思维，将通讯的过程比做迎接飞机，这不本质。现在让我们从最初的Linux调用先前一步一步推导设计，观察如何优化我们的调用。

###  回顾系统调用

当我们使用Java 的网络库在做网络编程的时候，本质上使用的还是操作系统的能力。也就是调用操作系统提供的api,  在Linux下面BIO调用的几个系统函数如下:

```C
int socket(int domain, int type, int protocol);
int setsockopt(int socket, int level, int option_name,const void *option_value, socklen_t option_len);
int bind(int sockfd, const struct sockaddr *addr,ocklen_t addrlen);
int listen(int sockfd, int backlog);
int accept(int sockfd, struct sockaddr *_Nullable restrict addr,socklen_t *_Nullable restrict addrlen);
ssize_t read(int fd, void buf[.count], size_t count);
ssize_t send(int sockfd, const void buf[.size], size_t size, int flags);
```

我们一般通过socket()调用创建Socket，然后通过setsockopt()设置参数，通过bind绑定地址，listen调用开启监听，accept接收连接。read函数读取数据，send函数写数据。

![](https://a.a2k6.com/gerald/i/2025/02/15/vxcl.jpg)

注意到里面有形参里面都有fd，这个fd事实上file descriptor，意为文件描述符。在Linux系统里面一切皆可以看成文件，当我们在操作这些文件的时候就需要获得这些文件的引用。如果用名字来进行查找，我们每操纵一次就要查找一次名字，这无疑效率比较慢。所以Linux规定每一个文件都对应一个索引，这样要操作文件的时候，我们直接找到索引就可以对其进行操作了。 通常文件描述符号是一个非负整数（通常是小整数）。

![](https://a.a2k6.com/gerald/i/2025/02/15/weqg.jpg)

现在我们再来看上面的调用，调用socket函数之后返回一个socket的fd，然后通过setsockopt函数传入fd，做参数设置。然后通过bind函数传入fd，绑定端口。然后listen也是接收fd。accept函数也是接收fd，返回是连接的fd。我们向read函数中传入fd，传入数组，读取数据。一切都是fd。

### 初步设计

那么我们自然就会萌生第一个想法 ， 能否请求操作系统为我们提供一个api，然后我们传入fd，询问是否可读，如果可读我们就进行读取。accept连接之后，拿到fd将其放入到一个集合里面。然后不断的去询问操作系统数据到了没有。

```c
// list在这里声明
while(1){
   int clientFD	= accept(); // 就执行不到语句一
   list.add(clientFd);   
   for(int fd : list) {
     int result = read(); // 这里调用的是加强过的read,
     if(result == 0){
          read();
     }           
   }  
}
```

但上面的代码还是有问题，比如一个连接进来之后，list里面加入了一个fd，然后不断的遍历这个集合，无法执行到语句一。所以我们需要有一个专门处理accept连接的线程，一个专门去轮询操作系统的线程。两个线程之间共享连接fd集合，需要保证是线程安全的。现在看起来很完美，但是这个加强过之后的read函数，仍然是系统调用，仍然需要上下文切换。那能不能把这个read() 函数做进内核里面。 这也就引出了Linux的select调用。

### select与poll

```java
int select(int nfds, fd_set *_Nullable restrict readfds,
  fd_set *_Nullable restrict writefds,
  fd_set *_Nullable restrict exceptfds,
 struct timeval *_Nullable restrict timeout);
```

- nfds: 监控的文件描述符集里最大文件描述符加1
- readfds: 监控有读数据到达文件描述符集合，传入传出参数
- writefds:  监控有写数据到达文件描述符集合，传入传出参数
- exceptfds:  监控异常发生到达文件描述符集合，传入传出参数

我们可以将fd理解为一个数组。所以有了select之后，我们的程序就可以这么写:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <sys/select.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>

#define PORT 8888
#define MAX_CLIENTS 10
#define BUFFER_SIZE 1024

// FD管理器结构
typedef struct {
    int server_fd;           // 服务器fd
    int client_fds[MAX_CLIENTS]; // 客户端fd数组
    int max_fd;             // 当前最大的fd
    int client_count;       // 当前客户端数量
} FDManager;

// 初始化FD管理器
void init_fd_manager(FDManager *manager) {
    manager->server_fd = -1;
    manager->max_fd = -1;
    manager->client_count = 0;
    // 初始化客户端数组为-1
    for(int i = 0; i < MAX_CLIENTS; i++) {
        manager->client_fds[i] = -1;
    }
}

// 更新max_fd
void update_max_fd(FDManager *manager) {
    manager->max_fd = manager->server_fd;
    for(int i = 0; i < MAX_CLIENTS; i++) {
        if(manager->client_fds[i] > manager->max_fd) {
            manager->max_fd = manager->client_fds[i];
        }
    }
    printf("Max FD updated to %d\n", manager->max_fd);
}

// 添加新的客户端fd
int add_client(FDManager *manager, int client_fd) {
    if(manager->client_count >= MAX_CLIENTS) {
        return -1;
    }
    
    for(int i = 0; i < MAX_CLIENTS; i++) {
        if(manager->client_fds[i] == -1) {
            manager->client_fds[i] = client_fd;
            manager->client_count++;
            if(client_fd > manager->max_fd) {
                manager->max_fd = client_fd;
                printf("New max FD: %d\n", manager->max_fd);
            }
            return i;
        }
    }
    return -1;
}

// 移除客户端
void remove_client(FDManager *manager, int client_fd) {
    for(int i = 0; i < MAX_CLIENTS; i++) {
        if(manager->client_fds[i] == client_fd) {
            close(client_fd);
            manager->client_fds[i] = -1;
            manager->client_count--;
            update_max_fd(manager);
            printf("Client %d removed\n", client_fd);
            break;
        }
    }
}

// 主服务器程序
int main() {
    FDManager fd_manager;
    struct sockaddr_in server_addr, client_addr;
    fd_set read_fds;
    char buffer[BUFFER_SIZE];
    // 初始化FD管理器
    init_fd_manager(&fd_manager);
    
    // 创建服务器socket
    fd_manager.server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd_manager.server_fd < 0) {
        perror("Socket creation failed");
        exit(EXIT_FAILURE);
    }
    
    // 设置socket选项
    int opt = 1;
    if(setsockopt(fd_manager.server_fd, SOL_SOCKET, SO_REUSEADDR, 
                  &opt, sizeof(opt)) < 0) {
        perror("Setsockopt failed");
        exit(EXIT_FAILURE);
    }
    
    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);
    
    // 绑定地址
    if(bind(fd_manager.server_fd, (struct sockaddr *)&server_addr,
            sizeof(server_addr)) < 0) {
        perror("Bind failed");
        exit(EXIT_FAILURE);
    }
    
    // 监听连接
    if(listen(fd_manager.server_fd, 5) < 0) {
        perror("Listen failed");
        exit(EXIT_FAILURE);
    }
    
    printf("Server listening on port %d\n", PORT);
    
    // 更新max_fd
    fd_manager.max_fd = fd_manager.server_fd;
    
    // 主循环
    while(1) {
        // 清空fd_set
        FD_ZERO(&read_fds);
        
        // 添加服务器fd到集合
        FD_SET(fd_manager.server_fd, &read_fds);
       
        // 添加所有客户端fd到集合
        for(int i = 0; i < MAX_CLIENTS; i++) {
            int client_fd = fd_manager.client_fds[i];
            if(client_fd > 0) {
                FD_SET(client_fd, &read_fds);
            }
        }
        
        printf("Waiting for activity... (max_fd = %d)\n", fd_manager.max_fd);
        
        // 使用select等待活动
        int activity = select(fd_manager.max_fd + 1, &read_fds, NULL, NULL, NULL);
        if(activity < 0) {
            perror("Select error");
            continue;
        }
        
        // 检查服务器fd是否有新连接
        if(FD_ISSET(fd_manager.server_fd, &read_fds)) {
            socklen_t addr_len = sizeof(client_addr);
            int new_client = accept(fd_manager.server_fd, 
                                  (struct sockaddr *)&client_addr,
                                  &addr_len);
                                  
            if(new_client < 0) {
                perror("Accept failed");
                continue;
            }
            
            printf("New connection from %s:%d\n",
                   inet_ntoa(client_addr.sin_addr),
                   ntohs(client_addr.sin_port));
                   
            // 添加新客户端
            if(add_client(&fd_manager, new_client) < 0) {
                printf("Cannot accept more clients\n");
                close(new_client);
            }
        }
        
        // 检查客户端fd是否有数据
        for(int i = 0; i < MAX_CLIENTS; i++) {
            int client_fd = fd_manager.client_fds[i];
            
            if(client_fd > 0 && FD_ISSET(client_fd, &read_fds)) {
                memset(buffer, 0, BUFFER_SIZE);
                int read_size = read(client_fd, buffer, BUFFER_SIZE);
                
                if(read_size <= 0) {
                    // 客户端断开连接或错误
                    printf("Client %d disconnected\n", client_fd);
                    remove_client(&fd_manager, client_fd);
                }
                else {
                    // 回显数据
                    buffer[read_size] = '\0';
                    printf("Received from client %d: %s", client_fd, buffer);
                    write(client_fd, buffer, strlen(buffer));
                }
            }
        }
    }
    
    // 关闭所有连接
    for(int i = 0; i < MAX_CLIENTS; i++) {
        if(fd_manager.client_fds[i] > 0) {
            close(fd_manager.client_fds[i]);
        }
    }
    close(fd_manager.server_fd);
    
    return 0;
}
```



###  epoll降临



### 小小的总结一下

 我们从第一性原理一次



### 那在Java里面 NIO该怎么写







## 由此引出AIO





## 由此引出io_uring





## 总结一下









## 参考资料

[1] Linux – IO Multiplexing – Select vs Poll vs Epoll  https://devarea.com/linux-io-multiplexing-select-vs-poll-vs-epoll/ 

[2 ]  read、write 与recv、send区别 gethostname   https://www.cnblogs.com/Malphite/p/11645067.html

[3] select函数及fd_set介绍   https://www.cnblogs.com/wuyepeng/p/9745573.html
