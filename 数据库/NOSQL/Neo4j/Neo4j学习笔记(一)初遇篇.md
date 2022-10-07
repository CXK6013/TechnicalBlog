# Neo4j 学习笔记(一) 初遇篇

> 本周让我们继续NoSQL家族的访谈之旅，本次我们的采访对象Neo4J先生。

## 为什么要有Neo4J？

各种数据库事实上都是在描述现实世界上中数据之间的联系，现实世界中一个比较典型的场景就是关系数据库力有不逮的地方，那就是社交网络，这种重联系的数据。如下图所示:

![图示意](https://tvax4.sinaimg.cn/large/006e5UvNly1h0f8tr1yzmj30wt0ikgnv.jpg)

A和B、C、D、E都是好朋友，这在关系型数据库是典型的多对多关系, 对于多对多的关系，一般关系型的策略是建立一张中间表来描述这种多对多的关系。但是在关系型数据库中描述联系更常见的场景是描述不同数据的建模关系，比如学生和课程表的关系，选课表就存储学生ID，课程表ID等其他必要信息，学生和课程描述的对象是不同的，上面的图描述的数据是相同的，可能都是社交圈的人。我们该如何存储这种联系，一张自联系的表？ 存储跟他联系的信息？ 像下面这样: 

![联系图](https://tvax4.sinaimg.cn/large/006e5UvNly1h0fazs9hr8j30vn0mi0vk.jpg)

那如果我现在的需求是查询A的朋友的朋友呢，那一个简单而无脑的操作就是我想看我朋友的朋友，这只是一层， 对应的SQL其实很好写:

```mysql
# 请原谅我还是从Student还是
 SELECT * FROM Student WHERE id in ( 
 SELECT studentReleationId  FROM student_releation WHERE studentId in     
 (SELECT studentReleationId FROM student_releation WHERE studentId  = '1')
# 转成join的话,先找出我朋友的朋友的Id,然后再做join，或者子查询
SELECT s2.studentReleationId  FROM student_releation s1 INNER JOIN  student_releation s2 on s1.studentReleationId = s2.studentId
WHERE s1.studentId = '1'
```

但其实并不推荐join，假设一个人的社交好友有100个，有的其实会更高，那我存储个人信息如果只有三万个人的话，那么存储联系的表就是三百万条数据，这个其实还不算交叉认识的。对于关系型数据库来说大表的join是一场灾难，但对于社交软件来说，三万个用户来说是相当少的数据量，对于成熟的社交产品来说，三百万，三千万都有可能。再比如电影关系图，查看某个导演的电影参演的演员，对于关系型数据库来说这个也不是不能做，只是数量上去之后，我们查询速度是不断在下降的，因为对于表与表之间的关系基本靠join，join的越多速度越慢。由此我们就引出了图数据库的应用场景，多对多建模、存取数据。图数据库对关联查询一般都进行针对性优化，比如存储模型上、数据结构、查询算法等，防止局部数据的查询引发全部数据的读取。

关系型数据库有不同的实现，MySQL，Oracle, SQL Server 一样，图数据库也有不同的实现: Neo4j，JanusGraph，HugeGraph。关系型数据库的查询语言是SQL，图数据库的查询语言是: Gremlin、Cypher.本次我们介绍的就是Neo4j。

## 用起来

我首先还是来到Neo4j官网希望有免费试用的，因为我不想自己装。然后我看到了免费试用:

![免费试用](https://tvax4.sinaimg.cn/large/006e5UvNly1h0fbuxktu8j31fc0m8n7t.jpg)

然后我就不装了，直接开始用。我们首先还是先介绍基本的语法:

- 创建

  - 创建结点

    ```cypher
    CREATE (Person { name: "Emil", from: "Sweden", klout: 99 }) 
    ```

    > Person是这个结点的标签,{}里面是这个标签的属性。

  - 创建联系

    ```cypher
    MATCH (ee:Person) WHERE ee.name = "Emil"  #  先查询一次是为了后面建立联系
    CREATE (js:Person { name: "Johan", from: "Sweden", learn: "surfing" }), # 创建一个结点,并将js指向该结点
    (ir:Person { name: "Ian", from: "England", title: "author" }),# 创建一个结点,并将ir指向 这个结点
    (rvb:Person { name: "Rik", from: "Belgium", pet: "Orval" }),
    (ally:Person { name: "Allison", from: "California", hobby: "surfing" }),
    (ee)-[:KNOWS {since: 2001}]->(js),(ee)-[:KNOWS {rating: 5}]->(ir),
    # (变量)-[关系描述]->(变量)  这两个结点建立联系 [中是对这个边的关系描述] KNOWS可以理解为了解,KNOWS{SINCE:2001}表示ee和js认识是在2001年
    (js)-[:KNOWS]->(ir),(js)-[:KNOWS]->(rvb),
    (ir)-[:KNOWS]->(js),(ir)-[:KNOWS]->(ally),
    (rvb)-[:KNOWS]->(ally)
    ```

- 删

  - 移除结点的属性

    ```cypher
    MATCH (a:Person {name:'Emil'}) REMOVE a.name
    ```

  - 删结点

    ```cypher
    MATCH (a:Person {name:'Ian'}) DELETE a
    ```

- 改

```cypher
MATCH(ee:Person) where name = 'Johan'
SET ee.name = '张三' RETURN ee; 
# 将name = Johan 的name改为张三
```

- 查

  - 查找单个结点

  ```cypher
  MATCH (ee:Person) WHERE ee.name = "Johan" RETURN ee; # 查出结点中name叫Johan的,然后返回 
  ```

  - 查找所有具有Knows关系的结点

  ```cypher
  MATCH (n)-[:KNOWS]-() RETURN n
  ```

  - 查找某人朋友的朋友, 这里我们将KNOWS看做朋友

  ```cypher
  MATCH (a:Person {name:'Ian'})-[r1:KNOWS]-()-[r2:KNOWS]-(friend_of_a_friend) RETURN friend_of_a_friend.name AS fofName
  # friend_of_a_friend 是返回的变量
  ```

## Java操纵Neo4J

Java操纵任何数据库都需要驱动, Neo4j的免费试用版有对应的语言示例.

![各种示例](https://tvax3.sinaimg.cn/large/006e5UvNly1h0ffgfrga4j316v0nc12k.jpg)



Neo4J 的驱动跟JDBC类似，原生都是我们拼SQL，然后提交。依赖:

```xml
 <dependency>
      <groupId>org.neo4j.driver</groupId>
      <artifactId>neo4j-java-driver</artifactId>
      <version>4.1.0</version>
 </dependency>
```

示例：

```java
public class DriverIntroduction implements AutoCloseable{

    private final Driver driver;

    public DriverIntroduction(String uri, String user, String password, Config config) {
        // The driver is a long living object and should be opened during the start of your application
        driver = GraphDatabase.driver(uri, AuthTokens.basic(user, password), config);
    }
    @Override
    public void close() throws Exception {
        driver.close();
    }
    public void createFriendship(final String person1Name, final String person2Name) {
        // 创建结点 然后建立联系,并返回结点
        String createFriendshipQuery = "CREATE (p1:Person { name: $person1_name })\n" +
                "CREATE (p2:Person { name: $person2_name })\n" +
                "CREATE (p1)-[:KNOWS]->(p2)\n" +
                "RETURN p1, p2";

        Map<String, Object> params = new HashMap<>();
        params.put("person1_name", person1Name);
        params.put("person2_name", person2Name);
        try (Session session = driver.session()) {
            Record record = session.writeTransaction(tx -> {
                Result result = tx.run(createFriendshipQuery, params);
                return result.single();
            });
        } catch (Neo4jException ex) {
            throw ex;
        }
    }
}
```

## 总结一下

本文简单介绍了一下Neo4J的由来，以及基本的增删改查，Java操纵数据库。Neo4j默认只有一个数据库，目前社区版似乎不支持语法创建数据库，需要修改配置文件。Neo4J也没有关系型数据库对应表的概念。数据库下面直接就是节点数据。Neo4j也有中文文档:

http://neo4j.com.cn/public/docs/index.html

## 参考资料

- 类似 Neo4j 这样的图数据库在国内会兴起么？为什么？  https://www.zhihu.com/question/19999933/answer/550019788
- 手把手教你快速入门知识图谱 - Neo4J教程  https://zhuanlan.zhihu.com/p/88745411?utm_source=wechat_session



