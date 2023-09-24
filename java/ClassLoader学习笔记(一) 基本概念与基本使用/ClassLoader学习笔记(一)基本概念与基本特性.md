# ClassLoader探索笔记

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

![](https://a.a2k6.com/gerald/i/2023/09/16/ht0i.jpg)

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

 那也就是说，父类加载器和子类加载器查找类的位置不同，如果查找的位置都是一个，那么父类一定能找到。那么我们来看看我们启动main函数的过程中涉及到几个类加载器，我们只需要在ClassLoader的loadClass中打个断点，然后写个main函数启动，根据我们上面说的，标准JDK用的是双亲委派模型，子类加载器收到一个加载类请求之后，会先委托给自己的父加载器，父加载器加载不到，子类才会去加载，这么做也是为了JVM平台的安全性，比如我自定义了个JDK标准的类，比如说String，然后请求JDK加载，那么如果直接加载进去加载，就会有很多意外之料的事情，我可以原封不懂复制一个String，然后处处报空指针异常，我也可以请求在我恶意写的String类中埋下危险操作，所以Tomcat也并未完全破坏双亲委派模型，在注释上我们也清楚的看到，上面写了防止web应用的类覆盖JavaSE的类，那现在我有一个问题，那么刚开始执行loadClass的时候，name是个啥造型，我们重点看findLoadedClass0方法的实现, 同样的我们在这个方法内部打上断点，瞧一瞧，看一看:

![](https://a.a2k6.com/gerald/i/2023/09/16/dkt.jpg)

那下一个问题来了，那么这个name是谁传给他的，我按着ctrl找这个方法的调用方，一无所获，难道是思路不对，那转移到类身上，按着ctrl找这个类的使用方：

![](https://a.a2k6.com/gerald/i/2023/09/16/3xv8r.jpg)

点进去就会发现TomcatEmbeddedWebappClassLoader被下面这个方法调用, 方法比较长，但是也不要怕，我们debug就能大致明白这每一行的作用:

```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
   File documentRoot = getValidDocumentRoot();
   TomcatEmbeddedContext context = new TomcatEmbeddedContext();
   if (documentRoot != null) {
      context.setResources(new LoaderHidingResourceRoot(context));
   } 
   context.setName(getContextPath());
   context.setDisplayName(getDisplayName());
   // 设置根路径 
   context.setPath(getContextPath()); // 语句一
   File docBase = (documentRoot != null) ? documentRoot : createTempDir("tomcat-docbase");
   context.setDocBase(docBase.getAbsolutePath());
   context.addLifecycleListener(new FixContextListener());
   // 设置父类加载器,我调试的时候这个resourceLoader不为空,拿到的ClassLoader是AppClassLoader 语句二
   context.setParentClassLoader((this.resourceLoader != null) ? this.resourceLoader.getClassLoader()
         : ClassUtils.getDefaultClassLoader());
   resetDefaultLocaleMapping(context);
   addLocaleMappings(context);
   try {
      context.setCreateUploadTargets(true);
   }
   catch (NoSuchMethodError ex) {
      // Tomcat is < 8.5.39. Continue.
   }
   configureTldPatterns(context);    
   WebappLoader loader = new WebappLoader(); // 语句三
   loader.setLoaderClass(TomcatEmbeddedWebappClassLoader.class.getName()); // 语句四
   loader.setDelegate(true); // 语句五
   context.setLoader(loader); // 语句六
   if (isRegisterDefaultServlet()) {
      addDefaultServlet(context);
   }
   if (shouldRegisterJspServlet()) {
      addJspServlet(context);
      addJasperInitializer(context);
   }
   context.addLifecycleListener(new StaticResourceConfigurer(context));
   ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
   host.addChild(context);
   configureContext(context, initializersToUse);
   postProcessContext(context);
}
```

语句三产生了一个WebappLoader，根据名字我推测这也是一个ClassLoader: 

![](https://a.a2k6.com/gerald/i/2023/09/16/l15j.jpg)

但是它不是，那根据名称推断他应该是负责加载war包和我们jar中的类，所以语句四的作用就是设置哪个类加载器来加载我们war包和jar中的类, 我们在审视一下这个类, 这个类是Loader的实现类，所以这里我推测Loader类规定了加载的行为，所以我们的目光转移到了Loader类上面, 在这个类的最上面我们可以看到注释:

>  Loader represents a Java ClassLoader implementation that can be used by a Container to load class files (within a repository associated with the Loader) that are designed to be reloaded upon request, as well as a mechanism to detect whether changes have occurred in the underlying repository.
>
> Loader代表一个Java ClassLoader实现，容器可以使用它来加载类文件(在与Loader关联的库中)，这些类文件可以根据请求重新加载，同时还提供了一种检测依赖的底层资源资源库是否发生改变的机制。
>
> In order for a Loader implementation to successfully operate with a Context implementation that implements reloading, it must obey the following constraints:
>
> 为了让Loader的实现和重新加载的Context一起运行，它必须遵循以下约束:
>
> - Must implement Lifecycle so that the Context can indicate that a new class loader is required.
>
>    一定要实现Lifecycle，以便Context可以指示Loader创建一个新的ClassLoader
>
> - The start() method must unconditionally create a new ClassLoader implementation
>
>   start方法一定要无条件的创建一个新的ClassLoader实现
>
> - The stop() method must throw away its reference to the ClassLoader previously utilized, so that the class loader, all classes loaded by it, and all objects of those classes, can be garbage collected.
>
>   stop方法一定要丢弃对ClassLoader的引用，以便在ClassLoader和对应加载的类，这些类对应的对象可以被垃圾回收。
>
> - Must allow a call to stop() to be followed by a call to start() on the same Loader instance.
>
>   对于一个Loader实例必须先调用start()方法，然后再调用stop方法。
>
> - Based on a policy chosen by the implementation, must call the Context.reload() method on the owning Context when a change to one or more of the class files loaded by this class loader is detected.
>
>   根据选择的实现策略，当检测到对应类加载器的一个或多个类文件发生更改的时候，必须调用所属Context的reload方法。

在这里又有意外发现，也就是加载我们的jar和应用的行为在Lifecycle里面，从上面的继承图我们可以知道WebappLoader继承了LifecycleMBeanBase，而LifecycleMBeanBase又继承了LifecycleBase，而LifecycleBase实现了Lifecycle ，我们可以看到start方法的实现, 但是在LifecycleBase的实现中又调用了startInternal()方法，这是一个抽象方法，实现还是在子类，具体的实现还是在WebappLoader中：

```java
protected void startInternal() throws LifecycleException {

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("webappLoader.starting"));
        }
		// 注意我们这里有一个context,我们上面是有prepareContext
    	// 所以prepareContext方法应当在startInternal被调用,我们看下resources方法
        if (context.getResources() == null) {
            log.info(sm.getString("webappLoader.noResources", context));
            setState(LifecycleState.STARTING);
            return;
        }

        // Construct a class loader based on our current repositories list
        try {
			// 然后创建类加载器
            classLoader = createClassLoader(); // 语句一
            classLoader.setResources(context.getResources());
            classLoader.setDelegate(this.delegate);

            // Configure our repositories
            setClassPath();

            setPermissions();

            classLoader.start(); // 语句二

            String contextName = context.getName();
            if (!contextName.startsWith("/")) {
                contextName = "/" + contextName;
            }
            ObjectName cloname = new ObjectName(context.getDomain() + ":type=" +
                    classLoader.getClass().getSimpleName() + ",host=" +
                    context.getParent().getName() + ",context=" + contextName);
            Registry.getRegistry(null, null)
                .registerComponent(classLoader, cloname, null);

        } catch (Throwable t) {
            t = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(t);
            throw new LifecycleException(sm.getString("webappLoader.startError"), t);
        }

        setState(LifecycleState.STARTING);
 }
```

我们看下createClassLoader的实现:

```java
private WebappClassLoaderBase createClassLoader()
        throws Exception {

        if (classLoader != null) {
            return classLoader;
        }
		// 如果parentClassLoader不为空,那么拿context对WebappLoader进行赋值
        if (parentClassLoader == null) {          
            parentClassLoader = context.getParentClassLoader();
        } else {
            context.setParentClassLoader(parentClassLoader);
        }
		// 如果加载类和ParallelWebappClassLoader,设置父加载器。
        if (ParallelWebappClassLoader.class.getName().equals(loaderClass)) {
            return new ParallelWebappClassLoader(parentClassLoader);
        }
		// 将创建对应的loadClass实例,返回类加载器
        Class<?> clazz = Class.forName(loaderClass);
        WebappClassLoaderBase classLoader = null;

        Class<?>[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        Constructor<?> constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoaderBase) constr.newInstance(args);

        return classLoader;
 }
```

语句二的调用的是Lifecycle 的start方法，我们简单看下实现:

```java
public void start() throws LifecycleException {

    state = LifecycleState.STARTING_PREP;
    WebResource[] classesResources = resources.getResources("/WEB-INF/classes");// 语句一
    for (WebResource classes : classesResources) {
        if (classes.isDirectory() && classes.canRead()) {
            localRepositories.add(classes.getURL());
        }
    }
    WebResource[] jars = resources.listResources("/WEB-INF/lib"); //语句二
    for (WebResource jar : jars) {
        if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
            localRepositories.add(jar.getURL());
            jarModificationTimes.put(
                    jar.getName(), Long.valueOf(jar.getLastModified()));
        }
    }

    state = LifecycleState.STARTED;
}
```

看到这个方法其实我是很激动的，在这里去对应文件夹下找了类，然后加入到localRepositories中，这是一个List，但是我打断点发现这个方法跑完之后，localRepositories还是空的，于是陷入了深思，难道是思路哪里不对嘛，仔细想了一下，思路并没有多大问题，问题出现在我对Spring Boot web 应用启动流程的认知错误，确实是在start方法里面去找我们部署的项目的，但是那是war包的形式，我之前写原生Servlet的时候就需要下载Tomcat，然后Eclipse关联Tomcat，项目就会自动部署到Tomcat的webapp文件夹下面，Tomcat的文件夹结构如下图所示:

![](https://a.a2k6.com/gerald/i/2023/09/16/o780.jpg)

然后我写了个简单的原生Servlet项目，打成war包放在了webapps里面: 

![](https://a.a2k6.com/gerald/i/2023/09/16/x0wpt.jpg)

然后再bin里面选择对应的启动脚本就行了, war包其实还是一个压缩包: 

![](https://a.a2k6.com/gerald/i/2023/09/16/6fx5p3.jpg)

所以Tomcat的start方法里面扫描的是war包，在Tomcat的逻辑里面是先启动Tomcat，然后我去扫描war包加载类，但是在Spring Boot的逻辑里是我先将bean加载的差不多了，再去启动Tomcat，所以我们如果有bean没加载成功，Spring Boot web项目会启动不起来。所以原生的Tomcat启动流程究竟是怎么样的呢？ 或者更进一步我们如何部署Java应用，我想起我刚学Java的时候，是在命令行里面写javac，java命令，java会给提示，如下所示:

![](https://a.a2k6.com/gerald/i/2023/09/16/4xvbj.jpg)

  java -jar jarfile 执行jar文件，那我一个jar里面有那么多文件，JVM该怎么知道加载哪一个文件呢？  当然是由jar来告诉jvm啊。

> If you have an application bundled in a JAR file, you need some way to indicate which class within the JAR file is your application's entry point. You provide this information with the `Main-Class` header in the manifest, which has the general form:
>
> 如果应用程序捆绑在一个 JAR 文件中，则需要某种方法来指明 JAR 文件中的哪个类是应用程序的入口点。您可以通过清单中的 Main-Class 头信息提供这一信息，该头信息的一般形式为:
>
> ```
> Main-Class: classname
> ```
>
> The value *`classname`* is the name of the class that is your application's entry point. 《The Java™ Tutorials》
>
> className是应用程序的入口。

上面说的清单指的是MANIFEST.MF，这里面有jar的元信息，所以Tomcat的脚本里面应当是启动了某个jar，我们来查一下，首先我们看的是startup.bat：

![](https://a.a2k6.com/gerald/i/2023/09/16/prj9.jpg)

​      ![](https://a.a2k6.com/gerald/i/2023/09/16/ppws.jpg)

jar是一个压缩包，我们去找MANIFEST.MF这个文件，Tomcat的这个文件稍微丰富一点:

```java
Manifest-Version: 1.0
Ant-Version: Apache Ant 1.9.16
Created-By: 11.0.14.1+1 (Eclipse Adoptium)
Main-Class: org.apache.catalina.startup.Bootstrap // 这就是我们想要的主类
Specification-Title: Apache Tomcat Bootstrap
Specification-Version: 8.5
Specification-Vendor: Apache Software Foundation
Implementation-Title: Apache Tomcat Bootstrap
Implementation-Version: 8.5.79
Implementation-Vendor: Apache Software Foundation
X-Compile-Source-JDK: 7
X-Compile-Target-JDK: 7
Class-Path: commons-daemon.jar
```

Bootstrap里面有一个main函数，我们看下这个main函数的逻辑:

```java
public static void main(String args[]) {

        synchronized (daemonLock) {
            if (daemon == null) {
                // Don't set daemon until init() has completed
                Bootstrap bootstrap = new Bootstrap();
                try {
                    bootstrap.init();
                } catch (Throwable t) {
                    handleThrowable(t);
                    t.printStackTrace();
                    return;
                }
                daemon = bootstrap;
            } else {
                // When running as a service the call to stop will be on a new
                // thread so make sure the correct class loader is used to
                // prevent a range of class not found exceptions.
                Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
            }
        }
     try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
            
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            } else if (command.equals("start")) {
            	 // 调用start()方法
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }
}
// 这里出现了三个类加载器
private void initClassLoaders() {
        try {
            commonLoader = createClassLoader("common", null);
            if (commonLoader == null) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader = this.getClass().getClassLoader();
            }
            catalinaLoader = createClassLoader("server", commonLoader);
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            handleThrowable(t);
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
}
public void init() throws Exception {
       initClassLoaders();
		// 设置当前上下文类加载器
        Thread.currentThread().setContextClassLoader(catalinaLoader);
		
        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        if (log.isDebugEnabled()) {
            log.debug("Loading startup class");
        }
    	// 用这个加载器加载Catalina加载Catalina类
        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    	// 然后调用构造函数,创建对象
        Object startupInstance = startupClass.getConstructor().newInstance();

        // Set the shared extensions class loader
        if (log.isDebugEnabled()) {
            log.debug("Setting startup class properties");
        }
    		
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
    	// 获取目标方法
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
    	// sharedLoader 调用 setParentClassLoader方法
        method.invoke(startupInstance, paramValues);
        catalinaDaemon = startupInstance;
}
```

流程图如下:  

![](https://a.a2k6.com/gerald/i/2023/09/17/ege.jpg)

setAwait方法只是将Catalina的await属性设置为true，我们重点看Catalina.load()方法和start方法，来看看扫描的类是如何被Bootstrap加载器加载的类加载进入JVM的，这里注意在Catalina有两个load方法，那么load方法究竟调用的是哪一个呢? 我们看调用处是如何调用的, 调用处是Bootstrap的main方法: 

```java
try {
    String command = "start";
    if (args.length > 0) {
        command = args[args.length - 1];
    }

    if (command.equals("startd")) {
        args[args.length - 1] = "start";
        daemon.load(args);
        daemon.start();
    } else if (command.equals("stopd")) {
        args[args.length - 1] = "stop";
        daemon.stop();
    } else if (command.equals("start")) {
        daemon.setAwait(true);
        daemon.load(args);
        daemon.start();
        if (null == daemon.getServer()) {
            System.exit(1);
        }
    } else if (command.equals("stop")) {
        daemon.stopServer(args);
    } else if (command.equals("configtest")) {
        daemon.load(args);
        if (null == daemon.getServer()) {
            System.exit(1);
        }
        System.exit(0);
    } else {
        log.warn("Bootstrap: command \"" + command + "\" does not exist.");
    }
} catch (Throwable t) {
    // Unwrap the Exception for clearer error reporting
    if (t instanceof InvocationTargetException &&
            t.getCause() != null) {
        t = t.getCause();
    }
    handleThrowable(t);
    t.printStackTrace();
    System.exit(1);
}
```

我们姑且就看参数为0的吧，

```java
 public void load() {
        if (loaded) {
            return;
        }
        loaded = true;

        long t1 = System.nanoTime();

        initDirs();

        // Before digester - it may be needed
        initNaming();

        // Parse main server.xml
     	// 我们看下parseServerXml做了什么
        parseServerXml(true);
        Server s = getServer();
        if (s == null) {
            return;
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // 这个姑且不看
        initStreams();

        // Start the new server
        try {
            // 然后这里初始化init方法
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error(sm.getString("catalina.initError"), e);
            }
        }

        if(log.isInfoEnabled()) {
            log.info(sm.getString("catalina.init", Long.toString(TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - t1))));
        }
 }
```

```java
// 省略
protected void parseServerXml(boolean start) {
    ServerXml serverXml = null;
    if (useGeneratedCode) {
        serverXml = (ServerXml) Digester.loadGeneratedClass(xmlClassName);
    }
    if (serverXml != null) {
        serverXml.load(this);
    } else {
        // 就猜走了这里吧
        try (ConfigurationSource.Resource resource = ConfigFileLoader.getSource().getServerXml()) {
			// createStartDigester
            Digester digester = start ? createStartDigester() : createStopDigester();
            InputStream inputStream = resource.getInputStream();
            InputSource inputSource = new InputSource(resource.getURI().toURL().toString());
            inputSource.setByteStream(inputStream);
            digester.push(this);
            if (generateCode) {
                digester.startGeneratingCode();
                generateClassHeader(digester, start);
            }
            digester.parse(inputSource);
            if (generateCode) {
                generateClassFooter(digester);
                try (FileWriter writer = new FileWriter(new File(serverXmlLocation,
                        start ? "ServerXml.java" : "ServerXmlStop.java"))) {
                    writer.write(digester.getGeneratedCode().toString());
                }
                digester.endGeneratingCode();
                Digester.addGeneratedClass(xmlClassName);
            }
        } catch (Exception e) {
            log.warn(sm.getString("catalina.configFail", file.getAbsolutePath()), e);
            if (file.exists() && !file.canRead()) {
                log.warn(sm.getString("catalina.incorrectPermissions"));
            }
        }
    }
}
```

```java
// 省略部分代码	  
protected Digester createStartDigester() {
        // 初始化一系列对象
        Digester digester = new Digester();
        digester.setValidating(false);
        digester.setRulesValidation(true);
        Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
        // Ignore className on all elements
        List<String> objectAttrs = new ArrayList<>();
        objectAttrs.add("className");
        fakeAttributes.put(Object.class, objectAttrs);
        // Ignore attribute added by Eclipse for its internal tracking
        List<String> contextAttrs = new ArrayList<>();
        contextAttrs.add("source");
        fakeAttributes.put(StandardContext.class, contextAttrs);
        // Ignore Connector attribute used internally but set on Server
        List<String> connectorAttrs = new ArrayList<>();
        connectorAttrs.add("portOffset");
        fakeAttributes.put(Connector.class, connectorAttrs);
        digester.setFakeAttributes(fakeAttributes);
        digester.setUseContextClassLoader(true);

        // Configure the actions we will be using
        digester.addObjectCreate("Server",
                                 "org.apache.catalina.core.StandardServer",
                                 "className");
        digester.addSetProperties("Server");
        digester.addSetNext("Server",
                            "setServer",
                            "org.apache.catalina.Server");

        digester.addObjectCreate("Server/GlobalNamingResources",
                                 "org.apache.catalina.deploy.NamingResourcesImpl");
        digester.addSetProperties("Server/GlobalNamingResources");
        digester.addSetNext("Server/GlobalNamingResources",
                            "setGlobalNamingResources",
                            "org.apache.catalina.deploy.NamingResourcesImpl");

        digester.addRule("Server/Listener",
                new ListenerCreateRule(null, "className"));
        digester.addSetProperties("Server/Listener");
        digester.addSetNext("Server/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        digester.addObjectCreate("Server/Service",
                                 "org.apache.catalina.core.StandardService",
                                 "className");
        digester.addSetProperties("Server/Service");
        digester.addSetNext("Server/Service",
                            "addService",
                            "org.apache.catalina.Service");

        digester.addObjectCreate("Server/Service/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Service/Listener");
        digester.addSetNext("Server/Service/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        //Executor
        digester.addObjectCreate("Server/Service/Executor",
                         "org.apache.catalina.core.StandardThreadExecutor",
                         "className");
        digester.addSetProperties("Server/Service/Executor");

        digester.addSetNext("Server/Service/Executor",
                            "addExecutor",
                            "org.apache.catalina.Executor");

        digester.addRule("Server/Service/Connector",
                         new ConnectorCreateRule());
        digester.addSetProperties("Server/Service/Connector",
                new String[]{"executor", "sslImplementationName", "protocol"});
        digester.addSetNext("Server/Service/Connector",
                            "addConnector",
                            "org.apache.catalina.connector.Connector");    

        return digester;

    }
```

在这里设置了StandardContext，为Server挂上实现类，Server内部又有Service成员变量，再为Service成员变量初始化Connector，为Connector初始化各种各样的属性，后面就能以Server为入口进行初始化了: 

```java
    /**
     * Start a new server instance.
     */
    public void load() {

        if (loaded) {
            return;
        }
        loaded = true;

        long t1 = System.nanoTime();

        initDirs();

        // Before digester - it may be needed
        initNaming();

        // Parse main server.xml
        parseServerXml(true);
        Server s = getServer();
        if (s == null) {
            return;
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // Stream redirection
        initStreams();

        // Start the new server
        try {
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error(sm.getString("catalina.initError"), e);
            }
        }

        if(log.isInfoEnabled()) {
            log.info(sm.getString("catalina.init", Long.toString(TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - t1))));
        }
    }
```

注意这里的StandardContext，在这个类里面的startInternal方法挂载了WebappLoader，然后由WebappLoader挂载webapps文件夹下面的类，那么我们看到在createStartDigester方法里只是将这个类放在fakeAttributes里面，并未执行初始化，那么他是在哪里执行的初始化呢, 我们找找看:

![](https://a.a2k6.com/gerald/i/2023/09/17/nmjf.jpg)

在ContextConfig我找到了createContextDigester方法:

```java
protected Digester createContextDigester() {
    Digester digester = new Digester();
    digester.setValidating(false);
    digester.setRulesValidation(true);
    Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
    List<String> objectAttrs = new ArrayList<>();
    objectAttrs.add("className");
    fakeAttributes.put(Object.class, objectAttrs);
    // Ignore attribute added by Eclipse for its internal tracking
    List<String> contextAttrs = new ArrayList<>();
    contextAttrs.add("source");
    fakeAttributes.put(StandardContext.class, contextAttrs);
    digester.setFakeAttributes(fakeAttributes);
    RuleSet contextRuleSet = new ContextRuleSet("", false);
    digester.addRuleSet(contextRuleSet);
    RuleSet namingRuleSet = new NamingRuleSet("Context/");
    digester.addRuleSet(namingRuleSet);
    return digester;
}
```

这个方法又被ContextConfig的init方法所调用, 而init方法又被ContextConfig的lifecycleEvent调用，lifecycleEvent又被LifecycleBase的fireLifecycleEvent所调用，而fireLifecycleEvent又是被StandServer的startInternal()所调用。 所以到现在我们的基本流程就连接起来了:

![](https://a.a2k6.com/gerald/i/2023/09/17/4nmh6.jpg)

load方法中解析server.xml，设置类加载器，创建对象，在start方法里面激活事件来做初始化动作。到此Tomcat启动完毕。让我们回到Spring Boot，我们直到在Spring Boot里面如果一个Bean初始化错误，Tomcat根本就启动不起来，那么也就是说在Spring Boot web应用中是先加载bean，在启动Tomcat的。那么在Spring Boot 中是如何加载类的呢，我们同样的还是在ClassLoader加载类上打断点来观察这个过程。

## 那Spring Boot 是如何扫描的呢？

首先JVM如要直到入口，入口在jar的MANIFEST.MF中描述，但是main函数所在的类名称不是固定的，那Spring Boot 是如何找到main函数所在类的呢? 下面是我做的Spring Boot 应用的的MANIFEST.MF:

```java
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: ssm
Implementation-Version: 0.0.1-SNAPSHOT
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
Start-Class: com.example.demo.SsmApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.4.11
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

主类是JarLauncher，我们看下这个类，在IDEA里面搜了一下没搜到，想来应该在打好的jar里面，然后果然在，但是class文件，所以就只好用IDEA反编译文件去看:

```java
// 省略无关方法
public class JarLauncher extends ExecutableArchiveLauncher {
    private static final String DEFAULT_CLASSPATH_INDEX_LOCATION = "BOOT-INF/classpath.idx";
    static final Archive.EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
        return entry.isDirectory() ? entry.getName().equals("BOOT-INF/classes/") : entry.getName().startsWith("BOOT-INF/lib/");
    };

    public JarLauncher() {
    }

    protected JarLauncher(Archive archive) {
        super(archive);
    }
    public static void main(String[] args) throws Exception {
        (new JarLauncher()).launch(args);
    }
}
```

这个lanuch方法在Launcher被定义，不是JDK的lanucher，Spring 创建了一个的Launcher，

```java
public abstract class Launcher {
    private static final String JAR_MODE_LAUNCHER = "org.springframework.boot.loader.jarmode.JarModeLauncher";
    public Launcher() {
    }
    protected void launch(String[] args) throws Exception {
        if (!this.isExploded()) {
            JarFile.registerUrlProtocolHandler();
        }
        ClassLoader classLoader = this.createClassLoader(this.getClassPathArchivesIterator());
        String jarMode = System.getProperty("jarmode");
        String launchClass = jarMode != null && !jarMode.isEmpty() ? "org.springframework.boot.loader.jarmode.JarModeLauncher" : this.getMainClass();
        // launch是通过反射方法来调用我们定义的main方法，然后项目就启动起来了。
        this.launch(args, launchClass, classLoader);
    }
    protected abstract String getMainClass() throws Exception;
}

```

在打断点的之后我们会发现自定义的类由AppClassLoader 加载，这个类位于 sun.misc包Launcher下面，Launcher里面有一些ClassLoader，我们都拿出来看看:

```
static class AppClassLoader extends URLClassLoader {
    final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

    public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
        final String var1 = System.getProperty("java.class.path");
        final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
        return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<AppClassLoader>() {
            public AppClassLoader run() {
                URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                return new AppClassLoader(var1x, var0);
            }
        });
    }
 }   
```

我们看下这个java.class.paths下面放的是什么：

![](https://a.a2k6.com/gerald/i/2023/09/17/tu9w.jpg)

这个路径下面的太长，就是我们依赖的JDK和maven填写的依赖。

```java
static class ExtClassLoader extends URLClassLoader {
        private static volatile ExtClassLoader instance;

        public static ExtClassLoader getExtClassLoader() throws IOException {
            if (instance == null) {
                Class var0 = ExtClassLoader.class;
                synchronized(ExtClassLoader.class) {
                    if (instance == null) {
                        instance = createExtClassLoader();
                    }
                }
            }

            return instance;
        }

        private static ExtClassLoader createExtClassLoader() throws IOException {
            try {
                return (ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction<ExtClassLoader>() {
                    public ExtClassLoader run() throws IOException {
                        File[] var1 = Launcher.ExtClassLoader.getExtDirs();
                        int var2 = var1.length;

                        for(int var3 = 0; var3 < var2; ++var3) {
                            MetaIndex.registerDirectory(var1[var3]);
                        }

                        return new ExtClassLoader(var1);
                    }
                });
            } catch (PrivilegedActionException var1) {
                throw (IOException)var1.getException();
            }
        }

        void addExtURL(URL var1) {
            super.addURL(var1);
        }


	 private static File[] getExtDirs() {
            String var0 = System.getProperty("java.ext.dirs");
            File[] var1;
            if (var0 != null) {
                StringTokenizer var2 = new StringTokenizer(var0, File.pathSeparator);
                int var3 = var2.countTokens();
                var1 = new File[var3];

                for(int var4 = 0; var4 < var3; ++var4) {
                    var1[var4] = new File(var2.nextToken());
                }
            } else {
                var1 = new File[0];
            }

            return var1;
        }
}      
```

我们看下这个属性下面有什么:

> C:\Program Files\Java\jdk1.8.0_202\jre\lib\ext;C:\WINDOWS\Sun\Java\lib\ext

![](https://a.a2k6.com/gerald/i/2023/09/17/4dbm.jpg)

我尝试在AppClassLoader上打断点，想看看是谁来调用这个类的，发现并没有停下来，想来估计又非Java代码构成的加载器触发了这两个加载器去加载, 查了一下资料，确实是，在JVM启动的时候，会运行一段特殊的机器代码来加载系统ClassLoader，这段代码被称为Boostrap加载器，它不是Java编写，Boostrap加载器是JVM特定的机器指令来触发，必须先启动系统类加载器，才能开始其他过程，系统加载器负责加载支持基本Java运行时环境(JRE)所需的所有代码。

## 总结一下

总结一下，我们写的类由类加载器进入JVM，一个类在被使用前需要经历加载进入JVM有加载、验证、准备、初始化，我们可以利用初始化阶段触发一些操作，由于初始化阶段只执行一次，我们可以借助这个特性来实现单例模式，为了保证JVM平台的安全性，类加载器在收到一个类加载请求的时候，会将这个类交给自己的父加载器进行加载，Tomcat破坏了一部分双亲委派机制。原因在于Servlet规范，需要允许不同的网络程序依赖不同版本的库。所以Tomcat重新实现了ClassLoader中的方法，但为了保证JVM平台的安全性，Tomcat在webapp下面的项目中无法查找到类的时候还是会调用JDK的标准加载器进行加载，这也是为了JDK平台的安全性，JVM平台分为核心类库和扩展类库，核心类库负责加载基本Java运行时环境，这个由系统引导器来加载，而扩展类库则由ExtClassLoader进行加载，AppClassLoader加载用户定义的类库，所以加载模型中为先AppClassLoader，再ExtClassLoader，再是系统类加载器。

我们也好奇，一个jar究竟是怎么被JDK启动的，于是我们下载了Tomcat查看了启动脚本，在启动脚本里面找到了要启动的jar，在jar里面我们发现了MANIFEST.MF里面指定的类，由此好奇Tomcat的整个启动流程，然后跟着代码一步一步打断点去验证。

## 写在最后

写这篇的时候，我有的时候会抗拒随机试探，我希望有步骤去一环套一环去得出某些结论，所以我看一篇文章的时候，我甚至更好奇作者是如何得出结论的，我认为何以知比知更为重要，但是我想了下人类的发展史，随机试探，大胆猜想是探索真理的通常步骤，我们并没有一套足够完善的发现真理的步骤，所以我们要大胆猜想，小心求证，猜想也是探索真理的方式，只是在我的脑海里，才是不确定的，有的时候猜的不见得准确，但不是因为猜想不是次次都对，我们就放弃猜想，因为很多时候都是试探出来的，所以这里我在看源码的时候，根据感觉，也就是猜想，猜完之后，再验证猜想，其实这也是一种探索的方式。

这篇按照规划，是只打算写双亲委派和Tomcat为什么破坏双亲委派的，但是一边写一边就会有新问题出现，我好奇Tomcat是怎么启动起来的，所以本篇的知识点像是网状的，结构并没有那么分明，也许知识本身就是网状的，我想起了大学图书馆的生活，我因为一本书看到了一本书，甚至一本书看了一本书，就是在探索知识。本篇名为探索笔记。



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

[13] 《**The Java™ Tutorials**》 https://docs.oracle.com/javase/tutorial/deployment/index.html

[14] Tomcat 源码阅读与调试环境搭建 - 基于Idea和Tomcat 8.5  https://zhuanlan.zhihu.com/p/597375198

[15]  How is the Java Bootstrap Classloader loaded? https://stackoverflow.com/questions/18214174/how-is-the-java-bootstrap-classloader-loaded
