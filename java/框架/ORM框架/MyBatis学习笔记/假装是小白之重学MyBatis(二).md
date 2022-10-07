# 假装是小白之重学MyBatis(二)

[TOC]

## 前言

本篇我们来介绍MyBatis插件的开发，这个也是来源于我之前的一个面试经历，面试官为我如何统计Dao层的慢SQL，我当时的回答是借助于Spring的AOP机制，拦截Dao层所有的方法，但面试官又问，这事实上不完全是SQL的执行时间，这其中还有其他代码的时间，问我还有其他思路吗？ 我想了想说没有，面试官接着问，有接触过MyBatis插件的开发吗？ 我说没接触过。 但后面也给我过了，我认为这个问题是有价值的问题，所以也放在了我的学习计划中。

看本篇之前建议先看:

- 《代理模式-AOP绪论》
- 《假装是小白之重学MyBatis(一)》

如果有人问上面两篇文章在哪里可以找的到，可以去掘金或者思否翻翻，目前公众号还没有，预计年中会将三个平台的文章统一一下。

## 概述

翻阅官方文档的话，MyBatis并没有给处插件的具体定义，但基本上还是拦截器，MyBatis的插件就是一些能够拦截某些MyBats核心组件方法，增强功能的拦截器。官方文档中列出了四种可供增强的切入点:

- Executor 

> 执行SQL的核心组件。拦截Executor 意味着要干扰或增强底层执行的CRUD操作

- ParameterHandler 

> 拦截该ParameterHandler，意味着要干扰SQL参数注入、读取的动作。

- ResultSetHandler 

> 拦截该ParameterHandler, 要干扰/增强封装结果集的动作

- StatementHandler 

> 拦截StatementHandler ，则意味着要干扰/增强Statement的创建和执行的动作

## 当然还是从HelloWorld开始

要做MyBatis的插件，首先要实现MyBatis的**Interceptor** 接口 , 注意类不要导错了，Interceptor很抢手，该类位于org.apache.ibatis.plugin.Interceptor下。实现该接口，MyBatis会将该实现类当作MyBatis的拦截器，那拦截哪些方法，该怎么指定呢? 通过**@Intercepts**注解来实现，下面是使用示例:

```java
@Intercepts(@Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}))
public class MyBatisPluginDemo implements Interceptor {
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("into invocation ..........");
        System.out.println(invocation.getTarget());
        System.out.println(invocation.getMethod().getName());
        System.out.println(Arrays.toString(invocation.getArgs()));
        return invocation.proceed();
    }
}
```

```
@Intercepts可以填多个@Signature,@Signature是方法签名,type用于定位类，method定位方法名，args用于指定方法的参数类型。三者加在一起就可以定位到具体的方法。注意写完还需要将此插件注册到MyBatis的配置文件中,让MyBatis加载该插件。
```

注意这个标签一定要放在environments上面，MyBatis严格限制住了标签的顺序。

```xml
<plugins>
    <plugin interceptor="org.example.mybatis.MyBatisPluginDemo"></plugin>
</plugins>
```

我们来看下执行结果:

![MyBatis插件](https://tvax2.sinaimg.cn/large/006e5UvNgy1h1t2s201pqj30rh056gro.jpg)

## 性能分析插件走起

那拦截谁呢？ 目前也只有Executor 和StatementHandler 供我们选择，我们本身是要看SQL耗时，Executor 离SQL执行还有些远，一层套一层才走到SQL执行，MyBatis中标签的执行过程在《MyBatis源码学习笔记(一) 初遇篇》已经讲述过了，这里不再赘述，目前来看StatementHandler 是离SQL最近的, 它的实现类就直接走到JDBC了，所以我们拦截StatementHandler ，那有的插入插了很多值，我们要不要拦截，当然也要拦截, 我们的插件方法如下:

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "query",
        args = {Statement.class, ResultHandler.class}), @Signature(type = StatementHandler.class,method =  "update" ,args = Statement.class )})
public class MyBatisSlowSqlPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("-----开始进入性能分析插件中----");
        long startTime = System.currentTimeMillis();
        Object result = invocation.proceed();
        long endTime = System.currentTimeMillis();
       // query方法入参是statement,所以我们可以将其转为Statement
        if (endTime - startTime > 1000){

        }
        return result;
    }
}
```

那对应的SQL该怎么拿？ 我们还是到StatementHandler去看下:

![MyBatis的StementHandler](https://tva1.sinaimg.cn/large/006e5UvNgy1h1t5ejxur6j30nm0eh0xw.jpg)

我们还是得通过Statement这个入参来拿, 我们试试看, 你会发现在日志级别为DEBUG之上，会输出SQL，像下面这样:

![日志级别为INFO](https://tva3.sinaimg.cn/large/006e5UvNly1h1tykqp98aj30ue07g0z3.jpg)

如果日志级别为DEBUG输出会是下面这样:

![日志级别为DEBUG](https://tva2.sinaimg.cn/large/006e5UvNly1h1tyma9yrgj313s09ethe.jpg)

这是为什么呢？ 如果看过《MyBatis源码学习笔记(一) 初遇篇》这篇的可能会想到，MyBatis架构中的日志模块，为了接入日志框架，就会用到代理，那么这个肯定就是代理类，我们打断点来验证一下我们的想法:

![Statement概述](https://tva4.sinaimg.cn/large/006e5UvNgy1h1tyt9w2nyj30q00mqqfm.jpg)



### 代理分析

我原本的想法是PreparedStatementLogger的代理类，仔细一想，感觉不对，感觉自己还是对代理模式了解不大透，于是我就又把之前的文章《代理模式-AOP绪论》看了一下，动态代理模式的目标:

- 我们有一批类，然后我们想在不改变它们的基础之上，增强它们, 我们还希望只着眼于编写增强目标对象代码的编写。
- 我们还希望由程序来编写这些类，而不是由程序员来编写，因为太多了。

在《代理模式-AOP绪论》中我们做的是很简单的代理: 

```java
public interface IRentHouse {
    void rentHouse();
    void study();
}
public class RentHouse implements IRentHouse{
    @Override
    public void rentHouse() {
        System.out.println("sayHello.....");
    }
    @Override
    public void study() {
        System.out.println("say Study");
    }
}
```

我们现在的需求是增强IRentHouse中的方法，用静态代理就是为IRentHouse再做一个实现类，相当于在RentHouse上再包装一层。但如果我有很多想增强的类呢，这样去包装，事实上对代码的侵入性是很大的。对于这种状况，我们最终的选择是动态代理，在运行时产生接口实现类的代理类，我们最终产生代理对象的方法是:

```java
/**
   * @param target 为需要增强的类
   * @return 返回的对象在调用接口中的任意方法都会走到Lambda回调中。
*/
private static  Object getProxy(Object  target){
        Object proxy = Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), (proxy1, method, args) -> {
            System.out.println("方法开始执行..........");
            Object obj = method.invoke(target, args);
            System.out.println("方法执行结束..........");
            return obj;
        });
        return proxy;
  }
```

接下来我们来看下MyBatis是怎么包装的，我们还是从PreparedStatementLogger开始看：

![PreparedStatementLogger](https://tvax2.sinaimg.cn/large/006e5UvNgy1h1u2koc6lpj30gq07n3z7.jpg)

InvocationHandler是动态代理的接口，BaseJdbcLogger这个先不关注。值得关注的是: 

```java
public static PreparedStatement newInstance(PreparedStatement stmt, Log statementLog, int queryStack) {
  InvocationHandler handler = new PreparedStatementLogger(stmt, statementLog, queryStack);
  ClassLoader cl = PreparedStatement.class.getClassLoader();
  return (PreparedStatement) Proxy.newProxyInstance(cl, new Class[]{PreparedStatement.class, CallableStatement.class}, handler);
}
```

可能有同学会问newProxyInstance为什么给了两个参数, 因为CallableStatement继承了PreparedStatement。 这里是一层，事实上还能点出来另外一层，在ConnectionLogger的回调中(ConnectionLogger也实现了InvocationHandler，所以这个也是个代理回调类)，ConnectionLogger的实例化在BaseExecutor这个类里面完成，如果你还能回忆JDBC产生SQL的话，当时的流程事实上是这样的：

```java
    public static boolean execute(String sql, Object... param) throws Exception {
        boolean result = true;
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        try {
            //获取数据库连接
            connection = getConnection();
            connection.setAutoCommit(false);
            preparedStatement = connection.prepareStatement(sql);
    		// 设置参数 
            for (int i = 0; i < param.length; i++) {
                preparedStatement.setObject(i, param[i]);
                preparedStatement.addBatch();
            }
            preparedStatement.executeBatch();
            //提交事务
            connection.commit();
        } catch (SQLException e) {
            e.printStackTrace();
            if (connection != null) {
                try {
                    connection.rollback();
                } catch (SQLException ex) {
                    ex.printStackTrace();
                    // 日志记录事务回滚失败
                    result = false;
                    return result;
                }
            }
            result = false;
        } finally {
            close(preparedStatement, connection);
        }
        return result;
    }
```

我们来捋一下，ConnectionLogger是读Connection的代理，但是Connection接口中有许多方法, 所以ConnectionLogger在回调的时候做了判断:

```java
@Override
public Object invoke(Object proxy, Method method, Object[] params)
    throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, params);
    }
    if ("prepareStatement".equals(method.getName()) || "prepareCall".equals(method.getName())) {
      if (isDebugEnabled()) {
        debug(" Preparing: " + removeExtraWhitespace((String) params[0]), true);
      }
      // Connection 的prepareStatement方法、prepareCall会产生PreparedStatement
      PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
      // 然后PreparedStatementLogger产生的还是stmt的代理类
      // 我们在plugin中拿到的就是  
      stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);
      return stmt;
    } else if ("createStatement".equals(method.getName())) {
      Statement stmt = (Statement) method.invoke(connection, params);
      stmt = StatementLogger.newInstance(stmt, statementLog, queryStack);
      return stmt;
    } else {
      return method.invoke(connection, params);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

![SQL执行链](https://tvax1.sinaimg.cn/large/006e5UvNgy1h1u3lzs78uj30xg0dagp5.jpg)

PreparedStatementLogger是回调类，这个PreparedStatementLogger有对应的Statement，我们通过Statement就可以拿到对应的SQL。那回调类和代理类是什么关系呢， 我们来看下Proxy类的大致构造:

![代理类是如何产生的](https://tva4.sinaimg.cn/large/006e5UvNgy1h1u6thqq28j30p00ndn7s.jpg)

所以我最初的想法是JDK为我们产生的类里面有回调类实例这个对象会有InvocationHandler成员变量，但是如果你用getClass().getDeclaredField("h")去获取发现获取不到，那么代理类就没有这个回调类实例，那我们研究一下getProxyClass0这个方法: 

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    // proxyClassCache 是 new WeakCache<>(new KeyFactory(), new ProxyClassFactory()) 的实例
    // 最终会调用ProxyClassFactory的apply方法。
    // 在ProxyClassFactory的apply方法中有 ProxyGenerator.generateProxyClass() 
    // 答案就在其中,最后调用的是ProxyGenerator的generateClassFile方法
    // 中产生代理类时,让代理类继承Proxy类。
    return proxyClassCache.get(loader, interfaces);
}
```

![动态代理类浅析](https://tvax2.sinaimg.cn/large/006e5UvNly1h1u79urjm0j30o70lbqau.jpg)

所以破案了，在Proxy里的InvocationHandler是protected，所以我们取变量应当这么取:

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "query",
        args = {Statement.class, ResultHandler.class}), @Signature(type = StatementHandler.class,method =  "update" ,args = Statement.class )})
public class MyBatisSlowSqlPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("-----开始进入性能分析插件中----");
        long startTime = System.currentTimeMillis();
        Object result = invocation.proceed();
        long endTime = System.currentTimeMillis();
       // query方法入参是statement,所以我们可以将其转为Statement
        Statement statement = (Statement)invocation.getArgs()[0];
        if (Proxy.isProxyClass(statement.getClass())){
            Class<?> statementClass = statement.getClass().getSuperclass();
            Field targetField = statementClass.getDeclaredField("h");
            targetField.setAccessible(true);
            PreparedStatementLogger  loggerStatement  = (PreparedStatementLogger) targetField.get(statement);
            PreparedStatement preparedStatement = loggerStatement.getPreparedStatement();
            if (endTime - startTime > 1){
                System.out.println(preparedStatement.toString());
            }
        }else {
            if (endTime - startTime > 1){
                System.out.println(statement.toString());
            }
        }
        return result;
    }
}
```

最后输出如下:

![慢SQL监控](https://tvax1.sinaimg.cn/large/006e5UvNly1h1u7nv41suj30kf061dkk.jpg)

但是这个插件还不是那么完美，就是这个慢SQL查询时间了，我们现在是写死的

这两个问题在MyBatis 里面都可以得到解决，我们可以看Interceptor这个接口:

```java
public interface Interceptor {
	
  Object intercept(Invocation invocation) throws Throwable;
 
  default Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  default void setProperties(Properties properties) {
    // NOP
  }
}
```

setProperties用于从配置文件中取值, plugin将当前插件加入，intercept是真正增强方法。那上面的两个问题已经被解决了:

- 硬编码

首先在配置文件里面配置

```xml
 <plugins>
        <plugin interceptor="org.example.mybatis.MyBatisSlowSqlPlugin">
            <property name = "maxTolerate" value = "10"/>
        </plugin>
 </plugins>
```

然后重写: 

```java
@Override
public void setProperties(Properties properties) {
	//maxTolerate 是MyBatisSlowSqlPlugin的成员变量
    this.maxTolerate = Long.parseLong(properties.getProperty("maxTolerate"));
}
```

回忆一下JDBC我们执行SQl事实上有两种方式:

- Connection中的prepareStatement方法
- Connection中的createStatement

在MyBatis中这两种方法对应不同的StatementType, 上面的PreparedStatementLogger对应 Connection中的prepareStatement方法， 如果说你在MyBatis中将语句声明为Statement，则我们的SQL监控语句就会出错，所以这里我们还需要在单独适配一下Statement语句类型。

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "query",
        args = {Statement.class, ResultHandler.class}), @Signature(type = StatementHandler.class,method =  "update" ,args = Statement.class )})
public class MyBatisSlowSqlPlugin implements Interceptor {

    private  long  maxTolerate;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("-----开始进入性能分析插件中----");
        long startTime = System.currentTimeMillis();
        Object result = invocation.proceed();
        SystemMetaObject
        long endTime = System.currentTimeMillis();
       // query方法入参是statement,所以我们可以将其转为Statement
        Statement statement = (Statement)invocation.getArgs()[0];
        if (Proxy.isProxyClass(statement.getClass())){
            Class<?> statementClass = statement.getClass().getSuperclass();
            Field targetField = statementClass.getDeclaredField("h");
            targetField.setAccessible(true);
            Object object = targetField.get(statement);
            if (object instanceof PreparedStatementLogger) {
                PreparedStatementLogger  loggerStatement  = (PreparedStatementLogger) targetField.get(statement);
                PreparedStatement preparedStatement = loggerStatement.getPreparedStatement();
                if (endTime - startTime > maxTolerate){
                    System.out.println(preparedStatement.toString());
                }
            }else {
                // target 是对应的语句处理器
                // 为什么不反射拿? Statement 对应的实现类未重写toString方法
                // 但是在RoutingStatementHandler 中提供了getBoundSql方法
                RoutingStatementHandler handler = (RoutingStatementHandler) invocation.getTarget();
                BoundSql boundSql = handler.getBoundSql();
                if (endTime - startTime > maxTolerate){
                    System.out.println(boundSql);
                }
            }
        }else {
            if (endTime - startTime > maxTolerate){
                System.out.println(statement.toString());
            }
        }
        return result;
    }

    @Override
    public void setProperties(Properties properties) {
        this.maxTolerate = Long.parseLong(properties.getProperty("maxTolerate"));
    }
}
```

事实上MyBatis里面写好了反射工具类，这个就是SystemMetaObject，用法示例如下:

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "query",
        args = {Statement.class, ResultHandler.class}), @Signature(type = StatementHandler.class,method =  "update" ,args = Statement.class )})
public class MyBatisSlowSqlPlugin implements Interceptor {

    private  long  maxTolerate;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("-----开始进入性能分析插件中----");
        long startTime = System.currentTimeMillis();
        Object result = invocation.proceed();
        long endTime = System.currentTimeMillis();
       // query方法入参是statement,所以我们可以将其转为Statement
        Statement statement = (Statement)invocation.getArgs()[0];
        MetaObject metaObject = SystemMetaObject.forObject(statement);
        if (Proxy.isProxyClass(statement.getClass())){
            Object object = metaObject.getValue("h");
            if (object instanceof PreparedStatementLogger) {
                PreparedStatementLogger  loggerStatement  = (PreparedStatementLogger) object;
                PreparedStatement preparedStatement = loggerStatement.getPreparedStatement();
                if (endTime - startTime > maxTolerate){
                    System.out.println(preparedStatement.toString());
                }
            }else {
                // target 是对应的语句处理器
                // 为什么不反射拿? Statement 对应的实现类未重写toString方法
                // 但是在RoutingStatementHandler 中提供了getBoundSql方法
                RoutingStatementHandler handler = (RoutingStatementHandler) invocation.getTarget();
                BoundSql boundSql = handler.getBoundSql();
                if (endTime - startTime > maxTolerate){
                    System.out.println(boundSql);
                }
            }
        }else {
            if (endTime - startTime > maxTolerate){
                System.out.println(statement.toString());
            }
        }
        return result;
    }

    @Override
    public void setProperties(Properties properties) {
        this.maxTolerate = Long.parseLong(properties.getProperty("maxTolerate"));
    }
}
```

那我有多个插件，如何指定顺序呢? 在配置文件中指定，从上往下依次执行

```java
 <plugins>
        <plugin interceptor="org.example.mybatis.MyBatisSlowSqlPlugin01">
            <property name = "maxTolerate" value = "10"/>
        </plugin> 
     <plugin interceptor="org.example.mybatis.MyBatisSlowSqlPlugin02">
            <property name = "maxTolerate" value = "10"/>
        </plugin> 
</plugins>
```

如上面所配置执行顺序就是MyBatisSlowSqlPlugin01、MyBatisSlowSqlPlugin02。  插件的几个方法执行顺序呢

![执行顺序](https://tvax4.sinaimg.cn/large/006e5UvNgy1h1ubdfbzekj30vg0g9q5g.jpg)

## 写在最后

感慨颇深，原本预计两个小时就能写完的，然后写了一下午，颇有种学海无涯的感觉。

## 参考资料

- mybatis拦截器插件实例-获取不带占位符的可直接执行的sql  B站教学视频  https://www.bilibili.com/video/BV1Fh411p7K1?spm_id_from=333.337.search-card.all.click
- MyBatis高级 https://www.bilibili.com/video/BV1Q4411579K?spm_id_from=333.337.search-card.all.click