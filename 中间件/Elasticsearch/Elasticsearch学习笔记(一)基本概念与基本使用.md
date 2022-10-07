# ElasticSearch 学习笔记(一) 基本概念与基本使用

> 一般我介绍某个框架、MQ、中间件，一般都是讲是啥，能帮助我们干啥，然后用起来，高级特性。这次打算换一种风格，穿插一些小故事。写到这篇的时候，我想起我刚入行的第一个项目，有一个页面查询，主表两百七十万条数据，join了七张表，第二张从表一百多万数据，剩余五张大概在四五十万条数据的上下，查询大概十几秒，这样的速度肯定是难以满足要求的，请DBA优化，优化到七秒，用户勉强可以接受了，我当时问带我的大哥，如果数据量再往上涨，更慢怎么办，我当时的大哥说，可以考虑ElasticSearch，ElasticSearch号称亿级数据，光速查询。

[TOC]



## 缘起

> 许多年前，一个刚结婚的名叫 Shay Banon 的失业开发者，跟着他的妻子去了伦敦，他的妻子在那里学习厨师。 在寻找一个赚钱的工作的时候，为了给他的妻子做一个食谱搜索引擎，他开始使用 Lucene 的一个早期版本。
>
> 直接使用 Lucene 是很难的，因此 Shay 开始做一个抽象层，Java 开发者使用它可以很简单的给他们的程序添加搜索功能。 他发布了他的第一个开源项目 Compass。
>
> 后来 Shay 获得了一份工作，主要是高性能，分布式环境下的内存数据网格。这个对于高性能，实时，分布式搜索引擎的需求尤为突出， 他决定重写 Compass，把它变为一个独立的服务并取名 Elasticsearch。
>
> 第一个公开版本在2010年2月发布，从此以后，Elasticsearch 已经成为了 Github 上最活跃的项目之一，他拥有超过300名 contributors(目前736名 contributors )。 一家公司已经开始围绕 Elasticsearch 提供商业服务，并开发新的特性，但是，Elasticsearch 将永远开源并对所有人可用。
>
> 据说，Shay 的妻子还在等着她的食谱搜索引擎…  《Elasticsearch: 权威指南》基于2.x 版本

那Lucene是啥，Lucene是一个全文搜索引擎库，属于Apache，全称为Apache Lucene，Luece可以说是当下最先进、高性能全功能的搜索引擎库, 但是Lucene仅仅只是一个库，使用起来比较复杂。Shay Banon构建了一个抽象层，试图对开发者屏蔽复杂细节，Java开发者使用它可以很简单的给他们的程序添加搜索功能，这也就是Compass，最后演变为ElasticSearch。

![演进](http://tva1.sinaimg.cn/large/006e5UvNly1h40qidehv0j30g409o3z1.jpg)

全文搜索？ 传统的数据库不行吗？ 《ElasticSearch权威指南》给出的理由是:

> 不幸的是，大部分数据库在从你的数据中提取可用知识时出乎意料的低效。 当然，你可以通过时间戳或精确值进行过滤，但是它们能够全文检索、处理同义词、通过相关性给文档评分么？ 它们能从同样的数据中生成分析与聚合数据吗？最重要的是，它们能实时地做到上述操作，而不经过大型批处理的任务么？

从上面的这句话我们可以提取的有效信息是，大部分关系数据不能够进行全文检索、处理同义词、通过相关性给文档进行评分。

- 什么是全文检索(搜索)?

> Full Text Searching (or just *text search*) provides the capability to identify natural-language *documents* that satisfy a *query*, and optionally to sort them by relevance to the query. The most common type of search is to find all documents containing given *query terms* and return them in order of their *similarity* to the query. Notions of `query` and `similarity` are very flexible and depend on the specific application. The simplest search considers `query` as a set of words and `similarity` as the frequency of query words in the document. [1]
>
> 全文搜索(或文本搜索）提供识别满足查询自然语言的能力，并且根据查询结果进行相关性进行排序。全文搜索最为常见的是从所有文档中找出满足条件的所有文档，并根据相似度进行排序。 注意查询和相似是非常灵活和依赖于具体应用的。最为简单的全文搜索是将“查询”视为一组词，将相似性当作查询次的频率。

- 那什么是Document?

> A *document* is the unit of searching in a full text search system; for example, a magazine article or email message. The text search engine must be able to parse documents and store associations of lexemes (key words) with their parent document. Later, these associations are used to search for documents that contain query words.
>
> Document是全文搜索系统的基本单元，比如杂志上的一篇文章或者一封邮件均可以被视为文档。全文搜索引擎一定要能够解析文档，并存储文档的关键词和文档的关联。这些关联将会被用来搜索包含查询关键词的文档。

上面的描述是不是跟我们平时用的搜索引擎有点类似呢，我们在搜索引擎框输入关键词，搜索引擎搜索整个Web，搜索引擎根据相关性进行网页排名。比如我在百度搜索Zookeeper，下面是百度的搜索结果:

![百度的搜索结果](http://tvax2.sinaimg.cn/large/006e5UvNly1h41tp0ustmj311u0p1wj5.jpg)

我在bing进行搜索: 

![bing的搜索结果](http://tva1.sinaimg.cn/large/006e5UvNly1h41tv5jv4ij312i0q5gqo.jpg)

这些网页是搜索引擎根据相关性排出来的。百度还处理了同义词，Zookeeper有动物园管理员的意思，所以动物园管理员也被带了出来。再比如apple官网和苹果官网就是同义词，你在百度输入这两个关键词，第一个都是苹果公司的官网。上面提到的这些在传统的关系型数据库去实现都是难以做到的，但是Elasticsearch就能为我们提供。其实这里还涉及一个分词的问题，举一个例子，我在搜索引擎上搜索ES分词，对于搜索引擎来说，这是一个输入，他去匹配文档的时候，事实上是将ES分词，当成两个词：ES 分词。文档中包含这两个词来计算相关性。既然你能为我们做这么多，那就只能，omg，学它。

以上功能都很像是搜索引擎的功能，所以ElasticSearch 将自己定位为: 

> Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例，适用于包括文本、数字、地理空间、结构化和非结构化数据等在内的所有类型的数据.

ES 当前最新的版本是8.3, 我们不选最新的，我们基于7.17这个版本来进行介绍。Linux服务器基于Centos 7

## 基本概念

![ES概念](http://tva3.sinaimg.cn/large/006e5UvNly1h41uv2h0x2j30p90f7q3d.jpg)

既然是ElstaticSearch可以充当搜索引擎，那么它们的数据源是哪里呢，ElstaticSearch有内置的数据源, 在某种形式可以理解为另一种形式的数据库。

### Index

看到这个词，我下意识的想到了MySQL的索引，但是在ES里，索引有两个含义:

- 名词: 一个索引相当于关系型数据库中的一个表(在6.x以前，一个index可以被认为是一个数据库)
- 动词: 将一份document保存在一个index里，这个过程也可以称为索引。

既然一个index 相当于关系型数据库的一个表，那ElstaticSearch有对应的建表语句吗？有的那就是Mapping。

#### Mapping

Mapping定义了索引里的文档到底有哪些字段及这些字段的类型，类似于数据库中表结构的定义。Mapping有两种作用:

- 定义了索引中各个字段的名称和对应的类型。
- 定义了各个字段、倒排索引的相关设置，如使用什么分词器等。

需要注意的是，Mapping一旦定义后，已经定义的字段类型是不可更改的

### type

在6.x之前，index可以被理解为关系型数据库中的数据库，而type则被认为是数据库中的表。使用type允许我们在index里面存储多种类型的数据，数据筛选时，可以指定type。type的存在从某种程度上可以减少index的数量，但是type存在以下限制:

- **不同 type 里的字段需要保持一致**。例如，一个 `index` 下的不同 `type` 里有两个名字相同的字段，他们的类型（string, date 等等）和配置也必须相同。
- 只在某个 `type` 里存在的字段，在其他没有该字段的 type 中也会消耗资源。
- 得分是由 `index` 内的统计数据来决定的。也就是说，一个 type 中的文档会影响另一个 type 中的文档的得分。

以上限制要求我们，只有同一个 `index` 的中的 type 都有类似的映射 (mapping) 时，才勉强适用 `type` 。否则，使用多个 `type` 可能比使用多个 `index` 消耗的资源更多。

6.00 之后开始逐步废弃type, 开始不支持一个index里面存在多个type。

7.00版本后正式废除单个索引下多Type的支持。

### Document

> Elasticsearch is a distributed document store. Instead of storing information as rows of columnar data, Elasticsearch stores complex data structures that have been serialized as JSON documents
>
> ElasticSearch存储文档的方式是分布式存储，与存储列式数据不同，ES存储的是序列为JSON的文档。

上面已经讲了一些基本概念了，下面我们将ElasticSearch装起来，在实践中介绍其他核心概念。

## 装起来

### 安装ES

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.0-linux-x86_64.tar.gz
mkdir es
mv elasticsearch-7.17.0-linux-x86_64.tar.gz  es
# ES 7 以后自带JDK,你在解压过程中会发现ES自带的JDK有module这个概念,已经不是JDK8喽
# 已经是JDK 17了
tar xvf elasticsearch-7.17.0-linux-x86_64.tar.gz 
# 为后面搭建集群做准备
mv elasticsearch-7.17.0  es_node1
```

#### ES 目录大致介绍

|  目录   |                             描述                             |
| :-----: | :----------------------------------------------------------: |
|   bin   |  包含一些执行脚本，其中 ES 的启动文件和脚本安装文件就在这里  |
| config  | 包含集群的配置文件（elasticsearch.yml）、jvm配置（jvm.options）、user等相关配置 |
|   JDK   |                7.0 后自带 jdk，java 运行环境                 |
|   lib   |                         java 的类库                          |
| plugins |                        插件安装的目录                        |
| modules |                      包含所有 ES 的模块                      |

#### 修改配置

虽然ES号称开箱即用，但我们还是要改一些配置的，ES的配置文件主要在config目录下，我们主关注的是两个目录：

- elasticsearch.yml

> elasticsearch.yml 是用来配置 ES 服务的各种参数的

- jvm.options

>  jvm.options 主要保存 JVM 相关的配置。

```shell
echo -e '\n' >> config/elasticsearch.yml
# 向yml追加配置
echo 'cluster.name: my_app' >> config/elasticsearch.yml

echo 'node.name: my_node_1' >> config/elasticsearch.yml

echo 'path.data: ./data' >> config/elasticsearch.yml

echo 'path.logs: ./logs' >> config/elasticsearch.yml

echo 'http.port: 9211' >> config/elasticsearch.yml
# 允许任意ip 直连,真实环境不要设置为0.0.0.0
echo 'network.host: 0.0.0.0' >> config/elasticsearch.yml

echo 'discovery.seed_hosts: ["localhost"]' >> config/elasticsearch.yml

echo 'cluster.initial_master_nodes: ["my_node_1"]' >> config/elasticsearch.yml

echo -e '\n' >> config/jvm.options

# 设置堆内存最小值

echo '-Xms1g' >> config/jvm.options

# 设置堆内存最大值

echo '-Xmx1g' >> config/jvm.options

sudo su

echo -e '\nvm.max_map_count=262144' >> /etc/sysctl.conf

sysctl -p

exit;
```

#### 运行ES

```shell
# 注意，ES 不能使用 root 来运行！！！！ 这里我们新建个用户
adduser es_study
# 这里我将密码设置为my_studyes001
passwd  es_study 
# 将esnode1 这个文件夹授予给es_study
chown -R es_study es_node1/
# 该limits.conf  
# 追加 * soft nofie 65536  
# 追加 * hard nofile 65536
vim /etc/security/limis.conf 
# 然后重新登录，如果下面命令输出65536代表生效
ulimit -H -n
# 切换用户
su es_study
# 到bin目录下 
 ./elasticsearch
```

然后访问ES所在主机ip:9211, 出现下面:

```json
{
  "name" : "my_node_1",
  "cluster_name" : "my_app",
  "cluster_uuid" : "G6WNl_15TFmJ6VhDJIXfCw",
  "version" : {
    "number" : "7.17.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "bee86328705acaa9a6daede7140defd4d9ec56bd",
    "build_date" : "2022-01-28T08:36:04.875279988Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

说明安装成功, 因为我配置了环境变量, 所以用的还是JDK 8, 但是这次我想让ES 用JDK 17.

```shell
# 在/etc/profile 中加入ES_JAVA_HOME即可
vim /etc/profile
source /etc/profile
export ES_JAVA_HOME=/usr/es/es_node1/jdk
```

### 安装Kibana

**Kibana 是官方的数据分析和可视化平台，但现在我们只需要把它当作 ES 查询的调试工具即可。** Kibana 与 ES 的版本是有对应关系的，所以需要下载与 ES 同版本的 Kibana。

```shell
# 注意kibana 也不能在root用户下面运行,我们还是要用上面的用户来启动 授予权限
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.17.0-linux-x86_64.tar.gz
tar xvf kibana-7.17.0-linux-x86_64.tar.gz
mv kibana-7.17.0-linux-x86_64 kibana
cd kibana
# 需要注意的是，线上一定不能配置ip为 0.0.0.0，这是非常危险的行为！！！
echo -e '\nserver.host: "0.0.0.0"' >> config/kibana.yml
echo -e '\nelasticsearch.hosts: ["http://localhost:9211"]' >> config/kibana.yml
chown -R es_study kibana
./bin/kibana >> run.log 2>&1 &
```

然后访问ES所在主机ip:5601, 看到如下图所示:

![选择DevTools](http://tva4.sinaimg.cn/large/006e5UvNly1h41z2qpgnjj31gu0pin0t.jpg)

## 用起来

费了那么大劲，终于装好了, 现在我们在Kibana中用起来。上面我们提到ES是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。考虑有的同学还不知道RESTFul风格，这里简单介绍一下, HTTP有若干种请求方式，让请求方式和CRUD能对的上：

- POST 新增
- PUT 更新
- DELETE 删除
- GET 获取资源

#### 对索引进行管理-Mapping

- 创建索引

```json
PUT books
{
  "mappings": {
    "properties": {
      "book_id": {
        "type": "keyword" 
      },
      "name": {
        "type": "text"
      },
   	  "price":{
        "type":"scaled_float",
         "scaling_factor": 100
      },
      "author": { 
       "properties": {
          "first": { "type": "text" },
          "last":  { "type": "text" }
       }  
    }
  }
}
}   
```

上面定义了一个通过ElasticSearch的Mapping简单定义了一个索引,  type代表数据类型，ES常见的数据类型有:

- keyword

> keyword 类型更适合存储简短、结构化的字符串。例如产品ID、产品名字等

- text

> text类型的字段适合存储全文本数据，如短信内容、邮件内容等。

- 数字类型(long, integer, short, byte, double, float, half_float, scaled_float...)
  - 就整数类型而言(byte short integer long)而言，应当选择满足要求的最小类型
  - 对于浮点类型来说, 使用缩放因子将浮点数据存储到整数中通常更为有效，这也就是我们上面用到的scaled_float. 如果输入是23.45，ES会将输入的价格是23.456 乘以100, 再取一个接近原始值的数字，得出2346。使用比例因子的好处是整型比浮点型更易压缩，节省空间，节省磁盘空间。
- 对象类型

> 简单来说就是字段的值还是一个json对象。

我们登上Kibana的DevTools输入上面的Mapping命令:

![创建成功](http://tva4.sinaimg.cn/large/006e5UvNly1h48n9qrgbbj31hf0c075z.jpg)

- 删除索引

```http
DELETE books
```

####  对文档的增删改查

- create

  - INDEX API 创建

    这种方式指定了文档的API

  ```json
  PUT books/_doc/1 
  {
    "book_id": "4ee82462",
    "name": "母猪的产后护理(一)",
    "price":"23.56",  
    "author": {"first":"wang","last":"小明"}
  }
  ```

  如果在Kibana中连续运行，不会报错，也不会重复创建，版本号会发生改变:

  ![ES的版本号](http://tva1.sinaimg.cn/large/006e5UvNgy1h48nnix7ufj30oj06p74g.jpg)

  在ES中索引一个文档的时候，如果文档ID已经存在，会先删除旧文档，然后再写入新文档的内容，并且增加文档的版本号。

  - 创建文档（create doc）

    ```json
    POST books/_doc
    {
      "book_id": "4ee824621",
      "name": "母猪的产后护理(二)",
      "price":"23.56",  
      "author": {"first":"wang","last":"小明"}
    }
    ```

    使用post方式创建文档由ES生成ID，连续运行上面的命令，会发现版本号没发生改变，说明是连续新增。

- QUERY

  - 查询ID 为1的文档

  ```http
  GET books/_doc/1
  ```

   GET API 提供了多个参数, 更多的信息请参考官方文档，

  - 批量查询

  ```json
  # 参数中指定index
  GET /_mget
  {
    "docs": [
      { "_index": "books", "_id": "1" },
      { "_index": "books", "_id": "00IDB4IBeGTeMZoVN5XN" }
    ]
  }
  # 在URL中指定index
  GET /_mget
  {
    "docs": [
      { "_index": "books", "_id": "1" },
      { "_index": "books", "_id": "00IDB4IBeGTeMZoVN5XN" }
    ]
  }
  # 简写
  GET /books/_mget
  {
    "ids" : ["1", "00IDB4IBeGTeMZoVN5XN"]
  }
  ```

- update

```
POST books/_update/1
{
  "doc": {
    "name":"伟大的傅里叶变换"
  }
}
```

 上面的索引文档也能更新，那跟这种方式的更新有什么区别呢，索引文档的更新效果是先删除数据，然后再写入旧数据。如果我们只想更新一个字段，并只给了更新的字段，那么其他字段就会被抹去。

- delete

 在参数上指定id即可。

```http
DELETE books/_doc/1
```

那么如果我们想批量操纵索引中的文档呢？ElstaticSearch为我们提供了Bulk API 操纵批量处理文档。Bulk API中可以同时支持4种类型的操纵：

- Index
- Create
- Update
- Delete

语法示例:

```
POST _bulk
{ "index" : { "_index" : "books", "_id" : "3" } }  #指定操纵类型 哪个索引和文档
{ "book_id": "4ee82462","name":"天生吾战"}
{ "delete":{"_index": "books", "_id":"2"}}
{"update":{"_index":"books","_id":3}}
{"doc":{"name":"母猪的产后护理(四)"}}
```

## 历程

一般我还是先去ElasticSearch的官网去看文档，ES有中文官网，但是汉化了一部分，ES官网放的有《Elasticsearch: 权威指南》，可惜是基于2.x版本:

- https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html

高版本放的都有用户指南:

![选DOCS](http://tva1.sinaimg.cn/large/006e5UvNly1h48pdfbwmxj31ab0kygok.jpg)

![用户指导](http://tvax2.sinaimg.cn/large/006e5UvNly1h48pfr18qoj314w0lkq56.jpg)

本文的安装教程基本上应用了掘金小册《Elasticsearch 从入门到实践》

## 参考文档

- **全文搜索**  https://docs.microsoft.com/zh-cn/sql/relational-databases/search/full-text-search?view=sql-server-ver16
- Elasticsearch(015)：es常见的字段映射类型之数字类型(numeric)   https://blog.csdn.net/weixin_39723544/article/details/104331885
- ElasticSearch GET 文档  https://www.elastic.co/guide/en/elasticsearch/reference/7.13/docs-get.html
- 《Elasticsearch 从入门到实践》 掘金小册 https://juejin.cn/book/7054754754529853475/section/7064921135250407437
- Elasticsearch 基础入门详文  公众号 腾讯技术工程
