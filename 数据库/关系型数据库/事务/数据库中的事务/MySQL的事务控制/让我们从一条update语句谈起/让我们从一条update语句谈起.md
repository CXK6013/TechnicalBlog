# 让我们从一条update语句谈起

[TOC]

## 前言

我们知道我们执行update语句更新同一行数据的时候是互斥的，也就是说假设有两条update语句:

```java
# 语句A
update  actor set  first_name = 'hello' where actor_id = 1; 
# 语句B
update  actor set  first_name = 'world' where actor_id = 1;
```

如果语句A先执行，语句B后执行，那么A和B不会并发执行，也就是说B会等待A执行完再执行:

![](https://a.a2k6.com/gerald/i/2024/04/04/xp57d.jpg)

![](https://a.a2k6.com/gerald/i/2024/04/04/3jft.jpg)

 我们可以看到如果A如果一直不提交，B在一直等到超时之后，就会报: Lock wait timeout exceeded; try restarting transaction。我们可以通过:

```mysql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
```

 来查看事务等待获取锁的时间，一般默认为50秒。我们常常用这个特性来做一些控制，比如扣减库存, 我们就可以这么写:

```mysql
update product_stock set stock - #{buyStock} where id = #{productId}  and  stock - #{buyStock} > 0;
```

 通常为了避免重复扣减，一般也会加一个防重表，来做幂等性，所以完整的扣减sql可能如下所示:

```mysql
 start transaction;
 update product_stock set stock - #{buyStock} where id = #{productId}  and  stock - #{buyStock} > 0;
 Insert into antiRe(code) value (‘订单号+Sku’)
 end;
```

这个例子也让我想起Java中的ReentrantLock和synchronized, ReentrantLock相关的api如下:

```java
public void lock();
public void unlock();
public boolean tryLock();
public boolean tryLock(long timeout, TimeUnit unit);
```

synchronizede和ReentrantLock具备共性，JVM虚拟机会为每个内部所分配一个入口集(Entry Set), 用于记录等待获取相应内部锁的线程。 多个线程申请同一个锁的时候，只有一个申请者能称为该锁的持有线程(即申请锁的操作成功)，而其他申请操作会失败，这些申请失败的线程并不会抛出异常，而是会被暂停(生命周期变为BLOCKED)并被存入相应锁的入口集中等待再次申请锁的机会。

![](https://a.a2k6.com/gerald/i/2024/04/04/4m4k.jpg)

ReentrantLock也有一个队列，用于存放获取锁失败的线程。那么说回数据库呢?  MySQL事务锁的结构如下所示:

```java
struct lock_t {
  /** transaction owning the lock */
  trx_t *trx;
 
  /** list of the locks of the transaction */
  UT_LIST_NODE_T(lock_t) trx_locks;
 
  /** Index for a record lock */
  dict_index_t *index;
 
  /** Hash chain node for a record lock. The link node in a singly
  linked list, used by the hash table. */
  lock_t *hash;
  union {
    /** Table lock */
    lock_table_t tab_lock;
 
    /** Record lock */
    lock_rec_t rec_lock;
};
```

我们平时所说的加锁就是在内存中生成这样的一个锁结构(除了生成锁结构，还有一种称作隐式锁的加锁方式，不用生成锁结构)。当我们在执行一条update语句的时候，如果另一个update语句也对这行数据进行修改，那么我们现在知道他们会排队执行，也是为了维护数据的一致性。那我们单独只执行一条update语句呢，会不会加锁呢，我们假定这条update语句执行时间比较长，我们也没有显式的使用start transaction 这个指令，那么会有锁生成嘛，答案是会，为什么呢，原因在于还有DDL语句，比如给这个表加个字段，这个时候需要锁表，但如果有update语句在执行的时候，锁表指令会被阻塞:

> Intention locks do not block anything except full table requests (for example, LOCK TABLES ... WRITE). The main purpose of intention locks is to show that someone is locking a row, or going to lock a row in the table
>
> 意向锁不会阻塞不会阻塞任何操作，除了完整的锁表请求。意向锁的主要目的就是显示有人正在锁定或将要锁定表的某一行。

也就是说，如果有有条update语句正在对表里面的数据进行更新，这个时候锁表会被阻塞住，那么MySQL是如何判断当前表由更新操作的呢，遍历每一行数据？ 这太慢了。也就是:

> Intention locks are table-level locks that indicate which type of lock (shared or exclusive) a transaction requires later for a row in a table
>
> 意向锁是表级锁，显示事务将要通过哪种类型的锁(共享或独占)，来锁定表里面的某一行。

也就是说在获取行锁之前，必须先获取意向锁，表示你想去锁住表里面的某一行或者多行记录。 

## 共享锁与独占锁

这里我们回忆一下共享锁和独占锁的语法: 

```mysql
SELECT ... LOCK IN SHARE MODE; 
在读取记录的时候获取该记录的共享锁 共享锁简称S锁
SELECT ... FOR UPDATE;
在该事务中获取该记录的X锁  排他锁简称X锁
```

如果事务T1持有对行r的X锁，那么事务T2去获取行r的S锁的时候将会被阻塞: 

![](https://a.a2k6.com/gerald/i/2024/04/05/pmus.jpg)

![](https://a.a2k6.com/gerald/i/2024/04/05/ywmlu.jpg)

一直阻塞到独占锁释放或超过锁等待时间为止才会释放锁, 获取共享锁超时也会报 Lock wait timeout exceeded; try restarting transaction这个错。 反过来如果事务T1先获得了共享锁，那么事务T2获取共享锁会成功，如果事务T2尝试去获取独占锁，在超过锁等待时间之前会 一直阻塞，直到事务T1释放锁。超过超时时间就会报Lock wait timeout exceeded; try restarting transaction。这让我想起了Java里面的读写锁ReentrantReadWriteLock:

```java
public class RWDictionary {
    private final Map<String, Object> map = new TreeMap<>();
    private final ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();

    private final Lock r = reentrantReadWriteLock.readLock();

    private final Lock w = reentrantReadWriteLock.readLock();
    
    private Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    private String[] allKeys() {
        r.lock();
        try {
            return map.keySet().toArray(new String[]{});
        } finally {
            r.unlock();
        }
    }
    private Object put(String key, Object value) {
        r.lock();
        try {
            return map.put(key, value);
        } finally {
            r.unlock();
        }
    }
    public void clear(){
        w.lock();
        try {
            map.clear();
        }finally {
            w.unlock();
        }
        
    }
}
```

也是一样的，读锁共享，如果一个线程获取到读锁之后，如果其他线程申请获取写锁会被阻塞，然后等待读线程释放之后唤醒写线程。共享锁和独占锁和ReentrantReadWriteLock的设计理念一致，允许多个线程同时读，但只允许一个线程去写，提高性能的同时也保证了安全。那么这里提出一个问题，为什么MySQL一定要等到commit了，才去释放锁呢?

现在我们假设，MySQL在执行完update语句之后就会释放锁，看看会发生什么？

![](https://a.a2k6.com/gerald/i/2024/04/05/5b1l0.jpg)

① 最开始id为1的这一行的value为1，然后这一行的数据被session A 读取到，并加上了独占锁。

②  然后将id为1的这一行的value 改为3， 这里我们假设直接释放了锁。

③  然后第三步，session B开启将id为1这一行改为4。

④  session A 读到 id =1这一行value的值为4。

这里有什么问题呢，让我们回忆一下事务的Atomic(原子性)、Consistent(一致性)、Isolation(隔离性)、Durability(持久性)。

### ACID

#### Atomic(原子性)和Durability (持久性)

在事务的实现机制上，MySQL采用的WAL: Write-ahead logging 预写式日志机制来实现的，也就是说所有的修改都先被写入到日志中，然后再被刷新到系统中。这是一种确保数据完整性的标准方法，WAL的核心理念是，只要在记录对数据文件(表和索引所在的文件)的更改后，也就是在将描述更改的WAL记录刷新到磁盘之后，才能将这些更改写入到数据文件上。

按照这个流程，我们就需要在每次提交事务时将修改刷新到磁盘，因为我们知道，如果发生崩溃，我们可以使用日志恢复数据库，任何尚未应用到数据页的修改都可以通过WAL重新进行(这也就是前滚恢复 roll-forward recovery , 也称为REDO)，然后再从数据页中删除未提交事务所做的更改(后滚恢复，也称UNDO)

采用WAL可以显著的减少磁盘写入次数，因为只需要将WAL文件刷新到磁盘就可以保证事务的提交，而不是事务的每个更改，都需要刷新磁盘。WAL文件是顺序写入的，因此，同步WAL的成本比刷新数据页的成本要低的多。

这也就是MySQL 中redo日志和undo 日志的由来，MySQL用redo log来实现持久性，在数据变更之前将操作写入redo log，这样当发生断电之类的情况，系统可以重启后继续操作。MySQL用undo 日志来实现原子性。

#### 隔离性

那隔离性呢，那么应当像现实世界一样，现实世界中的两次状态应该是互不影响的，比如转账操作，A同时向B同时进行两次金额为5元的转账，假设在两台ATM机上可以同时进行操作，那么A最后的账户肯定会少10元，B的账户上肯定会多10元。那么在数据库的世界A向B转账5元的操作是由下面几个步骤组成的:

1. 将小B的账户余额读取到变量A中，这一步骤简记为read(A)

   2 .将小B的余额减去账户余额，简记为 A = A - 5

   3 . 将小B修改过后的余额写入磁盘里，这一步骤简单写为write(A)

4. 读取小A账号的余额到变量B，这一步骤简写为read(B)

5. 将小A的账户余额加上小B转账过来的余额， 简单记为B = B + 5。

6. 将小A账户修改过的余额写到磁盘里，这一步骤简写为write(B)

为了说明问题，我们将两次转账操作记为T1、T2。在并发执行的场景下, T1在read(A)之后，很快T2也执行了read(A), 我们假设在转账操作进行之前，小B只有10元钱，也就是T1和T2在进行转账操作的时候都认为小B有十元钱，然后假定T1开始执行2,3,4,5,6之后，T2接着执行，在这种情况下，小B只转了五元钱，小A的账户确多出了十元，多出的五元，让银行补出来？ 这显然不合理。那为了避免T1，T2交替执行带来的问题，最简单的方法就是让T1，T2在数据库排队执行，这会慢的要死。 所以对于现实世界中状态转换对应的某些数据库操作来说，不仅要保证这些操作以原子性的方式来执行完成，而且要保证其他的状态转换不会影响到本次状态转换，这个规则我们称之为隔离性。

也就是说 ，一个事务的执行不应该受到其他并发事务的干扰。也就是说,事务应该像是独占地访问数据库,其结果应该与事务串行执行的结果相同。

###  回到上个问题

那有了对隔离性的理解之后，我们就可以







## 所以只有这些锁吗？



### 还有 Record Lock

​	

### 还有 gap lock



### 还有Next-Key Locks



### 还有Insert Intention Locks



## 总结一下对事务的理解



## 总结一下



## 参考资料

[1] 《 Mysql锁：灵魂七拷问 》 https://tech.youzan.com/seven-questions-about-the-lock-of-mysql/

[2]  电商库存系统的防超卖和高并发扣减方案 | 京东云技术团队 

[3]  What is an isolated transaction in Java?  https://stackoverflow.com/questions/5556146/what-is-an-isolated-transaction-in-java

[4] Transaction Isolation Levels and why we should  care https://www.metisdata.io/blog/transaction-isolation-levels-and-why-we-should-care

[5] ACID Explained: Atomic, Consistent, Isolated & Durable  https://www.bmc.com/blogs/acid-atomic-consistent-isolated-durable/

[6] MYSQL 事务的底层原理 | 京东物流技术团队  https://juejin.cn/post/7300855272913240073#heading-0

[7] Write-Ahead Logging (WAL)   https://www.postgresql.org/docs/current/wal-intro.html 