# 用Java的BIO和NIO、Netty实现HTTP服务器(六) 从问题中来学习Netty

[TOC]

## 前言

原本这一篇是打算放到《用Java的BIO和NIO、Netty实现HTTP服务器(四) 从问题中来学习Netty》里面的，但是想来放在一起跟四里面的内容整体连贯性不太强，索性就将这部分内容单独放出来一篇，本篇我们主要从《jdk17下netty导致堆内存疯涨原因排查 | 京东云技术团队》这篇文章的问题入手来学习Netty，首先对方给了一张图:

![](https://a.a2k6.com/gerald/i/2024/03/02/v5jy.jpg)

文章中对这个系统的描述是: “客户端和服务端底层通过netty直接进行tcp通信，且服务端也是基于netty将数据备份到对应的slave集群”。然后说要升级到JDK 17，然后用ZGC延迟相对低一点。回想一下我们在用《Java的BIO和NIO、Netty实现HTTP服务器(四)》里面讲的背压，是不是有一点感觉了？

这篇文章接着说: 采用jdk17+zgc经过相关的压测后，一切都在向着好的方向发展，但是在一种特殊场景压测，需要将数据从北京数据中心同步给宿迁数据中心的时候，发现了一些诡异的事情

1. 服务端容器的内存疯涨，并且停止压测后，内存只是非常缓慢的减少。

2. 相关机器cpu一直保存在20%（已经无流量请求）

3. 一直在次数不多的gc。大概每10s一次

##  开始分析问题

我们且不说最新的JDK 实现，我们目前只看JDK 17的ZGC特性，在JDK 17，ZGC倾向于等到堆填满的最后一刻再进行收集，目标是只在必要时运行GC，避免不必要的负载，尽早启动 GC，以便在堆真正填满之前完成（因为堆填满会导致应用程序暂时停滞）。要做到这一点，它需要测量应用程序分配内存的速度、GC 运行所需的时间，并预测应该在什么时候启动 GC。

ZGC的触发机制:  

- 预热规则: 服务刚启动时出现，一般不需要关注。日志中关键字是“Warmup”。 JVM启动预热，如果从来没有发生过GC，则在堆内存使用超过10%、20%、30%时，分别触发一次GC，以收集GC数据.

- 阻塞内存分配请求触发：当垃圾来不及回收，垃圾将堆占满时，会导致部分线程阻塞。我们应当避免出现这种触发方式。日志中关键字是“Allocation Stall”。

- 最主要的GC触发方式（默认方式），其算法原理可简单描述为”ZGC根据近期的对象分配速率以及GC时间，计算出当内存占用达到什么阈值时触发下一次GC”。通过ZAllocationSpikeTolerance参数控制阈值大小，该参数默认2，数值越大，越早的触发GC。日志中关键字是“Allocation Rate”。
- 基于固定时间间隔：通过ZCollectionInterval控制，适合应对突增流量场景。流量平稳变化时，自适应算法可能在堆使用率达到95%以上才触发GC。流量突增时，自适应算法触发的时机可能会过晚，导致部分线程阻塞。我们通过调整此参数解决流量突增场景的问题，比如定时活动、秒杀等场景。日志中关键字是“Timer”。
- 主动触发规则：类似于固定间隔规则，但时间间隔不固定，是ZGC自行算出来的时机，我们的服务因为已经加了基于固定时间间隔的触发机制，所以通过-ZProactive参数将该功能关闭，以免GC频繁，影响服务可用性。 日志中关键字是“Proactive”。
- 预热规则：服务刚启动时出现，一般不需要关注。日志中关键字是“Warmup”。
- 外部触发：代码中显式调用System.gc()触发。 日志中关键字是“System.gc()”。
- 元数据分配触发：元数据区不足时导致，一般不需要关注。 日志中关键字是“Metadata GC Threshold”。

理解ZGC日志: 

- 元数据分配触发

![](https://a.a2k6.com/gerald/i/2024/03/10/ki7c.jpg)

- 预热规则

![](https://a.a2k6.com/gerald/i/2024/03/10/kb3w.jpg)

- 最主要的GC触发方式（默认方式）

![](https://a.a2k6.com/gerald/i/2024/03/10/2xn6.jpg)

所以说上面说停止压测之后，内存很缓慢减少，那只说明里面有大量可达对象和强引用对象呗，我们在《JVM学习笔记(一) 初遇篇》已经简单做了讨论，况且根据ZGC的触发机制，我并不觉得这是奇怪的点，如果说奇怪那就奇怪在为什么会有这么多强引用对象。相关机器cpu一直保存在20%（已经无流量请求）, 难道是 -XX:ZCollectionInterval最小时间间隔在起作用？ 这篇文章没有给启动参数，我们无从得知。那么回到第三点，一直次数不多的GC, 每10s一次，那么也就是说在压测期间，频繁的快要将堆填满了，才会不断的触发gc，但是看回收情况，每次没有回收多少内存，那么会不会发生了内存泄漏呢，这篇文章也想到了此处，开始了内存泄漏排查，然后作者加上了JVM参数Dio.netty.leakDetection.level=PARANOID，进行排查，然后发现没有内存泄漏，接着怀疑到了JDK 身上，然后升级了JDK 也没解决问题。

终于怀疑到是自己代码身上了，作者指出:

- 发现回滚至jdk8的时候，对应宿迁中心的集群接受到的备份数据量比北京中心发送的数据量低了很多

- 为什么没有流量了还一直有gc，cpu高应该是gc造成的（当时认为是zgc的内存的一些特性）

- 内存分析:为什么netty的MpscUnboundedArrayQueue引用了大量的AbstractChannelHandlerContext$WriteTask对象，。MpscUnboundedArrayQueue是生产消费writeAndFlush任务队列，WriteTask是相关的writeAndFlush的任务对象，正是因为大量的WriteTask对象及其引用导致了内存占用过高。

只有跨数据中心出现该问题，同数据中心数据压测不会出现该问题 , 那也就是说跨数据中心延迟更高，数据量更大，本身消费能力不足导致的。这已经是定位到问题在哪里了，但是认为不是根本问题，于是作者接着分析问题, 也是在问为什么停止压测之后，内存只会缓慢减少，然后分析到了Netty分配内存的逻辑上，Netty通过PooledByteBufAllocator来分配内存，内存池机制，也就是里面的newHeapBuffer和newDirectBuffer方法，内存不足的话通过DirectArena的newChunk去预占申请内存，我们看下这个newChunk的逻辑:

```java
protected PoolChunk<ByteBuffer> newChunk(int pageSize, int maxPageIdx,
    int pageShifts, int chunkSize) {
    if (directMemoryCacheAlignment == 0) {      
        ByteBuffer memory = allocateDirect(chunkSize);
        return new PoolChunk<ByteBuffer>(this, memory, memory, pageSize, pageShifts,
                chunkSize, maxPageIdx);
    }
    final ByteBuffer base = allocateDirect(chunkSize + directMemoryCacheAlignment);
    final ByteBuffer memory = PlatformDependent.alignDirectBuffer(base, directMemoryCacheAlignment);
    return new PoolChunk<ByteBuffer>(this, base, memory, pageSize,
            pageShifts, chunkSize, maxPageIdx);
}
```

```java
private static ByteBuffer allocateDirect(int capacity) {
  return PlatformDependent.useDirectBufferNoCleaner() ? PlatformDependent.allocateDirectNoCleaner(capacity) : ByteBuffer.allocateDirect(capacity);
}
```

PlatformDependent.useDirectBufferNoCleaner() 返回的true和false是怎么控制的呢?   由USE_DIRECT_BUFFER_NO_CLEANER这个参数控制:

```java
public static boolean useDirectBufferNoCleaner() {
    return USE_DIRECT_BUFFER_NO_CLEANER;
}
```

那这个USE_DIRECT_BUFFER_NO_CLEANER通过谁来控制呢?  这个变量位于PlatformDependent中，通过静态代码块的初始化来控制，我在找这个变量的初始化的时候，还在找构造函数、get\set这样类似的函数去设置，但是忽略了静态代码块，这是以后看源码要值得注意的，静态代码块里面的逻辑是什么呢?

```java
long maxDirectMemory = SystemPropertyUtil.getLong("io.netty.maxDirectMemory", -1);
if (maxDirectMemory == 0 || !hasUnsafe() || !PlatformDependent0.hasDirectBufferNoCleanerConstructor()) {
    USE_DIRECT_BUFFER_NO_CLEANER = false;
    DIRECT_MEMORY_COUNTER = null;
} else {
    USE_DIRECT_BUFFER_NO_CLEANER = true;
    if (maxDirectMemory < 0) {
        maxDirectMemory = MAX_DIRECT_MEMORY;
        if (maxDirectMemory <= 0) {
            DIRECT_MEMORY_COUNTER = null;
        } else {
            DIRECT_MEMORY_COUNTER = new AtomicLong();
        }
    } else {
        DIRECT_MEMORY_COUNTER = new AtomicLong();
    }
}
```

io.netty.maxDirectMemory 这个参数用于控制堆外内存的最大使用量，在JDK 17之下没有特殊配置这个条件不满足，!hasUnsafe()这个条件也不满足，那么关键就在PlatformDependent0的hasDirectBufferNoCleanerConstructor这个方法上了:

```java
static boolean hasDirectBufferNoCleanerConstructor() {
    return DIRECT_BUFFER_CONSTRUCTOR != null;
}
```

在JDK 17下面拿到的也是null，所以最后分配逻辑就走到了ByteBuffer.allocateDirect(capacity):

```java
DirectByteBuffer(int cap) {                   // package-private
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    // 注意reserveMemory这个方法,里面会判断内存，如果不足会强制系统进行GC。
    // 然后在循环中等待MAX_SLEEPS看是否有直接内存。
    Bits.reserveMemory(size, cap);
    long base = 0;
    try {
        base = UNSAFE.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    UNSAFE.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

所以这里分析出来的根本原因是应用消费能力不足，导致申请最大直接内存之后，会同步等待在申请直接内存这里。注意这里已经注意到了消费不足，其实这里问题已经定位出来了，我们在《用Java的BIO和NIO、Netty实现HTTP服务器(五) 理解Netty的流水线》已经提到了背压:

> 在异步系统中处理,处理数据流,尤其是数据量未预先确定的实施数据要特别小心。最为突出而又常见的问题是资源消费控制的问题，以便防止大量数据快速到来淹没目的地。

很明显这就是下游被淹没了，根本原因就是没有做背压，Netty也预留了水位线，让开发者来判断消费积压，然后作者又开始分析为什么在JDK 8 下面没有这个问题，其实我感觉已经没有分析这个问题的必要了，然后再JDK 8下Netty分配内存走的是PlatformDependent.allocateDirectNoCleaner(capacity): 

```java
static ByteBuffer allocateDirectNoCleaner(int capacity) {
    return newDirectBuffer(UNSAFE.allocateMemory(Math.max(1, capacity)), capacity);
}
```

然后是用Unsafe来申请内存的，然后这里可以无限申请，没有内存检查。然后觉得自己定位到了问题，已经看到了根本原因，就是ByteBuffer.allocateDirect gc 同步等待直接内存释放导致消费能力严重不足导致的，并且在最大直接内存不足的情况下，大面积的消费阻塞耗时在申请直接内存，导致消费WriteTask能力接近于0，内存从而无法下降。所以解决方案是 netty在jdk高版本需要手动添加jvm参数 -add-opens=java.base/java.nio=ALL-UNNAMED和-io.netty.tryReflectionSetAccessible 来开启采用直接调用底层unsafe来申请内存，如果不开启那么netty申请内存采用ByteBuffer.allocateDirect来申请直接内存，如果EventLoop消费任务申请的直接内存达到最大直接内存场景，那么就会导致有大量的任务消费都会同步去等待申请直接内存上。并且如果没有释放足够的直接内存，那么就会成为大面积的消费阻塞，也同时导致大量的对象累积在netty的无界队列MpscUnboundedArrayQueue中。

所以就是加内存，无限内存申请，前面不都说了是内存消费能力不足了嘛，后面也说了没有加水位线判断，你都要开无限内存申请了，还判断这个干什么呢？ 我觉得根本原因还是在于没有做背压。

## 参考资料

[1]  Make ZGC run often https://stackoverflow.com/questions/68322932/make-zgc-run-often

[2] 新一代垃圾收集器：ZGC深度剖析，到底什么时候用？ https://juejin.cn/post/7309732396080906240#heading-19

[3] 新一代垃圾回收器ZGC的探索与实践 https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html
