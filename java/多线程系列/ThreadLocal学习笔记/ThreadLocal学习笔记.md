# ThreadLocal学习笔记

> 补齐多线程的最后一块木板

[TOC]

## 前言

我在刚工作的时候遇到日期格式化的时候，也就是用SimpleDateFormat，发现多个service中都用到了SimpleDateFormat对象时，我那个时候的操作是SimpleDateFormat提升为那个Service的静态成员变量，因为静态成员变量只加载一次，但是其实这样是有问题的，但是当时我对Java的代码执行模型理解的还是不全面，我确实背了Spring MVC的执行流程，但是一个请求到Tomcat，再到我写的Service，我确没有很好的理解。粗略的说，我们可以认为请求被处理的过程是这个样子的:

![](https://imglf6.lf127.net/img/b29MbXcxUWFDVGQ2UGF0SC9jK3NQb0Jac0FRNmtyTDV2aFVhNjFpYjd4NmJFZHZnMlJNZVJBPT0.png)

客户端向Tomcat发起一个请求，这个请求到达计算机之后，被操作系统转发给Tomcat，Tomcat的侦听线程侦听到请求之后，将请求给对应的线程池，线程池选取一个线程调用Spring MVC，下面就是Spring MVC的执行流程。Java中任何一段代码总是执行某个或多个线程之中。但后来我负责的那个模块也并没有出错，因为我刚毕业负责的那个模块分访问量比较小, 要产生这样的问题，需要很大的请求量才有一定的可能性会出现错误。那后来我是怎么发现的呢，我的IDEA是有一个Alibaba扫描插件, 我写完的时候会习惯性的扫一下，然后就给出了下面的提示: 

![](https://imglf4.lf127.net/img/QytaVTZWcGllV0FoaVJ3R0VPT01GVjdLcmZxTUZDL0cyQ2VXb3hVUWxQbWVGY09sQWxqMllnPT0.png)

图片可能有点不清楚，我将文字复制出来:

```java

SimpleDateFormat 是线程不安全的类，一般不要定义为static变量，如果定义为static，必须加锁，或者使用DateUtils工具类。 说明：如果是JDK8的应用，可以使用instant代替Date，LocalDateTime代替Calendar，DateTimeFormatter代替SimpleDateFormat，官方给出的解释：simple beautiful strong immutable thread-safe。
            
Positive example 1：
    private static final String FORMAT = "yyyy-MM-dd HH:mm:ss";
    public String getFormat(Date date){
        SimpleDateFormat dateFormat = new SimpleDateFormat(FORMAT);
        return sdf.format(date);
    }
        
        
            
Positive example 2：
    private static final SimpleDateFormat SIMPLE_DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    public void getFormat(){
        synchronized (sdf){
        sdf.format(new Date());
        ….;
    }
        
        
            
Positive example 3：
    private static final ThreadLocal<DateFormat> DATE_FORMATTER = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };
// 在JDK8下其实可以这么写
private static final ThreadLocal<DateFormat> DATE_FORMATTER = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd")); 
```

就看到了ThreadLocal这个类, 之前对于这个类还属于一知半解的，今天打算系统的学习一下这个类。

## 还是看注释

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

> 这个类提供了线程本地变量，这些变量不同于普通的变量, 像get或者set方法一样，这个变量和线程绑定起来, 只有存放进入的线程才能访问到他。 ThreadLocal对象通常以静态私有的字段在别的类中创建，以此来关联一个线程的某个状态（例如用户ID或事物ID）

那其实上面Alibaba给的建议还是示例三更好些，假设这个线程在不止一处的地方使用到了SimpleDateFormat，示例一的话也就是每次使用都要产生一个SimpleDateFormat对象, 但是用ThreadLocal的话，就只在第一次产生一个，后续从ThreadLocal变量中获取即可。另一个典型的应用场景是JDBC Connection。

我们在刚学JDBC的时候，一般都会尝试封装一个DBUtils, 像下面这样: 

```java
  private static final String DRIVER = "com.mysql.jdbc.Driver";
    private static final String USER = "root";
    private static final String PWD = "root";
    private static final String URL = "jdbc:mysql://localhost:3306/study?useUnicode=true&characterEncoding=utf8";

    //定义一个数据库连接
    private Connection conn = null;
    //获取连接
    public Connection getConnection() {
        try {
            Class.forName(DRIVER);
            if(conn == null) {
                conn = DriverManager.getConnection(URL, USER, PWD);
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return conn;
    }
    //关闭连接
    public void closeConnection() {
        if(conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
```

上面的示例是要求我们每次都new DBUtils来执行对应的数据库从操作, 这样的问题是getConnection本身是线程不安全的, 除此之外还有多个方法共享connection的问题，假设我们在Service层要开启事务, 我们首先就要调用getConnection, 然后Service层和Dao层本身共用一个conncetion，这个连接是在这个线程执行过程中所用到的比较多的，可以认为这个变量就专属于这个connection，这样其实我们可以将这个变量和绑定起来, 在用的时候直接取用就好。所以后面我们会将这个DBUtils中增加一个静态的全局变量ThreadLocal，像下面这样:

```java
public class DBUtils {
    // 这个例子随手写的只是为了说明问题,大家注意体会思想就好
    public static ComboPooledDataSource dataSource = new ComboPooledDataSource();
    private static final ThreadLocal<Connection> threadLocal = new ThreadLocal<Connection>();

    //定义一个数据库连接
    private Connection conn = null;
    //获取连接
    public static Connection getConnection() throws SQLException{
        Connection conn = tl.get();;
        if(conn == null){
            //conn为空就创建一个新的conn
            conn = dataSource.getConnection();
            //将conn存入到当前线程中去
            tl.set(conn);
        }
        return conn;
    }
    //关闭连接
    public void closeConnection() {
        if(conn != null) {
            try {
                conn.close();
                threadLocal.remove();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 使用示例

其实上面我们已经有意无意的用了ThreadLocal了, 使用ThreadLocal有几种方式:

- 为每个线程准备初始值，SimpleDateFormat就是一个典型的例子：

  - JDK8的写法

    ```java
    private static final ThreadLocal<DateFormat> DATE_FORMATTER = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
    ```

  - JDK8之前的写法

    ```
    private static final ThreadLocal<DateFormat> DATE_FORMATTER = new ThreadLocal<DateFormat>(){
            @Override
            protected DateFormat initialValue() {
                return new SimpleDateFormat();
            }
        };		
    ```

- 在get之前先set

> 如果没为线程准备初始值, 那在get的时候什么也get不到。也就是null。

示例: 

```java
public class Test {
    ThreadLocal<String> stringLocal = new ThreadLocal<String>();
    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(){
            public void run() {
                System.out.println(stringLocal.get());
            };
        };
        thread1.start();
        thread1.join();
		stringLocal.set("thread0")
    }
}
```

所以在get之前要确保set进去了值，要不然再用到这个值的时候，就会报空指针异常。有细心的同学可能会注意到我在closeConnection中调用了remove()方法，这里我们就要看一点源码来解释这个操作了。

## ThreadLocal源码简单解析

首先是get方法:

```java
 public T get() {
        Thread t = Thread.currentThread(); // 获取当前线程
        ThreadLocalMap map = getMap(t); // 这个看getMap的实现其实是的是线程的成员变量 
        // 每个线程内部维护了一个线程ThreadLocal.ThreadLocalMap
        // 在Thread中这个变量默认为null
        // 所以直接走到setInitialValue方法上
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
  }
```

setInitialValue的实现:

```java
  private T setInitialValue() {
        T value = initialValue(); // 如果我们传入了方法 或者初始值 此时执行
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
        if (map != null)  // 第一次进来map为空
            map.set(this, value);
        else
            createMap(t, value); // 走createMap方法
        return value;
    }
```

然后是createMap方法: 

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue); // 当前线程的当前的ThreadLocal对象初始化完成,key是线程 value是初始化的值
}
```

然后我们大致看下ThreadLocal

![](https://590233ee4fbb3.cdn.sohucs.com/auto/1-auto1f8da48613d040afb245fe386b92644c)

总结一下，我们看到这里应该已经明白了ThreadLocal本身不存储值,是当前线程存储。但是如果你使用的是线程池来用ThreadLocal变量，我们在回到线程池会复用线程，如果当前线程被线程池保留，一直不销毁，那么该线程将会一直持有对value的引用。我们来大致捋一下ThreadLocal Thread之间的引用关系:

Thread 持有对ThreadLocalMap的引用，ThreadLocalMap引用ThreadLocal,Value 也就是Entry。ThreadLocal本身还是某个静态类的成员变量。

ThreadLocalMap持有的是对ThreadLocal的弱引用，一旦对应ThreadLocal的强引用被移除，此时JVM执行GC，那么ThreadLocalMap中Entry持有的key就会被置为null，也就是内存被回收。但是这个Value对应的Entry还是被Thread强引用，如果这个线程被线程池复用，那么这个Value可能在很长一段时间都不会被回收，这也就是内存泄漏。 所以我们在线程中这个ThreadLocal的变量使用完毕，还是要手动的调用remove方法, 将对应的key和value置为空。

## 总结一下

假如有一个变量在线程整个执行流程中都需要使用到，我们不妨将其放入到ThreadLocal中，这样我们就避免了这个变量在线程的执行流程中，出现在多个方法参数中。

## 参考资料

- ThreadLocal解析  https://zhuanlan.zhihu.com/p/401422386
- ThreadLocal是鸡肋吗？ https://www.zhihu.com/question/391905541