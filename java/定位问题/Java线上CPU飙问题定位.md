# 如何定位线上Java应用CPU飙高

[TOC]



## 理清概念

对这个问题，感觉自己认知的还是不够清晰，于是今天就来理清楚思路，说到线上CPU飙高，说起这个问题的时候，我想到了什么呢? 我想起了处理器调度，我想起Linux CFS调度，CFS是 Completely Fair  Scheduler，也就是完全公平调度，也就是说它完全实现了所谓的“完全公平”调度算法，将CPU资源均匀的分配给各个进程(在内核中被称为任务，（Linux采用轻量级进程实现线程，所以这里说的线程可以认为线程的同义语)， 粗略的说，如果一台计算机有一个CPU，多个计算密集型进程，那么采用CFS调度器时，每个进程占用的时间为:

![](https://a.a2k6.com/gerald/i/2024/01/13/8hp.png)

我记得曾经我的某位老师告诉我，线程是去抢CPU的，但其实这是有点本末倒置，事实上是CPU从就绪队列里面选取任务进行执行，现代计算机基本上都是多核处理器，Linux默认的CFS调度采用的是多队列，也就是说应用程序提交的任务，由操作系统决定放入到哪个队列:

![](https://a.a2k6.com/gerald/i/2024/01/13/3i84f.jpg)

所谓抢占式调度不是线程去抢CPU，而是线程的时间片消耗尽，被CPU调度器强制打断，与协作式相相对，协作式调度则依靠被调度方主动弃权。到现在我们知道CPU是多核心的，每个核心都在不断的执行就绪队列里面的任务，如果就绪队列里面有任务的话，在这个角度上来说，多个核心可以同时执行任务，也就是并行，与并发相对，某个核心在一个时间点只能执行一个任务，如果我们将时间拉长，多个任务可以在这段时间都获得了执行，交错执行。也就是说一台计算机现在有多个核心，那么使用率应当是多个核心的累加，所以如果是八核十六线程，可以看到使用率最高达到1600%，这意味着每个核心都跑满了。等等你说到了八核十六线程，这是什么意思，不是一个核心跑一个任务嘛，怎么这一平均，就变成一个核心跑两个线程了, 我的电脑就是八核十六线程，在我的电脑上执行下面的代码输出就是16:

```java
System.out.println(Runtime.getRuntime().availableProcessors());
```

![](https://a.a2k6.com/gerald/i/2024/01/13/2rlp.jpg)

 那这是什么意思呢?  这其实要扯上超线程这种技术，通常情况下，一个核心可以在同一时间执行一个线程，当激活并支持超线程技术时，核心可以在同一时间执行两个线程。那么当你用top在Linux查看CPU使用率的时候事实上是多个核心使用率的累加, 所以如果你的如果你的逻辑处理器是四个核心，每个核心的使用率为百分之百二十，用top指令看总的使用率也就是百分之八十，那如何看每颗核心的使用率呢:

```java
#每秒统计一下CPU的使用率
mpstat -P ALL 3
# top 指令之后 摁1就行    
```

## 该如何理解CPU飙高？

我思考这个问题的时候，也是从CPU利用率这个角度来入手的，只不过理解CPU使用率的计算公式为，假设某核心的能力为100条指令，在某段时间内执行了90条指令，然后利用率就是百分之九十，但事实上的利用率的公式为1-空闲时间/CPU时间，这个公式暗含的意思是在一段时间内，CPU在不断被使用，这里我们没有用分类，分成几种情况，因为一种本质可以导出很多种不同的情况。这样讲可能有些抽象，我们看下实际的例子。

### 例子1: Nacos 出现大量线程创建的问题排查

Nacos做为服务中心和配置中心为大家所熟知，我从Nacos的Github上选取了一些例子来说明情况，其中一个例子是issue5764，标题为: “当项目用了多个数据源并配置spring.cloud.nacos.config.refreshable-dataids 时线程数不停飙升 ”:

![](https://a.a2k6.com/gerald/i/2024/01/13/42rn4.jpg)

这个issue也很贴心说了问题之后，还说了自己是怎么排查的，他真的，我哭死，这也是我们后面要介绍的工具。一个CPU高占用背后可能是多个问题，这个issue很有参考价值，我决定盘一盘，首先issue里面提到issue1605 ,我们顺着链接看一看, Issue1605的标题也很朴素:

![](https://a.a2k6.com/gerald/i/2024/01/13/l2et.jpg)

下面贴心的给了线程的堆栈，还有点美中不足的是这位兄弟上传的图裂了，但是没关系，他把堆栈信息也粘贴上来了, 堆栈信息比较长，这里我挑一些比较核心的，来说明一下问题:

![](https://a.a2k6.com/gerald/i/2024/01/13/sxrhb.jpg)

Runnable说明当前线程在执行或者在等待资源，例如I/O，或者处理器等。我们接着看堆栈信息:

![](https://a.a2k6.com/gerald/i/2024/01/13/46q8c.jpg)

下面还有不同的线程池，好像看不出来太多有价值的信息，但是柳暗花明下面放了一个分析的链接:

![](https://a.a2k6.com/gerald/i/2024/01/13/li76.jpg)

真的好贴心，我们顺着链接进去看:

![](https://a.a2k6.com/gerald/i/2024/01/13/lwsi.jpg)

![](https://a.a2k6.com/gerald/i/2024/01/13/34ep.jpg)

最后分析的结论是用户大量创建了NacosConfigService，链接中的图片已经裂了，但是没关系文章说了采样发现里面有大量的ClientWorker对象，有两千多个，然后一个对应这么多线程，而且都有任务在身，想来CPU高占用也不冤枉。

### Nacos Issue 859

我们接着看那个帖子里面的issue:

![](https://a.a2k6.com/gerald/i/2024/01/13/367n.jpg)

这个jmap我们在《JVM学习笔记(一) 初遇篇》里面已经让它漏过头了，可以用这个命令来导出dump文件，用来分析内存使用问题。现在还没有有价值的信息，我们接着往下翻:

![](https://a.a2k6.com/gerald/i/2024/01/13/4fydl.jpg)

Spring Cloud在配置刷新时触发Context的refresh操作，这里会导致nacos-client实例被重复创建。这个Nacos-Client实例也就是上面说的ClientWorker，也是一样的问题，大量线程被创建。

### Nacos Issue 5764

![](https://a.a2k6.com/gerald/i/2024/01/13/oo7o.jpg)

这位大哥说刷新配置导致Controller被销毁，销毁之后又要重建，我们看下这个大哥的分析之路，这位大哥首先还是用了jsstack去获取线程上下文信息:

```java
Thread 15906: (state = BLOCKED)
 - sun.misc.Unsafe.park(boolean, long) @bci=0 (Compiled frame; information may be imprecise)
 - java.util.concurrent.locks.LockSupport.park(java.lang.Object) @bci=14, line=175 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt() @bci=1, line=836 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireShared(int) @bci=83, line=967 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireShared(int) @bci=10, line=1283 (Compiled frame)
 - java.util.concurrent.locks.ReentrantReadWriteLock$ReadLock.lock() @bci=5, line=727 (Compiled frame)
 - org.springframework.cloud.context.scope.GenericScope$LockedScopedProxyFactoryBean.invoke(org.aopalliance.intercept.MethodInvocation) @bci=132, line=494 (Compiled frame)
 - org.springframework.aop.framework.ReflectiveMethodInvocation.proceed() @bci=120, line=186 (Compiled frame)
 - org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed() @bci=1, line=747 (Compiled frame)
 - org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(java.lang.Object, java.lang.reflect.Method, java.lang.Object[], org.springframework.cglib.proxy.MethodProxy) @bci=133, line=689 (Compiled frame)
```

这位大哥说，大量线程阻塞在这个接口上面，但是我们阻塞状态是等待获取锁，是不占用CPU的，那么看线程堆栈是阻塞在ReentrantReadWriteLock的lock方法，阻塞意味着此时有线程在写，所以读线程在不能读，读线程在lock的时候做了些什么呢? ReentrantReadWriteLock基于AQS:

```java
/**
 * Acquires the read lock.
 *  尝试获取读锁
 * <p>Acquires the read lock if the write lock is not held by
 * another thread and returns immediately.
 * 如果写锁没被另一个持有立刻返回
 * <p>If the write lock is held by another thread then
 * the current thread becomes disabled for thread scheduling
 * purposes and lies dormant until the read lock has been acquired.
 * 如果写锁被另一个线程所持有，当前线程将会被调度器禁用，直到获取读锁
 */
public void lock() {
    sync.acquireShared(1);
}
```

```java
public final void acquireShared(int arg) 0{
    if (tryAcquireShared(arg) < 0)      
        // 看上面的调用显然走到了这里
        doAcquireShared(arg);
}
```

这两个方法都有无限循环，所以也是有大量线程在不断使用CPU。

###  例子3:  GC 导致CPU高占用

上面的例子都是大量线程在不断使用CPU，粗略的说，发生GC动作的时候要STW，这个时候就算是只有聊聊几个线程也可以让CPU飙高，在搜索引擎上搜索FullGC  CPU，这样的分析文章并不少见:

![](https://a.a2k6.com/gerald/i/2024/01/13/50b5f.jpg)



## 那么如何定位问题?



### 有请Java Flight Record 

首先介绍Java Flight Record，中文名为飞行记录，暗含的意思是可以伴随Java应用启动，是 JVM 内置的基于事件的JDK监控记录框架。那为啥不先介绍jsstack呢，原因在于当你的服务器CPU告警的时候，可能已经影响用户使用了，为了不影响用户的使用，一种有效的选择是先停掉这个应用，再重新启动，导致现场信息丢失。另一种情况是复现条件比较复杂，而Java Flight Record是一直在记录的，可以获取现场信息。而且对性能的影响非常小。

#### 先介绍JDK 8中怎么用





#### 然后是JDK 11



### 有请async-profiler





### 有请jstack



### 有请Arthas 





## 参考资料

[1] 《Linux CFS 调度器：原理、设计与内核实现（2023）》  https://arthurchiao.art/blog/linux-cfs-design-and-implementation-zh/

[2] 《操作系统原理-处理器调度》 https://liuyehcf.github.io/2017/09/25/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86-%E5%A4%84%E7%90%86%E5%99%A8%E8%B0%83%E5%BA%A6/

[3] Why is the "top" command showing a CPU usage of 799%?  https://superuser.com/questions/457624/why-is-the-top-command-showing-a-cpu-usage-of-799/457634#457634

[4] Nacos 出现大量线程创建的问题排查  https://www.liaochuntao.cn/2019/09/04/java-web-53/

[5] 深度探索JFR - JFR详细介绍与生产问题定位落地 - 1. JFR说明与启动配置  https://zhuanlan.zhihu.com/p/122247741

[6] How to use JDK Flight Recorder (JFR)?  https://access.redhat.com/solutions/662203 