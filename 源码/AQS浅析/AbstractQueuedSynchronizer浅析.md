

# AQS浅析

> 衔接上一篇文章《CountDownLatch原理浅析》，在《CountDownLatch原理浅析》中我们已经和AbstractQueuedSynchronizer(后面我们)碰过面了，但是这种碰面只是属于“只是因为人群中多看了一眼”，本篇我们就属于和AQS相亲了。建议在看本篇之前，先看一下《CountDownLatch原理浅析》。

[TOC]

## 前言

其实在《CountDownLatch》中我们已经大致的介绍了AbstractQueuedSynchronizer了，我们这里再复习一下：

> 我们用CountDownLatch来实现线程协作，如果我们有一个任务，单线程处理比较耗时，我们将任务切割粒度，分割成若干个任务。
>
> 主处理逻辑等待上面的任务被多个线程执行完成再接着往下处理。

![CountDownLatch的使用场景](http://tva1.sinaimg.cn/large/006e5UvNgy1h51mlmovudj30tn0fz757.jpg)

其实这里就引出了AQS的定位:  

> Provides a framework for implementing blocking locks and related  synchronizers (semaphores, events, etc) that rely on  first-in-first-out (FIFO) wait queues。
>
> 提供了一种实现阻塞锁和同步工具的框架，这依赖于一个先进先出的等待队列。

我们来品一品这句话的实现阻塞锁和同步工具的框架，先来看阻塞锁，在没有JDK 1.5之前，Java的同步锁就只有一个：synchronized关键字。要实现线程协作，只能借助于notify、notifyAll、wait。 可能有的同学已经忘记了, 这三个Object中的方法是什么作用。 我们这里借助一个例子来复习一下：

```java
public class RomaticDateThreadDemo implements Runnable {
    // 画妆标志位
    private boolean flag;

    private static final String THREAD_GIRLFRIEND = "女朋友";
    
    public RomaticDateThreadDemo(boolean flag) {
        this.flag = flag;
    }
    @Override
    public void run() {
        synchronized (this) {
            // RomaticDateThreadDemo 的构造函数给的是false
            if (!flag) {
                flag = true;
                String currentThreadName = Thread.currentThread().getName();
                if (THREAD_GIRLFRIEND.equals(currentThreadName)) {
                    // 输出这句话说明女朋友线程先进来
                    System.out.println("你死定了,敢让你女朋友等");
                } else {
                    // 假设男孩子线程先进来
                    System.out.println("...........女朋友正在化妆中.................,请等一会儿");
                    try {
                        //等待女朋友唤醒,wait代表释放对象锁(释放许可证)
                        // 这个线程会放到队列里面
                        this.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            } else {
                String currentThreadName = Thread.currentThread().getName();
                if (THREAD_GIRLFRIEND.equals(currentThreadName)) {
                    // 走到这里,说明男孩子线程率先被线程调度器选中
                    System.out.println("..........要画十秒的妆.............");
                    try {
                        TimeUnit.SECONDS.sleep(5);
                        // 唤醒队列里面的线程
                        this.notify();
                        System.out.println(currentThreadName + "说:我们走吧");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else {
                    // 走到这里,说明女朋友线程率先被线程调度器选中
                    System.out.println(currentThreadName + "说:我们走吧");
                    this.notify();
                }
            }
        }
    }
}
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        RomaticDateThreadDemo romaticDateThreadDemo = new RomaticDateThreadDemo(false);
        Thread you = new Thread(romaticDateThreadDemo);
        you.setName("你");
        romaticDateThreadDemo.wait();
        Thread girlFriend = new Thread(romaticDateThreadDemo);
        girlFriend.setName("女朋友");
        you.start();
        girlFriend.start();
    }
}
```

在这个例子中，当前的锁在this也就是当前对象，其内部实现是JVM会为每个对象维护一个入口集(Entry Set) 用于存储该对象内部锁的线程。此外，JVM还会为每个对象维护一个被称为等待集(Wait Set)的队列，该队列用于存储该对象上的等待线程，Object.wait() 将当前线程暂停并释放相应的内部锁的同时会将当前线程的引用存入该方法所属对象的等待集中。 

执行一个对象的notify方法会使该对象的等待集中的一个任意线程被唤醒。被唤醒的线程仍然会停留在响应对象的等待集中，直到该线程再次持有相应内部锁的时候(此时Object.wait调用尚未返回，这里补充介绍一下，可能有的同学不大理解，这里解释一下在Java里面我们一般的理解是，我调用一个方法，一般都是执行完了才返回，这个Object.wait，可能有执行半截的感觉)。Object.wait会使当前线程从其所在的等待集中移出，接着Object.wait方法才执行完毕。



![锁队列](http://tvax2.sinaimg.cn/large/006e5UvNgy1h51rhrmp69j30no0fgjs1.jpg)



Object.wait/notify()实现的等待/通知中的几个关键动作，包括将当前线程加入等待集、暂停当前线程、释放锁以及将唤醒后的等待线程从等待集中移除等，都是在Object.wait()中实现的。Object.wait的部分内部实现相当于如下伪代码:

```java
public void wait(){
// 执行线程必须持有当前对象对应的内部锁 
if(!Thread.holdsLock(this)){
    throws new IllegalMonitorStateException();
}	
if(当前对象不在等待集中){
 // 将当前线程加入当前对象的等待集中
 addWaitSet(Thread.currentThread());
}
// 原子操作开始    
atomic{
 // 释放当前对象的内部锁
 releaseLock(this);
 block(Thread.currentThread());// 语句一   
}
acquireLock(this); // 语句二
// 将当前线程从wait set中移除  
removeFromWaitSet(Thread.currentThread());      
}
```

在Java中线程调用Object.wait方法时, 执行到语句一的位置被暂停，当被唤醒再次持有内部锁，wait方法接着从语句一往下执行。

但是这种通知唤醒有着相当大的缺陷，就是wait 、notity、notifyAll 只能在被synchronized修饰的代码块、方法中使用，不然就会抛出IllegalMonitorStateException。这种唤醒与等待极大的限制了Java中线程协作，原因在于多线程有的时候并不是互相竞争资源，只是在协作，因为无法确定线程完成的先后顺序，但是我们的处理逻辑是有先后顺序的，但是为了实现线程等待，我们就不得不使用synchroized。再有Object.notify唤醒的是任意一个线程，这样就可能导致出现线程饥饿，即部分线程在这种任意唤醒策略下，可能被唤醒的次数偏低，我们希望同步机制提供的锁尽可能公平一些，这也就是公平锁，公平的唤醒，多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁，公平锁的优点是等待锁的线程不会被饿死，总体来说公平锁适用于线程持有锁的时间比较长的场景。有了对synchronized讨论，我们就对AQS的 “Provides a framework for implementing blocking locks and related  synchronizers (semaphores, events, etc) that rely on  first-in-first-out (FIFO) wait queues。这依赖于一个先进先出的等待队列。 ”, 这句话有一个大致的理解了，即AQS 提供了一种同步的框架，用来实现同步锁和线程协作，AbstractQueuedSynchronizer是一个抽象类，是模板方式的实现，抽取了基本的逻辑骨架。

## 整体上看AQS

我们先来大致的看下AQS(后文都将AbstractQueuedSynchronizer简称为AQS)的基本结构，下面简单的将它的成员变量简单的列出来:

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
    // 这个就是我们上面提到的那个FIFO队列的结点
    
    static final class Node {
        static final Node SHARED = new Node();
        // 共享模式 独占模式
        static final Node EXCLUSIVE = null;
         
        static final int CANCELLED =  1;

        static final int SIGNAL    = -1;
      
        static final int CONDITION = -2;

        static final int PROPAGATE = -3;   
        
        //  线程的等待状态  
        volatile int waitStatus;  
        
        // 指向前驱结点
        volatile Node prev;
         
        // 指向后继结点 
        volatile Node next;
        
        // 指向处于等待的线程
        volatile Thread thread;
        
        // 指向下一个等待结点 
        Node nextWaiter;
     }       
    
    // 指向头结点
    private transient volatile Node head;
    
    // 指向尾结点
    private transient volatile Node tail;
    
    // 定义同步状态
    private volatile int state;
}
```

结点大致长成这个样子:

![AQS结点的内部构造](http://tva1.sinaimg.cn/large/006e5UvNgy1h51t44xwnxj30nr0hzwf1.jpg)

队列如下图所示:

![AQS的FIFO](http://tva3.sinaimg.cn/large/006e5UvNly1h51tbm63l0j30r40bp3yz.jpg)

在线程协作的场景中，有些场景是一些线程在进行某些操作的时候其他线程都不能执行该操作，比如持有锁的操作，在同一个时刻只能有一个线程持有锁，我们将这样的情景称之为独占模式。在另一些线程协调场景中，可以同时允许多个线程进行某种操作，我们称之为共享模式。

我们可以通过修改state字段代表的同步状态来实现多线程的独占模式或者共享模式，我们来看看AQS中暴露给我们的访问state变量的方法：

```java
// CAS 更改state的值
protected final boolean compareAndSetState(int expect, int update)    
// 这个方法通常被用来做初始化state    
protected final void setState(int newState)
// 获取state变量的值     
protected final int getState()
```

这几个方法都被final关键字所修饰，说明子类无法重写它们。另外还被protected修饰，所以只能在其子类中使用。

我们可以通过state变量来实现共享模式或独占模式，CountDownLatch 就是一个典型的共享模式实现，我们初始化的时候，给定任务数，每个线程完成任务，调用countDown，通过CAS操作将state变量进行递减。任务没完成的时候，调用CountDownLatch .await()方法就会陷入阻塞，这个方法的本质也是调用getState变量，判断state变量是否大于0，如果大于0，表示任务还未完成，初始化上面的队列，将调用await方法的线程陷入阻塞中。

独占模式是类似的，初始化的时候，我们将state变量设置未0，线程在进行某项独占操作之前，先判读state 变量 是否等于0，如果等于0，说明这个锁还未被其他线程所获取，当前线程通过CAS将state变量设置为1，开始进行某项独占操作，独占操作结束后，通过CAS将这个变量改为0，并唤醒其他线程 。如果不等于0，说明别的线程正在进行某项独占操作，当前线程进入阻塞状态中。

所以如果我们想自定义线程同步工具，就需要自定义获取同步状态与释放状态的方式，而AQS提供的几个方法刚好是做这个事的:

```java
// 独占模式下,获取同步状态
protected boolean tryAcquire(int arg)
// 独占模式下,释放同步状态
protected boolean tryRelease(int arg)    
// 共享模式下，获取同步状态
protected int tryAcquireShared(int arg)    
// 共享模式下，释放同步状态   
protected boolean tryReleaseShared(int arg)    
// 独占模式下，如果当前线程已经成功获取到了同步状态则返回true
// 否则就返回false. 该线程是否正在独占资源。只有用到Condition才需要去实现它。
protected boolean isHeldExclusively()    
```

其实看方法名也能大致看出来，Acquire 中文意为获取, Release释放。 上面几个方法都是空方法，交给子类去实现。

## 从CountDownLatch 看AQS

在《CountDownLacth原理浅析》一文中，我们已经分析了当我们new CountDownLatch、CountDownLatch.countDown、CountDownLacth.await方法实际上做了什么。这里我们再复习一下来加深一下理解。

```java
CountDownLatch countDownLatch = new CountDownLatch(10);
```

 这句new实例事实上是通过继承的setState方法来对AQS的state变量进行赋值。

```java
countDownLatch.countDown(); 
```

调用CountDownLatch的countDown方法，调用的是CountDownLatch内部类Sync实例的releaseShared方法，这个方法来自AQS。在releaseShared中有以下调用:

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
}
```

 CountDownLatch 中定义了一个内部类叫sync， 里面重写了tryReleaseShared方法：

```java
// 递减 无限循环 + CAS 递减state,同时返回state 是否等于0的结果
// 如果等于0,调用doReleaseShared,唤醒因为CountDownLatch.await方法陷入阻塞的线程.
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
}
```

doReleaseShared 中用到线程的等待状态:

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

> 当前线程处在SHARED情况下，该字段才会使用，下一次的同步状态将会被无条件的传播下去。

```java
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            // 如果头结点不为尾结点,则说明队列里面有等待线程
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 如果当前结点的状态SIGNAL说明后续结点可以被唤醒              
                if (ws == Node.SIGNAL) {                 
                    // 同过CAS来设置为0,代表默认值。 
                    // 如果失败,再获取一次头结点。
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))  // 语句一
                        continue;            // loop to recheck cases
                    // 唤醒起后继结点,
                    unparkSuccessor(h);
                } else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
 }
```

上面的doReleaseShared我们重点拿出来讲讲，上面是一个无限循环，本质上就是在不断的唤醒队列中的结点，那为什么CAS在什么情况下为失败呢，对应语句一。doReleaseShared的调用方有两处: 

- releaseShared 释放共享锁
- doAcquireSharedInterruptibly  线程在获取锁失败之后，进入队列之前，再检查一遍是否能获取锁。

   以ReentrantReadWriteLock为例，读写锁是一种改进型的排它锁，读写锁允许多个线程读取共享变量，但是一次只允许一个线程对共享变量进行更新。

   我们假设三个读线程在获取锁失败的时候，首先进入的是 doAcquireSharedInterruptibly 方法:

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
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

## 从ReetranLock 来看 AQS

ReetranLock通过构造函数来支持公平锁和非公平锁，属于独占模式，我们从ReetranLock来看AQS主要来看





## 总结一下



## 参考资料

- IllegalMonitorStateException on wait() call  https://stackoverflow.com/questions/1537116/illegalmonitorstateexception-on-wait-call
- 《Java 多线程编程实战指南》 黄文海 著
- 不可不说的Java“锁”事  https://tech.meituan.com/2018/11/15/java-lock.html
-  牛逼的AQS  https://juejin.cn/post/6844903839062032398#heading-4
- 读写锁doReleaseShared源码分析及唤醒后继节点的过程分析  https://www.cnblogs.com/bmilk/p/13037655.html
- JDK8 AQS中SIGNAL标志为什么要放前面节点上，放自己节点上不好吗?  https://www.zhihu.com/question/477546502
- Java并发锁框架AQS(AbstractQueuedSynchronizer)原理从理论到源码透彻解析  https://www.bilibili.com/video/BV1yJ411v7er?spm_id_from=333.337.search-card.all.click&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799
- 《The java.util.concurrent Synchronizer Framework》 JUC同步器框架（AQS框架）原文翻译  https://www.cnblogs.com/dennyzhangdd/p/7218510.html
- Java多线程第十篇--通过CountDownLatch再探AbstractQueuedSynchronizer的共享模式  https://juejin.cn/post/7115244103528349710
- CountDownLatch---源码分析 http://www.itabin.com/aqs-share/#CountDownLatch---%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90
