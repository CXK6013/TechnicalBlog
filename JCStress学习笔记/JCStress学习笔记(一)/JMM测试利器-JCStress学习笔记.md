# JMM测试利器-JCStress学习笔记

[TOC]

## 前言

我们前文提到，当我们对理论有一些了解，我们就渴望验证，如果无法对理论进行验证，那么我们就可能对理论将信将疑，那对于Java领域的并发理论，一向是难以测试的，更何况调试， 这不仅仅是我们的认知，OpenJDK的开发者也是这么认知的:

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

![image-20230505144522066](C:\Users\chenxingke\AppData\Roaming\Typora\typora-user-images\image-20230505144522066.png)

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



- Actor

> Actor is the central test annotation. It marks the methods that hold the actions done by the threads. The invariants that are maintained by the infrastructure are as follows:
>
> 1. Each method is called only by one particular thread.
> 2. Each method is called exactly once per State instance.

> Note that the invocation order against other Actor methods is deliberately not specified. Therefore, two or more Actor methods may be used to model the concurrent execution on data held by State instance.
> Actor-annotated methods can have only the State or Result-annotated classes as the parameters. Actor methods may declare to throw the exceptions, but actually throwing the exception would fail the test.

## 总结一下





## 跑一下看一下结果







## 跑出重排序







## 总结一下





## 参考资料





