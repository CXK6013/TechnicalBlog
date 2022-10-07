- MongDB学习笔记(一) 初遇篇

  > 本周去见了高中同学, 还是蛮开心的。本来打算的是写MySQL数据库事务的实现或者MySQL优化系列的文章，但是还没有想到如何组装这些知识。
  >
  > 索性这周换个方向，写写NOSQL。考虑到有些同学对JSON还不是很了解，这里我们先介绍MongDB的基本概念，再接着讲述为什么要引入MongDB，再讲解该如何使用。

  ##  是什么? what

  >  MongoDB is a document database designed for ease of application development and scaling  《MongDB官网》
  >
  >  MongDB是一个为高扩展性、高可用性而设计的文档数据库。

  注意到MongDB是一个文档数据库，文档数据库与我们常见的关系型数据库有什么区别呢？我们首先来看这个文档，也就是document究竟是什么含义?

  > MongoDB中的记录是一个文档，它是由字段和值对组成的数据结构。MongoDB文档类似于JSON对象。字段的值可以包括其他文档，数组和文档数组。 

  ![MongDB Document](https://tva3.sinaimg.cn/large/006e5UvNly1h006bjyij7j30mv05tdh8.jpg)

   关于什么是JSON?  参看我写的文章: 傻傻弄不清楚的JSON?

  所以MongDB的记录是一个类似于一个JSON对象的文档，MongDB将文档存储在集合中，集合类似于关系数据库中的表。存取关系类似于下面的图:

  ![MongDB VS 关系型数据库](https://tva4.sinaimg.cn/large/006e5UvNly1gzzzkit659j30yt0jwgqr.jpg)

  

  上面我们提到MongDB的记录是一个类似于JSON的文档，之所以是类似于，是因为MongDB采取的是BSON，BSON = Binary JSON， 也就是二进制形式的JSON，但同时又扩充了JSON，具备JSON中不具备的数据类型，像日期类型(Date type)、二进制数据类型(BinData type)。

  ## 为什么要引入MongDB？

  那为什么要引入MongDB呢?  或者说MongDB相对于关系型数据库有哪些优势呢？ 

  > 十年前，当 Dwight 和我开始这个后来成为 MongoDB 的项目的时候，我们绝对没有想到它今天的样子。我们只有一个信念：让开发者更高效。MongoDB 诞生于在庞大复杂的业务部署中使用关系型数据库给我们带来的沮丧。我们着手建造一个我们自己想用的数据库。这样，每当开发者想写一个应用时，他们可以专注于应用本身，而不用围着数据库转。  MongoDB 的十年，一个创始人的回顾

  所以就是在某些领域，MongDB可以让开发者更高效。

  知乎上有一个问题: MongoDB 等 NoSQL 与关系型数据库相比，有什么优缺点及适用场景？下面的有一个MongDB的开发者做了回答，这里我简单摘录一下他的回答：

  - 文档(JSON)模型与面向对象的数据表达方式更相似更自然。与关系型数据库中的表结构不同，文档(JSON)中可以嵌入数组和子文档(JSON)，就像程序中的数组和成员变量一样。这是关系型数据库三范式不允许。但实际中我们经常看到一对多和一对一的的数据。比如，一篇博客文章的Tag列表作为文章的一部分非常直观，而把Tag与文章的从属关系单独放一张表里就不那么自然。SQL语音可以很精确地形式化。然而三大范式强调的数据没有任何冗余，并不是今天程序员们最关心的问题，他们用着方便不方便才是更重要的问题。

  > 这里我们就得出MongDB相对于关系型数据库的第一个优势，在某些场景下MongDB比关系型数据库方便，注意是在某些场景下。

  - 性能

    三大范式带来的jojn，有的时候为了满足一个查询，不得不join多表，但join多表的代价是随着数据的提升，查询速度会变得很慢，更简单的访问模式可以让开发者更容易理解数据库的性能表现，举一个典型的场景就是，我join了十张表，在这样的情况下，我该如何加索引才能获得最优对的查询速度。

  - 灵活

     如果需要加字段，从数据库到应用层可能都需要改一遍，尽管有些工具可以把它自动化，但这仍然是一个复杂的工作。 MongDB没有Schema，就不需要改动数据库，只需要在应用层做必要的改动。(这句话可以这么理解，仔细看上面画的MongDB存取数据的图，集合中现在存取的学生都有三个字段，事实上MongDB允许文档拥有不同的字段)

  自定义属性也是一个问题，比如LCD显示器和LED显示的产品特性就不一样，数据的这种特性被称为多态，放在关系型数据库中，产品这个表应该怎么设计？ 一般来说最简单的方案是将每个可能的属性都变成单独一列，但如果显示器厂商每年推出一个新特性，那么这个表就要经常的改动，扩展性不强。还有一种更加极端的场景是，一个通讯录允许用户随意添加新的联系方式，一种方案是将这种自定义联系方式放在一列，用json存储。 

  MongDB的灵活还体现在非结构化和半结构化的数据上, MongDB提供全文索引，也支持地理位置查询和索引。例如一位用户想知道方圆五公里哪里有公共卫生间，这是「地理范围查询」。然后他搜索最近的单车。摩拜单车正是使用 MongoDB 完成这样的「距离排序查询」。强烈建议使用地理位置索引加速查询，因为我能理解这位用户的心情。

  - 可扩展性

    MongDB原生自带分片，对应的就是关系型数据库的分库分表。

  - 哪些场景下MongDB不适合

  > MongDB的开发者周思远举了一个场景，他在一个项目中需要地铁列车时刻表，官方提供的是一个满足行业标准的SQL格式的数据，也就是按照三大范式分成了好几张表。然后他花了一晚上把数据库导入到了MongDB中。但是他说如果再来一次，他可能会直接选取关系型数据库，因为如果源数据格式就是SQL数据，没法控制，数据量小；交叉引用关系丰富，查询模式丰富，应用又不需要高性能，那么关系型数据页也是一个务实的选择。

  我对这句话的理解是站在建模应用数据的角度不同，在某些场景下，关系型数据库提供的SQL在统计数据、查询、分析方面仍然具备强大的优势。

  ## 怎么用？

   有了，what和why之后，我们就可以开始用起来了，也就是开始安装使用，比较幸运的是MongDB有个国人维护的中文操作手册: 

   ![MongDB操作手册](https://tvax2.sinaimg.cn/large/006e5UvNly1h001f378hij31660nxalf.jpg)

  介绍的相当详细，也有安装教程，也有免费试用:

  ![免费试用](https://tva2.sinaimg.cn/large/006e5UvNly1h001gtluogj31cw0ni7h8.jpg)

  所以对于我这种我选择了面试试用，点进去，我选择的开发场景是开发微服务，你也可以选择在本地安装，这篇手册上的安装教程写的还是蛮详细的，这里我就放个地址，不做详细介绍了。https://docs.mongoing.com/install-mongodb

  但是这种在云端的数据库，安装MongDB的shell可能稍微比较麻烦一点，我在windows上安装了好久感觉都没成功，这里可以用MongDB的GUI，下载地址如下:

  https://downloads.mongodb.com/compass/mongodb-compass-1.30.1-win32-x64.exe

  ![MongDB GUI](https://tva1.sinaimg.cn/large/006e5UvNly1h003my5m86j30yx0mztec.jpg)

  ![增删改查](https://tva1.sinaimg.cn/large/006e5UvNly1h003r7260qj30z30n3wlv.jpg)

  会填JSON就行

  ## Java 操纵MongDB

  现在我们就尝试用起来，我在初遇篇讲的就是一些基本特性，CRUD之类的。  首先连接数据库肯定需要驱动:

  ```xml
  <dependency>
      <groupId>org.mongodb</groupId>
      <artifactId>mongo-java-driver</artifactId>
      <version>3.12.5</version>
  </dependency>
  ```

  增删改查，跟数据库打交道的基本步骤一般都是获取连接，然后发送语句，在MongDB的java驱动中，增删改查都和Document、Bson这两个类有关系:

  ![Document的UML图](https://tva1.sinaimg.cn/large/006e5UvNly1h0048wil4qj30k306kgm5.jpg)

  ![Document的方法结构图](https://tvax4.sinaimg.cn/large/006e5UvNly1h004a8ges3j30fq0ja11y.jpg)

   我们新增对象的时候就是new Document， 像里面丢值。Filter则用来构造条件对象，用于查询:

  ```java
  Filters.ne 是不等于
  Filters.gt 大于
  Filters.and(Filters.eq("age",2222)),Filters.eq("name","aaa") 连等于
  Filters.gte 大于等于
  Filters.lt 小于
  Filters.lte 小于等于
  Filters.in()
  Filters.nin() not in
  ```

  Filters 本质上来说就是来帮你传递了操作符，我们借助于Document也能实现也能实现一样的效果:

  ```java
   new Document("name","张三") = Filters.eq("name","张三")
   new Document("age",new Document("$eq",24))  = Filters.eq("name","张三")
   new Document("age",new Document("$ne",24))  = Filters.ne("name","张三")    
  ```

  我们主要借助于MongoCollection来完成对MongDB中的增删查改，增主要和Document有关系。改 删 查可以使用Filters和Document来实现。

  - 增

  ```java
  @Test
  void contextLoads() throws Exception {
      ConnectionString connectionString = new ConnectionString("mongodb+srv://study:a872455@cluster0.exlk2.mongodb.net/myFirstDatabase?retryWrites=true&w=majority");
      MongoClientSettings settings = MongoClientSettings.builder()
              .applyConnectionString(connectionString)
              .build();
      MongoClient mongoClient = MongoClients.create(settings);
      insertDocumentDemo(mongoClient);
  }
  
  public void insertDocumentDemo(MongoClient mongoClient) throws  Exception {
      MongoDatabase database = mongoClient.getDatabase("test");
      database.getCollection("study");
      MongoCollection<Document> study = database.getCollection("study");
      // study 这张表的存的有 number name id, 如果是关系型数据库,我们要添加属性,需要先在表中加字段,
      // 现在 在MongDB中我们直接添加
      Map<String,Object> map = new HashMap<>();
      map.put("number","bbb");
      map.put("name","aaa");
      map.put("id","1111");
      map.put("age",2222);
      Document document = new Document(map);
      study.insertOne(document);
  }
  ```

  结果: ![新增结果](https://tvax4.sinaimg.cn/large/006e5UvNly1h0045mj9zyj30sf0ie428.jpg)

  - 删

  ```java
  public void deleteDocument(MongoClient mongoClient){
          MongoDatabase database = mongoClient.getDatabase("test");
          database.getCollection("study");
          MongoCollection<Document> study = database.getCollection("study");
          // eq 是等于运算符,Filters.eq("age",2222) 表示查出定位的是age = 2222的对象
          study.deleteOne(Filters.eq("age",2222));
  }
  ```

  - 改

    ```java
    public void updateDocument(MongoClient mongoClient){
            MongoDatabase database = mongoClient.getDatabase("test");
            database.getCollection("study");
            MongoCollection<Document> study = database.getCollection("study");
            study.updateOne(Filters.eq("age",2222),new Document("$set",new Document("name","22222")));
     }
    ```

  - 查

  ```java
  public void selectDocument(MongoClient mongoClient){
          MongoDatabase dataBase = mongoClient.getDatabase("test");
          MongoCollection<Document> studyTable = dataBase.getCollection("study");
          // age 可以是一个数组, 所有的值都得是2222
          Bson condition = Filters.all("age", 2222);
          FindIterable<Document> documentList = studyTable.find(condition);
  }
  ```

  ## 最后总结一下

  ![程序操作数据](https://tvax3.sinaimg.cn/large/006e5UvNly1h0060ece5fj30w10f4goi.jpg)

  某种程度上我们的程度都是从现实数据采取数据经过处理之后送入数据库，然后根据用户请求再将数据库中的数据取出，但是现实世界的数据结构是形形色色的，关系型数据库也有力不能逮的场景，像是我们在上文讨论过的为为什么引入MongDB，还有一种场景就是描述数据之间的联系，比如组织架构，人际关系图，在数据量比较大的时候，关系数据库就面临着查询速度缓慢的问题，为了描述这种非线性的关系，图数据库应运而生。不同的数据库都是在应对不同的数据描述场景。

  ## 参考资料

  -  MongoDB中文手册|官方文档中文版   https://docs.mongoing.com/ 

  - MongoDB 等 NoSQL 与关系型数据库相比，有什么优缺点及适用场景？ https://www.zhihu.com/question/20059632
  - BSON 官网  https://bsonspec.org/#/
  - java操作mongodb——查询数据   https://www.cnblogs.com/simple-ly/p/5796440.html
