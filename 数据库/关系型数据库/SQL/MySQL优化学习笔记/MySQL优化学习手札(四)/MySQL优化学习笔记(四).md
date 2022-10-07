# MySQL优化学习笔记(三)

> 老实说这部分相关的知识点早就已经准备好了，只是苦于不知道该组织这些内容而已，昨晚想到了该如何组织这部分内容。
>
> 本系列的文章不加说明，一般都是在InnoDB数据引擎讨论。

在开始看本篇文章之前，建议先看:

- SQL查询模型和查疑补漏
- LeetCode刷题四部曲之SQL篇(一)
- MySQL优化学习手札(一)
- MySQL优化学习笔记手札(二)

## buffer pool 缓存池 的引入

### 简介

我们在那里已经唠叨过一条SQL由客户端发送给MySQL的服务端会发生些什么了，对此没有了解的，建议翻一下《MySQL优化学习手札(一)》

到目前为止，从宏观上来看，SQL被发送给MySQL服务端之后，MySQL的存储引擎根据SQL从磁盘中提取出来对应的数据，但你知道磁盘的速度相对于的内存的速度是很慢的，即使是固态硬盘。如果每次提取数据都从磁盘中提取数据，那未免有点太慢了吧。对于InnoDB作为存储引擎的表, MySQL的开发者设计了缓存，来避免每次提取数据都从缓存池里面提取数据。InnoDB存储引擎在处理客户端的请求时，当需要访问一个页的一条记录时, 就会把完整的页的数据加载到内存中，也就是说即使我们只需要访问一个页的一条记录，那也需要把整个页的数据加载到内存中。将整个页加载到内存中后就可以进行读写访问了, 在进行读写访问之后并不着急把该页对应的内存空间释放掉，而是将其缓存起来，这样将来有请请求再次访问该页面时，就可以避免直接从磁盘中提取数据了。

为了缓存磁盘中的页，在MySQL服务器启动的时候就向操作系统申请了一片连续的内存，这片内存也就是buffer pool，通过

```mysql
SHOW ENGINE INNODB STATUS;
```

可以看到缓存池的基本状态:

![缓存池.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz3vmysjk0j30xb0g7qbg.jpg)



Buffer Pool中默认的缓存页大小和在磁盘上默认的页大小是一样的，都是16KB。每个缓存页都会有一些对应的控制信息，包括页号、缓存页在Buffer Pool中的地址等，每个缓存页对应的控制信息占用内存大小是相同，我们称控制信息所占用的 内存为控制块，控制块和缓存页是一一对应的，都位于缓存池中，控制块位于缓存页之前。像下面这样:

![缓存页与控制块.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz3w9brnimj30xh0bb75i.jpg)

这个碎片是啥? 申请的内存空间在分配了控制块和缓存页，不够一个控制块和缓存页所需的内存空间呗，又假设控制块和缓存页所占用的内存一样，我们极端假设一下缓存池只有57KB，只够凑一个控制块和缓存页，剩下25KB，这25KB就是内存碎片。

每个控制块大约占用缓存页大小的5%，在MySQL 5.7.21这个版本占用的大小是808字节。而我们设置的innodb_buffer_pool_size并不包含这部分控制块占用的内存大小，也就是说InnoDB在Buffer Pool向操作系统申请连续的内存空间时，这片连续的内存空间一般会比innodb_buffer_pool_size的值大5%左右。Buffer Pool的大小可以通过MySQL的配置文件my.ini来指定，现在我们来看一下我MySQL下面Buffer Pool的配置:

>  InnoDB, unlike MyISAM, uses a buffer pool to cache both indexes and  row data. The bigger you set this the less disk I/O is needed to
>  access data in tables. On a dedicated database server you may set this parameter up to 80% of the machine physical memory size. Do not set it
>  too large, though, because competition of the physical memory may  cause paging in the operating system.  Note that on 32bit systems you  might be limited to 2-3.5G of user level memory per process, so do not set it too high.
>
>  InnoDB存储引擎，使用缓存池缓存索引和行数据，缓存池越大磁盘IO越少，在专门的数据库服务器上你可以将这个参数调整为机器物理内存的百分之八十以上。不要设置的太高，否则进程间竞争物理内存可能会导致操作系统的分页(关于操作系统管理内存相关的知识，我已经忘的差不多了，今年打算重修，对这块了解的可以在评论区留言)。注意在32位操作系统上，每个进程可能会被限制只能使用2-3.5G的内存。
>
>  innodb_buffer_pool_size=8M
>
>  总共缓存池的大小是8M
>
>  innodb_buffer_pool_instances=8
>
>  这个不代表有8个缓存池

 innodb_buffer_pool_size/innodb_buffer_pool_instances 是每个缓存池实例的大小，当innodb_buffer_pool_size的值小于1G，innodb_buffer_pool_instances这个参数设置就是无效的。MySQL官方推荐在innodb_buffer_pool_size大于1G的情况下，设置多个buffer pool实例。

### free链表(空闲链表)

MySQL启动的时候，就会完成对Buffer Pool的初始化过程，就是先向操作系统申请Buffer Pool的内存空间，然后把它划分成若干对控制块和缓存页，但目前缓存池中所有的控制块和缓存页还没有存储信息，随着MySQL开始接收查询请求，InnoDB开始从磁盘页中提取数据，缓存池的控制块和缓存页开始存储信息，为了确定从磁盘上提取的页该放到哪个缓存页，区分哪些缓存页是空闲的，哪些缓存页是已经被使用的，缓存页对应的控制块就派上了用场，MySQL的开发者将所有空闲的缓存页对应的控制块做为一个结点放到一个链表中，这个链表我们称之为free链表。刚刚初始化完成的缓存池所有的缓存页都是空闲的，所以每一个缓存页对应的控制块都会被加入到free链表中。

![空闲链表.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz41l944pvj30x407njtb.jpg)



黄色的结点是链表的头结点，记录链表首结点和尾结点，以及当前链表中结点的数量等信息。链表的头结点占用的内存空间并不大，在MySQL5.7.21这个版本，每个头结点只占用40字节大小。有了这个free链表之后事就好办了，每当需要从磁盘中加载一个页到缓存池中时，就从空闲链表中取一个空闲的缓存页，并且把该缓存页对应的控制块的信息填上(就是该页对应的页号之类的信息)，并且把该缓存页对应的free链表结点从链表中移除，表示该缓存页已经被使用了。

### LRU链表

Buffer Pool本质上是InnoDB向操作系统申请的一块连续的内存空间,是有限的,我们不可能将所有的磁盘页都加载进入内存，那我们该如何缓存数据页呢？或者说我们该如何制定缓存策略呢？ 理想的情景是访问某个数据页的时候，这个数据页已经在缓存池里面了。这也就是LRU，LRU全称: Least Recently Used, 按照最近最少使用的，也就是说当Buffer Pool中不再有空闲的缓存页时，就需要淘汰部分最近很少使用的缓存页，那么我们该如何知道哪些缓存页最近频繁使用，哪些最近很少使用呢？ 我们可以借助于链表，采取LRU策略:

当我们需要访问某个页时:

- 如果该页不在缓存池里，，就把该页从磁盘加载到缓存池中的缓存页时，就把该缓存页对应的控制块作为链表的头结点。
- 如果该页已经缓存在缓存池里，则直接把该页对应的控制块移动到链表头部。

那么该链表的尾部就是最近最少使用的缓存页喽，当Buffer Pool中的空闲缓存页使用完时，到LRU链表的尾部找些缓存页淘汰就可以了。

这个问题还没有被完全解决, 虽然从宏观上来看LRU策略没什么问题, 但我们仍然需要根据实际情况对LRU策略做些小补丁, 专门应对一些特殊的状况: 

- 需要扫描全表的查询语句(没有用到索引, 没有根WHERE子句的查询)，扫描全表意味着需要访问该表所有的数据页, 假设这个表中记录非常多，又假设这张表数据比较大，那么就意味着缓存池中的所有页都被换了一次血，其他查询语句在执行时又得执行一次从磁盘加载到缓存池的操作。这种全表扫描的语句执行频率也不高，但是每次执行都会把缓存池中的缓存页换一次血，严重影响其他查询对缓存池的使用，从而大大降低了缓存的命中率(缓存的命中率 = 缓存页的访问次数 除以 缓存页在缓存的次数)。

- MySQL的预判, 像是我们玩英雄联盟会预判对手的操作一样，MySQL也有对查询请求的预判，我们称之为预读, 所谓的预读，就是InnoDB不仅仅会加载查询请求所对应的数据页, 还会额外加载一些数据页, 根据触发方式不同，预读又可以细分为下边两种: 
  - 线性预读
  - 随机预读

 	介绍这两种预读, 需要懂一些MySQL如何组织数据，我们这里来简单介绍一下，到目前为止我们知道InnoDB以页为基本单位管理数据，每个索引都对应着一棵B+树，该B+树的每个结点都是一个数据页，数据页之间不是位于连续的内存空间，因为数据页之间有双向链表来维护着这些页的顺序。InnoDB的聚簇索引的叶子结点存储了完整的用户记录，也就是所谓的索引即数据，数据即索引。 

为了更好的管理数据页，MySQL在数据页的基础上设计了表空间(table space 有的也称file space)这个概念, 这个表空间是一个抽象的概念，它可以对应文件系统上一个或多个真实文件(不同的表空间对应的文件数量可能不同)。每一个表空间可以被划分为许多个页，我们的表数据就存放在表空间的数据页里。

InnoDB表空间又分为多种类型:

- 系统表空间: 需要注意的一点是，在MySQL服务器中，系统表空间只有一份。从MySQL 5.5.7到 MySQL5.6.6之间的各版本，我们表中的数据会被默认存储在这个系统表空间中。
- 独立表空间

那这些数据存储在磁盘上的哪里呢？ 我们可以通过:

```mysql
SHOW VARIABLES LIKE 'datadir';
```

命令来查看数据目录：

![数据目录.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz4vv280icj30xr0iq76y.jpg)

  我们看下这个数据目录下有什么:

![磁盘上的数据目录.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz4w1hwanpj30p00bjjwg.jpg)

数了数一共六个文件夹刚好对应六个数据库, 我们进studydatabase这个文件夹看一下，

![文件夹下的表.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz4weoicp3j30ix05cdi0.jpg)

studydatabase一共有三张表: score、student、student_info, 一张表对应两个文件, 表名.frm存储表的结构信息，ibd存储表的数据信息。

对于表空间来说，MySQL设立了区的概念来管理页(英文名: extent)，对于16KB的页来说，连续64个页就是一个区。有了区这个概念我们回头接着来介绍MySQL的预判: 

- 线性预读

> 如果顺序访问某个区的页面超过这个innodb_read_ahead_threshold这个系统变量的值，就会触发一次下一个区中全部的页面到Buffer Pool的请求。

- 随机预读

> 如果Buffer Pool中已经缓存了某个区的13个连续的页面，不论这些页面是不是顺序读取的，MySQL都会异步将这些页面所在区的所有页面加载到缓存池中。我们可以通过innodb_random_read_ahead来关闭开启随机预读。默认为关闭状态。

预判本来是个好事情, 但是如果预判出错, 这些预读的页都放到LRU链表的头部，碰巧我们的缓存池也不大，这就会导致LRU链表会被淘汰掉，大大降低缓存命中率。

针对这两种情况，MySQL将LRU链表按照一定比例分成两段。分别是:

- 一部分存储使用频率非常高的缓存页，这部分链表我们称之为热数据或者young区域
- 另一部分存储使用频率不是很高的缓存页，所以这一部分链表也叫冷数据，或者old区域。

热区域的数据和冷区域的数据是不固定的，冷区域的数据也可能被转化为热区域的数据。我们可以通过:

```mysql
 SHOW VARIABLES LIKE 'innodb_old_blocks_pct';
```

来查看热区域和冷区域的比例, 

![热冷区域比例.png](http://tva1.sinaimg.cn/large/006e5UvNly1gz4xi2xyupj30ud058abi.jpg)

默认情况下old区域占37%，有了这个划分之后，上面两种情况的补丁就好打了:

- 针对预读这种情况，初次加载数据页到缓存池中，会被放在old区域的头部，这样预读的数据页却不经常被访问到的，就会慢慢从链表的尾部淘汰。

- 针对全表扫描这种查询频率十分低的场景，对某个处在old区域的缓存页进行第一次访问时在它对应的控制块中记录下这个访问时间，如果后序的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会从old区域移动到young区域的头部。这个间隔时间是由innodb_old_blocks_time控制的 , 通过innoddb_old_blocks_time来控制:

  ```mysql
   SHOW VARIABLES LIKE 'innodb_old_blocks_time';
  ```

事实上仅仅这两个补丁还不够，还得继续打下去，但继续讲下去并不是本篇的主题。

### flush链表的管理

如果我们修改的某条数据，先将该页加载进入到缓存页中，然后直接对缓存页进行修改，那么此时缓存页的数据就和磁盘页的数据不一致了。当然，我们也可以修改完缓存页的时候马上同步到磁盘对应的页上，但是频繁的和内存进行交互相当影响性能，毕竟磁盘读写的速度相当慢。所以每次修改完缓存页的时候，MySQL并不会立即把修改同步到磁盘上，而是在某个时间点进行同步。

但如果不立即进行同步的话，那我们该如何知道Buffer Pool中的哪些页是脏页呢？链表同志，请再次出场，所有的脏页对应的控制块都会作为结点加入到一个链表中，因为这个链表对应的缓存页都是需要被刷新到磁盘上的，所以也叫flush链表。

InnoDB还有其他形式的链表，比如unzip LRU链表用于管理解压页等等，我们上面介绍buffer pool配置的时候，默认配置是8个，Buffer Pool本质上来说Buffer Pool是InnoDB向操作系统申请的一块连续的内存空间，在多线程环境下，访问缓存池的链表都需要加锁处理，在缓存池比较大且高并发访问下，单个buffer pool 可能会影响处理速度，所以如果单个buffer pool特别大的时候，我们可以将它们拆分为若干个小的Buffer Pool。每个Buffer Pool都是独立的，多线程并发访问并不会相互影响，从而提高处理并发的能力。

### buffer pool的一些注意事项

在MySQL 5.7.5 之前，Buffer Pool的大小只能在MySQL启动的时候指定，在MySQL 5.7.5之后的版本包括MySQL5.7.5，MySQL支持在运行时调整缓存池大小，MySQL以chunk为单位向操作系统申请内存地址空间，也就是缓存池又可以看做是若干chunk组成的。一个chunk代表一个连续的内存地址空间。这个chunk的大小由配置文件中的innodb_buffer_pool_chunk_size来指定。

为了保证每一个buffer pool实例中包含的chunk数量相同，innodb_buffer_pool_size 必须是 innodb_buffer_pool_chunk_size 乘 innodb_buffer_pool_instances的倍数。如果你设置的不是，MySQL会自动调整，举个例子，innodb_buffer_pool_chunk_size 乘 innodb_buffer_pool_instances = 2G，你指定的是9G，MySQL会自动将其提升为10G。

如果服务启动的时候，innodb_buffer_pool_size大于 innodb_buffer_pool_chunk_size 乘 innodb_buffer_pool_instances，那么innodb_buffer_pool_chunk_size 的值会被设置为 innodb_buffer_pool_size / innodb_buffer_pool_instances.举个例子, 假设你将innodb_buffer_pool_size设置为2G，innodb_buffer_pool_chunk_size  = 128M，innodb_buffer_pool_instances = 32，也就是4G。那么innodb_buffer_pool_chunk_size会被调整为64M。

## 总结一下

本文基本上的写作思路是，现在MySQL查询速度比较缓慢，在有效的利用索引的情况下，还能怎么提升MySQL的运行速度，带着疑问去阅读掘金小册《MySQL 是怎样运行的：从根儿上理解 MySQL》的buffer pool章节，最终的答案就是查看buffer pool的配置是否合理，多数的内容都从其摘录而来，用带着问题的方式又将其的内容又组合了一下。

