# ClassLoader学习笔记(一) 基本概念与基本特性

> DDD 推进不下去，来看看这个。

## 前言

最近打算学习一下Unsafe，然后看了一下其中的方法之后，然后轻车熟路的写下以下代码:

```java
public static void main(String[] args) {
    Unsafe unsafe = Unsafe.getUnsafe();
    unsafe.allocateMemory(100);
}
```

然后报了下面这个错:

```java
Exception in thread "main" java.lang.SecurityException: Unsafe
	at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
	at com.example.quicktest.generic.MyNodeTest.main(MyNodeTest.java:27)
```

不让我调是吧，我可以用好多种方式来调用，虽然本质上都是通过反射:

- 方式一:  通过反射调用构造函数，产生Unsafe对象。

```java
Class<Unsafe> unsafeClazz  = Unsafe.class;
Constructor<?> method = unsafeClazz.getDeclaredConstructor();
method.setAccessible(true);
Unsafe unsafe = (Unsafe)method.newInstance();
System.out.println(unsafe.allocateMemory(11));
```

- 方式二:   Unsafe类内部声明了一个成员变量是Unsafe类型, 变量名为: theUnsafe,  在JDK 8 中这个成员变量通过静态代码块进行初始化:

```java
static {
    // 省略无关代码
    theUnsafe = new Unsafe();
}
```

那有没有别的方式，能够触发这个静态代码块的执行呢?  Java是一门面向对象的编程语言，数据与行为的结合体就是对象，对象是具体的，在Java中要定义对象首先就要有一个类，类是对象的模板，在这个模板中定义了对象的行为、成员变量、静态变量等，在Java中要使用对象，首先我们要引入这个类，然后就可以通过类开放的方法来创建对象,  也就是说在使用对象之前JVM里面要将对应的类加载进入JVM，将类加载进入JVM的工作由类加载器承担，想起之前写的一篇文章《今天我们来聊聊单例模式和枚举》中提到的一个类从被加载到虚拟机内存为止，它的整个生命周期会经历加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、初始化(Initialization)、使用(Using)和卸载(Unloading) 七个阶段:

![](https://a.a2k6.com/gerald/i/2023/09/09/7ur1t.jpg)

如上图所示，加载、验证、准备、初始化和卸载这个五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，注意这里说的是按部就班的开始，意识是不是等加载完成之后，触发验证，这些阶段通常都是互相交叉地混合进行的，会在一个阶段执行的过程中调用、激活另一个阶段。解析则不一定，原因在于，它在某些情况下可以在初始化阶段之后，这是为了支持Java语言的运行时绑定特性，也称为动态绑定或晚期绑定。那什么是静态绑定、什么是动态绑定？ 让我们来看下面这个例子:

```java
public class NewClass {
   public static  class SuperClass {
     static void print(){
         System.out.println("print() superclass is called");
     }
   }
   public static class SubClass extends SuperClass {
       static void print(){
           System.out.println("print() subclass is called");
       }
   }

    public static void main(String[] args) {
        // 这里只是做演示，我们一般还是用类名.静态方法名调用静态方法
        SuperClass a = new SuperClass(); // 语句一
        SuperClass b = new SubClass();  //  语句二
        a.print();
        b.print();
    }
}
```

打印的结果是:  

```java
print() superclass is called
print() superclass is called
```





- 加载:  

  

- 链接

  - 验证
  - 准备
  - 解析

- 初始化

- 使用

- 卸载

//  到1点 不管写没写完，都切到一周所见所得

所以我们用反射来触发    

```java
public static Unsafe getUnsafe(){
   try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return  (Unsafe)field.get(null);
     } catch (Exception e) {
            e.printStackTrace();
            return null;
     }
   }
}
```

于是看Unsafe的源码:

```java
@CallerSensitive
public static Unsafe getUnsafe() {
   Class var0 = Reflection.getCallerClass(); // 语句一
  if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
       throw new SecurityException("Unsafe");
   } else {
      return theUnsafe;
   }
}
```

语句一我们通过方法名推断，是获取调用这个方法的类，我们调试一下验证一下我们的猜想:

![](https://a.a2k6.com/gerald/i/2023/09/09/4i47z.jpg)

还挺神奇,getCallerClass是一个native方法，这里就不做过多深究，在getUnsafe方法中，通过语句一拿到了调用类，然后再通过调用类的Class对象获取类加载器，然后调用isSystemDomainLoader方法: 

```java
public static boolean isSystemDomainLoader(ClassLoader var0) {
    return var0 == null;
}
```

逻辑很简单，我们在上面打断点的时候已经看到，调用类的类加载器不为空，所以这里返回false，然后抛出异常。注意到调用类的类加载器是AppClassLoader。我们本篇的主题 

这里引出类加载过程 双亲委派 Tomcat 破坏双亲委派 热加载



## 写在最后





##  参考资料

[1] Why Developers Should Not Write Programs That Call 'sun' Packages https://www.oracle.com/java/technologies/faq-sun-packages.html

[2] 如何查找 jdk 中的 native 实现 https://gorden5566.com/post/1027.html

[3] Java (programming language): What is the life cycle of an object in Java? https://www.quora.com/Java-programming-language-What-is-the-life-cycle-of-an-object-in-Java

[4] Chapter 5. Loading, Linking, and Initializing https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html

[5] 《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》周志明 著

[6] Static vs Dynamic Binding in Java  https://www.geeksforgeeks.org/static-vs-dynamic-binding-in-java/
