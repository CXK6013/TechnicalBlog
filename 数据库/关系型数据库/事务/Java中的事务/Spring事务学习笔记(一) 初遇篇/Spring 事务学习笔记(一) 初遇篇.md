# Spring 事务学习笔记(一) 初遇篇	

[TOC]

## 前言

在《数据库事务概论》、《MySQL事务学习笔记(一)》, 我们已经讨论过了事务相关概念，事务相关的操作，如何开始事务，回滚等。那在程序中我们该如何做事务的操作呢。在Java中是通过JDBC API来控制事务，这种方式相对来说有点原始，现代 Java Web领域一般都少不了Spring 框架的影子，Spring 为我们提供了控制事务的简介方案，下面我们就来介绍Spring 中如何控制事务的。 建议在阅读本文之前先预读:

- 《代理模式-AOP绪论》
- 《欢迎光临Spring时代(二) 上柱国AOP列传》

## 声明式事务: @Transational 注解

### 简单使用示例

```java
@Service
public class StudentServiceImpl implements StudentService{
    
    @Autowired
    private StudentInfoDao studentInfoDao;
    
    @Transactional(rollbackFor = Exception.class) // 代表碰见任何Exception和其子类都会回滚
    @Override
    public void studyTransaction() {
        studentInfoDao.updateById(new StudentInfo());
    }
}
```

这是Spring 为我们提供的一种优雅控制事务的方案，但这里有个小坑就是如果方法的修饰符不是public，则@Transational就会失效。原因在于Spring通过TransactionInterceptor来拦截有@Transactional注解的类和方法，

![Spring中的事务控制](https://tva2.sinaimg.cn/large/006e5UvNly1gzjwx3uyn4j30ww0abtbs.jpg)

注意看TransactionInterceptor实现了MethodInterceptor接口，如果对Spring比较熟悉的话，可以直到这是一个环绕通知，在方法要执行的时候，我们就可以增强这个类，在这个方法执行之前、执行之后，做些工作。

![事务拦截器方法调用](https://tva4.sinaimg.cn/large/006e5UvNly1gzjx0ycwzzj30tu0e57ch.jpg)

方法调用链如下: 

![事务拦截调用链](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ce164934ef54bf9936d1e36b9515b90~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![计算事务属性](https://tvax3.sinaimg.cn/large/006e5UvNly1gzjxc560nxj30w70iak21.jpg)

得知真相的我眼泪掉下来，我记得是我哪一次面试的时候，哪个面试官问我的，当时我是不知道Spring的事务拦截器会有这样的操作的，因为我潜意识中是觉得，AOP的原理是动态代理，不管是啥方法，我都能代理。我看@Transational注解中的注释也没说，以为什么方法修饰符都能生效呢。

### 属性选讲

上面我们只使用了@Transational的一个属性rollbackFor，这个属性用于控制方法在发生了什么异常的情况下回滚，现在我们进@Transational简单的看下还有哪些属性：

- value 和 transactionManager是同义语 用于指定事务管理器   4.2版本开始提供

>数据库的事务在被Java领域的框架控制，有不同的实现。比如Jdbc事务、Hibernate事务等
>
>Spring进行了统一的抽象，形成了PlatformTransactionManager、ReactiveTransactionManage两个事务管理器次顶级接口。两个类继承自TransactionManager.
>
>![事务管理器顶层接口](https://tva2.sinaimg.cn/large/006e5UvNly1gzk0vn0qlcj30la06uq3q.jpg)
>
>我们在用Spring整合的时候，如果是有连接池来管理连接，Spring有DataSourceTransactionManager来管理事务。
>
>![DataSourceTransactionManager](https://tvax4.sinaimg.cn/large/006e5UvNly1gzk0znuuubj30vv0ao0vx.jpg)

> 如果你用的是Spring Data JPA，Spring Data Jpa还带了一个JpaTransationManager.
>
> ![JpaTransationalManager](https://tvax3.sinaimg.cn/large/006e5UvNly1gzk1581lqpj31190cw77v.jpg)

> 如果你使用的是Spring-boot-jdbc-starter,那么Spring Boot 会默认注入DataSourceTransactionManager，当做事务管理器。如果使用了spring-boot-starter-data-jpa, 那么Spring Boot默认会采用 JpaTransactionManager。

- label 5.3 开始提供

>Defines zero (0) or more transaction labels.
>Labels may be used to describe a transaction, and they can be evaluated by individual transaction managers. Labels may serve a solely descriptive purpose or map to pre-defined transaction manager-specific options.
>
>定义一个事务标签，用来描述一些特殊的事务，来被一些预先定义的事务管理器特殊处理。

- Propagation  传播行为

  - REQUIRED 

    > 默认选项, 如果当前方法不存在事务则创建一个，如果当前方法存在事务则加入。

  - SUPPORTS

    > 支持当前事务，如果当前没有事务，就以非事务的方式来执行。

  - MANDATORY

    > 使用当前方法的事务，如果当前方法不存在事务，则抛出异常。

    ```java
    @Transactional(propagation = Propagation.MANDATORY)
    @Override
    public void studyTransaction() {
        Student studentInfo = new Student();
        studentInfo.setId(1);
        studentInfo.setName("ddd");
        studentInfoDao.updateById(studentInfo);
    }
    ```

    结果: 

    <img src="https://tva3.sinaimg.cn/large/006e5UvNly1gzk1u5wbqhj314g03zq8d.jpg" alt="抛了一个异常" style="zoom:200%;" />

    ```java
      @Transactional(rollbackFor = Exception.class)
      @Override
       public void testTransaction() {
            studyTransaction(); // 这样就不会报错了
       }
    ```

  - REQUIRES_NEW

    > Create a new transaction, and suspend the current transaction if one exists. Analogous to the EJB transaction attribute of the same name.
    >
    > 创建一个新的事务，如果当前已经处于一个事务内，则挂起所属的事务，同EJB事务的属性有相似的名字。
    >
    > NOTE: Actual transaction suspension will not work out-of-the-box on all transaction managers. This in particular applies to org.springframework.transaction.jta.JtaTransactionManager, which requires the javax.transaction.TransactionManager to be made available to it (which is server-specific in standard Java EE).
    >
    > 注意，不是所有的事务管理器都会承认此属性，挂起属性只被JtaTransactionManager事务管理器所承认。(也有对挂起的理解是先不提交, 等待其他事务的提交之后，再提交。我认为这个理解也是正确的。JtaTransactionManager是一个分布式事务管理器,)
    >
    > 所以我实测，没有挂起现象。现在我们来看看有没有开启一个新事务。

    ```mysql
    SELECT TRX_ID FROM information_schema.INNODB_TRX  where TRX_MYSQL_THREAD_ID = CONNECTION_ID(); // 可以查看事务ID
    ```

     我在myBatis里做了测试，输出两个方法的事务ID：

    ```java
      @Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
       @Override
        public void studyTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            studentInfoDao.selectById(1);
            System.out.println(studentInfoDao.getTrxId());
        }
    
        @Transactional(rollbackFor = Exception.class)
        @Override
        public void testTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            studentInfoDao.selectById(1);
            System.out.println(studentInfoDao.getTrxId());
            studyTransaction();
        }
    ```

    结果:  ![也没开启事务](https://tva1.sinaimg.cn/large/006e5UvNly1gzk2slz93uj30m804bgov.jpg)

    网上其他博客大多都是会开启一个事务，现在看来并没有，但是网上看到有人做测试的时候，发生回滚了，测试方法的原理是testTransaction()执行更新数据库，studyTransaction也更新数据库，studyTransaction方法抛异常看是否回滚，我们来用另一种测试，testTransaction更新，看studyTransaction中能不能查到，如果在一个事务中应当是能查到的。如果查不到更新那说明就不再一个事务中。

    ```java
    @Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
    @Override
    public void studyTransaction() {
        // 先执行任意一句语句,不然不会有事务ID产生
        System.out.println(studentInfoDao.selectById(2).getNumber());
        System.out.println(studentInfoDao.getTrxId());
    }
    
    @Transactional(rollbackFor = Exception.class)
    @Override
    public void testTransaction() {
        // 先执行任意一句语句,不然不会有事务ID产生
        Student student = new Student();
        student.setId(1);
        student.setNumber("LLL");
        studentInfoDao.updateById(student);
        studyTransaction();
    }
    ```

    然后没输出LLL, 看来确实是新起了一个事务。

  - NOT_SUPPORTED

    >Execute non-transactionally, suspend the current transaction if one exists. Analogous to EJB transaction attribute of the same name.
    >NOTE: Actual transaction suspension will not work out-of-the-box on all transaction managers. This in particular applies to org.springframework.transaction.jta.JtaTransactionManager, which requires the javax.transaction.TransactionManager to be made available to it (which is server-specific in standard Java EE).
    >
    >以非事务的方式运行，如果当前方法存在事务则挂起。仅被JtaTransactionManager支持。

    ```java
        @Transactional(propagation = Propagation.NOT_SUPPORTED,rollbackFor = Exception.class)
        @Override
        public void studyTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            System.out.println("studyTransaction方法的事务ID: "+studentInfoDao.getTrxId());
        }
    
        @Transactional(rollbackFor = Exception.class)
        @Override
        public void testTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            studentInfoDao.selectById(1);
            System.out.println("testTransactiond的事务Id: "+studentInfoDao.getTrxId());
            studyTransaction();
        }
    ```

    验证结果： ![NotSupported](https://tva2.sinaimg.cn/large/006e5UvNly1gzk2xvui7yj30it059ad7.jpg)

    似乎加入到了testTransaction中，没有以非事务的方式运行，我不死心，我要再试试。

    ```java
      @Override
        public void studyTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            Student student = new Student();
            student.setId(1);
            student.setNumber("cccc");
            studentInfoDao.updateById(student);
            // 如果是以非事务运行,那么方法执行完应当,别的方法应当立即能查询到这条数据。
            try {
                TimeUnit.SECONDS.sleep(30);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        @Transactional(rollbackFor = Exception.class)
        @Override
        public void testTransaction() {
            // 先执行任意一句语句,不然不会有事务ID产生
            System.out.println(studentInfoDao.selectById(1).getNumber());
        }
    ```

    输出结果:  testTransaction方法输出位cccc。 确实是以非事务方式在运行。

  - NEVER

    >Execute non-transactionally, throw an exception if a transaction exists
    >
    >以非事务的方式运行，如果当前方法存在事务，则抛异常。

    ```java
    	@Transactional(propagation = Propagation.NEVER)
        @Override
        public void studyTransaction() {
            System.out.println("hello world");
        }
        @Transactional(rollbackFor = Exception.class)
        @Override
        public void testTransaction() {
            studyTransaction();
        }
    ```

    没抛异常，难道是因为我没执更新语句？ 我发现我里面写了更新语句也是一样的情况，原先在于这两个方法在一个类里面，在另一个接口实现类里面调studyTransaction方法, 像下面这样就会抛出异常： 

    ```java
    @Service
    public class StuServiceImpl implements  StuService{
        @Autowired
        private StudentService studentService;
        
        @Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRED)
        @Override
        public void test() {
            studentService.studyTransaction();
        }
    
    }
    ```

    就会抛出如下的异常:

    ![传播行为为NEVER](https://tva1.sinaimg.cn/large/006e5UvNly1gzqqmq7m76j310309x15n.jpg)

    那这是事务失效吗？ 我们统一放到下文事务的失效场景来讨论。

  - NESTED

    >Execute within a nested transaction if a current transaction exists, behave like REQUIRED otherwise. There is no analogous feature in EJB.
    >Note: Actual creation of a nested transaction will only work on specific transaction managers. Out of the box, this only applies to the JDBC DataSourceTransactionManager. Some JTA providers might support nested transactions as well.
    >See Also:org.springframework.jdbc.datasource.DataSourceTransactionManager
    
    如果当前存在一个事务，当作该事务的子事务，同REQUIRED类似。注意，事实上这个特性仅被一些特殊的事务管理器所支持。在DataSourceTransactionManager可以做到开箱即用。
    
    那该怎么理解这个嵌套的子事务，还记得我们《MySQL事务学习笔记(一) 初遇篇》提到的保存点吗？ 这个NESTED就是保存点意思，假设 A方法调用B方法，A方法的传播行为是REQUIRED，B的方法时NESTED。A调用B，B发生了异常，只会回滚B方法的行为，A不受牵连。
    
    ```java
    @Transactional(propagation = Propagation.NESTED,rollbackFor = Exception.class)
    @Override
    public void studyTransaction() {
        Student student = new Student();
        student.setId(1);
        student.setNumber("bbbb");
        studentInfoDao.updateById(student);
        int i = 1 / 0;
    }
    @Transactional(rollbackFor = Exception.class)
    @Override
    public void testTransaction() {
        // 先执行任意一句语句,不然不会有事务ID产生
        Student student = new Student();
        student.setId(1);
        student.setNumber("LLL");
        studentInfoDao.updateById(student);
        studyTransaction();
    }
    ```
    
    这样我们会发现还是整个都回滚了,原因在于studyTransaction方法抛出的异常也被 testTransaction()所处理. 但是就是你catch住了会发现还是整个回滚，但是如果你在另一个service先后调用studyTransaction、testTransaction就能做到局部回滚。像下面这样:
    
    ```java
    @Service
    public class StuServiceImpl implements  StuService{
        @Autowired
        private StudentService studentService;
    
        @Autowired
        private StudentInfoDao studentInfoDao;
    
        @Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRED)
        @Override
        public void test() {
            Student student = new Student();
            student.setId(1);
            student.setNumber("jjjjj");
            studentInfoDao.updateById(student);
            try {
                studentService.studyTransaction();
            }catch (Exception e){
    
            }
        }
    ```
    
    或者在调用studyTransaction，自己注入自己也能起到局部回滚的效果，想下面这样：
    
    ```java
    @Service
    public class StudentServiceImpl implements StudentService, ApplicationContextAware {
    
        @Autowired
        private StudentInfoDao studentInfoDao;
    
        @Autowired
        private StudentService studentService
        
        @Transactional(rollbackFor = Exception.class)
        @Override
        public void testTransaction() {
            Student student = new Student();
            student.setId(1);
            student.setNumber("qqqq");
            studentInfoDao.updateById(student);
            try {
                studentService.studyTransaction();
            }catch (Exception e){
    
            }
        }
      }
    ```

- isolation 隔离级别

> 是一个枚举值, 我们在《MySQL事务学习笔记(一) 初遇篇》已经讨论过，可以通过此属性指定隔离级别，一共有四个:
>
> - DEFAULT 跟随数据库的隔离级别
> - READ_UNCOMMITTED
> - READ_COMMITTED
> - REPEATABLE_READ
> - SERIALIZABLE

- timeout 超时时间

> 超过多长时间未提交，则自动回滚。

- rollbackFor 

- rollbackForClassName
- noRollbackFor
- noRollbackForClassName

```java
 @Transactional(noRollbackForClassName = "ArithmeticException",rollbackFor = ArithmeticException.class )
   @Override
    public void studyTransaction() {
        Student studentInfo = new Student();
        studentInfo.setId(1);
        studentInfo.setName("ddd");
        studentInfoDao.updateById(studentInfo);
        int i = 1 / 0;
    }
```

noRollback和RollbackFor指定相同的类，优先走RollbackFor。	

### 事务失效场景

上面事实上我们已经讨论了一种事务的失效场景，即方法被修饰的方法是private的。如果想要对private方法级别生效，则需要开启AspectJ 代理模式。开启也比较麻烦，知乎搜索: Spring Boot教程(20) – 用AspectJ实现AOP内部调用 , 里面讲如何开启，这里就不再赘述了。

再有就是在事务传播行为中设置为NOT_SUPPORTED。

上面我们在讨论的事务管理器，如果事务管理器没有被纳入到Spring的管辖范围之内，那么方法有@Transactional也不会生效。

类中方法自调用，像下面这样:

```java
 @Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)
    @Override
    public void studyTransaction() {
        Student student = new Student();
        student.setId(1);
        student.setNumber("aaaaaaLLL");
        studentInfoDao.updateById(student);
        int i = 1 / 0;
    }
  @Override
  public void testTransaction() {
        studyTransaction();
 }
```

   这样还是不会发生回滚。原因还是才从代理模式说起，我们之所以在方法和类上加事务注解就能实现对事务的管理，本质上还是Spring再帮我们做增强，我们在调用方法上有@Transactional的方法上时，事实上调用的是代理类，像没有事务注解的方法，Spring去调用的时候就没有用代理类。如果是有事务注解的方法调用没事务注解的方法，也不会失效，原因是一样的，调用被@Transactional事实上调用的是代理类，开启了事务。

- 对应的数据库未开启支持事务，比如在MySQL中就是数据库的表指定的引擎MyIsam。
- 打上事务的注解没有使用一个数据库连接，也就是多线程调用。像下面这样:

```java
   @Override
    @Transactional
    public void studyTransaction() {
       // 两个线程可能使用不同的连接,类似于MySQL开了两个黑窗口,自然互相不影响。
        new Thread(()-> studentInfoDao.insert(new Student())).start();
        new Thread(()-> studentInfoDao.insert(new Student())).start();
    }
```



## 编程式事务简介

与声明式事务相反，声明式事务像是自动挡，由Spring帮助我们开启、提交、回滚事务。而编程式事务则像是自动挡，我们自己开启、提交、回滚。如果有一些代码块需要用到事务，在方法上加事务显得太过笨重，不再方法上加，在抽出来的方法上加又会导致失效，那么我们这里就可以考虑使用编程式事务.Spring 框架下提供了两种编程式事务管理:

- TransactionTemplate(Spring 会自动帮我们回滚释放资源)
- PlatformTransactionManager(需要我们手动释放资源)

```java
@Service
public class StudentServiceImpl implements StudentService{

    @Autowired
    private StudentInfoDao studentInfoDao;


    @Autowired
    private TransactionTemplate transactionTemplate;


    @Autowired
    private PlatformTransactionManager platformTransactionManager;

    @Override
    public void studyTransaction() {
        // transactionTemplate可以设置隔离级别、传播行为等属性
        String result = transactionTemplate.execute(status -> {
            testUpdate();
            return "AAA";
        });
        System.out.println(result);
    }

    @Override
    public void testTransaction() {
        // defaultTransactionDefinition   可以设置隔离级别、传播行为等属性
        DefaultTransactionDefinition defaultTransactionDefinition = new DefaultTransactionDefinition();
        TransactionStatus status = platformTransactionManager.getTransaction(defaultTransactionDefinition);
        try {
            testUpdate();
        }catch (Exception e){
            // 指定回滚
            platformTransactionManager.rollback(status);
        }
        studyTransaction();// 提交
        platformTransactionManager.commit(status);
    }

    private void testUpdate() {
        Student student = new Student();
        student.setId(1);
        student.setNumber("aaaaaaLLL111111qqqw");
        studentInfoDao.updateById(student);
        int i = 1 / 0;
    }
}
```

##  总结一下

Spring 为我们统一管理了事务，Spring提供的管理事务的方式大致上可以分为两种:

- 声明式事务 @Transactional 
- 编程式事务 TransactionTemplate 和 PlatformTransactionManager

如果你想享受Spring的提供的事务管理遍历，那么前提需要你将事务管理器纳入到容器的管理范围。

## 参考资料

- 详解Spring的事务管理PlatformTransactionManager  https://www.jianshu.com/p/903c01cb2a77
- 解惑 spring 嵌套事务  https://blog.csdn.net/z69183787/article/details/76212680?locationNum=2&fps=1
- Spring事务在哪几种情况下会失效？ https://www.zhihu.com/question/506754250/answer/2282397702
- Spring事务在哪几种情况下会失效？ https://www.zhihu.com/question/506754250/answer/2291365775
- Spring编程式事务管理 https://zhuanlan.zhihu.com/p/398713906
