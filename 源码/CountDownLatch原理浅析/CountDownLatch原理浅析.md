# CountDownLatch原理浅析

[TOC]

> 本文的副标题为由CountDownLatch到AQS

## 前言

我日常用到CountDownLatch的场景还是比较多，定时任务使用多线程去采集数据，然后采集完数据之后，做进一步的处理。一般我在程序里面会如下写:

```java
public class CountDownLatchDemo {

    // 避免指令重排序
    private volatile  ThreadPoolExecutor threadPoolExecutor;


    public void countDownLatchDemo(){
        // 一共五十个任务
        CountDownLatch countDownLatch = new CountDownLatch(50);
        for (int i = 0; i < 50; i++) {
            getThreadPoolExecutor().execute(()->{
                // 线程做完对应的任务,任务数减一
                countDownLatch.countDown();
            });
        }
        try {
            // main 线程走到这里,如果50个任务没完成,就会被阻塞在这这里。
            // hello world 不会输出
            countDownLatch.await();
            System.out.println("hello world");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
   		 // 接着做对应的业务处理
    }

    private ThreadPoolExecutor getThreadPoolExecutor() {
        // 获取当前系统的逻辑核心数,我的是八核十六线程。
        int cpuNumbers = Runtime.getRuntime().availableProcessors();
        if (Objects.isNull(threadPoolExecutor)) {
            synchronized (CountDownLatchDemo.class) {
                // 为线程池起名,便于定位问题
                // CustomizableThreadFactory 来自于Spring
                ThreadFactory threadFactory = new CustomizableThreadFactory("fetch-data-pool-");
                threadPoolExecutor = new ThreadPoolExecutor(cpuNumbers, cpuNumbers, 1L, TimeUnit.HOURS, new LinkedBlockingDeque<>(), threadFactory);
                return threadPoolExecutor;
            }
        }
        return threadPoolExecutor;
    }

    public static void main(String[] args) {
        CountDownLatchDemo countDownLatchDemo = new CountDownLatchDemo();
        countDownLatchDemo.countDownLatchDemo();
    }
}
```

所以不禁好奇CountDownLatch是如何实现的, 这也就是本篇的由来，我主要关心的问题主要有两个:

- 调用await方法的线程在任务完成之后，如何唤醒。
- 究竟是如何控制任务完成的，也就是countDown方法做了些什么

本篇的源码基于JDK8(高版本的JDK对AbstractQueuedSynchronizer有些调整，后面也会针对JDK 17的AbstractQueuedSynchronizer出一篇源码相关的文章，重点对比高版本JDK的优化). 我们首先打开CountDownLatch进行查看:

![CountDownLatch一览](http://tva2.sinaimg.cn/large/006e5UvNgy1h4i1n6t9tcj30q80kjac9.jpg)

我们看到调用CountDownLatch的构造函数初始化的时候，事实上间接的对CountDownLatch的静态内部类Sync进行了初始化。我们调CountDownLatch的countDown方法也是间接的调用Sync的releaseShared()方法. 这两个方法在Sync里面没有，那说明继承自AbstractQueuedSynchronizer里。这个类的作者仍然是Doug Lea，注释比较多，我还是喜欢注释多的源码。之前的看源码，我先看上面的注释，再看类的基本结构。这次我们调整下顺序，先大致看一下这个类的基本结构再看类上面的注释。

打开AbstractQueuedSynchronizer往下翻，我们首先看到的一个一眼能看懂的数据结构恐怕就是Node类了，Node是双向链表的节点, 似乎每一个节点主要数据是线程。

![JDK8中的AbstractQueuedSynchronizer](http://tvax2.sinaimg.cn/large/006e5UvNly1h4i4x7fv4zj30jr0fstb0.jpg)

![Node队列](http://tva1.sinaimg.cn/large/006e5UvNly1h4i51dqwwgj30yo04jmx8.jpg)

剩下的好像都围绕这个链表进行操作，本节我们关心的问题在于CountDownLatch的实现原理，然后才是AbstractQueuedSynchronizer。我们重点关注CountDownLatch中countDown和await的实现。

countDown方法的调用如下:

```java
// 这个sync事实上是CountDownLatch中静态内部类Sync的实例
public void countDown() {
  	//releaseShared调用的是父类AbstractQueuedSynchronizer的releaseShared方法
   sync.releaseShared(1);
 }
// tryReleaseShared为CountDownLatch中静态内部类Sync重写父类的方法
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
 }
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
		
        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            // 减少count
            for (;;) {
                // 我们看getState和setState方法,getState和setState是父类的方法
                // 获取state
                int c = getState();
  				// 如果等于0 返回false
                if (c == 0)
                    return false;
                // 否则就将state 变量 减一
                int nextc = c - 1;     
                // compareAndSetState调用的是AbstractQueuedSynchronizer的方法
                // 默认调用的是CAS,给AQS的state变量设置值。
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
 
//下面是AbstractQueuedSynchronizer的compareAndSetState
// 这里的stateOffset用可以理解为字段state的偏移地址,通过this+stateOffset可以定位到state变量的内存地址
 protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
 }
```

### 捋一下countDown方法究竟做了什么

![CountDownLatch初始化](http://tvax3.sinaimg.cn/large/006e5UvNly1h4osvge9drj310c04pt92.jpg)

我们使用CountDownLatch的时候是首先调用其构造函数进行初始化, 这会调用CountDownLatch的静态内部类Sync进行初始化, 传进来的任务数最终给了继承来自AbstractQueuedSynchronizerd的setState方法。

当线程调用countDown方法完成任务的时候，调用链如下：

![CountDown调用链](http://tva4.sinaimg.cn/large/006e5UvNly1h4oxu3qruyj30xe0g7ta4.jpg)



 注意tryReleaseShared虽然来自于AbstractQueued，但这个方法是空方法，留给子类重写，所以我们又需要回到CountDownLatch的Sync中。

![aqs-sync](http://tva1.sinaimg.cn/large/006e5UvNgy1h4ot2jwyhvj30y20bijsh.jpg)

tryReleaseShared代表当前完成任务，需要将总的任务数减一，如果任务数已经清0, 接着调用doReleaseShared方法。doReleaseShared方法需要配合CountDownLatch的await方法来看。到现在我们已经得到了第一个问题的答案:  划分的任务数通过CAS更新任务数，事实上在AQS里面我们可以称之为同步状态。

### await方法浅析

![AQS-await](http://tva3.sinaimg.cn/large/006e5UvNly1h4oxp7ep3aj30yw0hywg2.jpg)

我们调用CountDownLatch方法的await方法事实上调用的是CountDownLatch内部类的Sync的acquireSharedInterruptibly, 这个acquireSharedInterruptibly来自AbstractQueuedSynchronizer，tryAcquireShared在AbstractQueuedSynchronizer是一个空方法，留给子类重写，在CountDownLatch的实现是，如果state == 0，返回 1，我们重点看任务未完成的时候的doAcquireSharedInterruptibly方法，doAcquireSharedInterruptibly中又首先调用了addWaiter，所以我们接着看addWaiter方法，addWaiter，从方法名上来推上，这是添加一个等待者。

```java
/**
* 为当前线程创建一个排队结点,并且给一个模式
* Creates and enqueues node for current thread and given mode.
* mode 有两种类型, 一种是独占模式,一种是共享模式。有没有想到独占锁和共享锁呢
* @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
* @return the new node
*/
private Node addWaiter(Node mode) {
    	// 当前结点的下一个结点指向传入的结点
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
    	// 这个操作相当于设置尾结点
       // 首次添加为尾结点为null
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
    	// 走到这里说明 当前队列是空队列,直接添加结点
        enq(node);
        return node;
}

/* 
* 将当前结点插入到队列中,执行初始化。上面有图。突然觉得这里挺有意思的。
* 下面用图片演示一下这个过程
* Inserts node into queue, initializing if necessary. See picture above.
* @param node the node to insert
* @return node's predecessor // 返回插入结点的前结点
*/
private Node enq(final Node node) {
        for (;;) {
            // 获得tail的应用
            Node t = tail;
            // 如果tail 为空
            if (t == null) { // Must initialize
                // 设置头结点,此时头结点又是尾结点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 如果尾结点非空代表此时队列非空,尾结点成为该结点的前驱结点
                node.prev = t;
                // 将当前结点设置尾结点
                if (compareAndSetTail(t, node)) {
                    // 将尾结点的后继结点指向插入结点
                    t.next = node;
                    return t;
                }
            }
        }
 }
```

![New Node](http://tva2.sinaimg.cn/large/006e5UvNly1h4p5ylmqsoj312i0c4aaq.jpg)

   ![队列补充](http://tvax2.sinaimg.cn/large/006e5UvNly1h4p66g2vr2j30sj0jbaaw.jpg)



独占锁如果你感到陌生的话，我们换一个译名排他锁,  是指一个锁一次只能被一个线程所持有。如果线程A对数据A加上排他锁的话，其他线程不能再对A加任何类型的锁，获得排他锁的线程既能读数据又能修改数据，JDK中的synchronized和Lock的实现类就是排他锁, 所以ReentrantLock也借助AbstractQueuedSynronizer来实现。共享锁是指该锁可以被多个线程所持有，如果线程T对线程数据A加上共享锁后，其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

ReentranLock有一个内部类叫Sync，Sync有两个子类:FairSync,NonfairSync. Fair是公平的意思，如果你对ReentranLock比较熟悉的话, 会联想到ReetranLock的公平锁与非公平锁。

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    	// 创建一个结点,并返回该结点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                // 获取前驱结点
                final Node p = node.predecessor();
                // 如果是头结点
                if (p == head) {
                    // 获取state状态
                    // 大于零任务完成
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {                     
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
}
```

setHeadAndPropagate、shouldParkAfterFailedAcquire、parkAndCheckInterrupt这三个方法互有联系我们单独拎出来说，根据方法见名知义原则，我们来认单词, Propagate有扩散的意思也就是，我们讲CountDownLatch是线程协作利器，我们在上面讲的场景上某个线程需要启用多线程并发处理数据，后续线程里面的处理逻辑需要并发处理数据完毕之后，才进行下一阶段的处理，那这里我们可以推测调用CountDownLatch方法的await方法时由于任务没完成而陷入阻塞的线程，就是由这个方法进行唤醒的，目前队列中只有一个线程，属于比较简单的状况。现在这种情况时一个线程等多个线程，对外表现的形式是只有一个线程调用await方法。CountDownLatch的另一种使用场景是多等一，即多个线程调用await方法，一个线程调用countDown完成任务，要将处于等待状态的线程唤醒。

注意上面是一个无限循环，也就是线程调用CountDownLatch的await方法的时候并没有来自Object的wait、Condition队列来让当前线程陷入等待，而是不断的去获取state变量。Park有停车的意思，所以shouldParkAfterFailedAcquire是否是在获取状态失败之后，应当先休息一下？ shouldParkAfterFailedAcquire有涉及到结点状态的应用, 我们先来介绍状态，再来看shouldParkAfterFailedAcquire的源码.

waitStatus是一个整型变量,一共有以下几个候选值：

- 0  Node被初始化的时候默认值
- CANCELLED = 1, 

> 参考资料中的美团技术团队的文章《从ReentrantLock的实现看AQS的原理及应用》将该状态解释为:
>
> 表示线程获取锁的请求已经取消了。 我看了感觉有点理解不动, 这篇文章是从ReentrantLock入手的，但是我并不理解为什么获取锁的请求会被取消。
>
> This node is cancelled due to timeout or interrupt.  Nodes never leave this state. In particular,  a thread with cancelled node never again blocks.
>
> 由于超时或者该线程被打断，这个结点被取消，这个状态将会被固定，处于取消状态的线程结点不会再被阻塞。

- SIGNAL =  -1 

> 美团技术团队《从ReentrantLock的实现看AQS的原理及应用》将该状态解释为:  表示当前线程已经准备好了, 就等资源释放了。
>
> 《java并发编程系列：牛逼的AQS（上）》(文末放的有参考链接)：表示后边的节点对应的线程处于等待状态。
>
> 但两种解释是互补的，结合起来理解就是: 表示当前线程已经准备好了，就等资源释放了。同时后边的节点对应的线程处于等待状态。
>
> 这里将关于这个结点的相关注释列以下:
>
> waitStatus value to indicate successor's thread needs unparking 
>
> 后继结点线程需要解除等待状态。(Parking 对应的中文译为：泊车, 这里是将线程当作一辆车？ 队列是个停车场？ )
>
> The successor of this node is (or will soon be)   blocked (via park), so the current node must  unpark its successor when it releases or  cancels. To avoid races, acquire methods must  first indicate they need a signal, then retry the atomic acquire, and then,on failure, block.
>
> 当前结点的后继结点处于阻塞或者将要被阻塞(凭借park，难以翻译，只好中英混)。所以当前结点释放或者被取消一定要解除后继结点的阻塞状态. 为了避免竞争，acquire的相关方法一定是先从signal判断，再是原子重试获取，然后失败、阻塞。

- CONDITION = -2

> 表示节点在等待队列中，节点线程等待唤醒。

- PROPAGATE = -3

> 当前线程处在SHARED情况下，该字段才会使用，下一次的同步状态将会被无条件的传播下去。（这句话我们后面会讲，当前我们的目标是从CountDownLatch到AQS，来获得对AQS有一个宏观的理解）

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
      	// 获取前驱结点的状态
    	int ws = pred.waitStatus;
    	// 前继结点处于唤醒,当前结点可以被安全的被阻塞
    	if (ws == Node.SIGNAL)
            return true;
     	// 通过看注释可以看明白 ws > 0，说明当前结点pred处于取消状态。
    	// 将该结点pred从队列里面移除,接着向前遍历
        if (ws > 0) {         
            do {
               node.prev = pred = pred.prev;
                
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            // 将pred,也就是当前结点的前一个结点设置为-1.
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
 }
```

```java
private final boolean parkAndCheckInterrupt() {
     	// 用的是UnSafe,所以这里我们暂时不用关心它的实现原理，只需要知道线程调用这个方法
    	// 相当于请求操作系统阻塞线程
        LockSupport.park(this);
    	// 如果当前线程已经被中断,且满足shouldParkAfterFailedAcquire
    	// 抛出异常。中断了就没有被阻塞这个概念。
        return Thread.interrupted();
}
```

这里我们已经大致明白了，线程在调用CountDownLatch方法的await方法，事实上会进入到AQS的队列中，如果当前线程的前驱结点处于唤醒状态，那么就先将该线程阻塞一段时间。既然这里线程陷入了阻塞，那么任务完成的时候是如何通知这里的线程的呢？ 我们接着看doReleaseShared方法:

```java
private void doReleaseShared() {
    	// 无限循环
        for (;;) {
            // 获取头结点
            Node h = head;         
            if (h != null && h != tail) {
                // 获取头结点的状态
                int ws = h.waitStatus;
                // 如果当前结点处于唤醒状态
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    // 唤醒其后继结点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

### CountDownLatch的思路

现在我们可以捋一捋CountDownLatch的思路了，CountDownLatch借助于AbstractQueuedSynchronizer来实现， AbstractQueuedSynchronizer里面维护了一个state变量，调用countDown方法的时候调用的是CountDownLatch内部类的tryReleaseShared方法来对state进行递减。调用CountDownLatch事实是间接调用AQS的acquireSharedInterruptibly方法，acquireSharedInterruptibly方法会构造出一个队列，如果任务没完成，会调用LockSupport来让调用await方法的线程陷入阻塞。任务完成的时候，最终通过doReleaseShared，唤醒陷入阻塞的线程。

## 由CountDownLatch到AQS

上面我们已经基本引出了AQS，但是并没有很深入，有些方法我们没有细讲，只是当作一个黑盒，在探究CountDownLatch的过程中，我们发现目前JDK工具里面的大多数线程同步工具类，像ReentrantLock、ReentrantReadWriteLock、Semaphore、TheadPoolExecutor里面都有AbstractQueuedSynchronizer的出现这是一个强大的线程同步框架，本篇的思路是由CountDownLatch到AQS，从理解CountDownLatch的设计原理，然后再简单的引出AQS，为后面整体介绍AbstractQueuedSynchronizer做铺垫，重点体会思想。

## 参考资料

- Java魔法类：Unsafe应用解析  https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html

- CountDownLatch源码解析 https://qingtianblog.com/archives/countdownlatch%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90

- 大白话聊聊Java并发面试问题之谈谈你对AQS的理解？【石杉的架构笔记】  https://juejin.cn/post/6844903732061159437
- 不可不说的Java“锁”事  https://tech.meituan.com/2018/11/15/java-lock.html
- java并发编程系列：牛逼的AQS（上） https://juejin.cn/post/6844903839062032398#heading-0 
- 从ReentrantLock的实现看AQS的原理及应用  https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html 美团技术团队
