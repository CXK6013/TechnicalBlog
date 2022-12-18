# JDK 新特性学习笔记之模块系统

> 有两条小鱼快乐地游着，碰到一条老鱼从对面游过来。老鱼向他们点头问好：「早上好啊小伙子们，今天的水怎么样？」两条小鱼接着游了一会儿，突然停了下来，一脸懵逼地看着对方：水是个什么东西？

> 习以为常的就是水

> 模块系统是JDK 9的特性，后面的JavaFX学习笔记都会基于JDK 11，甚至更高版本。同时这个特性也是我比较感兴趣的，进一步强化了Java的封装能力。

[TOC]

## 回顾Java的特性

我想起刚毕业找工作时背的面试题，面向对象的三大特性是什么？ 想到这个问题我一瞬间居然没想到答案，愣了一下才想起答案:

- 封装
- 继承
- 多态

那什么是封装呢？ 我的答案是隐藏复杂实现过程对外提供功能，像智能手机一样，我们直接操作屏幕就能实现听音乐，看视频，打电话等操作。我觉得我的概括还是相当精准的，但我在搜索引擎上搜索了一下发现，我弄混了封装和信息隐藏，所谓封装是将数据和操作数据的方法都放在类里面，像胶囊一样:
![胶囊](http://tva1.sinaimg.cn/large/006e5UvNgy1h9012qvsjmj30j40atdhi.jpg)

封装是指将数据与操作该数据的方法捆绑在一起。而信息隐藏是隐藏实现细节。封装和信息隐藏常常出现在一起，以致于它们几乎成了同义词，在一些上下文，它们也的确是同义词。封装提供了边界，而信息隐藏则屏蔽复杂实现，这两个常常出现在一起，我们在封装的同时使用信息隐藏。

那继承呢？ 继承源于共性，不同的对象之间具备共性，那我们建模的时候就可以将共性抽出，将其当作父类，从而减少代码冗余，增强代码的简洁性。那多态呢？在《**The Java™ Tutorials**》(这个Java官方出的教程)在介绍多态的时候是这么介绍的:

> **The dictionary definition of** **polymorphism** **refers to a principle in biology in which an organism or species can have many different forms or stages. This principle can also be applied to object-oriented programming and languages like the Java language. Subclasses of a class can define their own unique behaviors and yet share some of the same functionality of the parent class.**
>
> 多态性的字典定义是指生物学中的一个原则，其中一个生物体或物种可以有许多不同的形式或阶段。这一原则也适用于面向对象编程和Java语言等语言。类的子类可以定义自己独特的行为，但也可以共享父类的一些相同功能。

多态进一步细分，功能(函数)重载与对象重载，函数重载就意味着函数在拥有不同的类型的参数或者不同数量参数的时候拥有相同的方法名。而对象则与继承有关，一个父类可以有多个子类，子类可以复用父类的行为，当然也可以进行重写，借助多态可继承我们重用已有的代码。

## 目前的问题

在《让我们来聊聊前端的工程化》我们已经讨论过了软件危机，这里再回忆一下:

> 1970年代和1980年代的软件危机。在那个时代，许多软件最后都得到了一个悲惨的结局，软件项目开发时间大大超出了规划的时间表。一些项目导致了财产的流失，甚至某些软件导致了人员伤亡。同时软件开发人员也发现软体开发的难度越来越大。在软件工程界被大量引用的案例是Therac-25的意外：在1985年六月到1987年一月之间，六个已知的医疗事故来自于Therac-25错误地超过剂量，导致患者死亡或严重辐射灼伤。
>
> 鉴于软件开发时所遭遇困境，北大西洋公约组织在1968年举办了首次软件工程学术会议，并于会中提出"软件工程"来界定软件开发所需相关知识，并建议"软件开发应该是类似工程的活动"。软件工程自1968年正式提出至今，这段时间累积了大量的研究成功案例，广泛地进行大量的技术实践，借由学术界和产业界的共同努力，软件工程正逐渐发展成为一门专业学科。
>
> 关于软件工程的定义，在GB/T11457-2006《消息技术 软件工程术语》中将其定义为"应用计算机科学理论和技术以及工程管理原则和方法，按预算和进度，实现满足用户要求的软件产品的定义、开发、和维护的工程或进行研究的学科"。

> Therac-25: 是加拿大原子能有限公司(AECL) 在 Therac-6 和 Therac-20 装置之后于 1982 年生产的一种计算机控制的放射治疗机。它有时会给患者带来比正常情况高数百倍的辐射剂量，导致死亡或重伤

那为了解决软件危机中的软件开发速度、软件开发越来越复杂，构建软件的编程语言引入了面向对象，着眼于强化于代码的重用性、可维护性。Java中我们如何复用别人的代码呢？ 如果在一个项目里面，我们通常会直接用，即在需要用到的类和方法里面写类名即可，IDE会帮我们自动引入。如果是第三方提供我们是通过jar这样的形式来引入，JDK为我们提供的类库，像HashMap、ArrrayList，这些都放在JDK中的一个jar中。任何一个Java文件总是有一个public class，如果我构建了一个类库，只对外提供一些类和接口该怎么做呢，在JDK 8之前包含JDK8是做不到的，原因在于反射，私有的方法我照样可以通过反射来访问。这为后期的维护带来了很大的麻烦，例如我原先的实现不够好，对外提供的类和接口我不做改动，但我想改动不对外提供类的实现，但改动了不确定有没有人用，别人在升级的时候可能就会问候我两句。再有JDK的jar也太过臃肿，太过庞大了, 只有相当粗略的划分, JDK 8大致提供的jar如下:

- rt.jar 、charset.jar 、ckdrdara.jar、dnsns.jar、jaccess.jar、jce.jar、jfr.jar
- jsse.jar、localedata.jar、management.jar、access-brige-64.jar、sunec.jar
- sunjce.jar、sunjce_provide.jar、sunmscapi.jar、sunpkcs11.jar、zipfs.jar、nashron.jar

这些jar我们好像都不认识，其实rt.jar是我们的老熟人，集合类、并发集合都在里面，所以rt.jar在JDK8有60m大小，Nashron是一个JavaScript引擎。所以如果我们开发服务端应用，我们不需要JavaScript引擎，在安装JDK8的时候这个也会被安装进去，包括AWT、SWING，尽管我们用不到，那我们自然提出这样一个问题，我们能否按需定制自己所需的JDK呢，让打的jar变的小一点。这也就是模块化的缘起。JDK 的模块来自Project Jigsaw，这个项目的主要目标为:

- 使开发人员更容易构建维护库和大型应用程序。
- 提升Java SE平台的安全性和可维护性，特别是对于JDK
- 让Java SE 和 JDK 能够以更小的体积用于小型设备和云部署。

## 模块系统

那么关于模块系统我们自然而然就有以下三个问题:

- 模块系统的目标?

- 什么是模块?

- 该如何使用模块?

### 模块系统的目标

- Reliable configuration(依赖可配置): 

> 模块化让JVM可以识别依赖关系，不管是在编译器还是运行时。系统根据这些依赖关系来确定程序所需要依赖的模块。

- Strong encapsulation (强封装)

> 模块的包只在模块显式的将其导出才可用，即使某个将其导出(export)，另一个模块如果不声明需要(对应require), 那么这个模块也不能使用导出模块的包。这样就提升了Java平台的安全性, 因为模块潜在的访问者更少，模块可以帮助开发者提出更简洁、更合乎逻辑的设计。

- Scalable Java platform (Java平台的可扩展性)

> 以前，Java平台是由一个大量包组成的整体，这给后续的开发和维护带来了相当大的挑战，现在Java平台被模块化为95个模块(这个数字可能随着Java的发展而改变)。现在您就可以自定义运行时，让您的程序可以在运行时拥有仅它需要的模块。例如，如果您只想做服务端开发，不想要GUI模块，那么在打包的时候，就可以不需要。从而显著的减少运行时大小。

这里我的解读是我们平时会讲职责单一，这一概念也应当不仅仅局限于我们的类、方法，同时也应当上升到jar。就rt.jar来言，这个jar太大了，集合类、并发框架、awt和swing的部分类都在其中。rt是run time的缩写，但是对于服务端的运行时一般不会用到这些类，从职责单一角度，awt和swing应当被划分到桌面的jar里面，现在放在rt.jar里，这让rt.jar显得十分庞大。对此我想到的一个比喻是，rt.jar像是一个杂物间，放了太多不应该放的东西。现在模块化对其进行了分类整理，rt.jar被拆分。下面是JDK 9之后的模块化图:

![module-graph](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8ztjkfrysj30m80hsalc.jpg)

 现在最为核心的是java.base，职责更为单一，每个模块都声明了自己依赖于哪些模块，原本这些可能放在文档中，现在这些依赖关系进入了代码。让JDK平台的可维护性得到了进一步的加强。上面这幅图来自GitHub的module-graph。

- Greater platform integrity(更高的平台完整性)

> 在JDK 9之前，许多JDK的类是可以无限制的使用的，尽管对于设计这些的人来说，这些类并不是暴露给开发者使用的。现在通过模块化，再次进行了封装，这些内部API被真正封装。如果您的代码使用了这些内部API，升级到JDK9之上就会出现问题。

大多数是 sun.misc.*打头的包中的类被隐藏，以Unsafe为例，sun.misc.Unsafe被移动到jdk.unsupported模块中，同时在java.base模块克隆了一个jdk.internal.misc.Unsafe类。jdk.internal包不开放给开发者使用。Unsafe在JDK 17上有了更好的替代者, 功能更强大，设计更为优秀，那就是MemorySegment，对于Java程序员来说MemorySegment更为友好。

### 什么是模块？

模块在包之上增加了更高级别聚合，我个人觉得模块相对于jar来说多了描述说明和限制，以前我们看一个jar该如何使用的时候，往往要从文档看起，现在我们可以从模块的描述符来看起, 模块描述符给出了允许外部使用的类。模块由一个唯一命名的包、资源和一个模块描述符(一个java文件)组成. 模块描述符指定了:

- the module’s *name*：模块名称
- the module’s *dependencies* (that is, other modules this module depends on):  模块依赖项
- the packages it explicitly makes available to other modules (all other packages in the module are *implicitly unavailable* to other modules)  允许哪些模块使用。
- the *services it offers* 它提供的服务

- the *services it consumes*  它消费的服务
- to what other modules it allows *reflection*  允许哪些模块反射

###  如何使用？

### 单模块示例

我们讲了那么多理论，现在就来实践一下, 注意模块化是JDK 9提供的特性，所以保证你的JDK版本要在8以上。我们本次演示的IDE是IDEA。

![模块化实例01](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8zvoo247rj30oj085aaw.jpg)

我们基于JDK 11建了一个项目, module-info就是模块描述符:

```java
module com.greetints {
    exports com.greetings;
}
```

module 关键字跟的是模块名, exports跟的是导出的包。我们现在来将这个项目做成jar给外部使用:

![导出模块化jar](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8zvs5opmqj30pf0ejjsz.jpg)

![模块化示例2](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8zvtpjm1fj30sf0hkgmx.jpg)



   ![模块化示例03](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8zvxnzowrj30si0n6gn1.jpg)

​	![模块化示例04](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8zvytwrpoj30wn0h0mz0.jpg)

![模块化示例05](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8zw0izxx0j30ws0crjs3.jpg)

jar会出现在这里:

![模块化示例06](http://tva4.sinaimg.cn/large/006e5UvNgy1h8zw4rzzpvj30r40f43zp.jpg)

然后我们再用IDEA建立一个项目叫moduleDemo02:

![moduleDemo02](http://tva1.sinaimg.cn/large/006e5UvNgy1h8zw2mjbotj30t00auac2.jpg)

然后在这个项目中将这个jar引进来:

![模块化示例07](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8zw68f7gcj30yn0i3wi6.jpg)

![模块化示例08](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8zw7avgf1j30sb0n0ta8.jpg)



![模块化示例09](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8zw8lisgwj30sk0n2dhg.jpg)

​	一般来说在JDK9之前，到这里我们就能用引入jar包的类了, 但是在JDK 9之后，对于模块后的jar，就用不了:

![无法引入](http://tva4.sinaimg.cn/large/006e5UvNgy1h8zwa48c9jj30gg075q35.jpg)

我们需要在module-info里，显式的声明一下我们需要使用引入的jar:

```java
module moduleDemo02 {
    requires com.greetints;
}
```

### 多模块示例

#### exports to

IDEA一次只能打开一个项目，Eclipse一次能打开多个项目，但IDEA支持一个项目多个module，下面的module语法用多模块演示:

![多module结构](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8zxdzp7gkj30re0au0tf.jpg)

moduleDemo01下面有两个文件：	

```java
package com.module01;

/**
 * @author xingke
 * @date 2022-12-11 15:39
 */
public class Module01 {

    public void sayHi(String msg){
        System.out.println(msg);
    }
}
```

```java
module com.module01{
    exports com.module01 to com.module02;
}
```

```java
exports package to module01,module02
代表导出该模块下的包给module01,module02用
仅限module01,和module02用。其他模块无法使用
```

上面我们就在moduleDemo01里面声明了导出com.module01包里面的类给module02使用。我们这个项目里面有三个module，那这就意味着在module03里面无法使用com.module01里面的类。尽管我们在moduleDemo03的文件声明了我们需要module01这个module，但是在IDEA中引入还是报错:

![不被允许使用](http://tva1.sinaimg.cn/large/006e5UvNgy1h8zxlzgytsj30yx0cw75o.jpg)

![Module03里面使用](http://tva4.sinaimg.cn/large/006e5UvNgy1h8zxn22rhlj30lv07s0sr.jpg)

 但在module02里面我们就可以使用, 在使用之前我们首先要在module-info里面声明一下:

![引入module模块](http://tva1.sinaimg.cn/large/006e5UvNgy1h8zxph66mgj30o307b3yz.jpg)

![ModuleDemo02示例](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8zxqlrlp7j30f806u3yv.jpg)

那曲线救国呢，现在moduleDemo02依赖module01，那我在moduleDemo03中引入可以使用moduleDemo01了吗？ 也是不可以的，因为我们上面使用的exports to语法限制了moduleDemo01的类仅能再moduleDemo02里使用。那去掉这个限制呢，也是不行？那我该如何使用呢？ 需要用到transitive关键字去声明：

```java
module com.module02 {
    requires transitive com.module01;
}
# 代表其他模块引入com.module02,也会将com.module01带入
```

 transitive 可以理解为传递依赖, 注意在JDK 9用public表达这个概念，JDK 11模块化被正式确立为永久特性，用transitive 表达。所以选择JDK 学习的时候尽量选择LTS版本。在JDK 里面有这样一个聚合模块叫java.se，这个模块将常用的模块通过transitive聚合在一起，我们引入此模块即能使用java.se开发所需的模块:

![java.se](http://tva4.sinaimg.cn/large/006e5UvNgy1h8zyoqmhjfj30k70cu75z.jpg)

我们在moduleDemo03的module-info文件中引入module02即可:

```java
module com.module03 {
    requires com.module02;
}
```

#### **opens…to**

有同学看到上面可能会想到，虽然模块com.module01里面声明了只给com.module02使用，但是我用反射，我照样也可以使用：

```java
public class Module03 {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clazz = Class.forName("com.module01.Module01");
        Object instance = clazz.getDeclaredConstructor().newInstance();
        for (Method method : clazz.getMethods()) {
            if (method.getName().contains("sayHi")){
                method.invoke(instance, "aaa");
            }
        }
    }
}
```

这种想法在JDK 9 之前是没问题的，在JDK 9之后就会报这个错:

![无权访问](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8zyc8em7rj31ea05e0uc.jpg)

去掉限制我们就可以用反射访问com.module01的类，那如何对反射也进行控制呢? 这也就是opens to语法:

```java
opens package to modulename
```

 允许模块通过反射访问包里非public的方法。

 示例:

```java
module com.module01{
    exports com.module01;
    opens com.module01 to com.module02;
}
```

我们允许com.module02来通过反射访问com.module01中的类里非公开的方法, 如果没有声明，访问非public级别的方法将会报下面的错:

```java
public class Module01 {
     void sayHi(String msg){
        System.out.println(msg);
    }
}
```

我们在module03里面通过反射进行访问:

```java
public class Module03 {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("com.module01.Module01");
        Object instance = clazz.getDeclaredConstructor().newInstance();
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.getName().contains("sayHi")){
                method.setAccessible(true);
                method.invoke(instance, "aaa");
            }
        }
    }
}
```

会报下面这个错: 

![opens报错](http://tva2.sinaimg.cn/large/006e5UvNgy1h8zz9ky9pjj31c508pgo4.jpg)

如果想允许模块下面的所有包的非public方法都可以通过访问，那么可以如下声明:

```java
open module com.module01{
    exports com.module01;
}
```

#### use 和 **provides…with.**

模块之间的桥梁通常会用接口来实现，这样可以实现解耦，提供接口的时候同时提供实现类, 下面是示例:

首先我们要定义接口及其实现:

```java
public interface SayMessage {
    void sayMessage();
}
```

```java
// 注意这两个实现不和sayMessage在一个包下,我们不对外暴露其实现
public class SayMessageImpl01 implements SayMessage {
    @Override
    public void sayMessage() {
        System.out.println("i'm SayMessageImpl01");
    }
}
public class SayMessageImpl02 implements SayMessage {
    @Override
    public void sayMessage() {
        System.out.println("i'm SayMessageImpl02");
    }
}
```

然后在module-info里面声明要暴露的服务:

```java
module com.module01{
    exports com.module01;
    provides com.module01.SayMessage with com.module01.impl.SayMessageImpl01,com.module01.impl.SayMessageImpl02;
}
```

然后在moduleDemo02里面使用moduleDemo01里提供的服务,首先在moduleDemo02的module-info里面声明一下:

```java
module com.module02 {
    requires   com.module01;
    uses com.module01.SayMessage;
}
```

然后我们在代码里面使用即可:

```java
public class ModuleDemo02 {
    public static void main(String[] args) {
        ServiceLoader<SayMessage> load = ServiceLoader.load(SayMessage.class);
        for (SayMessage sayMessage : load) {
            sayMessage.sayMessage();
        }
    }
}
```

 load方法会自动调用SayMessage实现类的无参构造函数, 那这里就会有同学问了，那我能让他调有参的构造函数吗？当然也是可以的，只不过要遵循一定的约定。我们首先改动SayMessageImpl02, 为其提供一个有参的构造函数:

```java
public class SayMessageImpl01 implements SayMessage {

    private String name;

    @Override
    public void sayMessage() {
        System.out.println("i'm SayMessageImpl01"+name);
    }

    public SayMessageImpl01(String name) {
        this.name = name;
    }

    public static SayMessage provider(){
        SayMessage  sayMessage = new SayMessageImpl01("aa");
        return sayMessage;
    }
}
```

然后load在加载实现类的时候就自动的会调用Provider方法, 使用有参的构造。

#### 模块相关的命令行选项

```java
# 列出JDK目前的模块
java --list-modules 
# java --describe-module 模块名
```

我们看下java.xml的模块描述:

![java.xml对外暴露的包](http://tvax1.sinaimg.cn/large/006e5UvNgy1h900s4f7r2j31080fmacb.jpg)

## 总结一下

模块系统让Java轻装出行，这也是趋势，模块系统的到来让JDK的运行时变的更小，可以让我们定制运行时，也能更好的构建大型程序和库。写本篇的时候想起了一个小故事:

> 有两条小鱼快乐地游着，碰到一条老鱼从对面游过来。老鱼向他们点头问好：「早上好啊小伙子们，今天的水怎么样？」两条小鱼接着游了一会儿，突然停了下来，一脸懵逼地看着对方：水是个什么东西？

也许过几年JDK 17全面普及，后面学习Java的人就会觉得模块化是理所当然的，就像水一样，但对于老鱼来说还是会问今天的水怎么样。

## 参考资料

- What is Encapsulation in Java and How to Implement It   https://www.simplilearn.com/tutorials/java-tutorial/java-encapsulation
- **Java 面向对象特征之一：封装和隐藏** https://tiantianliu2018.github.io/2019/10/01/Java-%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%89%B9%E5%BE%81%E4%B9%8B%E4%B8%80%EF%BC%9A%E5%B0%81%E8%A3%85%E5%92%8C%E9%9A%90%E8%97%8F/
- Encapsulation is not information hiding https://www.infoworld.com/article/2075271/encapsulation-is-not-information-hiding.html
- Understanding Java 9 Modules https://www.oracle.com/corporate/features/understanding-java-9-modules.html
- Everything You Need to Know About Polymorphism  https://betterprogramming.pub/everything-you-need-to-know-about-polymorphism-7a7976ca8987
- 如何从 Java 8 升级到 Java 12，升级收益及问题处理技巧  https://www.infoq.cn/article/osqc7lnqyg2hf*lm2h4h
- Java 17 更新（9）：Unsafe 不 safe，我们来一套 safe 的 API 访问堆外内存  https://www.bennyhuo.com/2021/10/02/Java17-Updates-09-foreignapi-memory/
- 神奇的魔法类和双刃剑-Unsafe  https://cloud.tencent.com/developer/article/1650017
- Java模块化开发，Java模块系统精讲 https://www.bilibili.com/video/BV1z5411o7nb?p=2&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799
