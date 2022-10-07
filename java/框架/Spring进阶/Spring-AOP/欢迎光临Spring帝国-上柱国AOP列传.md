# 欢迎光临Spring时代(二) 上柱国AOP列传

> 这里其实讲的是使用如何在Spring Framework如何使用AOP。在看本篇之前，建议先看《代理模式-AOP绪论》，《代理模式-AOP绪论》是本篇的基础理论，不看也行，本篇讲的也就是基本使用。


## AOP 简论
AOP: Aspect oriented programming 面向切面编程，是OOP(面向对象编程)的扩展。
如果说OOP是解决面向过程中遇到的一些问题(比如代码复用性不强等)的话，那么AOP解决的就是通用代码重复编写的问题。
这么说可能有点抽象，我们向现实的场景拉一拉，我们的方法通常是一步一步执行的，不同的方法有不同的调用时机。
![1614417184935](https://tvax4.sinaimg.cn/large/006e5UvNly1gzqzep6bovj31bk0g7t9w.jpg)
但总有一些方法似乎是在所有的代码流程里面都需要，比如检测是否登录，记录执行时间等。但是我们总不想做重复的事情，就是在每个代码中写一遍，我们的愿望是写一次，然后标定哪些代码在哪里需要执行呢? AOP解决的就是这类问题。那么这样做的话，我们首先就要标明哪里是需要重复执行的，也就是注入点，用AOP的术语就是切入点(pointcut)， 第二个就是声明在切入点在满足哪些时机时注入，比如可以选择一个方法在执行时注入，在方法发生异常时注入。像Spring Proxy并不支持多种多样的连接点，百分之九十九的情况是在方法执行的时候注入。这个时间我们用术语称之为连接点(joinPoint)。第三个就是注入的时机，是在方法前还是方法后，还是方法前后都来一遍。这个概念我们一般称之为建议(advice)。

pointCut+advice可以描述一段代码在什么时机注入(jointPoint)，注入到哪里(pointCut)，而被注入的代码我们用切面(Aspect)这个概念来描述，这也就是面向切面编程 (Aspect oriented programmin)。
![1614435175698](https://tvax1.sinaimg.cn/large/006e5UvNly1gzqzffjraaj31370hcacz.jpg)

仔细想一下，上面讲的AOP中的建议，是不是跟动态代理中的在不修改原来类的基础上增强一个类有点像呢? 没错AOP是借助于动态代理来实现的，如果你不懂什么是动态代理，请参看这篇文章:
- 代理模式-AOP绪论

其实不懂动态代理，也能看懂本篇。本篇主要介绍的是AOP在Spring中是如何使用的，AOP是一种理念，Spring实现了AOP。只不过理解动态代理对Spring实现的AOP会理解更深而已。

## 准备工作
在《欢迎光临Spring时代(一) 上柱国IOC列传》的依赖我们还要接着用，同时我们还要再补充以下依赖: 
```xml
<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-aop</artifactId>
			<version>5.2.8.RELEASE</version>
		</dependency>
<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjweaver</artifactId>
			<version>1.9.5</version>
</dependency>
```
不会maven想下jar包的，在《欢迎光临Spring时代(一) 上柱国IOC列传》中已经介绍了下载jar包的方法，这里不再重复介绍了。现在Spring Framework的轻量级已经体现出来了，依赖很少，大致上只需要七个jar包就能帮助我们优雅的管理对象之间的依赖关系和AOP了。

## AOP在Spring中的实现简介
Java中方法依附于类，而要使用Spring提供的AOP也就需要将需要增强的类加入到IOC容器中，而在《欢迎光临Spring时代(一) 上柱国IOC列传》我们已经介绍过了，IOC容器有两种形式，一种是基于配置文件的，一种是基于注解的。我们首先来介绍基于配置文件的。在Spring Framework中我们一般称建议为通知，在方法执行前执行，我们称之为前置通知，在方法执行后执行，我们称之为后置通知，在方法执行之前和在方法执行之后都执行我们称之为环绕通知，在方法发生异常的时候执行我们称之为异常通知。

在AOP中我们介绍了三个概念: 
- 切点(pointCut)描述在哪些方法上执行
- 连接点(joinPoint) 描述了注入时机
- 切面(Aspect) 描述的是被注入代码。
注意这三个概念，我们在用Spring Framework提供的AOP的时候常常会碰到。

如何让一个普通类中的方法称为一个切面呢? 那肯定要让Spring  Framework辨识到这个方法跟别的方法不一样，Spring  Framework提供了以下接口: 
- org.springframework.aop.MethodBeforeAdvice   
> 前置通知 在方法执行之前执行
> - org.springframework.aop.AfterReturningAdvice  
> 后置通知 在方法执行之后执行
> - org.aopalliance.intercept.MethodInterceptor     
> 环绕通知 拦截目标方法的调用，即调用目标方法的整个过程，即可以做到方法执行之前执行、方法执行之后执行、方法发生了异常之后执行
- org.springframework.aop.ThrowsAdvice 
> 异常通知 在方法发生了异常之后执行
> 实现它们，并将它们纳入到IOC容器的管辖，再准备切点，连接点。我们就能做到AOP。

我们上面讲切点是说要增强哪些方法，那方法在Java中应该怎么描述呢? 我们用全类名+方法名？ 这样写起来多麻烦啊！这也就是execution表达式出世的原因，简单点，再简单点

## execution表达式简介
格式通常如下: 
> execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)
- modifiers-pattern?: 指定方法的修饰符(public private protected)，支持通配符，该部分可以省略
- ret-type-pattern: 方法的返回值类型，支持通配符，可使用`*`来拦截所有的返回类型
- declaring-type-pattern?: 该方法所属的类，即全类名+方法名。也可以用*拦截所有方法
- name-pattern: 指定要匹配的方法名，支持通配符，可以使用`*`通配符来匹配所有方法。
- param-pattern: 指定方法声明中的形参列表，支持两个通配符，即 `*` 和..，其中*代表一个任意类型的参数，而..表零个或多个任意类型的参数。例如，() 匹配一个不接受任何参数的方法，而(..) 匹配一个接受任意数量参数的方法，(   `*`   )匹配了一个接受一个任何类型的参数的方法，(`*`,String)匹配了一个接受两个参数的方法，其中第一个参数是任意类型，第二个参数必须是String类型。
- throws-pattern：指定方法声明抛出的异常，支持通配符，该部分可以省略。

## 基于配置文件的AOP

#### 第一种配置文件的AOP方式

### 前置通知、后置通知、环绕通知
首先我们实现MethodBeforeAdvice: 
```java
public class LogBefore implements MethodBeforeAdvice{
    @Override
    public void before(Method method, Object[] args, Object o) throws Throwable {
        System.out.println("在方法之前执行");
    }
}
```
然后将LogBefore 加入到IOC容器中，方式有很多种，可参看《欢迎光临Spring时代(一) 上柱国IOC列传》，这里选取是在配置文件里配置: 
```java
<bean id = "logBefore" class="org.example.aop.LogBefore">
</bean>
 <bean id = "landlord" class = "org.example.aop.Landlord">
  </bean>
```
然后准备切点，并将切点和切面联系起来，如上文所说，Spring Framework并没有给我们提供多少连接点，我们选择的就是在方法执行这个时机，之前、之后，还是环绕。
```xml
<!-- 这个配置是开启AOP用的,记得一定要加-->
  <aop:aspectj-autoproxy/>
<aop:config>
        <aop:pointcut id = "pointCut" expression = "execution(public void org.example.aop.Landlord.rentHouse())"/>
        <aop:advisor advice-ref="logBefore" pointcut-ref="pointCut"></aop:advisor>
</aop:config>  
```
所以我们的切点就是位于org.example.aop包下的Landlord的 返回值类型为void的rentHouse方法。 `<aop:advisor>`用于关联切点和切面。advice-ref指向切面的id，pointcut-ref指向的是切点的id。
我们准备一下代码测试一下: 
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
 private static void testLogAop() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        Landlord landlord = applicationContext.getBean("landlord", Landlord.class);
        landlord.rentHouse();
    }
```
运行结果: ![1614481747612](https://tvax3.sinaimg.cn/large/006e5UvNly1gzqzhf6nxsj31bj07iwo9.jpg)
我放在配置文件中的不是org.example.aop.Landlord，为什么我现在取不出来了啊。因为你此时要增强Landlord中的rentHouse方法，恰巧你实现了接口，那我就用JDK的动态代理给你增强啊。你现再在把配置文件中的AOP配置去掉，就会发现能成功运行了，或者 我们现在换种接值方式，向容器中索取的是IRentHouse类型的Bean就不会出错了, 代码如下: 

```java
  private static void testLogAop() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        IRentHouse landlord = applicationContext.getBean("landlord", IRentHouse.class);
        landlord.rentHouse();
    }
```
这个例子是我特意准备的例子，让需要增强的类实现了一个接口，为的就是与《代理模式-AOP绪论》这篇文章形成映照，如果看过这篇文章的同学，就会知道动态代理还有另一种形式是基于Cglib库来做的，这种增强就是在运行时创建需要增强类的子类，所以此时你将上面的Landlord不实现IRentHouse接口，取Landlord类型的就可以了。这其中就有向上转型、向下转型的知识点。可以仔细体会一下。

后置通知和环绕通知基本上就是照葫芦画瓢而已，跟前置通知类似。后置通知我们实现AfterReturningAdvice接口，环绕通知我们实现MethodInterceptor接口: 
```java
public class LogAfter implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println("这是后置通知...............");
    }
}

public class LogAround implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("这是环绕通知,在方法执行之前执行.........");
        Object result = null;
        try {
         //  invocation 控制目标方法的执行,注释这段,则目标方法不会执行。
         result = invocation.proceed();
        }catch (Exception ex){
            System.out.println("这是环绕通知,在方法执行之后执行.........，可以当做异常通知");
            // ex.printStackTrace();
        }
        System.out.println("这是环绕通知,在方法执行之后执行.........");
        return result;
    }
}
```
在配置文件中配置: 
```xml
 <bean id = "logAfter" class="org.example.aop.LogAfter">
 </bean>
 <bean id = "logAround" class = "org.example.aop.LogAround">
 </bean>
<aop:config>
        <aop:pointcut id = "pointCut" expression = "execution(public void org.example.aop.Landlord.rentHouse())"/>
        <aop:advisor advice-ref="logAround" pointcut-ref="pointCut"></aop:advisor>
    </aop:config>
    <aop:config>
        <aop:pointcut id = "pointCut" expression = "execution(public void org.example.aop.Landlord.rentHouse())"/>
        <aop:advisor advice-ref="logException" pointcut-ref="pointCut"></aop:advisor>
    </aop:config>
    <aop:config>
        <aop:pointcut id = "pointCut" expression = "execution(public void org.example.aop.Landlord.rentHouse())"/>
        <aop:advisor advice-ref="logBefore" pointcut-ref="pointCut"></aop:advisor>
    </aop:config>
    <aop:config>
        <aop:pointcut id = "pointCut" expression = " execution(public void org.example.aop.Landlord.rentHouse())"/>
        <aop:advisor advice-ref="logAfter" pointcut-ref="pointCut"></aop:advisor>
    </aop:config>
```
测试代码: 
```java
private static void testLogAop() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        Landlord landlord = applicationContext.getBean("landlord", Landlord.class);
        landlord.rentHouse();
    }
```
运行结果: 
![1614483219931](https://tva4.sinaimg.cn/large/006e5UvNly1gzqzikvlzgj30yu04k408.jpg)

### 异常通知
异常通知跟其他接口的不一样之处，就在于这是一个空接口: 

![1614483315311](https://tva1.sinaimg.cn/large/006e5UvNly1gzqzjo9su3j30px05j0tn.jpg)

那么在方法在执行过程中，发生了异常，Spring该怎么回调这个方法呢? 别着急我们先看注释: 

![1614483582052](https://tvax3.sinaimg.cn/large/006e5UvNly1gzqzk9x05zj30wx0f9qe0.jpg)

问题解决了,我们照注释要求的来: 

```java
public class LogException implements ThrowsAdvice {
    public void afterThrowing(Method method, Object[] args, Object target, Exception ex){
        System.out.println(method.getName()+"方法发生了异常");
    }
}
```
然后再度测试，还是上面的测试代码，在目标代码中请做出一个Exception的子类，我做的是除零异常: 
![1614483748129](https://tva1.sinaimg.cn/large/006e5UvNly1gzqzkt686oj30yr08mn0k.jpg)

#### 第二种配置文件的AOP方式
上面实现的各种通知都是基于接口的，如果你不想实现接口，Spring也能将一个普通的类变成通知类，写到这里有想到了动态代理，想必还是基于动态代理的第二种形式来做的。我们首先准备一个通知类: 

```
Component
public class LogSchema {
    public void before(JoinPoint joinPoint) {
        System.out.println("z方法调用执行之前执行");
    }
    public void invoke(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("z环绕通知的前置通知........");
        Object result = null;
        try {
            // 控制方法的使用
            result = joinPoint.proceed();
            System.out.println("z打印方法的返回值..........." + result);
        } catch (Exception e) {
            System.out.println("z环绕通知的异常通知........");
        }
        System.out.println("z环绕通知的后置通知........");
    }
    public void after(JoinPoint joinPoint, Object returningValue) {
        System.out.println("z方法的返回值是:" + returningValue);
        System.out.println("z方法调用执行之后执行");
    }
    public void afterThrowing(JoinPoint joinPoint, ArithmeticException ex) {
        System.out.println("z这是异常通知的通知");
    }
}
```
然后将这个类在配置文件变成通知类: 
```xml
<aop:config>
        <aop:pointcut id="schema" expression="execution(public void org.example.aop.Landlord.rentHouse(int ))"/>
        <aop:aspect ref="logSchema">
            <aop:around method = "invoke" pointcut-ref="schema" arg-names="joinPoint"/>
            <aop:before method = "before" pointcut-ref="schema"  arg-names="joinPoint"/>
            <aop:after-returning method="after"  returning="returningValue" arg-names="joinPoint,returningValue" pointcut-ref="schema"/>
            <aop:after-throwing method="afterThrowing" pointcut-ref="schema" arg-names="joinPoint,ex" throwing="ex"/>
        </aop:aspect>
    </aop:config>
```
测试代码: 
![1614492156788](https://tva4.sinaimg.cn/large/006e5UvNly1gzqzlkx1jtj311809s78h.jpg)

## 基于注解的AOP
接下来我们介绍基于注解形式的Spring AOP，关于注解其实也有几种不同的写法，主要区别在于取参数的方式不同。一种是在注解中绑定参数，一种是用JoinPoint对象获取参数。两种我们都介绍，在使用注解的方式开发AOP之前，我们首先要Spring对AOP的支持，如果你是配置文件请在配置文件中加上: `<aop:aspectj-autoproxy/>`,如果是配置类请在配置类上加上@EnableAspectJAutoProxy注解。不然Spring就认为你不想用AOP。

### 基于@Pointcut的写法

#### 取目标方法参数的第一种形式
```java
@Aspect
// 该注解将该类纳入IOC容器
@Component 
public class LogAspectAnnotation {

    /**
     * Pointcut注解用于定义切点,可以被其他方法引用。
     * 相当于配置文件的: <aop:pointcut id = "pointCut" expression = "execution(public void    &org.example.aop.Landlord.rentHouse())"/>
     * Pointcut有两个属性一个是value用于指定execution表达式,argNames用于匹配参数取参数。
     * 记得语法是跟在配置文件中不同的是写完execution表达式,用&&args写参数名,多的用逗号隔开,在argNames中写对应的参数名
     * 然后在切点的方法中也要写,用于绑定参数。
     * @param i
     * @return
     */
    @Pointcut(value = "execution(public void org.example.aop.Landlord.rentHouse(int)) && args(i)",argNames = "i")
    public int  pointCut(int i){
        return i;
    }
    @Before(value = "pointCut(i)",argNames = "i")
    public void before(int i){
        System.out.println("方法调用执行之前执行");
    }
}
```
#### 取目标方法参数的第二种形式
```java
@Pointcut(value = "execution(public void org.example.aop.Landlord.rentHouse(int))")
    public void  pointCut(){
    }
    @Before(value = "pointCut()")
    public void before(JoinPoint joinPoint){
        for (Object arg : joinPoint.getArgs()) {
            System.out.println(arg);
        }
        System.out.println("方法调用执行之前执行");
    }
```
JoinPoint 可以获取目标类的所有元数据，我们来大致看一下JoinPoint类: 
![1614488941645](https://tva2.sinaimg.cn/large/006e5UvNly1gzqzmw8a43j30nl0hzdo5.jpg)

#### 其实还可以这么写
```java
// value 就直接相当于切点
@Before(value = "execution(public void org.example.aop.Landlord.rentHouse(int))")
    public void before(JoinPoint joinPoint){
        // 这里是打印切点的参数
        for (Object arg : joinPoint.getArgs()) {
            System.out.println(arg);
        }
        System.out.println("方法调用执行之前执行");
    }
```
#### 后置通知、环绕通知、异常通知
```java
@Aspect
@Component
public class LogAspectAnnotation {

    /**
     * Pointcut注解用于定义切点,可以被其他方法引用。
     * 相当于配置文件的: <aop:pointcut id = "pointCut" expression = "execution(public void org.example.aop.Landlord.rentHouse())"/>
     * Pointcut有两个属性一个是value用于指定execution表达式,argNames用于匹配参数取参数。
     * 记得语法是跟在配置文件中不同的是写完execution表达式,用&&args写参数名,多的用逗号隔开,在argNames中写对应的参数名
     * 然后在切点的方法中也要写,用于绑定参数。
     *
     * @param
     * @return
     */
    @Pointcut(value = "execution(public void org.example.aop.Landlord.rentHouse(int))")
    public void pointCut() {
    }

    @Before(value = "pointCut()")
    public void before(JoinPoint joinPoint) {
        System.out.println("方法调用执行之前执行");
    }

    /**
     * 相对于前置通知,AfterReturning中多了一个属性就是returning,用于获取目标方法的返回值。
     * returning中的值要和后置通知的参数名保持一致
     *
     * @param joinPoint
     * @param returningValue
     */
    @AfterReturning(value = "execution(public void org.example.aop.Landlord.rentHouse(int))", returning = "returningValue")
    public void after(JoinPoint joinPoint, Object returningValue) {
        System.out.println("方法的返回值是:" + returningValue);
        System.out.println("方法调用执行之后执行");
    }

    /**
     * 环绕通知用ProceedingJoinPoint来控制方法的执行
     *
     * @param joinPoint
     * @throws Throwable
     */
    @Around("execution(public void org.example.aop.Landlord.rentHouse(int))")
    public void around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("环绕通知的前置通知........");
        Object result = null;
        try {
            // 控制方法的使用
            result = joinPoint.proceed();
            System.out.println("打印方法的返回值..........." + result);
        } catch (Exception e) {
            System.out.println("环绕通知的异常通知........");
        }
        System.out.println("环绕通知的后置通知........");
    }

    /**
     * throwing属性会将发生异常时的对象传递给方法的参数,所以throwing的属性值和参数名要保持一致
     * 发生了方法中的异常就会触发异常通知,当前方法就是ArithmeticException时触发
     *
     * @param joinPoint
     * @param ex
     */
    @AfterThrowing(value = "execution(public void org.example.aop.Landlord.rentHouse(int))", throwing = "ex")
    public void afterThrowing(JoinPoint joinPoint, ArithmeticException ex) {
        System.out.println("这是异常通知的通知");
        ex.printStackTrace();
    }
}
```
还是上面的测试代码，我在Landlord的rentHouse方法中做了一个1除以0操作，我们来看一下测试结果: 
![1614490626900](https://tvax2.sinaimg.cn/large/006e5UvNly1gzqznke61hj30zb099k03.jpg)

## 总结一下
之前我学Spring Framework的时候就是看的颜群老师在B站发布的视频，这个教程也确实不错，当时学的时候心里还存在些疑问，比如AOP的实现，IOC究竟做了什么，打算以后有时间将这些疑问全部解决等。到现在这些问题算上之前的三篇博客大多都得以解决，这也算是重新梳理了一下自己对Spring Framework的认识，因为我还是希望自己的知识成系统一点，不是零零碎碎的。希望对大家学习Spring Framework有所帮助。

## 参考资料

- [如何用通俗易懂方式解释面向（切面）AOP编程 和AOP的实现方式？]     (https://www.zhihu.com/question/318501676/answer/1606839429)
- [Java进阶知识23 Spring execution 切入点表达式]           (https://www.cnblogs.com/dshore123/p/11823849.html)
- [Spring视频教程]                (https://www.bilibili.com/video/BV1ds411V7HZ)  颜群