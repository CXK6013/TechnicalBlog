# 让我们从synchronized和volatile谈起

[TOC]

## 让我们先从一个例子谈起

```java
public class AtomicDemo {
    private   static final AtomicInteger sum = new AtomicInteger(0);
    public static  void increment(){
        sum.getAndIncrement();
    }
    public static void main(String[] args) {
        for (int i = 0; i < 2 ; i++) {
             new Thread(()->{
                 for (int j = 0; j < 100 ; j++) {
                     increment();
                     System.out.println(sum); // 语句一
                 }
             }).start();
        }
    }
}
```

输出结果会有重复的:  ![](https://a.a2k6.com/gerald/i/2024/03/23/w2tg.jpg)

 执行到语句一的时候，已经自增完成了，那为什么会线程一和线程二会出现重复值呢？我们看下AtomicInteger.getAndIncrement的源码:

```java
// 基于OpenJDK 11
private volatile int value;

public AtomicInteger(int initialValue) {
    value = initialValue;
}
// U是unsafe,VALUE为字段value的内存偏移地址，通过字段VALUE的值可以定位到AtomicInteger对象中value的内存地址
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value"); // 语句二

public final int getAndIncrement() {
    return U.getAndAddInt(this, VALUE, 1);
}
```

我们接着看getAndAddInt方法的实现:

```java
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        // 获取AtomicInteger的value的值
        v = getIntVolatile(o, offset);
        // 
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

```java
public final boolean weakCompareAndSetInt(Object o, long offset,
                                          int expected,
                                          int x) {
    //执行比较并交换操作。
    return compareAndSetInt(o, offset, expected, x);
}
```

注意AtomicInteger的value变量有volatile修饰，那么根据我们在《JMM学习笔记(二) 规则和volatile》讲的内容:

> 当我们编写Java程序的时候，Java Memory Model屏蔽掉了各种硬件和操作系统的内存访问差异，Java内存模型给出了一组规则或规范， 定义了程序中各个变量(包括示例字段，静态字段和构成数组对象的元素)的访问方式，规范了Java虚拟机与计算机内存是如何协同工作的。
>
> JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(有些文献也称为栈空间)，用于存储线程私有的数据，而Java内存模型中规定所有变量都必须存储在内存，主内存是共享内存区域，所有线程可以访问。
>
> 但线程对变量的操作(读取赋值等)必须在工作内存中进行，首先要将变量从主内存拷贝的自己的工作内存空间，然后对变量进行操作，操作完成后再将变量写会主内存，不能直接操作主内存中的变量，工作内存中存储着主内存中的变量副本拷贝。
>
> 工作内存是每个线程的私有数据区域，因此不同的线程无法访问对方的工作内存，线程间的通信(传值) 必读通过主内存来完成。

- **volatile变量规则**：对于同一个volatile变量，如果对于这个变量的写操作先行发生于这个变量的读操作，那么对于这个变量的写操作所产的影响对于这个变量的读操作是可见的。

注意读volatile变量规则，对于同一个volatile变量，对于这个变量的写操作先行发生于这个变量的读操作，那么这个变量的写操作所产生的影响对于这个变量的读操作是可见的。下面是对这个例子的说明:

```java
public class VolatileRule {
    private volatile static boolean stop = false;
    public static void main(String[] args) {     
        Thread updater = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                stop = true;
                System.out.println("updater set stop true");
            }
        }, "updater");

        Thread getter = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (stop) {
                        System.out.println("getter stopped");
                        break;
                    }
                }
            }
        }, "getter");
        getter.start();
        updater.start();
    }
}
```

updater线程对stop这个变量的操作之后，getter操作之后读取到了这个操作产生的影响。那么上面AtomicDemo中的重复值该怎么解释呢?  我们会想一下我们在

《JMM学习笔记(二) 规则和volatile》里面讲的内容: 

> 在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。

```java
System.out.println(sum);
```

```java
public String toString() {
    return Integer.toString(get());
}
public final int get() {
    return value;
}
```

在getAndIncrement之后，应当能看到自增操作产生的影响，我们提出第一种假说，假设Thread-0线程读取到变量之后的代码是1，在调用getAndAddInt没完成自增操作时间片耗尽，于是Thread-1线程被CPU执行，然后也读取到了主存的变量是1，然后Thread-0开始自增，自增成功完成。然后Thread-1开始跟上，由于Thread-1此时工作内存的值还是1，比较主存和工作内存的变量，然后发现不同，再读取一遍主存的值到共享内存变量里面，完成自增，此时是3，但是还没刷新到主存中，于是打印语句输出了2，但是我们看到上面的规则: “在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作”。根据这条规则应当能读取到共享变量操作产生的影响啊，那为什么会输出两次2呢？ 我们将上面的例子做进一步的变型:

```java
public class AtomicDemo {
    private   static final AtomicInteger sum = new AtomicInteger(0);
    public static int  increment(){
       return sum.incrementAndGet();
    }
    public static void main(String[] args) {
        for (int i = 0; i < 2 ; i++) {
             new Thread(()->{
                 for (int j = 0; j < 100 ; j++) {
                     System.out.println("increment结果:"+increment() + " " +Thread.currentThread().getName()+"------->"+sum);
                 }
             }).start();
        }
    }
}
```

![](https://a.a2k6.com/gerald/i/2024/03/23/6q961.jpg)

我们看到increment给我们返回的和我们打印出来的sum是不同的，Thread-0 自增的结果是2，虽然Thread-0输出最早，但是我们每次自增一，在自增到2之前，肯定有一个线程自增到1，那也就是说从结果上分析来说，应当是Thread-1先抢到CPU，线程去抢CPU这个说法倒是怪怪的，我们在《如何定位线上Java应用CPU飙高》中已经提到了，CPU从就绪队列中选取任务进行执行，Linux默认的CFS调度采用的是多队列，也就是说应用程序提交的任务，由操作系统决定放入到哪个队列，于是想起来我的电脑是八核十六个核心的，所以这个任务可能是

突然想起来，之前和一个朋友讨论过单核心是否不存在线程安全问题，

##  理一理synchronized和volatile





## CopyOnWriteList中的volatile与synchronized







## 简单读写锁中的Volatile与synchronized



##  总结一下







## 参考资料 

[1] Why CopyOnWriteArrayList uses Volatile keyword along with Lock   https://stackoverflow.com/questions/70445308/why-copyonwritearraylist-uses-volatile-keyword-along-with-lock

[2] Why setArray() method call required in CopyOnWriteArrayList https://stackoverflow.com/questions/28772539/why-setarray-method-call-required-in-copyonwritearraylist



​                                                                                                                                                                                                                                                                                                                                                                                                 

