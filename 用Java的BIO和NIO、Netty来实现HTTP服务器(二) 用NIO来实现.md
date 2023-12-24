# 用Java来实现BIO和NIO模型的HTTP服务器(二) NIO模型

> 翻了一下(一)发现整体还是不大好, 这里重新再梳理一下

[TOC]

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

```java
ServerSocket serverSocket = new ServerSocket();
// 绑定在8080端口
serverSocket.bind(new InetSocketAddress(8080));
// 监听连接,该方法会阻塞到这里直到有连接建立完成
// 发起系统调用
while (true) {
    try (Socket socket = serverSocket.accept()) {
        try (InputStream socketInputStream = socket.getInputStream()) {
            byte[] readByte = new byte[4096];
            // 这里其实数据不见得立马可以读, 因为数据不代表立马可以读
            // 发起系统调用
            int readTotalNumber = socketInputStream.read(readByte);
            String s = new String(readByte, 0, readTotalNumber);
            System.out.println(s);
        }
    }
}
```

在这种模型下同时只能处理一下一个连接，因为我们只有一个线程，这个链接的读取逻辑没处理完毕，下一个得等待在那里。我们这里解释一下，我们调用SocketInputStream的read函数的时候为什么不是立刻能读取，一般来说，一台能联网的计算机首先得有网卡不管是有线网卡还是无线网卡，数据经过路由器之后网线到达网卡，然后将数据包从网卡硬件缓存转移到内存中，然后通知内核处理，然后经过TCP/IP协议层处理，最后应用程序通过系统调用读取到发送过来的数据。

![](https://a.a2k6.com/gerald/i/2023/12/16/wwak.jpg)



上面的写法还面临的一个问题就是没有判断什么时候报文结束，如果是短链接即传输一次消息连接就关闭，那么read函数返回-1就代表数据结束，如果我们希望TCP连接保活，即保持这个链接，我们只是做示例，完整的会在下面构建HTTP服务器中详细讲述。

如果你熟悉网络编程，还有一个网络异常会经常碰到: 

```java
Exception in thread "main" java.net.SocketException: Connection reset by peer
```

被对等方重置连接，这是啥意思？  相当于突然挂电话，这比单纯的不回应、让人等着更有礼貌。但这并不是真正有礼貌的TCP/IP的关闭方式。也就是说连接建立了，某一方突然关闭连接，另一方还在使用这个连接，就会出现这个异常。

在《用Java的BIO和NIO、Netty来实现HTTP服务器(一) 》里面我们用的是在1.4引入的新API，这套API的优势就是比较统一，可以通过ServerSockeChannel的configureBlocking来制定使用BIO还是NIO，所以上面服务端的写法可以等价转换为: 

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.bind(new InetSocketAddress(8080));
while (true) {
    try (SocketChannel socketChannel = serverSocketChannel.accept()) {
        ByteBuffer byteBuffer = ByteBuffer.allocate(4096);
        socketChannel.read(byteBuffer);
        byteBuffer.flip();
        byte[] readDataArray = new byte[byteBuffer.limit()];
        byteBuffer.get(readDataArray);
        String readData = new String(readDataArray);
        System.out.println(readData);
    }
}
```

ByteBuffer基于byte数组封装了一些常见的操作，可以理解为一个容器，put存，get取，但是在取之前我们需要知道取到哪个位置，调用flip方法之后，调用limit方法之后就能知道ByteBuffer中放了多少元素。

## 前面面临的问题

(一)是七月份写的,这里我们在复习一下里面的内容, 我们希望构建用Java标准库，以及Java的NIO框架Netty实现一个简单的HTTP服务器，(一)是BIO模型，不管是NIO还是BIO，还是用框架构建，基于TCP协议的网络编程都会面临这样一个问题，首先是管理连接，也就是连接建立，连接建立之后我们读取数据，那么HTTP报文不固定大小，我们需要根据报文结束标志来判断是否读取结束，然后读取完整之后交给下一层去处理。但是NIO还是BIO他们具备共性，所以我们用继承来实现，我们首先抽象了一个Server的基础类:

```java
public abstract class Server {

    protected ServerSocketChannel serverSocketChannel;

    protected SSLContext sslContext;

    static int port = 8000;

    static int backlog = 1024;

    static boolean secure = false;

    public Server(int port, int backlog, boolean secure) throws IOException {
        // 创建一个ServerSocketChannel
        serverSocketChannel =  ServerSocketChannel.open();
        /**
         * 可以重用ip+端口这个链接,TCP以链接为单位,当TCP链接要关闭的时候，会等待一段时间再进行关闭,
         * 如果我想要重用端口,那么channel就无法绑定,在绑定到对应地址之前,设定重用地址。即使在这个端口上的tcp连接
         * 处于处于TIME_WAIT状态,我们仍然可以使用
         */
        serverSocketChannel.socket().setReuseAddress(true);
        // 绑定端口和backlog
        serverSocketChannel.bind(new InetSocketAddress(port),backlog);

    }

    /**
     * 这里是抽象方法,我们后面要用NIO再实现一遍
     * 所以这里交给子类来实现
     */
    protected  abstract void runServer();

    /**
     * 我们要写的是一个简单的HTTP服务器,
     * 这个服务器可以从命令行方式启动的时候接收参数
     * 我们可以选择从main函数
     * @param args
     */
    public static void main(String[] args) throws IOException {
        Server server = null;
        if (args.length == 0){
            System.out.println("http server running default model");
            server = new BlockingServer(port,backlog,secure);
            server.runServer();
        }
        // 端口目前先固定死, 我们目前只读一个参数
        if ("B".equals(args[0])){
            server = new BlockingServer(port,backlog,secure);
        }else if ("N".equals(args[0])){
            server = new NonBlockingServer(port,backlog,secure);
        }else{
            System.out.println("input args error only support B OR N");
            return;
        }
        server.runServer();
    }
}

public class BlockingServer extends Server {
    
    public BlockingServer(int port, int backlog, boolean secure) throws IOException {
        super(port, backlog, secure);
    }
    
    @Override
    protected void runServer() throws IOException {
        for (;;){
            SocketChannel socketChannel = serverSocketChannel.accept();
            ChannelIO channelIO = ChannelIO.getInstance(socketChannel, true);
            RequestServicer requestServicer = new RequestServicer(channelIO);
            requestServicer.run();
        }
    }
}
```

我们用ServerSocketChannel这个为核心来构建HTTP服务器，原因是实现上更为统一，我们最终可以做成一个jar，所以我们根据命令行参数来决定是BIO还是NIO模型的服务器。启动之后应当是一个无限循环，不断接收连接，不断处理请求。连接建立之后，我们需要不断的读数据，这是NIO和BIO共同的特征，所以我们写了一个ChannelIO工具类，来实现对数据的读取：

```java
public class ChannelIO {

    private SocketChannel socketChannel;

    private ByteBuffer requestBuffer;

    int defaultByteBufferSize = 4096;

    private ChannelIO(SocketChannel socketChannel, boolean blocking) throws IOException {
        this.socketChannel = socketChannel;
        this.requestBuffer = ByteBuffer.allocate(4096);
        this.socketChannel.configureBlocking(blocking);
    }


    public static ChannelIO getInstance(SocketChannel socketChannel, boolean blocking) throws IOException {
        return  new ChannelIO(socketChannel,blocking);
    }

    public int read() throws IOException {
        // 剩余的小于百分之五自动扩容
        resizeByteBuffer(defaultByteBufferSize / 20);
        return socketChannel.read(requestBuffer);
    }

    private void resizeByteBuffer(int remaining) {
        if (requestBuffer.remaining() < remaining){
            // 扩容一倍
            ByteBuffer newRequestBuffer = ByteBuffer.allocate(requestBuffer.capacity() * 2);
            // 转为读模式
            requestBuffer.flip();
            //  将旧的buffer放入到新的buffer中
            newRequestBuffer.put(requestBuffer);
            requestBuffer = newRequestBuffer;
        }
    }

    public ByteBuffer getReadBuf(){
        return this.requestBuffer;
    }

    public int write(ByteBuffer byteBuffer) throws IOException{
        return socketChannel.write(byteBuffer);
    }

    public void close() throws IOException{
        socketChannel.close();
    }
}
```

ChannelIO主要的几个作用就是读和写，默认的ByteBuffer为4096，但是报文大小有可能超过，所以这里我们读之前看看需不需要自动扩容，这个类被请求处理者所处理, 请求处理者要负责解析HTTP报文，HTTP报文有请求方式，有结束标志，有uri，这个解析的任务我们放在Request这个类来处理: 

```java
public class Request {

    private Action action;

    private URI uri;

    private String version;

    public Request(Action action, URI uri, String version) {
        this.action = action;
        this.uri = uri;
        this.version = version;
    }

    static class Action{
        private String name;

        static Action GET = new Action("GET");

        static Action POST = new Action("POST");

        static Action PUT = new Action("PUT");

        static Action HEAD = new Action("HEAD");

        public Action(String name) {
            this.name = name;
        }
        public String toString(){
            return this.name;
        }
        static Action parse(String s){
            if ("GET".equals(s)){
                return GET;
            }
            if ("POST".equals(s)){
                return POST;
            }
            if ("PUT".equals(s)){
                return PUT;
            }
            if ("HEAD".equals(s)){
                return HEAD;
            }
            // 参数不合法
            throw new IllegalArgumentException(s);
        }
    }


    public  static boolean isComplete(ByteBuffer byteBuffer){
        int position = byteBuffer.position() - 4;
        if (position < 0){
            return false;
        }
        return byteBuffer.get(position + 0) == '\r'
                && byteBuffer.get(position + 1) == '\n'
                && byteBuffer.get(position + 2) == '\r'
                && byteBuffer.get(position + 3) == '\n';
    }

    private static Charset ascii = StandardCharsets.US_ASCII;

    /**
     * 正则表达式 用来分割请求报文
     * http 请求的报文是: GET /dir/file HTTP/1.1
     *  Host: hostname
     *  被正则表达式分割以后:
     *      group[1] = "GET"
     *      group[2] = "/dir/file"
     *      group[3] = "1.1"
     *      group[4] = "hostname"
     */
    private static Pattern requestPattern
            = Pattern.compile("\\A([A-Z]+) +([^ ]+) +HTTP/([0-9\\.]+)$"
                    + ".*^Host: ([^ ]+)$.*\r\n\r\n\\z",
            Pattern.MULTILINE | Pattern.DOTALL);

    public static  Request parse(ByteBuffer byteBuffer) throws RequestException {
        // byte to char
        CharBuffer charBuffer = ascii.decode(byteBuffer);
        Matcher matcher = requestPattern.matcher(charBuffer);
        // 未匹配
        if (!matcher.matches()){
            throw new  RequestException();
        }
        Action a;
        try {
            a = Action.parse(matcher.group(1));
        }catch (IllegalArgumentException  e){
            throw new RequestException();
        }
        URI u = null;
        try {
            u = new URI("http://" + matcher.group(4) + matcher.group(2));
        }catch (URISyntaxException e){
           throw new RequestException(e);
        }
        return new Request(a,u,matcher.group(3));
    }
}
```

这个类主要封装HTTP报文的请求方式、版本、URI。有Request就有Reply，一个HTTP响应通常情况下会有状态码和内容，这里我们的HTTP服务器将来要扩展到各种类型： 

```java
public interface Sendable {
    // 做转码
    void prepare() throws IOException;
	// 发送动作
    boolean send(ChannelIO channelIO);
    
    void release();
}
public interface Content extends Sendable {
    // 发送类型
    String type();
    
	// 长度
    long length();
}
public class StringContent implements Content{

    private String type;    // MIME type

    private String content;

    private ByteBuffer byteBuffer;

    private static final Charset ascii = StandardCharsets.US_ASCII;

    StringContent(CharSequence c, String t) {
        content = c.toString();
        type = t + "; charset=iso-8859-1";
    }

    StringContent(CharSequence c) {
        this(c, "text/plain");
    }

    StringContent(Exception x) {
        StringWriter sw = new StringWriter();
        x.printStackTrace(new PrintWriter(sw));
        type = "text/plain; charset=iso-8859-1";
        content = sw.toString();
    }

    @Override
    public String type() {
        return type;
    }

    @Override
    public long length() {
        return byteBuffer.remaining();
    }

    @Override
    public void prepare() throws IOException {
        encode();
        // 在写入之前就需要调用一下rewind方法
        byteBuffer.rewind();
    }

    private void encode() {
        if (byteBuffer == null){
            byteBuffer =  ascii.encode(CharBuffer.wrap(content));
        }
    }

    @Override
    public boolean send(ChannelIO channelIO) throws IOException {
        if (byteBuffer == null)
            throw new IllegalStateException();
        // 写的时候 不见得一次写完
        channelIO.write(byteBuffer);
        // hasRemaining 代表是否还有剩余
        // 如果有剩余就可以写着写
        return  byteBuffer.hasRemaining();
    }

    /**
     * 这是个空方法
     * 后面只是为了统一调用
     */
    @Override
    public void release() {

    }
}
```

```java
public class Reply implements Sendable {

    static class Code{

        private int number;

        private String description;

        public Code(int number, String description) {
            this.number = number;
            this.description = description;
        }

        static Code OK = new Code(200,"OK");

        static Code BAD_REQUEST = new Code(400,"Bad Request");

        static Code NOT_FOUND = new Code(404,"Not Found");

        static Code METHOD_NOT_ALLOWED = new Code(405,"Method Not Allowed");
    }

    private Code code;

    private Content content;

    private ByteBuffer headerBuffer;


    private static String CRLF = "\r\n";

    private static Charset ascii = Charset.forName("US-ASCII");

    public Reply(Code code, Content content) {
        this.code = code;
        this.content = content;
    }

    /**
     * 这个方法负责添加请求头
     * @return
     */
    private ByteBuffer headers(){
        CharBuffer cb = CharBuffer.allocate(1024);
        cb.put("HTTP/1.0 ").put(code.toString()).put(CRLF);
        cb.put("Server: niossl/0.1").put(CRLF);
        cb.put("Content-type: ").put(content.type()).put(CRLF);
        cb.put("Content-length: ")
                .put(Long.toString(content.length())).put(CRLF);
        cb.put(CRLF);
        cb.flip();
        return ascii.encode(cb);
    }

    @Override
    public void prepare() throws IOException {
        content.prepare();
        headerBuffer = headers();
    }

    @Override
    public boolean send(ChannelIO channelIO) throws IOException {
         // 先写请求头
        if (headerBuffer.hasRemaining()){
            if (channelIO.write(headerBuffer) <= 0)
                return true;
        }
        // 再写响应内容
        if (content.send(channelIO))
            return true;
        return false;
    }

    @Override
    public void release() {
        content.release();
    }
}
```

连接建立之后开始提取数据:

```java
public class RequestServicer implements Runnable {


    private ChannelIO channelIO;

    public RequestServicer(ChannelIO channelIO) {
        this.channelIO = channelIO;
    }

    private void service() throws IOException {
        ByteBuffer byteBuffer = receive(); // 接收数据
        Request request = null;
        Reply reply = null;
        try {
            request = Request.parse(byteBuffer);
        } catch (RequestException e) {
            reply = new Reply(Reply.Code.BAD_REQUEST, new StringContent(e));
        }
        // 说明正常解析
        if (reply == null) {
            reply  = build(request); // 构建回复
        }
        reply.prepare();
        do {} while (reply.send(channelIO));         // Send
    }

    @Override
    public void run() {
        try {
            service();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

	
    ByteBuffer receive() throws IOException {
        for (; ; ) {
            int read = channelIO.read();
            ByteBuffer bb = channelIO.getReadBuf();
            if ((read < 0) || (Request.isComplete(bb))) {
                bb.flip();
                return bb;
            }
        }
    }

    Reply build(Request rq) throws IOException {

        Reply rp = null;
        Request.Action action = rq.action();
        if ((action != Request.Action.GET)) {
            rp = new Reply(Reply.Code.METHOD_NOT_ALLOWED,
                    new StringContent(rq.toString()));
            rp.prepare();
            return rp;
        }
        rp = new Reply(Reply.Code.OK,
                new StringContent("hello world"));
        rp.prepare();
        return rp;
    }
}
```

## NIO简介

上面模型也被称为BIO模型，也就是Blocking Input/Output, 其实上面已经分析出来在哪里了，也就是读数据的时候未必可以读，但是我们的read调用就被阻塞在那里，我们自然能够想到能否让操作系统为我们提供一个非阻塞的read函数, 这个 read 函数的效果是，如果没有数据到达时（到达网卡并拷贝到了内核缓冲区），立刻返回一个错误值（-1），而不是阻塞地等待。操作系统提供了这样的功能，只需要在调用 read 前，将文件描述符设置为非阻塞即可。这样我们在线程里面调用read函数，直到返回值不为-1的，再开始处理业务。 但是在数据到达内核缓冲区，这个阶段仍然是阻塞的，需要等待数据从内核缓冲区拷贝到用户缓冲区，才能返回。

这里可以为每个连接准备一个线程来处理，这其实也是解决问题的方案，一些连接请求不多的HTTP服务器现在还是这么处理的，那么对于连接过多的，多线程就有些乏力了，当然也可以有聪明的方法，我们可以每accept一个连接之后，将这个文件描述符(可以理解为Socket的引用)放在一个数组里面，然后弄一个新的线程去不断遍历这个数组，调用每一个元素的非阻塞 read 方法，这样，我们就成功用一个线程处理了多个客户端连接。

这看起来就有多路复用的意思了，但这和我们用多线程去将阻塞 IO 改造成看起来是非阻塞 IO 一样，这种遍历方式也只是我们用户自己想出的小把戏，每次遍历遇到 read 返回 -1 时仍然是一次浪费资源的系统调用。所以，还是得恳请操作系统老大，提供给我们一个有这样效果的函数，我们将一批文件描述符通过一次系统调用传给内核，由内核层去遍历，才能真正解决这个问题。

select 是操作系统提供的系统调用函数，通过它，我们可以把一个文件描述符的数组发给操作系统， 让操作系统去遍历，确定哪个文件描述符可以读写， 然后告诉我们去处理。

但是这个函数仍然不完美，原因在于: 

1. select 调用需要传入 fd 数组，需要拷贝一份到内核，高并发场景下这样的拷贝消耗的资源是惊人的。（可优化为不复制）

2. select 在内核层仍然是通过遍历的方式检查文件描述符的就绪状态，是个同步过程，只不过无系统调用切换上下文的开销。（内核层可优化为异步事件通知）
3. select 仅仅返回可读文件描述符的个数，具体哪个可读还是要用户自己遍历。（可优化为只返回给用户就绪的文件描述符，无需用户做无效的遍历）

但也不是不能用，但select还有限制，这个限制就是select 只能监听 1024 个文件描述符的限制，后面的poll去掉了这个限制。最终解决select函数的大boss叫epoll，针对select函数的三个不完美的点进行了修复:

1. 内核中保存一份文件描述符集合，无需用户每次都重新传入，只需告诉内核修改(添加、修改、监控的文件描述符)的部分即可。
2. 内核不再通过轮询的方式找到就绪的文件描述符，而是通过异步 IO 事件唤醒。
3. 内核仅会将有 IO 事件的文件描述符返回给用户，用户也无需遍历整个文件描述符集合。

## 重回NIO

“内核仅会将有IO事件的文件描述符返回给用户”，仔细读这一句话，我以为是select函数的返回值是一个集合，但是我去看了一下这个函数:

```c
int select(int maxfd, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);
```

fd是一个集合类型，从参数名字上我们来推断readfds传入需要监视读事件的文件描述符，writefds是需要监视读事件的文件描述符，exceptfds是异常事件的文件描述符，这里我们提到了文件描述符，这个文件描述符是用来代表一个打开的文件、或者socket、或者其他数据源。定义了能对该文件做的操作。当select函数有所返回的时候，会修改传入的集合。select函数是系统调用，Java层面对应的抽象也就是Selector，使用起来倒是简单:

```java
Selector selector = Selector.open();
selector.select();
```

默认选择当前操作系统的实现，我们看下Open的实现，我的电脑装的操作系统是Windows，select的实现在Oracle 的hotspot VM中是闭源的，观察他的实现要在OpenJDK上，我这里随手选了一个JDK 11版本的实现:

![](https://a.a2k6.com/gerald/i/2023/12/23/4rhl.jpg)

​	![](https://a.a2k6.com/gerald/i/2023/12/23/19mubx.jpg)

那怎么让这个选择器知道我对某个事件感兴趣 , 读事件就绪、写事件就绪其实通道(通道也就是对连接的抽象)上发生的事件，按照我之前的想法Selector这个类里面应该会有一个register之类的方法，但是没找到，不在Selector就在ServerSocketChannel里面，果然我在ServerSocketChannel找到了register方法, 这个register来自AbstractSelectableChannel。

```java
// ops是一个枚举值,att是对应的事件触发之后,交付给哪个对象处理
public final SelectionKey register(Selector sel, int ops, Object att)
```

所以我们可以写成下面这样:



## NIO的实现









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

[18]   What does "connection reset by peer" mean?  https://stackoverflow.com/questions/1434451/what-does-connection-reset-by-peer-mean

[19] linux select函数解析以及事例 https://zhuanlan.zhihu.com/p/57518857
