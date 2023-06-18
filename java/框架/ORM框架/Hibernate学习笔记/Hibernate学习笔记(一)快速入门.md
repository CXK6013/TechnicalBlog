# Hibernate学习笔记(一)快速入门

[TOC]

## 前言

毕业以来，我一直用MyBatis比较多，像另一种思路的ORM框架Hibernate，还一直没用过，也想起实习的架构师吐槽MyBatis蠢，今天就来换一种思路来学习一下Hibernate。学习本篇要求懂maven和jdbc。不懂maven参看下面的文章:

- Maven学习笔记 https://segmentfault.com/a/1190000019897882

   不懂JDBC，我还写JDBC相关的文章，哈哈，可以在B站上随手搜个视频看下。照以往文章的思路肯定是先讲下使用原生JDBC的痛点，然后引出ORM框架，但是本篇想换个思路，我们先用起来，在实践中体会Hibernate带给我们的便利，现在我们将Hibernate理解为一个ORM框架，给我们提供了一组API，可以让我们方便快捷的实现CRUD。

## 先用起来

目前我学习一个框架、中间件，一般都会习惯性的去官网，看看有没有详细的用户指导，Hibernate这一点做的相当优秀:

![Vb5ri8.jpeg](https://i.imgloc.com/2023/06/17/Vb5ri8.jpeg)



之前听到一种论调说Hibernate已经被淘汰了，看看官网Hibernate还活的好好的，我们本篇只看Hibernate ORM：

![Vb53bZ.jpeg](https://i.imgloc.com/2023/06/17/Vb53bZ.jpeg)



我本来想介绍6.2版本的，但是6.2最低是JDK 11，5.6最低是JDK 1.8，那这里就介绍两个版本，5.6和6.2。 我们先开始用，首先我们引入maven依赖:

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

我们已经用Hibernate完成了表的创建、新增数据、查询数据。 我们来分析一个流程。

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

## HQL





## JPA





## 总结一下