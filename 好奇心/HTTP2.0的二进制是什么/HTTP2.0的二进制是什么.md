# HTTP/2.0的二进制是什么?

> 这篇纯粹满足自己的好奇心

> 我好像是一个在海边玩耍的孩子，不时为拾到比通常更光滑的石子或更美丽的 贝壳而欢欣鼓舞，而展现在我面前的是完全未探明的真理之海。牛顿

写本文的时候，想起高中物理课本的一句话:

> 我好像是一个在海边玩耍的孩子，不时为拾到比通常更光滑的石子或更美丽的贝壳而欢欣鼓舞，而展现在我面前的是完全未探明的真理之海。

那个时候不懂这句话，忙于刷分，如今纯粹是为了自己的好奇心而探究一些问题，脑海中又开始复现这句话。本文的问题来自于前面的一篇文章:《HTTP学习笔记(三) HTTP/2》， 这篇文章里我们提到了HTTP/2的几个特点:

1. is binary, instead of textual

> 二进制代替了文本

2. is fully multiplexed, instead of ordered and blocking

> 多路复用

3. can therefore use one connection for parallelism

> 并行请求

4. uses header compression to reduce overhead

> 压缩请求头，减少消耗

5. allows servers to “push” responses proactively into client caches

> 允许服务器主动推送响应进入客户端的缓存中

其实对于1我是不理解的，毕竟在计算机的世界都是“二进制”嘛，当时我的想法是难道是跟JDK处理String一样的操作，在JDK8之前，String本身是借助于char来存储的:

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

到了JDK 8之后, JDK借助byte来存储字符串:

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc {
    @Stable
    private final byte[] value;
}    
```

毕竟一个char占两个字节, 一个byte只占一个字节，因为我之前用程序连接过充电桩，接收充电桩的报文，给的报文都是byte类型的，byte更小，像String就带了一些额外的信息，所以我猜想，是这个意义上的二进制，但是这只是猜想，我想过用抓包工具去验证我的猜想，但是发现抓包工具我用的并部署，再加上HTTP/2.0都是加密报文，抓包挺麻烦的，我也想过看HTTP Client的源码，但是这两个见效都太慢了，最近偶然翻看MongDB的文档，翻到了这方面的说明，这个问题就有了答案。其实HTTP也对上面的二进制进行了解释:

> Why is HTTP/2 binary?
>
> Binary protocols are more efficient to parse, more compact “on the wire”, and most importantly, they are much less error-prone, compared to textual protocols like HTTP/1.x,  because they often have a number of affordances to “help” with things like whitespace handling, capitalization, line endings, blank lines and so on.
>
> 二进制协议相对于文本协议，比如HTTP/1.x ，解析效率、传输效率更高，有更好的容错性。还提供了一些机制可以帮助处理空白字符、大小写、空行等等。
>
> For example, HTTP/1.1 defines four different ways to parse a message; in HTTP/2, there’s just one code path.
>
> 例如，HTTP/1.1定义了四种解析数据的方式，但是在HTTP/2, 只有一种代码路径。
>
> It’s true that HTTP/2 isn’t usable through telnet, but we already have some tool support, such as a Wireshark plugin.
>
> 虽然HTTP/2已经不能再使用Telnet了，但是我们也有其他工具的支持，比如 Wireshark plugin。

所以HTTP说自己是二进制的，潜台词是HTTP/2的数据包采取了高度结构化的格式，因为在计算机中，最终一切都是二进制形式存在。在HTTP/2中传输的数据会被格式化为帧(frame),  每个帧都会被分配一个流。HTTP/2的帧具备特定的格式，如下图所示:

```http
 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```

在HTTP/2中，每个帧都由两部分组成: 帧头(9个字节)，帧头是固定长度的，占据9个字节，包含了关于该帧的一些信息，比如长度、类型等。帧头后面是有效载荷，长度可变，取决于帧的类型和内容。这有点类似于TCP数据包，读取HTTP/2帧可以遵循定义好的过程(先读取数据长度，然后帧的类型)。相比之下，HTTP/1.1是由一个ASCII编码的文本行组成的非结构化格式，虽然这些文本最终将以二进制形式传输，但是基本上它是一串字符的流，而不是明确地被分为独立的帧。

![From a user-, script-, or server- generated event, an HTTP/1.x msg is generated, and if HTTP/2 is in use, it is binary framed into an HTTP/2 stream, then sent.](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages/httpmsg2.png)



HTTP/1.1的消息通过逐个字符地读取字符来解析，直到达到换行字符为止，这种方式有点混乱，但是因为无法知道每行的长度，所以必须逐个字符进行处理。对于HTTP正文的长度可以提前知道，我们可以在HTTP头里获知到这个信息。

这让我想起了MongDB的BSON，我想BSON中的binary的语义应当和HTTP/2的binary语义是对等的，在BSON规范的官网可以看到我们的猜想是正确的:

> BSON is a binary format in which zero or more ordered key/value pairs are stored as a single entity. We call this entity a *document*.
>
> BSON是一种二进制格式，其中有零个或多个有序的键/值对被存储为一个单一的实体，我们将这个实体称之为文档。

下面是一个JSON和其对应的BSON格式的示例:

```java
{"hello": "world"} →
\x16\x00\x00\x00           // total document size 总的文档大小 算上大小字段本身
\x02                       // 0x02 = type String 
hello\x00                  // field name 字段值
\x06\x00\x00\x00world\x00  // field value 字段值
\x00                       // 0x00 = type EOO ('end of object')
```

总结一下，在计算机中最终一切都是二进制格式，当我们在数据格式中看到二进制时，我们可以理解为这种存储结构是高度结构化的 , 读取效率更高，更为紧凑，将数据重新进行布局。



