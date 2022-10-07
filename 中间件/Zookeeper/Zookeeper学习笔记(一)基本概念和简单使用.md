# Zookeeper学习笔记(一)基本概念和简单使用

>闲扯两句, 一般学习一门新技术我现在会直接看文档，文档上的一些核心观点放到文章中，原因在于我希望记录我的思考过程，同时也是锻炼自己阅读英文文档的能力。

[TOC]

## 概述

首先我们打开bing搜索引擎，搜索Zookeeper，有同学可能会问，你为什么让打开bing搜素引擎，而不是百度呢。那是因为目前在百度搜索Zookeeper，第一页没找到官网：

![百度的搜索结果](http://tva2.sinaimg.cn/large/006e5UvNly1h3dluxw67bj30vx0o67ju.jpg)



![百度第二页](http://tvax4.sinaimg.cn/large/006e5UvNly1h3dlvjo1a1j30ts0ktwqs.jpg)

但是你打开bing搜索Zookeeper:

![bing的搜索结果](http://tva3.sinaimg.cn/large/006e5UvNly1h3dlwzag9jj31260od4d2.jpg)

个人感觉百度的搜索质量有变差的迹象，所以我最近用bing比较多。

![Zookeeper文档](http://tva4.sinaimg.cn/large/006e5UvNly1h3dlz9s8gyj31fv0l4tsr.jpg)

![bird](http://tvax1.sinaimg.cn/large/006e5UvNly1h3dm0a8z3ij31b30knay4.jpg)

> ZooKeeper is a high-performance coordination service for distributed applications. It exposes common services - such as naming, configuration management, synchronization, and group services - in a simple interface so you don't have to write them from scratch. 
>
> Zookeeper为分布式应用提供高性能协调服务，用简单的接口提供了许多服务，有域名服务、配置管理、分布式同步、组服务。

解读1:  域名服务、配置管理、分布式同步、组服务这四个似懂非懂. 

解读2  为分布式应用提供了高性能的协调服务，高性能我们是喜欢的，协调了啥？ 

### 再聊聊分布式吧

我们再来聊聊服务端系统架构的演进吧，最初的时候我们的服务只在一台服务器上运行，慢慢的随着用户量的不断提升，单机部署已经无法再满足访问量了。于是人们自然就想到了集群，即相同的应用再部署一遍，由Nginx或其他中间件根据负载均衡算法将请求分摊到集群的机器上。但是为了追求高可靠，我们不能将一个做单机集群，将应用在配置文件中改端口实现集群，但是这并不可靠，假如这台计算机出现了什么问题，整个服务都会变得不可用，为了不让鸡蛋放在一个篮子里，维持服务的高可靠，我们将服务在多台计算机上，这样就算一台计算机上的服务出现了问题，服务还是可用的(现在我们的服务还是一个单体应用)，这也就是分布式部署，所以分布式并不一定要和微服务挂钩。

![单机部署到集群部署](http://tvax2.sinaimg.cn/large/006e5UvNly1h3jmw5jb17j30x50hbta3.jpg)

但是这又引入了新的问题:

- 一个节点挂掉不能提供服务时如何被集群知晓并由其他节点接替任务

> 例子: 当数据量与访问量不断上升，单机的MySQL无法再支撑系统的访问量，我们开始搭建集群，提升数据库的访问能力，为了增加可靠性，我们多机部署，甚至多地部署。

![数据库集群](http://tva3.sinaimg.cn/large/006e5UvNgy1h3k8iotrvfj30zx0k6acs.jpg)

一般来说增删改消耗的性能远小于查询的性能，所以我们选若干台数据库节点做写入，对于用户的新增数据请求，会分摊到写节点，写节点写入完成要将这个数据扩散到其他节点, 但这里有一个问题就是如果写节点挂掉呢，那一个自然而然的操作是从从库中再选一个读库回应请求，同时将挂掉的结点从集群中剔除.

- 在分布式的场景下如何保证任务只被执行一次。

> 例子: 分布式下的定时任务，在计算机A和B上都部署了服务，如何保证定时任务只执行一次。

 这也就是Zookeeper的协调。

### 基本概念与设计目标

> 在设计目标里面能看到核心概念。

- ZooKeeper is simple. (Zookeeper 是简单的)

> ZooKeeper allows distributed processes to coordinate with each other through a shared hierarchical namespace which is organized similarly to a standard file system. The namespace consists of data registers - called znodes, in ZooKeeper parlance - and these are similar to files and directories. Unlike a typical file system, which is designed for storage, ZooKeeper data is kept in-memory, which means ZooKeeper can achieve high throughput and low latency numbers.
>
> Zookeeper通过类似于文件系统的命名空间来实现对分布式进程的协调，命名空间是由一个一个数据寄存器组成，在Zookeeper中它们被称为znode, ZNode与文件系统的文件夹是相似的，但是Zookeeper选择将数据保存在内存中，这意味着Zookeeper可以实现高吞吐和低延迟。

 ![Zookeeper的命名空间](http://tvax2.sinaimg.cn/large/006e5UvNly1h3k9hhp714j30qb0g540p.jpg)

这就是Zookeeper的命名空间，像不像Linux的文件系统, 一个典型的树结构， 其实你也可以类比到windows的文件系统，/是根目录，这是硬盘，下面是文件夹。像一个文件夹有多个子文件夹一样，一个znode也拥有多个结点，以key/value形式存储数据。Znode有两种，分为临时节点和永久节点，节点的类型在创建时被确定，并且不能改变。 临时节点的生命周期依赖于创建它们的会话。一旦会话结束，临时节点将会被自动删除，当然也可以手动删除，临时节点不允许拥有子节点。永久节点的生命周期不依赖于会话，并且只有在客户端显式执行删除操作的时候，才能被删除。Znode还有一个序列化的特性，如果创建的时候指定的话，该Znode的名字后面会自动追加一个递增的序列号。序列号对于此节点的父节点来说是唯一的，这样便会记录每个子节点的创建的先后顺序。

Znode节点的特性:

1. 兼具文件和目录特点 既像文件一样维护着数据、信息、时间戳等数据，又像目录一样可以作为路径标识的一部分，并可以具备子Znode。用户对Znode具有增、删、改、查等操作

2. Znode具有原子性操作  读操作将获取与节点相关的所有数据，写操作也将替换节点的所有数据
3.  Znode 存储数据大小有限制 每个Znode的数据大小至多1M，但是常规使用中应该远小于此值。
4. Znode 通过路径引用，路径必须是绝对的。

​	

- ZooKeeper is replicated

  > Like the distributed processes it coordinates, ZooKeeper itself is intended to be replicated over a set of hosts called an ensemble.
  >
  > 
  >
  > The servers that make up the ZooKeeper service must all know about each other. They maintain an in-memory image of state, along with a transaction logs and snapshots in a persistent store. As long as a majority of the servers are available, the ZooKeeper service will be available.
  > Clients connect to a single ZooKeeper server. The client maintains a TCP connection through which it sends requests, gets responses, gets watch events, and sends heart beats. If the TCP connection to the server breaks, the client will connect to a different server.
  >
  > 如同被其协调的分布式应用一样，Zookeeper本身也维持了一致性，集群中的Zookeeper同步内存状态、以及持久化的日志和快照，只要大部分的服务器是可用的，那么对应的Zookeeper就是可用的。
  >
  > 客户端连接到单台Zookeeper，通过该连接发送请求、获取响应、获取监听事件并发送心跳，如果客户端的连接断开，客户端将会连接到其他的Zookeeper上。

![zkservice](http://tva3.sinaimg.cn/large/006e5UvNly1h3ka09ga5bj30go055wfe.jpg)



- Conditional updates and watches

  > ZooKeeper supports the concept of *watches*. Clients can set a watch on a znode. A watch will be triggered and removed when the znode changes. When a watch is triggered, the client receives a packet saying that the znode has changed.
  >
  > Zookeeper 支持监听的概念，客户端可以监听Znode，当节点被移除或者改变的时候，会通知监听的客户端，当节点发生改变的时候将收到消息。

### 小小总结一下

Zookeeper借助以上特性来实现上面我们提到的功能特性:

- 域名服务  将ip映射为服务名，如果我们的服务集群中需要互相调用，那么我们可以选择将ip和域名存储到Zookeeper的节点中，在需要调用的时候去用域名来换取到对应的ip地址
- 配置管理  动态刷新配置, 基于监听机制，我们将配置文件存储在Znode中，应用监听对应的Znode，Znode改变会将改变推送给对应的应用。也就是动态刷新配置
- 数据的发布与订阅 同样是基于监听机制
- 分布式锁   不同主机上的进程竞争统一资源，可以借助Zookeeper做分布式锁，举一个例子在服务A身上配置的有定时任务，我们集群部署为了保证定时任务A只在一台上跑，我们可以借助分布式锁来完成这个任务。

 为了让我们让我们的服务实现更强的吞吐能力和高可用，我们选择了分布式部署，但是在计算机的世界里通常是通过某种技术手段解决一个问题，就会引入新的问题，分布式部署的过程中，我们又遇到了新的问题，比如节点之间的协调(主从集群中选中Leader)，资源的竞争问题，为了解决这些问题Zookeeper应运而生。

  为什么会将Zookeeper的官方文档拎出来呢，因为希望将自己的学习过程也记录下来，我记得刚学Java Web的时候会去B站上找视频，但是我看视频的时候有的时候会想，视频作者是怎么得出这个结论的，他们是怎么得出Zookeeper可以这么用的，因为我想直接获取第一手的资料，有自己的思考过程。

## 先装起来

说了这么多，我们先将zookeeper用起来再说。

![Zookeeper下载](http://tvax1.sinaimg.cn/large/006e5UvNly1h3kg1g9kquj31eo0nlh4d.jpg)

本次我们通过在Linux下进行安装部署, 国内进入Zookeeper官网下载比较慢，我们通过镜像进行下载:

```shell
# 首先在cd 到usr下建zookeeper目录，然后在这个目录下建zk1、zk2、zk3.我们本次做集群部署
# zk1 下面执行下面命令 
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz --no-check-certificate 
# 解压
tar -xzvf apache-zookeeper-3.7.1-bin.tar.gz
# 然后创建data logs  目录
mkdir data logs
# 将zk1 下面的所有文件复制到 zk2 zk3 下面一份
cp -r /usr/zookeeper/zk1/*  /usr/zookeeper/zk2/
cp -r /usr/zookeeper/zk1/*  /usr/zookeeper/zk3/
# zk1/data 下面建立myud 文件，此文件记录节点id,每个zookeeper节点都需要一个myid文件来记录节点在集群中的id,此文件只能由一个数字。
echo "1" >> /usr/zookeeper/zk1/data/myid
echo "2" >> /usr/zookeeper/zk2/data/myid
echo "3" >> /usr/zookeeper/zk3/data/myid
# 然后进入 apache-zookeeper-3.7.1-bin的conf文件夹下面，将配置文件zoo_sample.cfg重名为zoo.cfg。对该文件进行如下配置
mv zoo_sample.cfg  zoo.cfg
# 加入以下配置 dataDir 存储数据  dataLogDir 存储日志  clientPort 监听端口
dataDir=/usr/zookeeper/zk1/data 
dataLogDir=/usr/ZooKeeper/zk1/logs
clientPort=2181
server.1=127.0.0.1:8881:7771
server.2=127.0.0.1:8882:7772
server.3=127.0.0.1:8883:7773
#集群配置中模版为 server.id=host:port:port，id 是上面 myid 文件中配置的 id；ip 是节点的 ip，第一个 port 是节点之间通信的端口，第二个 port 用于选举 leader 节点
# 第一个编辑完,我们用复制指令将这个配置文件复制到zk2和zk3中。注意要改clientPort dataDir dataLogDir
 /usr/zookeeper/zk1/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
 /usr/zookeeper/zk2/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
 /usr/zookeeper/zk3apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
 # 正常启动会输出 Starting zookeeper ... STARTED 如果不放心可以用jps指令进行监测
```

### 节点的增删改查

像是Redis 有Redis Cli一样，Zookeeper也有对应的客户端我们借助这个客户端来实现创建节点操作。

- 永久节点

```shell
#连接zk1
/usr/zookeeper/zk1/apache-zookeeper-3.7.1-bin/bin/zkCli.sh -server 127.0.0.1:2181
# 创建一个节点 dog 是key 123 是value
create /dog 123 
# 获取目录中存储的值
get /dog
# 现在连接zk2 获取dog节点
/usr/zookeeper/zk2/apache-zookeeper-3.7.1-bin/bin/zkCli.sh -server 127.0.0.1:2181
# 获取dog目录中存储的值 会发现能够获取的到
get /dog
```

- 临时节点

  ```shell
  # 连接zk1 创建临时节点 -e 代表临时节点
  create -e /dog/cat  123
  # 连接zk2 获取/dog/cat
  get /dog/cat
  # 在zk1中输入quit指令,断掉当前会话
  quit
  # 在zk2就获取不到了
  ```

  ![创建临时节点](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3kioqst15j30kf0230th.jpg)

   ![zk2能获取到](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3kiq5uz53j310s0dy1dt.jpg)

![zk2能获取到](http://tvax2.sinaimg.cn/large/006e5UvNgy1h3kiq5uz53j310s0dy1dt.jpg)

![zk2就获取不到了](http://tva2.sinaimg.cn/large/006e5UvNgy1h3kirymd7nj30p902k3zk.jpg)

- 经典案例基: 基于Znode临时顺序节点+Watcher机制实现公平分布式锁

​        原理如下:  

![公平分布式锁](http://tva2.sinaimg.cn/large/006e5UvNly1h3kj693n9kj313u0f7gok.jpg)

请求A首先来到Zookeeper请求创建临时顺序节点，Zookeeper为请求A生成节点，请求A检查lock目录下自己的序号是否最小，如果是代表加锁成功，B监听节点顺序值小于自己的节点的变化，如果A执行则B去获取锁，如果有C、D等更多的客户端监听，道理是一样的。

```
create -s -e /dog/pig  s #在dog下创建临时顺序节点
# 返回值Created /dog/pig0000000001 
```

## 写在最后

其实Zookeeper还有其他功能，如下:

- 数据的发布和订阅
- 服务注册与发现
- 分布式配置中心
- 命名服务
- 分布式锁
- Master 选举
- 负载均衡
- 分布式队列

这里只介绍了基本的概念和应用，希望会对大家学习Zookeeper有所帮助，放英文注释也是提升自己阅读英文技术文档的水平。

## 参考资料

- 从0到1详解ZooKeeper的应用场景及架构  微信公众号  腾讯技术工程

- zookeeper 知识点汇总 https://www.cnblogs.com/reycg-blog/p/10208585.html#zookeeper-%E6%98%AF%E4%BB%80%E4%B9%88

- zookeeper入门  https://zookeeper.readthedocs.io/zh/latest/index.html#

- Nginx负载均衡当其中一台服务器挂掉之后，Nginx负载将会怎样呢？  https://blog.csdn.net/Tomwildboar/article/details/115382121

- 基于zookeeper的MySQL主主负载均衡的简单实现  https://www.cnblogs.com/TomSnail/p/4389297.html

  
