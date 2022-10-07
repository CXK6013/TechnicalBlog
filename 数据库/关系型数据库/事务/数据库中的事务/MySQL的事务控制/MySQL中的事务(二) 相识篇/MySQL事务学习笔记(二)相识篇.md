# MySQL事务学习笔记(二) 相识篇

 天色将晚， 在我看着你的眼里色彩斑斓 《娱乐天空》

## 序

在《MySQL事务学习笔记(一) 初遇篇》我们已经交代了如何开启事务、提交、回滚。但是还有一个小尾巴被遗漏了，就是如何设置事务的隔离级别。本篇我们就来介绍MySQL中是如何设置隔离级别ySQL中是如何实现事务的ACID以及隔离级别。写作这篇文章的时候，我也在思考如何组织这些内容，是再组织一下自己看的资料上的内容，还是笔记式的，罗列一下知识点。坦率的说我不是很喜欢罗列知识点这种形式的，感觉没有一条线组织起来，我个人比较喜欢的是像是树一般的知识组织结构，有一条主干。所以本篇在介绍MySQL是实现事务实现的时候，会先从宏观上介绍其组织，部分知识点不会太详细，这样的方式可以让我们先把握其主干，不会迷失在细节中。

## 设置事务隔离级别

```mysql
select @@tx_isolation;
```

![事务的隔离级别](https://tvax2.sinaimg.cn/large/006e5UvNly1h06yryvmotj30e604o753.jpg)

我的MySQL默认隔离级别为可重复读，SQL事务的隔离级别:

- 未提交读
- 已提交读
- 可重复读
- 可串行化。

MySQL支持在运行时和启动时设置隔离级别:

- 启动时设置隔离级别:

  windows下的配置文件取 my.ini

  Linux下的配置文件取 my.cnf 

  在配置文件中添加:  transaction-isolation =  隔离级别

  隔离级别的候选值: READ COMMITTED， REPEATABLE READ， READ UNCOMMITTED，SERIALIZABLE

- 运行时设置隔离级别

​		SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;

​    	LEVEL的候选值:  READ-COMMITTED， REPEATABLE READ， READ UNCOMMITTED，SERIALIZABLE

​        GLOBAL的关键字在全局范围内影响，在执行完下面语句之后:

```mysql
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

 后面所有的会话的隔离级别都会变为可串行化；

 ![读已提交](https://tvax4.sinaimg.cn/large/006e5UvNly1h074g8g8gpj30iz05rmz7.jpg) 

   ![隔离级别变为可串行化](https://tvax1.sinaimg.cn/large/006e5UvNly1h074hiycr4j30eq046js2.jpg)

​	而SESSION关键字则是只在会话范围内影响，如果事务还未提交则只对后面的事务有效。

​    如果GLOBAL和SESSION都没有，则只对当前会话中下一个即将开启的事务有效，下一个事务执行完毕，后序事务将恢复到之前的隔离级别。该语句不能在已经开启的事务中执行，会报错。

 ![事务的隔离级别报错](https://tva3.sinaimg.cn/large/006e5UvNly1h074n5bisij30mn03l0u5.jpg)

​	下面我们就来演示事务在不同的隔离级别会出现的问题:

- 事务的隔离级别为未提交读:

```mysql
SET GLOBAL TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

​     ![这个事务还未提交](https://tvax2.sinaimg.cn/large/006e5UvNly1h074zoy78hj30fq083q5c.jpg)

​      然后再打开一个窗口: 

​      ![查到了一个未提交的事务提交的数据](https://tvax3.sinaimg.cn/large/006e5UvNly1h0750w4zq4j30fr04xab5.jpg)

​	  发生了脏读

-  事务的隔离级别为已提交读

```mysql
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

![已提交读](https://tvax1.sinaimg.cn/large/006e5UvNly1h075fyw160j30mc0bp42i.jpg)

再开一个会话:

​	![Snipaste_2022-03-12_15-52-17](https://tva3.sinaimg.cn/large/006e5UvNly1h075gxr0ynj30eg04zabe.jpg)

![不可重复读](https://tva4.sinaimg.cn/large/006e5UvNly1h075k5h54bj30kr08ewho.jpg)

出现了不可重复读:

![不可重复读-1](https://tvax4.sinaimg.cn/large/006e5UvNly1h075lt0l21j30dw0a3418.jpg)

- 隔离级别为可重复读

```mysql
SET GLOBAL TRANSACTION ISOLATION LEVEL  REPEATABLE READ;
```

![可重复读测试](https://tva4.sinaimg.cn/large/006e5UvNly1h075wedjfpj30he08a77l.jpg)

![Snipaste_2022-03-12_16-06-09](https://tvax3.sinaimg.cn/large/006e5UvNly1h075xipx9tj30hv08tq5s.jpg)

 上面我们讲到MySQL在该级别下可以做到禁止幻读的，我们这里来测试一下:

  ![幻读演示-1](https://tvax3.sinaimg.cn/large/006e5UvNly1h0761tld2ej30m30gpq8k.jpg)

​    这张图打错了，是5和6才对。

![幻读演示-2](https://tvax2.sinaimg.cn/large/006e5UvNly1h0761ze53lj30hx03etac.jpg)

   

下面我们来分别讲述MySQL是如何实现隔离级别、ACID的。

## redo 原子性 持久性

在《MySQL优化学习手札(一)》，我们讲到MySQL以页为单位作为磁盘和内存的基本的交互单位，增删改查事实上都是在访问页面(读、写、创建新页面)，虽然我们是访问页面但是我们访问的并不是磁盘的页面，而是缓存池的页面，由工作线程定时将缓存池的更新页面刷新到磁盘上，那么问题来了，某个页面的数据被改变，还没有来得及将此页面刷新到磁盘上，碰到了一些故障，MySQL是如何保证持久性呢？ 所谓持久性就是指对一个已经提交的事务，在事务提交后，即使系统发生了崩溃，这个事务对数据库中所做的更改也不能丢失。

简单而无脑的做法是在更新buffer pool的数据页之后，立刻将该页刷新到磁盘上，但是刷新一个完整的数据页太浪费了，有的时候我们可能只改动了某个页面中的某行数据的一个字段，这刷新到磁盘上花费的代价有点大。 其次假设这个事务虽然只有一条语句，但是修改了很多页的数据，又不巧，这些页不相邻，这就很慢。

MySQL的做法是存储修改数据的元信息，比如将Student的id = 1这一列的name改为张三, MySQL就会存储这条数据在那个数据页的某某行数据的某某列改为张三，取增量，记录变化。这样在我们事务提交后，我们将改变刷新到磁盘中，即使工作线程还没有来得及将缓存池的页刷新到磁盘上，系统崩溃了，再重启的时候我们根据这些记录的改变再恢复一下数据即可，记录改变的数据在MySQL中被称为重做日志，特就是redo  log。 与事务提交时将所有修改过的内存中页面刷新到磁盘中相比，直接将事务执行过程中产生的redo日志刷新到磁盘好处如下:

- redo 日志占用的空间非常小
- redo日志按顺序写入磁盘

那原子性呢，其实也是借助redo日志，在执行这些保证原子性的操作时必须以组的形式来记录redo 日志，在进行数据恢复的时候，系统中的某个组的日志要么全部恢复，要么全部不恢复。redo 日志也有自己的缓存区，也并不是直接刷新到磁盘上。

## undo  日志  回滚

如果事务执行了一半，系统断电了怎么办，又或者手动执行了回滚，我们该如何回滚，答案是记录一下改变，即将什么改变成了什么(这里的改动指的是UPDATE INSERT，UPDATE)，MySQL将这些记录改变的数据称为undo log ，不同类型的update log 不同。如果某个事务对某个表执行了增、删、改这样的操作，InnoDB引擎为这个事务分配一个唯一的事务id。上面我们唠叨了，MySQL以页为单位作为磁盘和内存的基本交互单位，页里面是行记录，每行会有多个隐藏列:

- trx_id： 每次一个事务对某条聚簇索引记录进行改动时，都会把该事务得到事务id赋值给trx_id隐藏列。
- roll_pointer: 每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo 日志中，然后这个隐藏列就相当于一个指针，可以通过它来记录修改前的信息。

那可能有同学会问，那多条事务更新一条记录怎么办，MySQL会让他们排队执行，可以理解为锁，我们来试试看，两个事务同时更新一条记录怎么办?

![先不提交](https://tva2.sinaimg.cn/large/006e5UvNly1h077nd02l9j30gp086778.jpg)

![另一条语句卡在这里](https://tvax1.sinaimg.cn/large/006e5UvNly1h077nowedtj30om04qgmn.jpg)

过了一会就会出现 Lock wait timeout exceeded; try restarting transaction.

每次对记录进行改动，都会记录一条undo 日志，每条undo 日志 也都会有roll_pointer属性，这些日志可以串起来成一条链表。版本链的头结点记录的是当前记录最新的值，每个版本还包含一个事务 ID。对于隔离级别是READ UNCOMMITED的事务来说，由于可以读取到未提交事务修改过的数据，所以直接读取最新版本就好。对于READ COMMITED和REPEATABLE READ隔离级别的事务来说，都必须保证读到已经提交过的事务，也就是说如果当前事务未提交，是不能读取最新的版本记录的，那现在的问题就是该读取链表中的哪条记录，由此我们就引出READ VIEW这个概念。

## READ VIEW的生成时机 MVCC

READ VIEW有四个比较重要的内容:

- m_ids:  表示在生成ReadView时当前系统中活跃的读写事务的事务ID列表
- min_trx_id: 表示在生成ReadView时当时系统活跃的读写事务中的最小事务ID，也就是m_ids的最小值。
- max_trx_id: 表示在生成ReadView时系统应该分配给下一个事务的ID。
- creator_trx_id: 表示生成该ReadView的事务的事务Id。

 如果访问版本的trx_id与READ VIEW中的creator_trx_id表名当前事务再访问它自己修改的记录，直接访问链表最新的头结点即可。

 如果被访问版本的trx_id小于Read View中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView之前已经提交，所以该版本可以被当前事务访问。

如果被访问版本的trx_id 大于等于或Read View中的max_trx_id，表明生成版本的事务在当前事务生成Read View才开启，所以该版本不可以被当前事务访问。

如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断trx_id的属性值在不在m_ids中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以访问，如果不在，说明创建ReadView时生成该版本的事务已经被提交。

现在访问数据的方式就是在遍历数据对应的undo 链表，按照步骤判断可见性，如果遍历到最后都不可见，那就是真的不可见。

在MySQL中， READ COMMITED 和REPEATABLE READ隔离级别的一个非常大的区别就是生成ReadView时机不同。事务在执行过程中，只有在第一次真正修改记录时(INSERT DELETE UPDATE)，才会被分配一个单独的事务id, 这个事务id是递增的。

下面我们举一些例子来说明在不同隔离级别下，查询时的过程。在故事的开始我们依然是准备一个表:

```mysql
CREATE TABLE `student`  (
  `id` int(11) NOT NULL COMMENT '唯一标识',
  `name` varchar(255) COMMENT '姓名',
  `number` varchar(255) COMMENT '学号',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB 
```

### READ COMMITTED 每次查询都生成一个 Read View

现在有两个事务ID为200和300的正在执行，像下面这样:

```mysql
# 事务ID 为 200， id = 1 的name 在这个事务开始之前为 王哈哈
BEGIN;
update  student set name =  '李四' where id = 1;
update  student set name =  '王五' where id = 1;
```

```mysql
# 事务ID 为 300
BEGIN;
# 做更新其他表的操作
```

这个时候id 为1这行记录的版本链如下图所示:

![版本链](https://tvax2.sinaimg.cn/large/006e5UvNly1h0791tm9h4j30ud0gbq5g.jpg)

现在另一个事务开始查询id=1这条记录，执行SELECT语句即会生成一个Read View，Read View的值为[200,300],min_trx_id为200，max_trx_id为301，creator_trx_id为200。然后开始遍历undo 链表，最新的版本是王五，trx_id = 200, 在min_ids中不符合可见性原则，访问下一条记录，下一条记录的trx_id 为200，跳到下一个记录。王哈哈的trx_id小于min_trx_id，然后将这行记录返回给用户。

### REPEATABLE READ 第一次读的时候生成一个Read View 

还是上面的更新语句：

```mysql
# 事务ID 为 200， id = 1 的name 在这个事务开始之前为 王哈哈
BEGIN;
update  student set name =  '李四' where id = 1;
update  student set name =  '王五' where id = 1;
```

```mysql
# 事务ID 为 300
BEGIN;
# 做更新其他表的操作

```

然后使用REPEATABLE READ的隔离级别来查询:

```mysql
begin;
SELECT * FROM Student Where id = '1';
```

 上面这个SELECT查询会生成一个Read View:  m_ids[200,300], min_trx_id=200,max_trx_id=301,creator_trx_id=0。

  ![版本链](https://tvax2.sinaimg.cn/large/006e5UvNly1h0791tm9h4j30ud0gbq5g.jpg)

最新的版本的trx=id在min_ids中，该版本不可见，到下一条记录，李四的trx_id也为200，也在min_id中，也不可见。王哈哈的版本id小于read view中的min_trx_id, 表明这个记录在Reada View之前产生，返回该记录。然后提交一下事务ID=200的操作。

```mysql
BEGIN;
update  student set name =  '李四' where id = 1;
update  student set name =  '王五' where id = 1;
COMMIT;
```

然后事务ID 为 300也对id = 1进行修改。

```mysql
begin;
update  student set name =  '徐四' where id = 1;
update  student set name =  '赵一' where id = 1;
```

现在的版本链就如下图所示:

![可重复读-1](https://tvax2.sinaimg.cn/large/006e5UvNly1h07a7v013oj30x10l9n1f.jpg)

查询id = 1的记录:

```mysql
begin;
SELECT * FROM Student Where id = '1';
```

之前已经产生过read view了，复用上面的read view， 然后当前记录的事务id在min_ids[200,300]中，该记录不可见, 跳到下一条记录中，下一条的trx_id 为300，也在min_ids中，不可见，然后跳到下一条记录，下条记录的trx_id也在min_ids中，不可见。直到“王哈哈”，这也就是可重复读的含义。即使事务ID为300的事务提交了，其他事务读到了也会是“王哈哈”。对于这条记录的事务全部提交之后，再次查询该记录会重新再产生Read View。这也就是MVCC(Multi-Version Concurrency Control 多版本并发访问控制)，在READ COMMITTED、REPEATABLE READ隔离级别，避免脏读、不可重复读所采取的策略。READ COMMITE每次查询都会生成一个Read View。 而REPEATABLE  READ则是在第一次进行相关的记录查询的时候生成Read View，之后查询复用这个Read View，被这些事务中查询操作复用的Read View，在提交之后。再查询对应的记录的时候，再重新产生。

## 总结一下

MySQL下借助undo、redo实现原子性、持久性、已提交读，可重复读。redo记录记录发生了什么改变，undo用于回滚。为了支持MVCC，对应的记录删除之后删掉不会立马删除而是会打上标记。好像是在刚毕业的时候就接触到了MVCC，那个时候觉得这个很是高端和复杂，今天在写这篇文章的时候，还是先从宏观入手，没有介绍之前打算undo、redo日志的格式相关的细节，根据我的经验，介绍这些格式会很让人头晕，迷失在细节之中，其实初衷只是为了了解MySQL对ACID和事务的实现。所以本文只介绍了必要的内容，到后面的文章如果必须引用这些日志的详细介绍的时候会再介绍一遍。

## 参考资料

- MySQL 事务的四种隔离级别   https://blog.51cto.com/moerjinrong/2314867
- MySQL 是怎样运行的：从根儿上理解 MySQL   https://juejin.cn/book/6844733769996304392/section/6844733770071801870
- [Mysql]——通过例子理解事务的4种隔离级别  https://www.cnblogs.com/snsdzjlz320/p/5761387.html
