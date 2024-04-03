# 让我们从synchronized和volatile谈起(一) 

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

updater线程对stop这个变量的操作之后，getter操作之后读取到了这个操作产生的影响。也许JIT编译器发现了变量没有volatile修饰，将getter线程的代码直接变成了：

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
                   if (true) {
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

于是想起了JITWatch, JITWatch能够靠JIT编译器在运行时生成的代码:

![](https://a.a2k6.com/gerald/i/2024/04/01/phx4.png)

然后发现我用的最新版本好像有点问题，这个界面没展示出来JIT代码的手脚，我只好用沙盒环境去跑:

![](https://a.a2k6.com/gerald/i/2024/04/01/qkqp.png)

倒是在日志中看到了这个手脚，只是好像JIT没有按我想的来。这个想法只好作罢。

那么上面AtomicDemo中的重复值该怎么解释呢?  我们回想一下我们在

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

我们看到increment给我们返回的和我们打印出来的sum是不同的，Thread-0 自增的结果是2，虽然Thread-0输出最早，但是我们每次自增一，在自增到2之前，肯定有一个线程自增到1，那也就是说从结果上分析来说，应当是Thread-1先抢到CPU，线程去抢CPU这个说法倒是怪怪的，我们在《如何定位线上Java应用CPU飙高》中已经提到了，CPU从就绪队列中选取任务进行执行，Linux默认的CFS调度采用的是多队列，也就是说应用程序提交的任务，由操作系统决定放入到哪个队列，于是想起来我的电脑是八核十六个核心的，所以这个任务可能是Thread-1开始执行被分配到了某颗核心上开始自增，从0自增到1，然后获取当前线程的执行名称，此时Thread-0也开始执行，将结果自增到-2，然后输出，而System.out.println上有synchronized，所以到真正执行输出语句的时候，变成了排队执行，等到Thread-0执行完毕之后，才轮到Thread-1执行。

```java
public void println(String x) {
    synchronized (this) {
        print(x);
        newLine();
    }
}
```

但是线程在哪个核心上执行是不确定的，也可能是在一颗核心上并发执行，但是Voaltile保证了可见性，也就是说我们提到的Volatile规则: 

> 对于同一个volatile变量，如果对于这个变量的写操作先行发生于这个变量的读操作，那么对于这个变量的写操作所产的影响对于这个变量的读操作是可见的。

又在公司的电脑上跑了几次，发现没出现相同值，那我该如何让读者诸君相信我呢，我想起了JCStress这个专门用来测并发的工具, JCStress是什么，怎么用，参看《JMM测试利器-JCStress学习笔记》,  JCStress跑出来的结果是确定的，于是敲下了下面这个例子:

```java
@JCStressTest
@Outcome(id = "1, -1", expect = Expect.ACCEPTABLE)
@Outcome(id = "-1, 1", expect = Expect.ACCEPTABLE_INTERESTING)
public class DuplicateValueTest {
    private final SharedState state = new SharedState();

    private static AtomicInteger atomicInteger = new AtomicInteger();

    @State
    public static class SharedState {
        public int value1;
        public int value2;
    }

    @Actor
    public void actor1(SharedState state) {
        atomicInteger.incrementAndGet();
        state.value1 = atomicInteger.intValue();
    }

    @Actor
    public void actor2(SharedState state) {
        atomicInteger.incrementAndGet();
        state.value2 = atomicInteger.intValue();
    }

    // 这里犯了一个错误
    @Arbiter
    public void arbiter(SharedState state, II_Result result) {
        if (state.value1 == state.value2) {
            result.r1 = 1;
            result.r2 = -1;
        } else {
            result.r1 = -1;
            result.r2 = 1;
        }
    }
}
```

实在不瞒着诸君，当我们根据理论进行预测的时候，我总是想验证，要不然我就会对我的理论有疑心。 这里再简单介绍一下这段代码表达的意思：

- @Actor 是核心测试注解，被标记的方法将会被JCStress启动一个线程执行。
- @State  用于标记这个类会被多个线程共享，SharedState用于收集线程执行结果。
- @Arbiter  在所有被@Actor标记的方法执行之后，会执行被 @Arbiter标记的方法，这个方法执行之后的结果被放入II_Result
- @Outcomes是代表我们预期的结果,   被@Arbiter标记的方法执行完毕之后，会和@Outcomes之后的
- 结果进行对比，如果相等, 就会出现在测试结果里面。

跑出来的结果: 

![](https://a.a2k6.com/gerald/i/2024/04/02/77h0q.png)

每次修改我们只修改了jcstress-samples这个模块，于是我就想只编译这个模块，命令如下:

```java
mvn clean install -pl jcstress-samples -am -DskipTests -T 1C 
```

按道理到这里应该结束了，上面的问题也得到了良好的解决，但是又想了一下出现重复值是因为volatile，那没有volatile呢，于是我将AtomicInteger的源码复制了出来，移除了volatile，按照我的预期应该是不出现重复值(其实这里的想法是想错了的，参看VolatileRule这个类的测试) , 然后还是跑出来了重复值: 

![](https://a.a2k6.com/gerald/i/2024/04/02/11ago.png)

JCStress测试代码:

```java
@JCStressTest
@Outcome(id = "1, -1", expect = Expect.ACCEPTABLE)
@Outcome(id = "-1, 1", expect = Expect.ACCEPTABLE_INTERESTING)
public class DuplicateValueTest {
    private final SharedState state = new SharedState();

    private static AtomicIntegerPlus atomicInteger = new AtomicIntegerPlus();

    @State
    public static class SharedState {
        public int value1;
        public int value2;
    }

    @Actor
    public void actor1(SharedState state) {
        atomicInteger.incrementAndGet();
        state.value1 = atomicInteger.intValue();
    }

    @Actor
    public void actor2(SharedState state) {
        atomicInteger.incrementAndGet();
        state.value2 = atomicInteger.intValue();
    }

    @Arbiter
    public void arbiter(SharedState state, II_Result result) {
        if (state.value1 == state.value2) {
            result.r1 = 1;
            result.r2 = -1;
        } else {
            result.r1 = -1;
            result.r2 = 1;
        }
    }
}
```

在JDK 8之上，Unsafe密封，不能被外部模块所访问，因此我们需要在jcstress-samples中的的编译插件中加上:

```xml
<compilerArgs>
    <arg>--add-exports</arg>
    <arg>java.base/jdk.internal.misc=ALL-UNNAMED</arg>
</compilerArgs>
```

最后跑出来的结果:

![](https://a.a2k6.com/gerald/i/2024/04/02/s57.png)

跑结果的是也要加特别参数:

```java
java --add-exports  java.base/jdk.internal.misc=ALL-UNNAMED -jar   jcstress-samples/target/jcstress.jar -t DuplicateValueTest
```

于是感到疑惑，为什么从AtomicInteger移除volatile关键字之后，测试结果没变化呢，究竟是哪里出现了问题呢。 同时我们在VolatileRule这个例子中可以看到移除Volatile之后，getter 线程读不到这个结果了，也就是说我们对AtomicInteger的理解一定是哪里出现了问题。

于是一边瞎看AtomicInteger的注释，一边用AtomicInteger remove Integer 或AtomicInteger Volatile 去搜索，然后就在StackOverFlow上搜到了我想要的答案(见参考资料[4]):

![](https://a.a2k6.com/gerald/i/2024/04/02/12shq.png)

这个回答很贴心还给出了自己的引用来源，也就是JDK的包下面会有package-summary.html，会有对这个类的说明，所以以后看并发的时候可以先从package-summary.html来看。

> I believe that Atomic* actually gives both atomicity and volatility. So when you call (say) AtomicInteger.get(), you're guaranteed to get the latest value. This is documented in the java.util.concurrent.atomic package documentation:
>
> 我相信原子类实际上具备两重特性: 原子性和易变性。所以当你调用AtomicInteger的get方法，你将会确保会拿到相对比较新的值。
>
> 这在java.util.concurrent.atomic的包下面有文档。

所以我看AtomicInteger还是太想当然了，没有好好的看这个注释吗？

> The memory effects for accesses and updates of atomics generally follow the rules for volatiles, as stated in section 17.4 of The Java™ Language Specification.  对原子变量的访问和更新通常遵循 Java 语言规范第 17.4 节中关于 `volatile` 变量的规则。
>
> - `get` has the memory effects of reading a `volatile` variable.
>
>   `get` 操作具有读取 `volatile` 变量的内存效果
>
> - `set` has the memory effects of writing (assigning) a `volatile` variable.
>
>   `set` 操作具有写入(赋值)一个 `volatile` 变量的内存效果。
>
> - `lazySet` has the memory effects of writing (assigning) a `volatile` variable except that it permits reorderings with subsequent (but not previous) memory actions that do not themselves impose reordering constraints with ordinary non-`volatile` writes. Among other usage contexts, `lazySet` may apply when nulling out, for the sake of garbage collection, a reference that is never accessed again.
>
>   `lazySet` 操作具有写入(赋值)一个 `volatile` 变量的内存效果,但它允许与后续(而非之前)的内存操作重新排序,这些操作本身不会对普通的非 `volatile` 写入施加重新排序的约束。在其他使用情况下,`lazySet` 可用于将一个不再访问的引用设置为 null,以便进行垃圾收集。
>
> - `weakCompareAndSet` atomically reads and conditionally writes a variable but does *not* create any happens-before orderings, so provides no guarantees with respect to previous or subsequent reads and writes of any variables other than the target of the `weakCompareAndSet`.
>
>    原子地读取和有条件地写入一个变量,但不创建任何 happens-before 排序,因此对于 `weakCompareAndSet` 目标之外的任何变量的先前或后续读写,不提供任何保证。
>
> - `compareAndSet` and all other read-and-update operations such as `getAndIncrement` have the memory effects of both reading and writing `volatile` variables.
>
>   `compareAndSet` 和所有其他读取并更新的操作,如 `getAndIncrement`,具有读取和写入 `volatile` 变量的内存效果。

好消息是我们知道了AtomicInteger中的volatile不是给getAndIncrement用的，而是给get/set方法用的。坏消息是又看到了新的方法需要去理解:

- lazySet
- weakCompareAndSet
- compareAndSet

知识就是这样一串接一串, 那么原子类CAS失败是如何获取最新的变量的呢, 让我们接着看源码:

```java
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

注意这里的getIntVolatile， 这是一个Native方法，我们直接看对应的实现：

```c
#define GET_FIELD_VOLATILE(obj, offset, type_name, v) \
oop p = JNIHandles::resolve(obj); \
volatile type_name v = OrderAccess::load_acquire((volatile type_name*)index_oop_from_field_offset_long(p, offset));
```

```java
UNSAFE_ENTRY(jlong, Unsafe_GetLongVolatile(JNIEnv *env, jobject unsafe, jobject obj, jlong offset))
  UnsafeWrapper("Unsafe_GetLongVolatile");
  {
    if (VM_Version::supports_cx8()) {
      GET_FIELD_VOLATILE(obj, offset, jlong, v);
      return v;
    }
    else {
      Handle p (THREAD, JNIHandles::resolve(obj));
      jlong* addr = (jlong*)(index_oop_from_field_offset_long(p(), offset));
      ObjectLocker ol(p, THREAD);
      jlong value = *addr;
      return value;
    }
  }
```

getIntVolatile是将某个变量当做volatile变量去读取，我们可以看到OrderAccess::load_acquire这句调用用于加载内存屏障，我们将会在《JMM 学习笔记(三) volatile与内存屏障》里面详细探讨内存屏障这个概念。

## CopyOnWriteList中的volatile与synchronized

happens-before的关于volatile和synchronized的论述如下:

1. volatile规则: 对于同一个volatile变量，如果对于这个变量的写操作先行发生于这个变量的读操作，那么对于这个变量的写操作所产的影响对于这个变量的读操作是可见的。 
2. 管程锁定规则 **（Monitor Lock Rule）**: 一个unlock操作先行发生于后面对同一个锁的lock操作。这里再对管程锁定规则解释一下，这句陈述给人一种没上锁先解锁的感觉，事实上我们可以将现行发生理解为一种主动的规则要求，而先行发生事实上是程序的客观运行结果，正确的解读方式是，对于同一把锁，如果程序在运行过程中一个Unlock操作先行发生于同一把锁的lock操作，那么该unlock操作产生的影响(修改共享变量的值，发送了消息，调用了方法)，对该lock操作是可见的。我们结合这段代码来理解这个规则:

```java
public class WithoutMonitorLockRule {
    
    private static boolean stop = false;
    
    public static void main(String[] args) {
        Thread updater = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            stop = true;
            System.out.println("updater set stop true");
        }, "updater");

        Thread getter = new Thread(() -> {
            while (true) {
                if (stop) {
                    System.out.println("getter stopped");
                    break;
                }
            }
        }, "getter");
        getter.start();
        updater.start();
    }
}
```

getter线程会一直陷入循环中，无法读取到updater线程对共享变量操作的影响。现在我们用synchronized改造一下:

```java
public class MonitorLockRuleSynchronized {
    private static boolean stop = false;
    private static final Object lockObject = new Object();

    public static void main(String[] args) {
        Thread updater = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            synchronized (lockObject) {
                stop = true;
                System.out.println("updater set stop true.");
            }
        }, "updater");
        Thread getter = new Thread(() -> {
            while (true) {
                synchronized (lockObject) {
                    if (stop) {
                        System.out.println("getter stopped.");
                        break;
                    }
                }
            }
        }, "getter");
        updater.start();
        getter.start();
    }
}
```

```java
# 输出结果为:
updater set stop true.
getter stopped.
```

 所以synchronized和volatile有重合的地方，synchronize能够替代volatile ?  带着这个问题我们看下CopyOnWriteArrayList的源码:

```java
public class CopyOnWriteArrayList<E>{  
    
  private transient volatile Object[] array;
    
  public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
  }	
  private E get(Object[] a, int index) {
     return (E) a[index];
  }    
  public E get(int index) {
     return get(getArray(), index);
  }   
}    
```

假设一个线程调用add，setArray是在操纵volatile变量，根据volatile规则，对于同一个volatile变量，如果对于这个变量的写操作先行发生于这个变量的读操作，那么对于这个变量的写操作所产的影响对于这个变量的读操作是可见的。我们就能在setArray操作完成之后，读线程就能调用get方法拿到这个更新。这里不知道怎么把CopyOnWriteArrayList用的锁记成synchronized了，其实这里换成synchronized也是一样的效果，想起来早起看的一个很粗糙的例子:

```java
public class SimpleReadAndWriteLock {
	
    private volatile int value;
   
    public int getValue(){
        return value;
    }  
    public void increment(){
        synchronized (this){
            value++;
        }
    }
}
```



## volatile的场景







##  思考路程



## 参考资料 

[1] Why CopyOnWriteArrayList uses Volatile keyword along with Lock   https://stackoverflow.com/questions/70445308/why-copyonwritearraylist-uses-volatile-keyword-along-with-lock

[2] Why setArray() method call required in CopyOnWriteArrayList https://stackoverflow.com/questions/28772539/why-setarray-method-call-required-in-copyonwritearraylist

[3] JMM测试利器-JCStress学习笔记 https://segmentfault.com/a/1190000044324553?_ea=342952266

[4] How do I Understand Read Memory Barriers and Volatile https://stackoverflow.com/questions/1787450/how-do-i-understand-read-memory-barriers-and-volatile

[5] AtomicInteger and volatile [duplicate] https://stackoverflow.com/questions/14338533/atomicinteger-and-volatile

[6] 绿色线程  https://zh.wikipedia.org/wiki/%E7%BB%BF%E8%89%B2%E7%BA%BF%E7%A8%8B 

​                                                                                                                                                                                                                                                                                                                                                                                      

