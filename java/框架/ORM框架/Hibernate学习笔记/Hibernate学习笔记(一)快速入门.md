# Hibernate学习笔记(一)快速入门

[TOC]

## 前言

毕业以来，我一直用MyBatis比较多，像另一种思路的ORM框架Hibernate，还一直没用过，也想起实习的架构师吐槽MyBatis蠢，今天就来换一种思路来学习一下Hibernate。学习本篇要求懂maven和jdbc。不懂maven参看下面的文章:

- Maven学习笔记 https://segmentfault.com/a/1190000019897882

   不懂JDBC，哈哈，可以在B站上随手搜个视频看下。照以往文章的思路肯定是先讲下使用原生JDBC的痛点，然后引出ORM框架，但是本篇想换个思路，我们先用起来，在实践中体会Hibernate带给我们的便利，现在我们将Hibernate理解为一个ORM框架，给我们提供了一组API，可以让我们方便快捷的实现CRUD。

## 先用起来

目前我学习一个框架、中间件，一般都会习惯性的去官网，看看有没有详细的用户指导，Hibernate这一点做的相当优秀:

![Vb5ri8.jpeg](https://i.imgloc.com/2023/06/17/Vb5ri8.jpeg)



之前听到一种论调说Hibernate已经被淘汰了，看看官网Hibernate还活的好好的，我们本篇只看Hibernate ORM：

![Vb53bZ.jpeg](https://i.imgloc.com/2023/06/17/Vb53bZ.jpeg)



我本来想介绍6.2版本的，但是6.2最低是JDK 11，5.6最低是JDK 1.8，那这里就介绍5.6版本。 我们先开始用，首先我们引入maven依赖:

```xml
 <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-core</artifactId>
      <version>5.6.6.Final</version>
 </dependency>
<dependency>
   <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.29</version>
</dependency>
<dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>1.7.5</version>
</dependency>
```

我们本篇是快速入门，看的是Hibernate 文档中的《Getting Started Guide》: 

![Vbs73v.jpeg](https://i.imgloc.com/2023/06/17/Vbs73v.jpeg)

这个文档还是点名表扬一下的，非常详细，真的非常详细，里面有下载jar的链接，maven的依赖。还是从配置文件开始讲起。本篇指导用到的代码还给了下载链接：

![VbvNnQ.jpeg](https://i.imgloc.com/2023/06/17/VbvNnQ.jpeg)

然后我们下载下来，参考一下。在base模块我找到了配置文件: 

```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>

    <session-factory>

        <!-- 这里配置数据相关的信息驱动类  -->
        <property name="connection.driver_class">org.h2.Driver</property>
        <!-- JDBC  -->
        <property name="connection.url">jdbc:h2:mem:db1;DB_CLOSE_DELAY=-1</property>
        <property name="connection.username">sa</property>
        <property name="connection.password"/>
		
        <!-- JDBC connection pool (use the built-in) 连接池大小1 -->
        <property name="connection.pool_size">1</property>

        <!-- SQL dialect 指定方言 不同的数据库有不同的方言 -->
        <property name="dialect">org.hibernate.dialect.H2Dialect</property>

        <!-- Disable the second-level cache 禁用缓存  -->
        <property name="cache.provider_class">org.hibernate.cache.internal.NoCacheProvider</property>

        <!-- Echo all executed SQL to stdout -->
        <property name="show_sql">true</property>

        <!-- 自动根据实体来创建表-->
        <property name="hbm2ddl.auto">create</property>
		
        <!--在这里指定映射关系-->
        <mapping resource="org/hibernate/tutorial/hbm/Event.hbm.xml"/>

    </session-factory>
</hibernate-configuration>
```

Hibernate的配置文件叫hibernate.cfg.xml，注意这个文件放在resources下面。然后我们建实体类和映射文件:

```java
package org.hibernate.tutorial.hbm;

import java.util.Date;

public class Event {
	private Long id;

	private String title;
	private Date date;

	public Event() {
		// this form used by Hibernate
	}

	public Event(String title, Date date) {
		// for application use, to create new events
		this.title = title;
		this.date = date;
	}

	public Long getId() {
		return id;
	}

	private void setId(Long id) {
		this.id = id;
	}

	public Date getDate() {
		return date;
	}

	public void setDate(Date date) {
		this.date = date;
	}

	public String getTitle() {
		return title;
	}

	public void setTitle(String title) {
		this.title = title;
	}
}
```

```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC
        "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping package="org.hibernate.tutorial.hbm">
	 <!-- 这里将实体类和表名进行绑定 -->
    <class name="Event" table="EVENTS">
        <id name="id" column="EVENT_ID">
      	 <!--设定ID产生策略-->
            <generator class="increment"/>
        </id>
        <property name="date" type="timestamp" column="EVENT_DATE"/>
        <property name="title"/>
    </class>
</hibernate-mapping>
```

下面是测试类:

```java
package org.hibernate.tutorial.hbm;

import java.util.Date;
import java.util.List;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;

import junit.framework.TestCase;
/**
 * 
 *
 * 这个会启动setUp 再执行testBasicUsage，再执行 tearDown
 */
public class NativeApiIllustrationTest extends TestCase {
	private SessionFactory sessionFactory;
	
  
	@Override
	protected void setUp() throws Exception {
		// A SessionFactory 在一个应用只设置一次
		final StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
				.configure() // configure可以指定Hibernate的配置文件, 这里让Hibernate去找默认的配置文件
				.build();
		try {
			sessionFactory = new MetadataSources( registry ).buildMetadata().buildSessionFactory();
		}catch (Exception e) {	
            // registry  会被SessionFactory锁销毁,我们会在构建Session可能遇到一些问题
            // 所以要手动销毁它
			StandardServiceRegistryBuilder.destroy( registry );
		}
	}

	@Override
	protected void tearDown() throws Exception {
        // 执行完毕,关闭sessionFactory	
		if ( sessionFactory != null ) {
			sessionFactory.close();
		}
	}

	@SuppressWarnings("unchecked")
	public void testBasicUsage() {
		// 创建一对Event,开启一个Session,
		Session session = sessionFactory.openSession();
        // 开启事务
		session.beginTransaction();
        // 插入一条记录
		session.save( new Event( "Our very first event!", new Date() ) );
        // 插入一条记录
		session.save( new Event( "A follow up event", new Date() ) );
        // 提交事务
		session.getTransaction().commit();
        // 关闭session
		session.close();
		
		session = sessionFactory.openSession();
		session.beginTransaction();
        // 从数据库里面拉取记录,这个语法被称为HQL
		List result = session.createQuery( "from Event" ).list();
		for ( Event event : (List<Event>) result ) {
			System.out.println( "Event (" + event.getDate() + ") : " + event.getTitle() );
		}
        // 提交事务
		session.getTransaction().commit();
        // 关闭
		session.close();
	}
}
```

运行这个测试用例就会发现, 数据库里面多了一张表:

![VjZHgd.jpeg](https://i.imgloc.com/2023/06/17/VjZHgd.jpeg)

我们已经用Hibernate完成了表的创建、新增数据、查询数据。 我们来分析一下这个流程。

## 梳理流程

除开引入依赖这个步骤，我们一共有三个步骤:

- 编写Hibernate的配置文件，在这个文件中，我们声明了数据库相关的配置，设置展示生成实际的SQL，禁用缓存，设置了dialect(方言)。

  > 我们来解释一下这个方言,  关系型数据库的一些语法是通用的，比如join、group by，因为关系型数据库都遵循一个标准，但是每个数据库也有自己独有的语法，函数。 比如oracle的开窗函数，到MySQL 8.0才有。在标准之外的地方，关系型数据库独有的实现，被称为方言。

- 然后编写实体类，映射文件，在表和实体类之间建立映射关系。
- 然后编写测试类，加载配置文件，构造SessionFactory，获得一个session，我们用这个session就完成了新增、查询、建表。

对比MyBatis的话，就是SQL几乎完全不用我们编写。

## Hibernate概述

### 理念

> 在面向对象的软件和关系型数据库工作可能会感到繁琐和耗时。由于对象和关系数据库在数据表示存在着不匹配，开发成本显著增加。Hibernate是针对Java中的对象和关系型数据映射的解决方案(ORM), ORM是指将数据从对象模型映射到关系型数据，再从关系型数据映射到对象模型。
>
> Hibernate不仅负责将Java中的类映射到数据库表(将Java中的数据类型映射到SQL数据类型)，还提供数据查询和检索功能。可以显著减少开发时间，避免手动编写SQL和JDBC进行数据处理。Hibernate的设计目标是将开发者从95%的数据持久化性相关的编程任务解脱出来。然而，与其他许多持久化解决方案不同，Hibernate并没有向你隐藏SQL的力量，你在关系型技术和知识方面的投资依然有效。

能帮我从百分之九十五的数据持久化相关的编程任务解脱出来？ 我喜欢。我们接着往下看。

### 架构

![Data Access Layers](https://docs.jboss.org/hibernate/orm/6.2/userguide/html_single/images/architecture/data_access_layers.svg)

> Hibernate, as an ORM solution, effectively "sits between" the Java application data access layer and the Relational Database, as can be seen in the diagram above. 
>
> Hibernate是一个ORM解决方案，位于Java数据访问层和关系型数据库之间，如上图所示。

> The Java application makes use of the Hibernate APIs to load, store, query, etc. its domain data. Here we will introduce the essential Hibernate APIs. This will be a brief introduction; we will discuss these contracts in detail later.
>
> Java应用可以使用Hibernate APIs 对数据进行增删改查。 这里我们将介绍基本的Hibernate APIs，这是一个简单的介绍，后面详细讨论这些细节。

> As a Jakarta Persistence provider, Hibernate implements the Java Persistence API specifications and the association between Jakarta Persistence interfaces and Hibernate specific implementations can be visualized in the following diagram:
>
> 作为Jakarta Persistence  的提供者，Hibernate 实现了Java 持久性API规范，Jakarta持久性接口和Hibernate具体实现之间的联系可以在下图中得到体现。

![image](https://docs.jboss.org/hibernate/orm/6.2/userguide/html_single/images/architecture/JPA_Hibernate.svg)

The Java Persistence API provides Java developers with an object/relational mapping facility for managing relational data in Java applications. Java Persistence consists of four areas:

> Java持久化API向Java开发者提供了一个在Java应用程序中管理对象和关系映射的工具。Java 持久化设计了四个方面: 

- The Java Persistence API

> 持久化API

- The query language

> 查询语言

- The Java Persistence Criteria API

> Java持久化标准API

- Object/relational mapping metadata

> 对象/关系映射元数据。

总结一下，JPA 是一组接口，提供了操作数据库的API，而Hibernate提供了实现。Hibernate提供了两方面的能力，一方面是JPA的实现，另一方面是Hibernate自己提供的能力。我们先看Hibernate提供的能力，再看JPA的实现。

###  简单介绍JDBC

软件的世界有着形形色色的关系型数据库，这些数据库天生的存在差异，虽然他们都遵循一个标准，但是也有自己的实现。那Java该如何操作各个数据库呢，由JDK的维护人员为每个数据库都开发一个SDK，开发人员需要哪个数据库，就去找哪个数据库的SDK?  那是为每一个数据库都专门设计一套工具包，还是每个数据库的SDK都是由一组规范约束，每个数据库的SDK实现了规范，但是实现不同。当然是每个数据库的SDK由一组规范约束，能够减少开发者的心智负担更好，那为什么JDK的开发人员不给规范呢？ 为什么不让各大数据库厂商给实现呢？ Java中的接口就可以当作规范，所以JDK的开发人员就给了一组接口，也就是JDBC:

> Java DataBase Connectivity   API为Java提供了访问各种数据源的通用数据访问方式。通过JDBC API你可以几乎访问任何数据源，包括关系型数据库，excel，二进制格式的文件。JDBC包括两个包:
>
> - java.sql
> - javax.sql
>
> 要使用JDBC API与特定的数据库进行交互，你需要一个基于JDBC技术的驱动程序来作为数据库和JDBC之间的中介。 驱动程序可以完全使用Java语言来编写，也可以使用Java编程语言和JNI的混合形式来编写。

![VIURKx.jpeg](https://i.imgloc.com/2023/06/22/VIURKx.jpeg)

那现在来说操纵数据的方法有了，那么从数据里面查出来的数据怎么变成Java的类呢，我们还要解决的问题就是，数据库的类型和Java数据类型的映射关系，JDBC定义了数据库的基本数据类型，是一个枚举，叫JDBCType，位于java.sql，但是具体数据库的类型应该对应Java中的什么类型，不同的数据库有不同的答案，MyBatis的答案参看下面这个页面:

- https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers

我们可以看到MyBatis的答案基本是符合我们直觉的，BOOLEAN对boolean，Byte对Byte。由不同的类型处理器帮我们将对应的数据库数据类型转换成Java对应的数据类型。那Hibernate是如何处理的呢？

### Hibernate的答案是

这次我们也依然看例子，再介绍理念，要操作数据库，首先我们要有一张表:

```sql
create table Contact (
    id integer not null,
    first varchar(255),
    last varchar(255),
    middle varchar(255),
    notes varchar(255),
    starred boolean not null,
    website varchar(255),
    primary key (id)
)
```

然后我们需要有一个实体类:

```java
@Entity(name = "Contact")
public static class Contact {

	@Id
	private Integer id;

	private Name name;

	private String notes;

	private URL website;

	private boolean starred;

	//Getters and setters are omitted for brevity
}

@Embeddable
public class Name {

	private String firstName;

	private String middleName;

	private String lastName;

	// getters and setters omitted
}
```

在Hibernate中，Contact中的所有字段被称为值类型，值类型又分为三种: 基本类型，嵌入类型，集合类型。 在Contact类里除了name，都被映射到Contact表里，这些就是基本类型。name属性被称为嵌入类型。集合类型并没有在上面的例子中出现，但集合类型也是值类型中的一个独特类型。

#### 值类型

##### 基本类型

所谓基本类型，数据库类型到Java类型的映射，通常是单个数据库列，Hibernate内置了一些基本类型，我们简单介绍几个:

![VIUp3L.jpeg](https://i.imgloc.com/2023/06/22/VIUp3L.jpeg)

这些类型映射由Hibernate中的BasicTypeRegistry进行处理。

##### 嵌入类型

 在一个实体类里，如果某个字段的类型是由另一个实体类，那么这个实体类，我们就称之为嵌入类型。

##### 集合类型

Hibernate也允许持久性集合，持久化集合里面可以容纳任何其他Hibernate类型，如下所示:

```java
@Entity(name = "Person")
public static class Person {

	@Id
	private Long id;

	@ElementCollection
	private List<String> phones = new ArrayList<>();
    
	//Getters and setters are omitted for brevity
}
Person person = entityManager.find( Person.class, 1L );
//Throws java.lang.ClassCastException: org.hibernate.collection.internal.PersistentBag cannot be cast to java.util.ArrayList
ArrayList<String> phones = (ArrayList<String>) person.getPhones()
```

#### 实体类型简介

实体类型简单的说也就是Java中的类，由值类型组成，需要一个唯一标识符，Contact就是一个简单示例。要进行对关系型数据库进行CRUD，我们首先要有一个表, 但是在Hibernate里面，我们可以指定自动创建表，只用建立实体类即可:

```java
@Entity
@Table(name = "person")
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY )
    private Long id;

    private String name;

    private String nickName;

    private String address;
	
    // 这个字段用于将Java的日期类型映射到数据库的类型
    @Temporal(TemporalType.TIMESTAMP )
    private Date createdOn;

    @Version
    private int version;
    //Getters and setters are omitted for brevity
}

```

在mapping里指定这个实体类，即能够根据实体类实现建表:

```xml
 <mapping class="org.example.tutorial.hbm.entity.Person"></mapping>
```

## 使用Hibernate进行CRUD

有了这些关系，我们就开始CRUD吧。

### CREATE、DELETE 、 UPDATE

我们的第一个例子就是新增，Hibernate中还支持其他写法:

```java
int insertedEntities = session.createQuery(
	"insert into Partner (id, name) " +
	"select p.id, p.name " +
	"from Person p ")
.executeUpdate();
```

删除更新的也可以通过在createQuery方法的参数中指定SQL，来实现删除和更新:

```java
session.createQuery("update from Person set address = :address  where id = :id")
                .setParameter("id",2L).
                 setParameter("address","测试修改").executeUpdate();
session.createQuery("delete from Person where id = :id").setParameter("id",1L).executeUpdate();
```

这里注意在createQuery里写的SQL，from后面要跟实体名，Hibernate会自动寻找实体上table注解对应的表名，不能写表名。:参数被称为占位符用于替换参数，注意填入的参数类型要和实体中的那个属性对应的上，否则就会报参数类型转换错误。

### READ

查也可以通过createQuery中写SQL，但from后面要跟实体名，不能直接写表名，这在Hibernate中被称为HQL，下面是查询示例:

```java
List list =  session.createQuery(" select p  from Person p where id = :id").setParameter("id",2L).list();
List<Person> personList = session.createQuery(" select p  from Person p where address is null", Person.class).list();
```

比较有意思的是Hibernate还提供了逐行加载数据的查询，也就是滚动查询和Stream(这个Stream 非JDK 8的Stream)

## JPA

前面我们提到JPA是一组标准接口，Hibernate提供了实现，我们这里也简单介绍一下JPA，要使用Hibernate对JPA的实现，还是要读配置文件，只不过构造出来的对象不同: 

```java
public class JPATest extends TestCase {

    private EntityManagerFactory entityManagerFactory;

    @Override
    protected void setUp() throws Exception {
        entityManagerFactory = Persistence.createEntityManagerFactory( "org.hibernate.tutorial.jpa" );
    }
    // createQuery的语法与我们上文介绍的相同
    public void test(){
        entityManagerFactory.createEntityManager().createQuery("select p from Person p where id = :id").setParameter("id",2L);
    }

    @Override
    protected void tearDown() throws Exception {
        entityManagerFactory.close();
    }
}
```

## 总结一下

相对于MyBatis来说，Hibernate和JPA感觉有些庞大，这一点从两者的用户指导书就可以看出:

[![VIweJX.jpeg](https://i.imgloc.com/2023/06/23/VIweJX.jpeg)](https://imgloc.com/i/VIweJX)

[![VIwFrJ.jpeg](https://i.imgloc.com/2023/06/23/VIwFrJ.jpeg)](https://imgloc.com/i/VIwFrJ)

总结一下，我们介绍了Hibernate的基本使用，Hibernate和JDBC、JPA的关系。想起实习的时候，架构师吐槽MyBatis蠢的要命，架构师很喜欢Hibernate，但是MyBatis确实比较轻量级，很快就能上手。现在我还没体会到Hibernate的精妙之处，可能还是需要再学习一下吧。

## 参考资料

- Hibernate用户指导 https://docs.jboss.org/hibernate/orm/5.6/userguide/html_single/Hibernate_User_Guide.html#jpql-api