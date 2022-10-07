# 代理模式-AOP绪论
>  本篇可以理解为Spring AOP的铺垫，以前大概会用Spring框架提供的AOP，但是对AOP还不是很了解，于是打算系统的梳理一下代理模式和AOP。

## 简单的谈一下代理
软件世界是对现实世界抽象，所以软件世界的某些概念也不是凭空产生，也是从现实世界中脱胎而来，就像多态这个概念一样，Java官方出的指导书 《The Java™ Tutorials》在讲多态的时候，首先提到多态首先是一个生物学的概念。那么设计模式中的代理模式，我们也可以认为是从现实世界中抽象而来，在现实世界代理这个词也是很常见的，比如房产中介代理房东的房子，经纪人代理明星谈商演。基本上代理人都在一定范围内代理被代表人的事务。

  那我们为什么需要代理模式呢?  我们事实上也可以从现实世界去寻找答案，为什么房东要找房产中介呢? 因为房东也是人，也有自己的生活，不可能将全身心都放置在将自己的房屋出租上，不愿意做改变。那么软件世界的代理模式，我认为也是有着同样的原因，原来的某个对象需要再度承接一个功能，但是我们并不愿意修改对象附属的类，原因可能有很多，也许旧的对象已经对外使用，一旦修改也许会有其他意想不到的反应，也许我们没办法修改等等。原因有很多，同时对一个过去的类修改也不符合开闭原则，对扩展开放，对修改封闭。于是我们就希望在不修改目标对象的功能前提下，对目标的功能进行扩展。

网上的大多数博客在讲代理模式的时候，大多都会从一个接口入手,然后需求是扩展实现对该接口实现类的增强，然后写代码演示。像是下面这样: 
```java
public interface IRentHouse {
    /**
     * 实现该接口的类将能够对外提供房子
     */
    void rentHouse();
}
public class Landlord implements IRentHouse {
    @Override
    public void rentHouse() {
        System.out.println(" 我向你提供房子..... ");
    }
}
public class HouseAgent implements IRentHouse {

    @Override
    public void rentHouse() {
        System.out.println("在该类的方法之前执行该方法");
        IRentHouse landlord = new Landlord();
        landlord.rentHouse();
        System.out.println("在该类的方法之后执行该方法");
    }
}
public class Test {
    public static void main(String[] args) {
        IRentHouse landlord = new HouseAgent();
        landlord.rentHouse();
    }
}
```
测试结果: 
![1614393461311](https://tva1.sinaimg.cn/large/006e5UvNly1gzqz3m3ifcj30wy07c755.jpg)
许多博客在刚讲静态代理的时候通常会从这里入手，在不改变原来类同时，对原来的类进行加强。对，这算是一个相当现实的需求，尤其是在原来的类在系统中使用的比较多的情况下，且运行比较稳定，一旦改动就算是再小心翼翼，也无法保证对原来的系统一点影响都没有，最好的方法是不改，那如何增强原有的目标对象，你新增强的一般就是要满足新的调用者的需求，那我就新增一个类吧，供你调用。很完美，那问题又来了，为什么各个博客都是从接口出发呢？

因为代理类和目标类实现相同接口，是为了尽可能的保证代理对象的内部结构和目标对象保持一致，这样我们对代理对象的操作都可以转移到目标对象身上，我们只用着眼于增强代码的编写。

从面向对象的设计角度来看,如果我这个时候我不放置一个顶层的接口，像上面我将接口中的方法移动至类中，不再有接口。那你这个时候增强又改该怎么做呢？这就要聊聊类、接口、抽象类的区别了，这也是常见的面试题，在学对象的时候，我们常常和过去的面向过程进行比较，强调的一句话是类拥有属性和行为，但在面向过程系语言本身是不提供这种特性的。在学习Java的时候，在学完类，接着就是抽象类和接口，我们可以说接口强调行为，是一种契约，我们进行面向对象设计的时候，将多个类行为抽象出来，放到接口中，这样扩展性更强，该怎么理解这个扩展性更强呢？就像上面那样，面向接口编程。

假设我们顽固的不进行抽象，许多类中都放置了相同的方法，那么在使用的时候就很难对旧有的类进行扩展，进行升级，我们不得不改动旧有的类。那抽象类呢？ 该怎么理解抽象类呢? 如果接口是对许多类行为的抽象，那么抽象类就是对这一类对象行为的抽象，抽象的层次是不一样的。就像是乳制品企业大家都要实现一个标准，但是怎么实现的国家并不管。抽象类抽象的是鸟、蜂鸟、老鹰。这一个体系的类的共性，比如都会飞。

其实到这里，静态代理基本上就讲完了，代理模式着眼于在不改变旧的类的基础上进行增强，那么增强通常说的就是方法，行为增强，属性是增加。那么为了扩展性强，我们设计的时候可以将行为放置在接口中或者你放在抽象类里也行，这样我们就可以无缝增强。关于设计模式，我觉得我们不要被拘束住，我觉得设计模式是一种理念，践行的方式不一样而已。

静态代理着眼于增强一个类的功能，那么当我们的系统中有很多类都需要增强的时候，就有点不适合了，假设有三十个类都需要增强，且设计都比较好，都将行为抽象放置到了接口中，这种情况下，你总不能写三十个静态代理类吧。当然不能让我们自己写，我们让JVM帮我们写。这也就是动态代理。

## 动态代理

### 换个角度看创建对象的过程
对于Java程序员来说，一个对象的创建过程可能是这样的:
![1614396121343](https://tvax2.sinaimg.cn/large/006e5UvNly1gzqz4g5q7nj31a20ivjtb.jpg)
我们在思考下，将面向对象的这种思想贯彻到底，思考一下，作为Java程序员我们使用类来对现实世界的事物进行建模，那么类的行为是否也应该建模呢？也就是描述类的类，也就是位于JDK中的类: java.lang.Class。每个类仅会有一个Class对象，从这个角度来看，Java中类的关系结构如下图所示: 
![1614397435334](https://tvax3.sinaimg.cn/large/006e5UvNly1gzqz4udn3kj30v40ekdk2.jpg)
所以假如我想创建一个对象，JVM的确会将该类的字节码加载进入JVM中，那么在该对象创建之前，该对象的Class对象会先行创建,所以对象创建过程就变成了下面这样: 
![1614398753119](https://tvax3.sinaimg.cn/large/006e5UvNly1gzqz57o1q1j31d90jrjuh.jpg)
在创建任何类的对象之前，JVM会首先创建给类对应的Class对象，每个类仅对应一个，如果已经创建则不再创建。然后在创建该类的时候，用于获取该类的元信息。

### 基于接口的代理

我们这里再度强调一下我们的目标: 

- 我们有一批类，然后我们想在不改变它们的基础之上，增强它们, 我们还希望只着眼于编写增强目标对象代码的编写。
- 我们还希望由程序来编写这些类，而不是由程序员来编写，因为太多了。

第一个目标我们可以让目标对象和代理对象实现共同的接口，这样我们就能只着眼于编写目标对象代码的编写。

那第二个目标该如何实现呢? 我们知道接口是无法实例化的，我们上面讲了目标对象有一个Class类对象，拥有该类对象的构造方法，字段等信息。我们通过Class类对象就可以代理目标类，那关于增强代码的编写，JDK提供了java.lang.reflect.InvocationHandler(接口)和 java.lang.reflect.Proxy类帮助我们在运行时产生接口的实现类。


我们再回想一下我们的需求，不想代理类，让JVM写。那么怎么让JVM知道你要代理哪个类呢？一般的设计思维就是首先你要告知代理类和目标类需要共同实现的接口，你要告知要代理的目标类是哪一个类由哪一个类加载器加载。这也就是Proxy类的: 
getProxyClass方法，我们先大致看一下这个方法: 

```
public static Class<?> getProxyClass(ClassLoader loader,  Class<?>... interfaces)throws IllegalArgumentException
```

该方法就是按照上述的思想设计的，第一个参数为目标类的类加载器，第二个参数为代理类和目标类共同实现的接口。
那增强呢?  说好的增强呢? 这就跟上面的InvocationHandler接口有关系了，通过getProxyClass获取代理类，这是JDK为我们创建的代理类，但是它没有本体(或者JDK在为我们创建完本地就把这个类删除掉了)，只能通过Class类对象，通过反射接口来间接的创建对象。
所以上面的静态代理如果改造成静态代理的话，就可以这么改造:
```java
 private static void dynamicProxy() throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        /**
         *  第一个参数为目标类的加载器
         *  第二个参数为目标类和代理类需要实现的接口
         */
        Class<?> rentHouseProxy = Proxy.getProxyClass(IRentHouse.class.getClassLoader(), IRentHouse.class);
        //这种由JDK动态代理的类，会有一个参数类型为InvocationHandler的构造函数。我们通过反射来获取
        Constructor<?> constructor = rentHouseProxy.getConstructor(InvocationHandler.class);
        // 通过反射创建对象，向其传入InvocationHandler对象，目标类和代理类共同实现的接口中的方法被调用时,会先调用                               
        // InvocationHandler的invoke方法有目标对象需要增强的方法。为目标对象需要增强的方法调用所需要的的参数
        IRentHouse iRentHouseProxy = (IRentHouse) constructor.newInstance(new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                IRentHouse iRentHouse = new Landlord();
                System.out.println("方法调用之前.............");
                Object result = method.invoke(iRentHouse, args);
                System.out.println("方法调用之后.............");
                return result;
            }
        });
        iRentHouseProxy.rentHouse();
    }   
```
上面这种写法还要我们在调用的时候显式的new一下我们想要增强的类，属于硬编码，不具备通用性，假设我想动态代理另一个类，那我还得再写一个吗？ 事实上我还可以这么写: 
```
   private static Object getProxy(final Object target) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

        Class<?> proxyClazz = Proxy.getProxyClass(target.getClass().getClassLoader(), target.getClass().getInterfaces());
        Constructor<?> constructor = proxyClazz.getConstructor(InvocationHandler.class);
        Object proxy = constructor.newInstance(new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println(method.getName() + "方法开始执行..........");
                Object object = method.invoke(target, args);
                System.out.println(method.getName() + "方法执行结束..........");
                return object;
            }
        });
        return proxy;
    }
```
这样我们就将目标对象，传递进来了，通用性更强。事实上还可以这么写: 
```
private static Object getProxyPlus(final Object target) {
        Object proxy = Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("方法开始执行..........");
                Object obj = method.invoke(target, args);
                System.out.println("方法执行结束..........");
                return obj;
            }
        });
        return proxy;
    }
```
### CGLIB代理简介
然后我想增强一个类，这个类恰巧没有实现接口怎么办？ 这就需要Cglib了，其实道理倒是类似的，你有接口，我就给你创建实现类，你没接口还要增强，我就给你动态的创建子类。通过“继承”可以继承父类所有的公开方法，然后可以重写这些方法，在重写时对这些方法增强，这就是cglib的思想。

Cglib代理一个类的通常思路是这样的，首先实现MethodInterceptor接口，MethodInterceptor接口简介: 
![1614415722058](https://tva3.sinaimg.cn/large/006e5UvNly1gzqz79r4asj30nv0hv775.jpg)
我们可以这么实现: 

```java
public class MyMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("方法执行前执行");
        System.out.println(args);
        Object returnValue = methodProxy.invokeSuper(obj, args);
        System.out.println("方法执行后执行");
        return returnValue;
    }
}
```
注意调用在intercept方法中调用代理类的方法，会再度回到intercept方法中，造成死循环。intercept就可以认为是代理类的增强方法，自己调用自己会导致递归，所以搜门上面的调用是用methodProxy调用继承的父类的函数，这就属于代理类。
测试代码: 
```java
private static void cglibDemo() {
        // 一般都是从Enhancer入手
        Enhancer enhancer = new Enhancer();
        // 设置需要增强的类。
        enhancer.setSuperclass(Car.class);
        // 设置增强类的实际增强者
        enhancer.setCallback(new MyMethodInterceptor());
        // 创建实际的代理类
        Car car = (Car) enhancer.create();
        System.out.println(car.getBrand());
    }
```
这种增强是对类所有的公有方法进行增强。这里关于Cglib的介绍就到这里，在学习Cglib的动态代理的时候也查了网上的一些资料，怎么说呢? 总是不那么尽如人意，总是存在这样那样的缺憾。想想还是自己做翻译吧，但是Cglib的文档又稍微有些庞大，想想还是不放在这里吧，希望各位同学注意体会思想就好。

## 总结一下

不管是动态代理还是静态代理都是着眼于在不修改原来类的基础上进行增强，静态代理是我们手动的编写目标类的增强类，这种代理在我们的代理类有很多的时候就有些不适用了，我们并不想为每一个需要增强的累都加一个代理类。这也就是需要动态代理的时候，让JVM帮我们创建代理类。创建的代理类也有两种形式，一种是基于接口的(JDK官方提供的)，另一种是基于类的(Cglib来完成)。基于接口的是在运行时创建接口的实现类，基于类是在运行时创建需要增强类的子类。



## 参考资料
- [设计模式（四）——搞懂什么是代理模式]            (https://zhuanlan.zhihu.com/p/70098824)
- [Java 动态代理作用是什么？]                       (https://www.zhihu.com/question/20794107)
- [Java bean 是个什么概念？]                  (https://www.zhihu.com/question/19773379/answer/31625054)
- [类加载与Class对象]                          (https://zhuanlan.zhihu.com/p/60294147)
- [CGLIB(Code Generation Library)详解]                   (https://blog.csdn.net/danchu/article/details/70238002)
- [Java两种动态代理JDK动态代理和CGLIB动态代理]                          (https://blog.csdn.net/flyfeifei66/article/details/81481222?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&dist_request_id=c81d098d-b936-4863-8f44-cce026f4f914&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control)
- [Java Proxy和CGLIB动态代理原理]            (https://www.cnblogs.com/CarpenterLee/p/8241042.html)