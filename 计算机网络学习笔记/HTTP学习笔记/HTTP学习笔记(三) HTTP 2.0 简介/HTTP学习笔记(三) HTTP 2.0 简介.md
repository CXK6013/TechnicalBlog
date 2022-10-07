# HTTP学习笔记(三) HTTP/2

> 这里简单的介绍一下HTTP 2.0。

[TOC]

## 由HTTP 1.1 走向 HTTP 2.0

写这篇文章的时候我在听B站UP主翻唱的歌曲，然后我心血来潮打算看看B站现在用的是HTTP的哪个版本,于是我摁下了F12键。

![B站使用的网络协议版本](http://tvax4.sinaimg.cn/large/006e5UvNly1h48prtiyxfj31gv0mijvx.jpg)

这个h2和h3代表的是HTTP 2.0 和3.0? 这版本号刷的这么快的吗？ 不应该是2.1==>2.5 ==>3.0这样吗？为了验证我的想法，我打开了火狐浏览器。

![火狐浏览器](http://tvax3.sinaimg.cn/large/006e5UvNly1h48q3rr2t3j31h30cojwx.jpg)

所以就很突然，本来按照计划只介绍HTTP/2.0的，然后HTTP/3.0也只得加入到学习计划中，所以每次学习一个知识点的时候，总会碰到新的，生也有涯，知也无涯的感觉，在前面两篇文章了，我们已经大致的介绍HTTP 0.9 、1.0、1.1。

## 2.0 与1.1的不同

### 1.1 的新特性

我们这里来再度介绍一下HTTP 1.1带来的改进:

- 连接可以复用，节省了多次打开 TCP 连接加载网页文档资源的时间。

> HTTP 1.1 之前的连接模型是短连接，也是HTTP/1.0是短连接，每一个HTTP请求都由它自己独立的连接完成；这意味着发起每一个HTTP请求之前都会有一次TCP握手。 TCP协议本身握手本身就是耗费时间的，所以TCP可以保持更多的热连接来适应负载。
>
> 火狐开发者文档是如是吐槽这个模型的: 除非是要兼容一个非常古老的，不支持长连接的系统，没有一个令人信服的理由继续使用这个模型。为了缓解这些问题，长连接的概念便被设计出来了，甚至在HTTP/1.1之前。或者这被称为一个keep-alive连接。
>
> 一个长连接会保持一段时间，重复用于发送一系列请求，节省了新建TCP连接握手的时间，还可以利用TCP性能增强能力。当然这个连接也不会一直保留着: 连接在空闲一段时间后会被关闭(服务器可以使用Keep-Alive协议头来指定一个最小的连接保持时间)
>
> 但是长连接也不是完美无缺的；就算是在空闲状态，它还是会消耗服务器资源，而且在重负载时，还有可能遭受Dos attacks攻击。在这种场景下，可以使用非长连接，既尽快关闭那些空闲的连接，也能对性能有所提升。

- 增加管线化技术，允许在第一个应答被完全发送之前就发送第二个请求，以降低通信延迟。

> 管线化的英文是pipelining，pipelining还有另一个意思是流水线。我看火狐开发者文档的时候，中文版将pipelining在介绍HTTP的演变为翻译为管线化，但是在介绍连接管理的时候，又将其译为流水线。我刚看的时候还以为是两种技术，事实上是一个名词的翻译。
>
> 但是火狐的开发者文档评价管线化技术，比较难用，现代浏览器默认不会启用此特性。这个特性被更好的算法所代替，也就是HTTP/2

- 支持响应分块。

> 简单的说这个代表分块传输，通常情况下，HTTP应答消息中发送的数据是整个发送的，Content-Length消息头部表示数据的长度，客户端需要知道哪里是应答消息的结束，以及后续应答消息的开始。一个比较常见的应用是断点续传，实现H5页面的大视频播放，实现渐进式播放，不需要将整个文件加载到内存中。
>
> 值得注意的是此特性在HTTP/2.0中并不支持，HTTP/2.0提供了一种更加有效率的数据流方式。

- 引入额外的缓存控制机制。

> 简单的说就是引入了Cache-Control, 用Cache-control区分了缓存的类型，制定了缓存过期策略。

- 引入内容协商机制，包括语言，编码，类型等，并允许客户端和服务器之间约定以最合适的内容进行交换。

- 凭借Host头, 能够使不同域名配置在同一个ip地址的服务器上。

![HOST头](http://tvax3.sinaimg.cn/large/006e5UvNly1h49qksb7s8j31gt0ozwml.jpg)

有的同学看到这里会问，众所周知，HTTP用的是TCP协议，发送HTTP请求的时候，IP地址和端口必然是已知的，那么HOST的意义在哪里呢？举一个实际的例子是，Tomcat中部署多个服务，事实上一个Tomcat可以对应多个服务，现在是Spring Boot时代，Tomcat已经默认集成在里面，是一个应用对一个Tomcat。如果你开发过Servlet，还没到Spring Boot，那个时候是打war包的，war被放在webapps下面，我们在Tomcat中可以配置Host头，Tomcat会根据不同的host头转发到对应的服务上。打开Tomcat的server.xml:

![Server.xml](http://tvax1.sinaimg.cn/large/006e5UvNgy1h49r5rxq38j30u40jen0i.jpg)

  ### 流水线(pipelining)的设计问题

上面我们提到了HTTP 1.1引入了流水线，允许第一个响应被完全返回前，客户端发送第二个请求，我认为这是合理的设计，但是为什么现代浏览器大多没启用此特性吗？

> I would always recommend going to the authoritative source when trying to understand the meaning and purpose of HTTP headers. 
>
> 如果你在尝试理解HTTP头的含义和用途时，我总是建议你去原始权威文档. 

上面这句话出自StackOverflow中What is HTTP "Host" header?的答案下，我挺喜欢这句话的。其实这个答案我在StackOverFlow上已经看到了答案，有同学可能会问，百度找不到吗？说句惭愧的话，找的到，但是比较费劲，Stackoverflow这种问答式网站找起问题来更快。

答案是:  流水线要求响应按请求顺序返回。

![流水线](http://tvax4.sinaimg.cn/large/006e5UvNly1h49shjvvskj311o0brt9d.jpg)

这种设计会带来队头阻塞问题(**Head of line Blocking**), 只要第一个HTTP流遭遇到阻塞，那么第二个响应就得一直排队等待。那为什么你要按顺序返回呢？

>  答案是: Because there is no identifier in the response indicating to which request it belongs.
>
>   响应没有标识符区别响应式属于哪一个请求的。

那么请问HTTP/2.0是如何解决这个问题的？

> HTTP/2 solves this with stream identifiers.
>
> HTTP / 2使用流标识符来解决了这个问题。

### 为了更快的速度-来吧HTTP/2

#### HTTP/2 的改进

HTTP/2是有官网的, 我们可以在官网看到对HTTP/2的认知

- is binary, instead of textual

> 传输的是二进制数据，不再是文本。

- is fully multiplexed, instead of ordered and blocking

> 多路复用取代了顺序和阻塞。

- can therefore use one connection for parallelism

> 可以并行请求

- uses header compression to reduce overhead

> 压缩请求减少消耗

- allows servers to “push” responses proactively into client caches

> 其允许服务器在客户端缓存中填充数据，通过一个叫服务器推送的机制来提前请求。

#### is fully multiplexed 多路复用与并行请求

我们举一个小例子来说明HTTP 1.0 ==> HTTP1.1 ==> HTTP/2的变化， 为了说明这个变化，我们需要请出网购达人小明。

如果小明需要在一个淘宝店网购多个东西，在HTTP 1.0下, 不能放在一个订单里面(一个TCP连接里面)，只能是一个商品一个订单。

这也太浪费时间了吧，于是小明切换到了HTTP/1.1, 虽然不用开多个订单，但其实限制也比较大，也还是只能一次订购一个，虽然最后都被算在了一个订单中(一个TCP连接)。于是小明切换到了HTTP/1.1的流水线模式，在一个订单里买了10件商品，但是他们会按照请求顺序依次到达，如果某件商品缺货或者遗失，那么小明必须等待这件商品到货或者补发到货之后，才能收到下一个商品。这也太复杂了吧。

小明马上切换到了HTTP/2模式，在这个模式下，小明可以按照任意顺序购买商品，哪个商品准备好了，哪个商品就率先发货，所以他们的到达顺序和你的购买顺序可能是不一样的，为了加速，商家还可能拆分商品。这也就是HTTP/2的多路复用，一个连接上多个请求，不再像HTTP/1.1要求的那样，要求响应按请求顺序返回。

上面这个绝妙的比喻来于StackOverFlow下What does multiplexing mean in HTTP/2这个问题的回答，这个结果颇为简单易懂。

#### is binary, instead of textual 二进制的而不是文本的

上面的话也就是在说，HTTP/2之前使用的是文本的，但是最终传输的不都是二进制的吗？ 我们可以说二进制是令人困惑的术语，因为数据在计算机中最后都是二进制形式的。所以有什么区别呢？

为了解决我的问题，我决定使用Wireshark进行抓包，对比HTTP/1.1和HTTP/2的区别。但是我忽略了一点，目前基本上使用HTTP/2网站都是HTTPS，然后我在抓包工具中看到的就是TLS V 1.2这样的东西，这些都是密文。所以我又想用Spring Boot做一个支持HTTP/2的后台，然后自己抓包分析一下，然后发现Spring Boot 不支持HTTP/2的明文版本, 原来HTTP/2分成两个版本，一个是明文版，一个是基于加密的。但是目前浏览器和Spring Boot 只支持基于加密的版本，所以必须有加密证书，如果你禁用加密，通过浏览器进行访问的时候，浏览器会进行自动降级，将HTTP/2降级为HTTP/1.1。

所以通过自己搭建支持HTTP/2网站的想法报销了，因为比较麻烦。我只能将希望转到有支持HTTP/2明文的网站, 于是我找到了下面这篇文章:

- HTTP/2 协议抓包实战  https://zhuanlan.zhihu.com/p/415291905

里面给了一个HTTP/2的明文网站，遗憾的是在我去抓的时候，人家升级成HTTP/2密文了，同样的curl命令，他抓到请求中有请求升级协议的，我这里没抓到，我进到网站一看，发现人家已经是HTTPS了。所以思来想去，可能我们提出的问题本身就是有问题的，我们的问题是最终传输的时候都是二进制，所以提出的问题是有什么区别？我想下面的回答可以局部回答我们的问题:

> The first is that some people have suggested that the bottom layer of the HTTP protocol is still 0 and 1. How can it be a text protocol? The answer is whether a protocol is a text protocol or not, it has nothing to do with the upper and lower layers of the network model, but only with itself.
>
> 有人认为HTTP协议的底层仍然是0和1，那为什么HTTP就算是文本协议了呢？ 答案是一个协议是基于文本还是其他，不在于他的上层还是下层，仅仅在于它本身。

那你还是没回答HTTP/1.1交付给传输层的数据和HTTP/2交付给传输层的有什么区别啊？ 我想大概就是在编码和格式上了。这里简单的介绍一下文本编码和二进制编码的区别，他们都是表示数据的形式。对于数字来说，一个数字比如说用二进制协议，比如98765，这是个五位数，两个字节不大够，我们大致要上三个字节来表示。那如果用文本协议呢，那就是一个字符一个字节，五个数字也就是五个字节。这就节省了两个字节。第二个就是固定了格式，将原来的Header+Body变为数个分散的帧:

![报文演变](http://tva3.sinaimg.cn/large/006e5UvNly1h4gx3wgenej30rz0bi3z0.jpg)

#### 为什么是HTTP/2而不是HTTP 1.2

最初的HTTP/2被称为HTTP/2.0，HTTP/2工作组给出了解释，他们认为以前的1.0,1.1造成了很多混乱和误解，让人在实际的使用难以区分差异，所以将这次的版本号定位HTTP/2，弃用次要版本号“.0”。

## Java 领域对HTTP/2的支持

虽然现在HTTP/3已经出来了，但是Java领域对此支持的HTTP Client，就我熟悉的而言，还处于积极的适配HTTP/2中，Maven 仓库中的Apache HttpClient最新版本还处于4.5.2，这个目前还只是支持HTTP1.1版本。

## 写在最后

本篇的定位是简介，简单介绍了HTTP/2的主要特性，本身是想做抓包的，但是抓包的相关的解析又会加大大本篇的篇幅，所以考虑将报文解析移动后面的文章序列了。其实这篇文章，本身我预计打算一天完成的，后面查二进制协议和文本协议的区别，倒是让我花了不少时间，这到让我颇为意外。

## 参考资料

- HTTP/2 中的多路复用是什么意思 https://stackoverflow.com/questions/36517829/what-does-multiplexing-mean-in-http-2
- Transfer-Encoding:chunked详解  https://blog.csdn.net/qq_32331073/article/details/82148409
- How does HTTP2 solve Head of Line blocking (HOL) issue  https://stackoverflow.com/questions/45583861/how-does-http2-solve-head-of-line-blocking-hol-issue
- HTTP Head of line blocking: Why responses must come back in order? https://stackoverflow.com/questions/65597477/http-head-of-line-blocking-why-responses-must-come-back-in-order
- What is HTTP "Host" header?  https://stackoverflow.com/questions/43156023/what-is-http-host-header
- Why is it said that HTTP2 is a binary protocol?  https://stackoverflow.com/questions/58498116/why-is-it-said-that-http2-is-a-binary-protocol?noredirect=1&lq=1
-  How to enable http2 using spring boot and tomcat without SSL configuration   https://stackoverflow.com/questions/54454044/how-to-enable-http2-using-spring-boot-and-tomcat-without-ssl-configuration
- Binary vs Text  https://dev.to/cacilhas/binary-vs-text-2blo
- 透视 HTTP 协议  https://zq99299.github.io/note-book2/http-protocol/
- HTTP/2 官网 https://httpwg.org/specs/rfc9113.html#FramingLayer

