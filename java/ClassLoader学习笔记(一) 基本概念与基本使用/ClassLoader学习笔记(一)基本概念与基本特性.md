# ClassLoader学习笔记(一) 基本概念与基本特性

> DDD 推进不下去，来看看这个。

[TOC]

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

## 通过类的初始化来触发

那有没有别的方式，能够触发这个静态代码块的执行呢? 类整个生命周期会经历加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、初始化(Initialization)、使用(Using)和卸载(Unloading) 七个阶段:

![](https://a.a2k6.com/gerald/i/2023/09/09/7ur1t.jpg)

如上图所示，加载、验证、准备、初始化和卸载这个五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，注意这里说的是按部就班的开始，意思是不是等加载完成之后，触发验证，这些阶段通常都是互相交叉地混合进行的，会在一个阶段执行的过程中调用、激活另一个阶段。解析则不一定，原因在于，它在某些情况下可以在初始化阶段之后，这是为了支持Java语言的运行时绑定特性，也称为动态绑定或晚期绑定。那什么是静态绑定、什么是动态绑定？所谓绑定指的是:

> Connecting a method call to the method body is known as Binding.  Static binding uses Type(Class in Java) information for binding while Dynamic binding uses Object to resolve binding.
>
> 将方法调用和方法体连接起来我们称之为绑定，静态绑定使用类型信息，在Java中类型也就是类，动态绑定使用对象来解决绑定问题。

 让我们来看下面这个例子:

```java
// 例子来自于: https://www.geeksforgeeks.org/static-vs-dynamic-binding-in-java/
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
// 例子来自于: https://www.geeksforgeeks.org/static-vs-dynamic-binding-in-java/
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

 我们将print方法变成了非静态方法，并且在SubClass里面重写了print方法，打印结果就发生了改变，原因在于在编译过程中不知道该调用Superclass的print方法，还是Subclass重写的print方法，因为引用变量都是SuperClass类型，因此绑定会延迟到运行时，到运行时根据引用变量指向的对象来决定调用Superclass中的print方法，还是子类重写过的print方法。在不同阶段会有不同的动作被执行:

- 加载:  

  - 通过一个类的全限定名来获取定义此类的二进制字节流。 
  - 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
  - 在内存中生成一个代表这个类的java.lang.Class对象, 作为方法区这个类的各种数据的访问入口。

- 链接

  - 验证

    > 验证是连接阶段的第一步，这一阶段的目的是确保Class文件的字节流中包含的信息符合《Java虚 拟机规范》的全部约束要求，保证这些信息被当作代码运行后不会危害虚拟机自身的安全。

  - 准备

    > 在准备阶段是正式为类中定义的变量(即静态变量，被static修饰的变量，后文简称为类变量)分配内存并设置类变量初始值的阶段，从概念上讲，这些变量所使用的内存都应当在方法区中进行分配，但必须注意到方法区本身是一个逻辑上的区域，在JDK 7 及之前，HotSpot使用永久代来实现方法区，我们可以将其类比为接口和实现类关系，在JDK 8及之后，类变量则会随着Class对象一起存放在Java堆中，我们这个时候使用类变量存储在方法区是一种逻辑概念的描述，因为虚拟机不止hotspot一个。
    >
    > 这里重点说明一下，初始值是指数据类型的零值，假设我们有一个静态变量定义为:

    ```java
    public static int value = 123;
    ```

    那变量value在准备阶段过后的初始值为0而不是123。

  - 解析

    > 解析阶段是Java虚拟机将常量池内的符号引用替换为直接引用的过程，

- 初始化

> 类的初始化阶段是类加载过程的最后一个步骤，前面的几个类加载动作里，除了在加载阶段用户应用程序可以通过自定义类加载的方式局部参与外，其余动作都完全由Java虚拟机来主导控制。直到初始化阶段，Java虚拟机才真正开始执行类中编写的Java程序代码，将主导权移交给应用程序。在准备阶段，变量已经赋过一次系统要求的初始零值(比如int，初始零值就是0)，而在初始化阶段编译器会自动收集类中所有的类变量的赋值动作和静态语句块(static{}块)

- 使用

  > 粗略的说也就是我们可以使用类里面向我们暴露的静态方法、变量，我们可以用类和new来产生对象。

- 卸载

  > 类对应的类加载器被回收的时候，类会被卸载。这意味着，对每个类的引用和对该类ClassLoader本身的引用都需要被淘汰。

那在什么情况下会触发类的初始化动作呢? 

1. 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始 化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有

   1.1 使用new关键字实例化对象的时候。

   1.2 ·读取或设置一个类型的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外） 的时候。

   1.3 调用一个类型的静态方法的时候。

2.   使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需 要先触发其初始化

3.  当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化

4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先 初始化这个主类。

5.  当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解 析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句 柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。

6. 当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有 这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

也就是说我们可以通过反射调用来触发类的初始化从而触发静态代码块的调用，所以我们可以写出下面的代码。

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

## 所谓双亲委派

在上面的过程中，我们其实已经提到类加载器的作用了，见名知义，类加载器就是将类加载进JVM的对象，既然是Java的对象，那我们知道对象是类的实例，那么就有类，这也就是ClassLoader，ClassLoader是一个抽象类，在它的注释中，它是这么介绍自己的:

> 一个类加载器是负责类的对象，ClassLoader类是一个抽象类。 给定一个类的二进制名称，类加载器应该尝试定位或生成该类定义的数据。典型的策略是将类名转换为文件名，然后从文件系统重读取该名称的文件。
>
> 每个类对象都包含都包含加载该类的ClassLoader的引用。
>
> 数组类的类对象不是由类加载器常见的，而是根据Java运行时的需要自动创建的，通过Class.getClassLoader()方法返回的数组类类加载器与其元素类型的类加载器相同；如果数组中的元素类型是基本类型，则数组类没有类加载器。
>
> 开发者可以选择继承ClassLoader，来扩展Java动态加载类的方式。类加载器通常可以被安全管理用来表示安全域(我的解读是在Java中一切代码都在类里面，而类需要类加载器加载，所以我们可以通过类加载来区别当前代码属于什么安全级别)。

> ClassLoader使用委托模型来搜索类和资源，每个ClassLoader实例都有一个关联的父类加载器。当类加载器被请求查找一个类或者资源的时候，当前类加载器并不会直接执行查找和加载动作，而是将查找类或者资源的任务委托给其父加载器。虚拟机内置的类加载器，称之为bootstrap class loader，BootStrapLoader本身没有父类，但可以作为其他类加载器的父类加载器。

听起来是不是有点熟悉，这也就是我们常说的双亲委派机制，这一点我们可以在ClassLoader的源码中直接印证:

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
       // 首先检查这个类是否已经被加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 然后父加载器不为空,请求父加载器加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

看到ClassLoader是一个抽象类，于是我就在其中找抽象方法，然后只找到了一个空方法:

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

方法上的注释是这么写的:

> Finds the class with the specified binary name. This method should be overridden by class loader implementations that follow the delegation model for loading classes, and will be invoked by the loadClass method after checking the parent class loader for the requested class.
>
> 查找具备指定二进制名称的类文件，这个方法应当由遵循委托模型的加载器实现然后重载，然后在调用ClassLoader的loadClass方法中检查父类加载器之后没查找到将会被调用。
>
> The default implementation throws a ClassNotFoundException.
>
> 默认实现的事抛ClassNotFoundException。

我们接着看注释:

> Class loaders that support concurrent loading of classes are known as parallel capable class loaders and are required to register themselves at their class initialization time by invoking the ClassLoader.registerAsParallelCapable method. 
>
> 支持并发加载类的类加载器被称为并行功能加载器，需要在初始化的时候调用ClassLoader的registerAsParallelCapable方法进行注册。

registerAsParallelCapable从JDK 1.7开始加入，在一个普通的Spring Boot web项目中我们来看看调用者有多少:

![](https://a.a2k6.com/gerald/i/2023/09/12/127hb.png)

TomcatEmbeddedWebappClassLoader带个Tomcat，我们点进去看一下, 发现TomcatEmbeddedWebappClassLoader的包名是org.springframework.boot.web.embedded.tomcat，那么这个也就是Spring项目扩展的，TomcatEmbeddedWebappClassLoader继承ParallelWebappClassLoader，ParallelWebappClassLoader继承自WebappClassLoaderBase，继承图如下所示:

![](https://s2.wzznft.com/i/2023/09/12/z6mf6h.png)

然后我们看下WebappClassLoaderBase，就可以看到ClassLoader中的方法被重写了两个:

- loadClass

```java
// 省略部分方法
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (JreCompat.isGraalAvailable() ? this : getClassLoadingLock(name)) {
            if (log.isDebugEnabled()) {
                log.debug("loadClass(" + name + ", " + resolve + ")");
            }
            Class<?> clazz = null;

			// 这个方法检查类加载器是否被停止	
            checkStateForClassLoading(name);
			
            // 检查该类是否已经被加载,缓存是一个ConcurrentHashMap
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (log.isDebugEnabled()) {
                    log.debug("  Returning class from cache");
                }
                if (resolve) {
                    resolveClass(clazz);
                }
                return clazz;
            }

            // isGraalAvailable 判断Tomcat是否是由GraalVM加载,调用findLoadedClass方法来进行加载,
            // 这里的name事实上就是对应的路径,已经出发
            clazz = JreCompat.isGraalAvailable() ? null : findLoadedClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled()) {
                    log.debug("  Returning class from cache");
                }
                if (resolve) {
                    resolveClass(clazz);
                }
                return clazz;
            }

            // (0.2) Try loading the class with the bootstrap class loader, to prevent
            //       the webapp from overriding Java SE classes. This implements
            //       SRV.10.7.2
            // 尝试调用BootStrapClassLoader 阻止webapp应用程序覆盖Java SE的类，
            String resourceName = binaryNameToPath(name, false);		
            ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {          
                URL url;
                if (securityManager != null) {
                    PrivilegedAction<URL> dp = new PrivilegedJavaseGetResource(resourceName);
                    url = AccessController.doPrivileged(dp);
                } else {
                    url = javaseLoader.getResource(resourceName);
                }
                tryLoadingFromJavaseLoader = (url != null);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);          
                tryLoadingFromJavaseLoader = true;
            }
			// 如果尝试用ClassLoader能够加载到,则直接调用javaseLoader加载这个类
            if (tryLoadingFromJavaseLoader) {
                try {
                    clazz = javaseLoader.loadClass(name);
                    if (clazz != null) {
                        if (resolve) {
                            resolveClass(clazz);
                        }
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
          }	
           boolean delegateLoad = delegate || filter(name, true);

            // (1) 然后委托给我们的父classLoader来加载这个类
            if (delegateLoad) {
                if (log.isDebugEnabled()) {
                    log.debug("  Delegating to parent classloader1 " + parent);
                }
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (log.isDebugEnabled()) {
                            log.debug("  Loading class from parent");
                        }
                        if (resolve) {
                            resolveClass(clazz);
                        }
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }             
        throw new ClassNotFoundException(name);
    }
```

所以Tomcat加载的class的流程按我们展示的代码来说就是:

1. 加载一个类之前先去缓存里面看这个类是不是已经被记载了。
2. 如果缓存没有，就直接根据name去查找类,然后对class进行解析
3. 还没加载到，就调BootStrapLoader加载这个类
4. 如果还没加载到，就调用父加载器进行加载。

这也就是说Tomcat破坏了双亲委派模型，那Tomcat为什么要破坏双亲委派模型呢，对此Tomcat的官方文档也做了解释:

> Like many server applications, Tomcat installs a variety of class loaders (that is, classes that implement `java.lang.ClassLoader`) to allow different portions of the container, and the web applications running on the container, to have access to different repositories of available classes and resources. This mechanism is used to provide the functionality defined in the Servlet Specification, version 2.4 — in particular, Sections 9.4 and 9.6.
>
> 像一些网络应用程序一样，Tomcat里面有不同的类加载器(也就是对JDK内置ClassLoader的扩展)，让Tomcat中不同部分，以及运行在容器的Web应用程序访问不同的可用类和资源存储库。这个机制用于实现Servlet规范2.4版本，特别的参见9.4节，9.6节。
>
> In a Java environment, class loaders are arranged in a parent-child tree. Normally, when a class loader is asked to load a particular class or resource, it delegates the request to a parent class loader first, and then looks in its own repositories only if the parent class loader(s) cannot find the requested class or resource. 
>
> 在Java世界里面，类加载器以父子树结构的形式排列，也就是说，当类加载器被要求加载一个特定的类，它会先将这个请求委托给父类加载器，只有当父类加载器查找不到资源的时候，它才会在自己的本地仓库中去寻找。

 那也就是说，父类加载器和子类加载器查找类的位置不同，如果查找的位置都是一个，那么父类一定能找到。那么我们来看看我们启动main函数的过程中涉及到几个类加载器，我们只需要在ClassLoader的loadClass

## JDK中的类加载器



​	

## JDK 8之后对类加载器的调整









## 写在最后





##  参考资料

[1] Why Developers Should Not Write Programs That Call 'sun' Packages https://www.oracle.com/java/technologies/faq-sun-packages.html

[2] 如何查找 jdk 中的 native 实现 https://gorden5566.com/post/1027.html

[3] Java (programming language): What is the life cycle of an object in Java? https://www.quora.com/Java-programming-language-What-is-the-life-cycle-of-an-object-in-Java

[4] Chapter 5. Loading, Linking, and Initializing https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html

[5] 《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》周志明 著

[6] Static vs Dynamic Binding in Java  https://www.geeksforgeeks.org/static-vs-dynamic-binding-in-java/

[7] Static Vs. Dynamic Binding in Java https://stackoverflow.com/questions/19017258/static-vs-dynamic-binding-in-java

[8] JLS 12.7  https://docs.oracle.com/javase/specs/jls/se15/html/jls-12.html#jls-12.7

[9] Unloading classes in java?  https://stackoverflow.com/questions/148681/unloading-classes-in-java

[10]  Java系列 | 远程热部署在美团的落地实践 https://tech.meituan.com/2022/03/17/java-hotswap-sonic.html

[11]  Java classloaders: why search the parent classloader first?  https://stackoverflow.com/questions/5650334/java-classloaders-why-search-the-parent-classloader-first

[12] Tomcat web doc   https://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html 
