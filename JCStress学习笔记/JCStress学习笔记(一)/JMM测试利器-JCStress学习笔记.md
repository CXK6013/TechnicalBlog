# JMM测试利器-JCStress学习笔记

[TOC]

## 前言

我们前文提到，当我们对理论有一些了解，我们就渴望验证，如果无法对理论进行验证，那么我们就可能对理论将信将疑，那对于Java领域的并发理论，一向是难以测试的，更何况调试， 这不仅仅是我们的认知，OpenJDK的	者也是这么认知的:

> Circa 2013 (≈ JDK 8) ， we suddenly realized there are no regular concurrency tests that ask hard questions about JMM conformance
>
> 大约在2013年，也就是JDK8发布的时候，我们突然意识到没有常规的并发测试可以针对JMM的一致性进行深入探究。

> Attempts to contribute JMM tests to JCK were futile: probabilistic tests
>
> 尝试将JMM测试贡献给JCK是徒劳的：这是概率性测试。

> JVMs are notoriously awkward to test: many platforms, many compilers, many compilation and runtime modes, dependence on runtime profile
>
> JVM是出了名的难以测试， 涉及多个平台，多个编译器、多种编译和运行时模式，以及依赖的配置文件，所以测试起来比较麻烦。

这也就是JCStress诞生的原因，我们希望有一个测试框架对JMM进行测试，方便我们验证我们的猜想是否正确。但是关于JCStress这一方面的资料一向少之又少，我期待自己是第一手资料的获得者，于是我尝试看JCStress给出的文档,  首先我来到了JCStress在GitHub上的仓库, 看到了下面一段话:

In order to understand jcstress tests and maybe write your own, it might be useful to work through the  jcstress-samples. The samples come in three groups:

通过JCStress-sample，可以帮助你理解JCStress测试，帮助你写出自己的测试

- **APISample** target to explain the jcstress API;

​       APISample 的目标是解释jcstress的API 

- **JMMSample** target to explain the basics of Java Memory Model;

​      JMM的Sample的目标是解释Java内存模型的基本概念

- **ConcurrencySample** show the interesting concurrent behaviors of standard library.

​     展示标准库有趣的并发行为   

fine，接下来我们就直接尝试阅读范例来学习一下JCStress是如何使用的。

## 开始阅读

这些范例我们有两种方式可以拿到:

- 从Github上下载一下JCStress的源码 , 地址如下: https://github.com/openjdk/jcstress.git
- maven仓库直接获取对应的依赖:

```xml
<dependency>
	<groupId>org.openjdk.jcstress</groupId>
    <artifactId>jcstress-samples</artifactId>
    <version>0.3</version>
 </dependency>
```

我们首先阅读的第一个类叫:

```java
/*
    This is our first concurrency test. It is deliberately simplistic to show
    testing approaches, introduce JCStress APIs, etc.
	这是我们第一个并发测试，这个测试被刻意的做的很简单,以便展示测试方法、介绍JCStress API等方法。
    Suppose we want to see if the field increment is atomic. 
    我们想看下字段的递增是否是原子性的.
    We can make test with two actors, both actors incrementing the field and recording what value they observed into the result     object. 
    我们可以用两个actor进行测试，两个actor都对字段进行自增，并将观察到的值记录到result对象里面
    As JCStress runs, it will invoke these methods on the objects holding the field once per each actor and instance, and record     what results are coming from there.
    当JCStress运行时，她会对每个"actor"和实例进行一次方法调用，并记录调用的结果。
    Done enough times, we will get the history of observed results, and that
    would tell us something about the concurrent behavior. For example, running
    this test would yield:
    进行充足的测试之后, 我们将会得到观察结果的历史，将会告诉我们并发行为发生了什么,运行这些测试将产生:
      [OK] o.o.j.t.JCStressSample_01_Simple
      (JVM args: [-server])  请求虚拟机以server方式运行
      观察状态           发生的情况		   期待	          解释    
      Observed state   Occurrences   Expectation  Interpretation
       											  // 两个线程得出了相同的值: 原子性测试失败 	
                1, 1    54,734,140    ACCEPTABLE  Both threads came up with the same value: atomicity failure.
                								  // actor1 先新增  后actor2新增
                1, 2    47,037,891    ACCEPTABLE  actor1 incremented, then actor2.
                								  // actor2 先增加, actor1后增加		
                2, 1    53,204,629    ACCEPTABLE  actor2 incremented, then actor1.
      如何运行JCStress测试,            
     How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t JCStress_APISample_01_Simple
 */
 标记这个类是需要JCStress测试的类
// Mark the class as JCStress test.
@JCStressTest
// These are the test outcomes.
@Outcome(id = "1, 1", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Both actors came up with the same value: atomicity failure.")
@Outcome(id = "1, 2", expect = Expect.ACCEPTABLE, desc = "actor1 incremented, then actor2.")
@Outcome(id = "2, 1", expect = Expect.ACCEPTABLE, desc = "actor2 incremented, then actor1.")
// 状态对象     
// This is a state object
@State
public class APISample_01_Simple {
    
    int v;
    
    @Actor
    public void actor1(II_Result r) {
        // 将结果actor1的行为产生的记录进r1
        // record result from actor1 to field r1 
        r.r1 = ++v; 
    }
    
    @Actor
    public void actor2(II_Result r) {
        //将actor2的结果记录进r2
        // record result from actor2 to field r2
        r.r2 = ++v; 
    }
}
```

我们上面已经对这个测试示例进行了简单的解读，这里我们要总结一下JCStress的工作模式 ,  我们在类里面声明我们希望JCStress执行的行为，这些行为的结果被记录到结果中,  最后和@Outcome注解中声明的内容做对比, 输出对应的结果:

![iqv8hp.png](https://i.328888.xyz/2023/05/12/iqv8hp.png)

现在这段代码, 我们唯一还不明白作用的就是@outcome、@State、   @Actor。我们还是看源码上的注释:

- State

> State is the central annotation for handling test state. It annotates the class that holds the data mutated/read by the tests.
> Important properties for the class are: 
>
> State 是处理测试状态的中心注解，有这个注解的类，在测试中负责读取和修改数据。这些类要求以下两个属性
>
> 1. State class should be public, non-inner class.
>
>    State 类应当是public ，而不是内部类
>
> 2. State class should have a default constructor.
>
> ​    State 类应当有一个默认的构造函数
>
> During the run, many State instances are created, and therefore the tests should try to minimize state instance footprint. All actions in constructors and instance initializers are visible to all Actor and Arbiter threads.
>
> 在运行期间，有State注解的类会被大量创建，因此在测试中应当尽量减少State实例的占用，所有构造函数中的操作和变量初始化代码块对所有的Actor 和Arbiter 可见。

constructor 这个我是知道的，instance initializers 是 指直接对类的成员变量进行初始化的操作:

```java
class Dog {
  private String name = "旺财";
  // 这种声明 我是头一次见  
  {name = "小小狗"}  
}
```

- outcome

> Outcome describes the test outcome, and how to deal with it. It is usually the case that a JCStressTest has multiple outcomes, each with its distinct id(). id() is cross-matched with Result-class' toString() value. id() allows regular expressions. 
>
> OutCome注解用于描述测试结果, 以及如何处理它。通常情况下，一个JCStress测试有多个输出结果，每个结果都有一个唯一id 。id和观测结果类的toString方法返回值相匹配，id允许正则表达式。
>
> For example, this outcome captures all results where there is a trailing "1":   \@Outcome(id = ".*, 1", ...) When there is no ambiguity in what outcome should match the result, the annotation order is irrelevant. When there is an ambiguity, the first outcome in the declaration order is matched. There can be a default outcome, which captures any non-captured result. It is the one with the default id().
>
> 例如，我们在id里面填入的属性为.*, 1，那么将会匹配的字符串形式为尾巴为, 1的结果，比如2,1。当预计结果与实际输出结果匹配且没有歧义，OutCome的注解顺序是任意的。当预计结果与实际输出结果有歧义时，指存在两个OutCome中的id值存在交集，那么将会匹配第一个注解。所以也可以设置一个默认的输出结果，当实际输出和预计输出不匹配的时候，匹配默认输出。

我们结合APISample_01_Simple的代码来解释，注意在这个类里面两个方法将结果都放在了II_Result上，这是实际输出，II_Result代表有两个变量记录值，JCStress里面有各种各样类似的类来记录结果，II_Result代表提供了两个变量来记录实际运行结果，II代表两个，那么我们是可以大胆推导一下，有I_Result 、III_Result、IIII_Result。是的没有错，如果你觉得你需要测试的结果需要更多的变量来记录，那么需要变量的数量I_Result去JCStress里面找就行了。 然后我们在看下I_Result 的toString方法:

```java
//省略II_Result的其他方法和成员变量
@Result
public class II_Result implements Serializable {
    public String toString() {
        return "" + r1 + ", " + r2;
    }
}
//省略III_Result的其他方法和成员变量
@Result
public class IIII_Result implements Serializable {
    public String toString() {
        return "" + r1 + ", " + r2 + ", " + r3 + ", " + r4;
    }
}
```

我们会发现II_Result、IIII_Result的toString方法都是将成员变量用逗号拼接。那上面的例子我们就能大致看懂了，JCStress执行APISample_01_Simple，然后将结果和Outcome中的id做比较，如果相等会被记录，expect属性是什么意思呢？对我们希望出现的结果进行评级，哪些是我们重点的结果，一共有四个等级: 

ACCEPTABLE: Acceptable result. Acceptable results are not required to be present 

> 可接受的结果，可接受的结果不要求出现

ACCEPTABLE_INTERESTING:   Same as ACCEPTABLE, but this result will be highlighted in reports.

> 和ACCEPTABLE一样，但是结果出现了就会在报告里面被高亮。

FORBIDDEN:  Forbidden result. Should never be present.

> 禁止出现的结果 应当永远不出现

UNKNOWN:   Internal expectation: no grading. Do not use.

> 未知，不要用

- Actor

> Actor is the central test annotation. It marks the methods that hold the actions done by the threads. The invariants that are maintained by the infrastructure are as follows:
>
> Actor 是核心测试注解，被它标记的方法将会一个线程所执行，基础架构会维护一组不变的条件:
>
> 1. Each method is called only by one particular thread.
>
>    每个方法会被一个特殊的线程所调用。
>
> 1. Each method is called exactly once per State instance.
>
> ​    每个方法被每个State实例所调用

> Note that the invocation order against other Actor methods is deliberately not specified. 
>
> 注意Actor所执行的方法调用顺序是故意不指定的。
>
> Therefore, two or more Actor methods may be used to model the concurrent execution on data held by State instance.
>
> 因此，可以使用两个或多个Actor方法来模拟对State实例持有数据的并发执行。
>
> Actor-annotated methods can have only the State or Result-annotated classes as the parameters. 
>
> 只有被@Result或@State标记的类才能作为Actor方法(拥有Actor注解)的参数
>
> Actor methods may declare to throw the exceptions, but actually throwing the exception would fail the test.
>
> Actor方法可能声明抛出异常，但是实际上抛出的异常会导致测试失败。

这里我来解读一下，II_Result 这个类上就带有@Result注解。现在我们要解读的就是++v了，除了++v之外，我们熟悉的还有v++，如果现在问我这俩啥区别，我大概率回答的是++v 先执行自增，然后将自增后的值赋给v，对应的代码应该是像下面这样:

```java
v = v + 1;
```

那v++呢，先使用值，后递增，我理解应该是像下面这样:

```java
v = v; 
v = v + 1;
```

我们日常使用比较多的应该就是v++了:

```java
for(int v = 0 ; v < 10 ; v++){}
```

for循环的括号是个小代码块，第一次执行的时候首先声明了一个v = 0，然后判断v是否小于10，如果小于执行v++，这个时候v的值仍然是0，然后走循环体里面的内容。结合我们上面的理解，这似乎有些割裂，如果按照我对v++的理解，一条语句等价于两条语句，那么这两条语句还分开执行，这看起来非常怪。如果我们将v++对一个变量进行赋值，然后这个变量拿到的就是自增之后的值，我们也可以对我们的理解打补丁，即在循环中两条语句不是顺序执行，先执行v = v，循环体里面的内容执行完毕再执行v = v + 1。那再完善一下呢，i++事实上被称为后缀表达式，语义为i+1，然后返回旧值。++i被称为前缀表达式, 语义为自增返回新值。乍一看好像是自增的，但是我们在细化一下，在自增之前都要读取变量的值，所以无论是++v，还是v++，都需要先读值再自增，所以这是两个操作，在Java里面不是原子的，这是我们的论断，我们跑一下上面的例子，看下结果。

## 跑一下看一下结果

一般来说JCStress是要用命令行跑的，但是这对于新手不友好，有些概念还需要理解一下，但是IDEA有专门为JCStress制作的插件，但这个插件并没有做到全版本的IDEA适配，目前适配的版本如下:

> Aqua — 2023.1 (preview)

> IntelliJ IDEA Educational — 2022.2 — 2022.2.2

> IntelliJ IDEA Community — 2022.2 — 2023.1.2 (eap)

> IntelliJ IDEA Ultimate — 2022.2 — 2023.1.2 (eap)

> Android Studio — Flamingo | 2022.2.1 — Hedgehog | 2023.1.1 Canary 2

本来这个插件跑的是好好的，但是我换了台机器就发现好像有点问题，还是用命令行吧，首先我们去GitHub将源码下载下来:

```http
https://github.com/openjdk/jcstress.git
```

编译JCStress这个项目的master分支最好是JDK 17，我试了其他版本，发现都会有点奇奇怪怪的问题，我们先不必深究这些问题，先跑出来结果再说. 首先我们在命令行执行：

```java
mvn clean install -DskipTests -T 1C
```

本篇的例子来自于JCStress tag中的0.3，但是跑JCStress测试的时候，发现0.3上好像有点水土不服，但提供的例子都是差不多，基本没变化。这里跑测试的时候，用的就是master分支下面的示例，我们来跑下示例:

```java
java -jar jcstress-samples/target/jcstress.jar -t API_01_Simple
```

![VZuzyU.jpeg](https://i.328888.xyz/2023/05/15/VZuzyU.jpeg)

跑完测试之后，测试报告会会出现在results文件夹下面:

![VZuGqy.jpeg](https://i.328888.xyz/2023/05/15/VZuGqy.jpeg)

我们打开看下测试报告:

![VZuKyE.jpeg](https://i.328888.xyz/2023/05/15/VZuKyE.jpeg)

运行的次数比我们想象的要多，1877050次。

## 例子解读

### API_02_Arbiters

Arbiter 意为冲裁。

```java
/*
    Another flavor of the same test as APISample_01_Simple is using arbiters.
    另一个和APISample_01_Simple一样味道的测试是使用仲裁者。
    
    Arbiters run after both actors, and therefore can observe the final result. 
    Arbiters方法在actor方法运行之后执行,因此可以观察到最后的结果。

	This allows to observe the permanent atomicity failure after both actors  finished.
	在actor方法执行之后,Arbiters方法将会观察到原子性失败的结果	
    How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t API_02_Arbiters
        ...

      RESULT         SAMPLES     FREQ       EXPECT  DESCRIPTION
           1     888,569,404    6.37%  Interesting  One update lost: atomicity failure.
           2  13,057,720,260   93.63%   Acceptable  Actors updated independently.
 */

@JCStressTest

// These are the test outcomes.
@Outcome(id = "1", expect = ACCEPTABLE_INTERESTING, desc = "One update lost: atomicity failure.")
@Outcome(id = "2", expect = ACCEPTABLE,             desc = "Actors updated independently.")
@State
public class API_02_Arbiters {

    int v;

    @Actor
    public void actor1() {
        v++;
    }

    @Actor
    public void actor2() {
        v++;
    }

    @Arbiter
    public void arbiter(I_Result r) {
        r.r1 = v;
    }
}
```

### API_03_Termination

```java
/*
    Some concurrency tests are not following the "run continously" pattern.
    一些并发测试可能并不适用"持续运行"模式。
    One of interesting test groups is that asserts if the code had terminated after a signal.
    一个有趣的测试是确定代码是否在接收到信号之后终止
    Here, we use a single @Actor that busy-waits on a field, and a @Signal that
    sets that field.
    在下面的例子中，我们使用@Actor来让一个线程在一个字段上忙等，然后用@Signal方法去设置这个字段。
    JCStress would start actor, and then deliver the signal.
    JCStress会首先启动actor，然后启动signal.
    If it exits in reasonable time, it will record "TERMINATED" result, otherwise
    record "STALE".
    如果在一定的时间退出，它将会记录"TERMINATED",否则将会记录state。
    How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t API_03_Termination

        ...

          RESULT  SAMPLES     FREQ       EXPECT  DESCRIPTION
           STALE        1   <0.01%  Interesting  Test hung up.
      TERMINATED   13,168   99.99%   Acceptable  Gracefully finished.

      Messages:
        Have stale threads, forcing VM to exit for proper cleanup.
 */

@JCStressTest(Mode.Termination)
@Outcome(id = "TERMINATED", expect = ACCEPTABLE,             desc = "Gracefully finished.")
@Outcome(id = "STALE",      expect = ACCEPTABLE_INTERESTING, desc = "Test hung up.")
@State
public class API_03_Termination {

    @Actor
    public void actor1() throws InterruptedException {
        synchronized (this) {
            wait();
        }
    }

    @Signal
    public void signal() {
        synchronized (this) {
            notify();
        }
    }
}
```

这里我们不认识的也就是  @Signal，我们看下这个类上的注释:

```java
/**
 * {@link Signal} is useful for delivering a termination signal to {@link Actor}  in {@link Mode#Termination} tests.
 * Signal 被用于 在中断测试中发送中断信号给Actor。
 *  It will run after {@link Actor} in question  started executing.
 *  它将会在相应的Actor运行之后开始执行。
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Signal {
}

```

### API_04_Nesting

```java
/*
It is sometimes convenient to put the tests in the same source file for better comparison. JCStress allows to nest tests like this:
有时候将一些测试放入一个源文件可以更好的比较，JCStress允许将下面一样嵌套

    How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t API_04_Nesting

        ...

        .......... [OK] org.openjdk.jcstress.samples.api.APISample_04_Nesting.PlainTest

          RESULT      SAMPLES    FREQ       EXPECT  DESCRIPTION
            1, 1   21,965,585    4.5%  Interesting  Both actors came up with the same value: atomicity failure.
            1, 2  229,978,309   47.5%   Acceptable  actor1 incremented, then actor2.
            2, 1  232,647,044   48.0%   Acceptable  actor2 incremented, then actor1.

        .......... [OK] org.openjdk.jcstress.samples.api.APISample_04_Nesting.VolatileTest

          RESULT      SAMPLES    FREQ       EXPECT  DESCRIPTION
            1, 1    4,612,990    1.4%  Interesting  Both actors came up with the same value: atomicity failure.
            1, 2   95,520,678   28.4%   Acceptable  actor1 incremented, then actor2.
            2, 1  236,498,350   70.3%   Acceptable  actor2 incremented, then actor1.
 */

public class API_04_Nesting {

    @JCStressTest
    @Outcome(id = "1, 1", expect = ACCEPTABLE_INTERESTING, desc = "Both actors came up with the same value: atomicity failure.")
    @Outcome(id = "1, 2", expect = ACCEPTABLE,             desc = "actor1 incremented, then actor2.")
    @Outcome(id = "2, 1", expect = ACCEPTABLE,             desc = "actor2 incremented, then actor1.")
    @State
    public static class PlainTest {
        int v;

        @Actor
        public void actor1(II_Result r) {
            r.r1 = ++v;
        }

        @Actor
        public void actor2(II_Result r) {
            r.r2 = ++v;
        }
    }

    @JCStressTest
    @Outcome(id = "1, 1", expect = ACCEPTABLE_INTERESTING, desc = "Both actors came up with the same value: atomicity failure.")
    @Outcome(id = "1, 2", expect = ACCEPTABLE,             desc = "actor1 incremented, then actor2.")
    @Outcome(id = "2, 1", expect = ACCEPTABLE,             desc = "actor2 incremented, then actor1.")
    @State
    public static class VolatileTest {
        volatile int v;

        @Actor
        public void actor1(II_Result r) {
            r.r1 = ++v;
        }

        @Actor
        public void actor2(II_Result r) {
            r.r2 = ++v;
        }
    }
}
```

### API_05_SharedMetadata

```java
/*
    In many cases, tests share the outcomes and other metadata. To common them,
    there is a special @JCStressMeta annotation that says to look up the metadata
    at another class.
    在一些情况下，测试需要共享输出结果和其他元数据，为了共享他们,有一个特殊的注解@JCStressMeta来指示其他类在另一个类里面查找元数据。
    How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t API_05_SharedMetadata

        ...

        .......... [OK] org.openjdk.jcstress.samples.api.APISample_05_SharedMetadata.PlainTest

          RESULT      SAMPLES    FREQ       EXPECT  DESCRIPTION
            1, 1    6,549,293    1.4%  Interesting  Both actors came up with the same value: atomicity failure.
            1, 2  414,490,076   90.0%   Acceptable  actor1 incremented, then actor2.
            2, 1   39,540,969    8.6%   Acceptable  actor2 incremented, then actor1.

        .......... [OK] org.openjdk.jcstress.samples.api.APISample_05_SharedMetadata.VolatileTest

          RESULT      SAMPLES    FREQ       EXPECT  DESCRIPTION
            1, 1   15,718,942    6.1%  Interesting  Both actors came up with the same value: atomicity failure.
            1, 2  120,855,601   47.2%   Acceptable  actor1 incremented, then actor2.
            2, 1  119,393,635   46.6%   Acceptable  actor2 incremented, then actor1.
 */

@Outcome(id = "1, 1", expect = ACCEPTABLE_INTERESTING, desc = "Both actors came up with the same value: atomicity failure.")
@Outcome(id = "1, 2", expect = ACCEPTABLE,             desc = "actor1 incremented, then actor2.")
@Outcome(id = "2, 1", expect = ACCEPTABLE,             desc = "actor2 incremented, then actor1.")
public class API_05_SharedMetadata {

    @JCStressTest
    @JCStressMeta(API_05_SharedMetadata.class)
    @State
    public static class PlainTest {
        int v;

        @Actor
        public void actor1(II_Result r) {
            r.r1 = ++v;
        }

        @Actor
        public void actor2(II_Result r) {
            r.r2 = ++v;
        }
    }

    @JCStressTest
    @JCStressMeta(API_05_SharedMetadata.class)
    @State
    public static class VolatileTest {
        volatile int v;

        @Actor
        public void actor1(II_Result r) {
            r.r1 = ++v;
        }

        @Actor
        public void actor2(II_Result r) {
            r.r2 = ++v;
        }
    }
}
```

我们首先来看一下 @JCStressMeta这个注解: 

```java
/**
 * {@link JCStressMeta} points to another class with test meta-information.
 *	@JCStressMeta注解将指向另一个包含测试元信息的类。
 * <p>When placed over {@link JCStressTest} class, the {@link Description}, {@link Outcome},
 * {@link Ref}, and other annotations will be inherited from the pointed class. This allows
 * to declare the description, grading and references only once for a group of tests.</p>
 *  JCStressMeta 
 * 当它和@JCStressTest一起出现，@Description、@Outcome、@Ref等其他注解将会从指向的类继承。这样就可以在一组测试中只声明一次描* 述、评分和参考信息
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface JCStressMeta {

    Class value();

}
```

### API_06_Descriptions

```java
/*
    JCStress also allows to put the descriptions and references right at the test.
    This helps to identify the goal for the test, as well as the discussions about
    the behavior in question.
	JCStress 还允许将引用和描述放入测试中,这将帮我们认清测试目标, 以及相关问题的讨论
    How to run this test:
       $ java -jar jcstress-samples/target/jcstress.jar -t API_06_Descriptions
 */

@JCStressTest

// Optional test description
@Description("Sample Hello World test")

// Optional references. @Ref is repeatable.
@Ref("http://openjdk.java.net/projects/code-tools/jcstress/")

@Outcome(id = "1, 1", expect = ACCEPTABLE_INTERESTING, desc = "Both actors came up with the same value: atomicity failure.")
@Outcome(id = "1, 2", expect = ACCEPTABLE,             desc = "actor1 incremented, then actor2.")
@Outcome(id = "2, 1", expect = ACCEPTABLE,             desc = "actor2 incremented, then actor1.")
@State
public class API_06_Descriptions {

    int v;

    @Actor
    public void actor1(II_Result r) {
        r.r1 = ++v;
    }

    @Actor
    public void actor2(II_Result r) {
        r.r2 = ++v;
    }
}
```

## 总结一下

本篇我们介绍的JCStress注解有:

- @JCStressTest  标定这个类需要被JCStress测试
- 输入参数 II_Result 用于记录测试结果 ，这个类型的结果几个I，就是由几个变量，记录的结果最后会被用逗号拼接，然后和outcome注解的id来匹配。
- Outcome 的id属性用于设置我们希望测出来的结果，expect 用于标定测试等级。
- @Actor 标定的方法 会被一个线程执行，只有类上出现了@State，类上的方法才可以允许出现@Actor注解。
-  @Arbiter 在actor方法之后执行
- @Signal 被用于在中断测试中发送中断信号给Actor。
- 你也可以将一些测试放在一个类中
- 有的时候 你想做的并发测试，预期输出结果都一样，我们可以用@JCStressMeta注解，共享属性。
- 有的时候我们也想和别人讨论并发问题，并希望在测试输出的文档中附带参考链接和描述，我们可以用@Ref和@Description。

## 写在最后

这个框架倒是我头一次只看官方文档，根据例子写的教程，探索的过程倒是摔了不少跟头，刚开始不想用命令行，想用IDEA的一个插件JCStress来跑，换了一台电脑之后，发现这个插件有点水土不服，本来想看下问题出在哪里，但是想想要花不少的时间，这个想法也就打消了。其实本篇还打算介绍一下线程中断机制，但是感觉这个内容一篇之内也介绍不完，所性也砍了，本来想解读测试报告的时候，把JCStress的执行机制也顺带介绍一下，Java的两个编译器C1、C2。但是刚敲几个字就发现也是不小的篇幅，所性也砍了。后面我们介绍JMM，将用JCStress来验证我们的猜想。





## 参考资料

[1] Difference between pre-increment and post-increment in a loop?  https://stackoverflow.com/questions/484462/difference-between-pre-increment-and-post-increment-in-a-loop

[2] Why is i++ not atomic?  https://stackoverflow.com/questions/25168062/why-is-i-not-atomic



