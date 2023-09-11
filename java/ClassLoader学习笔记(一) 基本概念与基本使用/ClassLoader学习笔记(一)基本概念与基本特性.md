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

那有没有别的方式，能够触发这个静态代码块的执行呢? 类整个生命周期会经历加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、初始化(Initialization)、使用(Using)和卸载(Unloading) 七个阶段:

![](https://a.a2k6.com/gerald/i/2023/09/09/7ur1t.jpg)

如上图所示，加载、验证、准备、初始化和卸载这个五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，注意这里说的是按部就班的开始，意思是不是等加载完成之后，触发验证，这些阶段通常都是互相交叉地混合进行的，会在一个阶段执行的过程中调用、激活另一个阶段。解析则不一定，原因在于，它在某些情况下可以在初始化阶段之后，这是为了支持Java语言的运行时绑定特性，也称为动态绑定或晚期绑定。那什么是静态绑定、什么是动态绑定？所谓绑定指的是:

> 





 让我们来看下面这个例子:

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

我们通过语句一和语句二创建了一个SuperClass对象，和一个SubClass对象，因为SuperClass是SubClass的父类，所以我们可以用SuperClass的指向我们创建的子类对象，由于静态方法不会被重载，所以编译器在编译期就知道要调用哪个打印方法，也没有任何歧义。这也就是静态绑定，于编译期就能知道该调用父类的方法。现在让我们将上面的方法稍作改动:

```java
public class GFG {
    public static class Superclass {
        void print() {
            System.out.println("print in superclass is called");
        }
    }
    public static class Subclass extends Superclass {
        @Override
        void print() {
            System.out.println("print in subclass is called");
        }
    }
    public static void main(String[] args) {
        Superclass A = new Superclass();
        Superclass B = new Subclass();
        A.print();
        B.print();
    }
}
```

输出结果为:

```java
print in superclass is called
print in subclass is called
```

 我们将print方法变成了非静态方法，

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

[7] Static Vs. Dynamic Binding in Java https://stackoverflow.com/questions/19017258/static-vs-dynamic-binding-in-java
