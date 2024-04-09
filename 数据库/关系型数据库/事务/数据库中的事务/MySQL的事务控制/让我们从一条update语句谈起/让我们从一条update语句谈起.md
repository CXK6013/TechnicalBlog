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

1. 将小B的账户余额读取到变量A中，这一步骤简记为read(A)、
2.  将小B的余额减去账户余额，简记为 A = A - 5
3.  将小B修改过后的余额写入磁盘里，这一步骤简单写为write(A)
4. 读取小A账号的余额到变量B，这一步骤简写为read(B)
5. 将小A的账户余额加上小B转账过来的余额， 简单记为B = B + 5。
6. 将小A账户修改过的余额写到磁盘里，这一步骤简写为write(B)

![](https://a.a2k6.com/gerald/i/2024/04/08/12lx5f.png)



我们可以看到在排队执行的情况下，不会有任何问题，如果是并发执行呢？ 就会出现下面这种状况:

![](https://a.a2k6.com/gerald/i/2024/04/08/40vn.png)	

为了说明问题，我们将两次转账操作记为T1、T2。在并发执行的场景下, T1在read(A)之后，很快T2也执行了read(A), 我们假设在转账操作进行之前，小B只有10元钱，也就是T1和T2在进行转账操作的时候都认为小B有十元钱，然后假定T1开始执行2,3,4,5,6之后，T2接着执行，在这种情况下，小B只转了五元钱，小A的账户确多出了十元，多出的五元，让银行补出来？ 这显然不合理。那为了避免T1，T2交替执行带来的问题，最简单的方法就是让T1，T2在数据库排队执行，这会慢的要死。 所以对于现实世界中状态转换对应的某些数据库操作来说，不仅要保证这些操作以原子性的方式来执行完成，而且要保证其他的状态转换不会影响到本次状态转换，这个规则我们称之为隔离性。

也就是说 ，一个事务的执行不应该受到其他并发事务的干扰。也就是说,事务应该像是独占地访问数据库,其结果应该与事务串行执行的结果相同。

###  回到上个问题

那有了对隔离性的理解之后，我们接着讨论上个问题，事务B的修改居然让事务A看到了，各个事务之间应该是像排队一样执行，这违反了事务ACID的Isolation(隔离性)。所以MySQL必须等commit之后才能释放这个锁。

那么接着有人会问了，那如果我将事务的隔离级别改到读未提交呢(Read Uncommited),可以读到对方未提交的东西，是不是就不需要满足隔离性，是不是就可以不用等到 commit 才释放锁了？

让我们举一个例子来说明:

![](https://a.a2k6.com/gerald/i/2024/04/08/ldd.png)

在这种情况如果没commit就释放锁，Session A 就会读到别的事务没提交的东西，而Session A的隔离级别还是RC，这样事务A的隔离级别也变成了RU。所以就算你是读未提交，也要等到commit之后再释放锁。

## 所以还有哪些锁

上面我们从一条update语句的时候，论证了意向锁的必要性，也就是说在为表加字段、索引的时候，需要锁定表，就需要看看当前表中，有没有哪行数据在被事务锁定更新，为了提升性能，引入了意向锁，这是一个表级锁。意向锁也可以分为两类:

- 意向共享锁(IS):  表示一个事务打算对表中单个行设置共享锁
- 意向独占锁(IX): 表示事务打算对表中的单个行设置排他锁

这里我们在回忆一下共享锁， 共享锁从字面上来理解就是允许多个事务来共享某个资源的锁，一般我们用S来表示

```mysql
#mysql 8.0之前的版本通过 lock in share mode给数据行添加share lock
select * from user where id = 1 lock in share mode;
#mysql 8.0以后的版本通过for share给数据行添加share lock
select * from user where id = 1 for share
```

排他锁只允许一个事务来持有某个资源的锁，用X来表示:

```mysql
# 通过for update可以给数据行加exclusive lock
select * from user where id = 1 for update;
# 通过update或delete同样也可以
update user set age = 16 where id = 1;
```

举个例子, 假如事务T1持有了某一行的共享锁(S),当事务T2也想获得该行的锁，分为如下两种情况:

- 如果T2申请的是共享锁，会被立刻允许，此时T1和T2同时持有该行的共享锁
- 如果T2申请的是排他锁(X)，那么必须等T1释放才能成功获取。

反过来说，如果T1持有该行的排他锁，那不管T2申请的是共享锁还是独占锁都必须等待T1释放之后才能持有。表级锁之间的兼容性如下图所示:

|      | X              | IX               | S                | IS               |
| ---- | -------------- | ---------------- | ---------------- | ---------------- |
| X    | Conflict(互斥) | Conflict(互斥)   | Conflict(互斥)   | Conflict(互斥)   |
| IX   | Conflict(互斥) | Compatible(兼容) | Compatible(兼容) | Compatible(兼容) |
| S    | Conflict(互斥) | Conflict(互斥)   | Compatible(兼容) | Compatible(兼容) |
| IS   | Conflict(互斥) | Compatible(兼容) | Compatible(兼容) | Compatible(兼容) |

- Before a transaction can acquire a shared lock on a row in a table, it must first acquire an `IS` lock or stronger on the table. 

  > 一个事务想要获取表中某一行共享锁的时候，它首先要获取意向共享锁或意向排他锁

- Before a transaction can acquire an exclusive lock on a row in a table, it must first acquire an `IX` lock on the table.

  > 一个事务想要获取表中某一行独占锁的时候，那它必须先取得对表的IX锁。

在行锁之上，有根据不同的作用点分为记录锁(Record Lock)、间隙锁(Gap Lock)、Next-Key Locks。值得注意的是，在可重复读取这个基础上，间隙锁和next-key 锁才会起作用。

```mysql
# 通过该指令来看事务的隔离级别
show variables like '%isolation%';
```

### 还有 Record Lock(记录锁)

记录锁是作用于索引记录，索引是一种数据结构，例如: 

```mysql
 SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE; 
```

 这条语句会阻止其他事务插入、更新或删除 t.c1 值为 10 的记录。示例:

![](https://a.a2k6.com/gerald/i/2024/04/08/vmlb.png)

### 还有 gap lock(间隙锁)

```mysql
SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;
```

上面的语句会阻止 防止其他事务向 t.c1 列插入值为 15 的内容，这也就是间隙锁，记录锁锁定一个记录，而间隙锁锁定一个区间。那么为什么要引入间隙锁呢，上面提到间隙锁只在可重复读的情况下发生作用，我们接着回忆一下我们上面讲的，如果我们要求事务排队执行，跟现实世界一样，那么就不会有任何问题，但是排队性能很弱，这无疑是我们不想看到的，为了追求性能，我们不得已要放松对事务隔离性的要求，而换取性能的提升，那么事务并发执行会有什么问题呢?

- 脏写(Dirty  Write), 如果一个事务修改了另一个未提交事务修改过的数据，那就意味着发生了脏写。如下图所示:

  ![](https://a.a2k6.com/gerald/i/2024/04/08/19xneq.png)

- 脏读（Dirty Read): 如果一个事务读到了另一个事务未提交事务修改过的数据，那就意味着发生了脏读。

​		![](https://a.a2k6.com/gerald/i/2024/04/08/1a0bzo.png)

- 不可重复读（Non-Repeatable Read) : 如果一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值，那就意味着发生了不可重复读。

  ![](https://a.a2k6.com/gerald/i/2024/04/08/6ky68.png)

- 幻读（Phantom):  如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来，那就意味着发生了幻读. 示意图如下:

​		![](https://a.a2k6.com/gerald/i/2024/04/08/xv3n.png)

这些问题都可以用一个视角来统一，就是如果我们让事务排队执行，互不干扰的情况，就不会出现这些问题，也就是隔离性的真义所在，每个事务在执行的时候都像是在独占。

因此也就引出了四种隔离级别:

- READ UNCOMMITTED 隔离级别, 可能发生脏读、不可重复读、幻读这些问题。
- READ COMMITTED  隔离级别下，可能发生不可重复读和幻读，但是不可以发生脏读问题
- REPEATABLE READ 隔离级别下，可能发生幻读问题，但是不可以发生脏读和不可重复读问题
- SERIALIZABLE  隔离级别下，各种问题都不可以发生。

但是不同的数据库厂商对SQL标准中规定的四种隔离级别支持不一样，比方说Oracle 就只支持RC和SERIALIZABLE  隔离级别，而MySQL虽然支持4种隔离级别，MySQL在RR这个隔离级别下是可以禁止幻读发生的。我们在《MySQL事务学习笔记(二) 相识篇》里面讨论了，MySQL采取的方式是一种实现方式是MVCC，所谓MVCC也就是 Multiversion Concurrency Control 多版本并发控制，核心要义在于，生成一个Read View，READ COMMITTED 每次查询都生成一个 Read View，REPEATABLE READ 第一次读的时候生成一个Read View。 

而间隙锁也是MySQL实现的禁止幻读的一种方式。

那像下面的语句呢?

```mysql
SELECT * FROM child WHERE id = 100;
```

如果id这一行上面有唯一索引，那么并不会触发间隙锁(这不包括搜索条件只包括多列唯一索引中的某些列的情况；在这种情况下，间隙锁定会发生)， 那如果没有唯一索引则会触发间隙锁，我们看下MySQL中对间隙锁的描述: 

> A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record. 
>
> 间隙锁是对索引记录之间的间隙的锁定，或者对第一条记录之前或最后一条索引记录之后的间隙的锁定。

那么问题来了，上面的语句是锁定区间(-oo,100)呢，还是(100,oo)呢，注意隔离级别要在RR，让我们首先来一个表:

```mysql
CREATE TABLE `student`  (
  `id` int(11) NOT NULL,
  `number` int(11) NULL DEFAULT NULL,
  `student_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = DYNAMIC;
```

然后插入两条数据:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (1, 4, '6666');
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (2, 8, '777');
```

然后我们开启一个窗口，执行下面语句:

```mysql
begin;
select * from student where number = 8 for update;
```

先不执行commit语句，然后执行一条插入语句:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (3, 6, '777');
```

然后会发现一直会被阻塞住，超时了就会报Lock wait timeout exceeded; try restarting transaction。

然后我们插入比number值比8大的，看看行不行:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (3, 6, '777');
```

所以锁住的区间是(-oo,8),(8,oo)?   带着这个疑问我执行了下面的SQL语句:

```mysql
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
```

![](https://a.a2k6.com/gerald/i/2024/04/09/rzib.png)

 这个supremum pseudo-record是什么意思，是另一外一种锁，当锁定的记录处于末尾的时候，就会创建supremum 锁，也就是说你的事务锁定记录没有大于你请求范围的现有记录。

我们插入几条数据, 再尝试锁定一个区间试试看，看看锁的情况如何:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (3, 11, '777');
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (4, 3, '777');
begin
select * from student where number between  4 and 8  for update;
# 这个语句在插入的时候才会有结果
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
```

![](https://a.a2k6.com/gerald/i/2024/04/09/5l2cp.png)

还是没发现间隙锁，我们向里面再插入一条记录试试看:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (15, 7, '17');
```

然后执行下面语句:

```mysql
begin
select * from student where number between  4 and 8  for update;
# 开启一个事务执行
begin
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (13, 5, '777');
```

![](https://a.a2k6.com/gerald/i/2024/04/09/43lo.png)

所以锁定范围里面有值触发的是间隙锁，没有值是supremum 锁。

###  还有next-key锁

其实MySQL会组合使用是Gap lock和Record Lock。Gap lock中的索引间隙是一个左开右开的区间，在next-key 锁中，变成左开右闭。也就是(4,8)变成(4,8]。我们现在表里面的数据是:

| id   | number | student_name |
| ---- | ------ | ------------ |
| 1    | 4      | 6666         |
| 3    | 11     | 777          |
| 4    | 2      | 777          |
| 11   | 9      | 777          |
| 15   | 7      | 17           |

```mysql
begin;
select * from student where number = 7 for update;
```

那么区间应该是(4,7]和[7,9),  我们插入一条语句试试看:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (16, 8, '17');
```

![](https://a.a2k6.com/gerald/i/2024/04/09/5p20e.png)

看来是因为7对应的id是最大的，导致触发suprem锁，我们用id为3这条记录试试看:

```mysql
begin;
select * from student where number = 9 for update;
```

 然后区间应该是(7,9], [9,11), 我们插入一条10试试看:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (16, 10, '17');
```

然后因为区间里面没有值接着触发了suprem锁。 由此就引出了插入意向锁，假设我们用间隙锁，锁定了一个区间，然后尝试往这个区间之外插值是可以的，像这个区间之内插值将会被阻塞：

```mysql
begin;
select  * from student where number between 2 and 7;
```

然后再执行一个事务执行下面的语句:

```mysql
INSERT INTO `student` (`id`, `number`, `student_name`) VALUES (18, 6, '17');
```

  插入语句会等到锁释放才能执行插入。

## 总结一下对事务的理解

总结一下，在MySQL中单纯执行两条update语句对一行数据进行更新，会排队执行，这样是为了避免事务并发执行带来的问题，我们希望事务在执行的时候就像是在排队执行一样，这也就是事务的隔离性，但是排队执行的性能我们无法接受，于是我们只能舍弃一点隔离性而换取隔离性的提升，放松对隔离性的要求，会带来脏写、脏读、不可重复读，幻读。 我们看出来这些操作有问题的一个预先设定就是我们希望事务看起来就像是排队执行，各自执行各自的，这也就是隔离性的最本质的所在。由这些问题我们引出了隔离级别: 

1. 读未提交    可能会发生脏读
2. 读已提交     可能发生不可重复读和幻读，但是不可以发生脏读问题
3. 可重复读    可能会发生幻读
4.  串行化   以上都不可能发生。

一般来说，隔离级别越高，吞吐量越低，但是不同的数据库厂商对SQL标准中规定的四种隔离级别支持不一样，比方说Oracle 就只支持RC和SERIALIZABLE  隔离级别，而MySQL虽然支持4种隔离级别，MySQL在RR这个隔离级别下是可以禁止幻读发生的。

那单纯的执行一条update语句会有哪些锁呢，我们可以想到数据库还会执行加字段之类的请求、加索引的请求， 在这种情况下无疑是要锁表的，那么在锁表之前无疑是看当前表是否有行锁，那如何看是否有行锁呢，遍历嘛，这太慢了，所以update语句除了上自己的行锁之外，还会上一个意向锁用于告知要锁表的请求。

在MySQL中将select语句分为快照读和当前读，也就是带for update的和不带for update的，快照读通过mvcc来实现隔离级别，而当前读则通过锁，锁又分为独占锁和共享锁，独占锁和共享锁互斥，独占锁我们用X表示，共享锁我们用S来表示。IS表示意向共享锁，IX表示意向独占锁。

在当前读之上，MySQL又引入了记录锁和间隙锁，看输出其实都是作用于索引，记录锁表示锁定的记录不能被更新、删除。间隙锁用于锁定一个区间，如果锁定的记录上面有唯一索引，此时是记录锁，如果没有MySQL会使用间隙锁扩大锁定范围，间隙锁我们可以理解为一个区间，如果锁定的那一行所在列没有索引或者不是唯一索引，会触发间隙锁，如果是表的最后一行记录，当前读的区间会扩大。

MySQL采用WAL，也就是预写日志来实现，也就是说所有的修改都先被写入到日志中，然后再被刷新到系统中。这是一种确保数据完整性的标准方法。redo 实现持久性，undo 实现原子性。

## 参考资料

[1] 《 Mysql锁：灵魂七拷问 》 https://tech.youzan.com/seven-questions-about-the-lock-of-mysql/

[2]  电商库存系统的防超卖和高并发扣减方案 | 京东云技术团队 

[3]  What is an isolated transaction in Java?  https://stackoverflow.com/questions/5556146/what-is-an-isolated-transaction-in-java

[4] Transaction Isolation Levels and why we should  care https://www.metisdata.io/blog/transaction-isolation-levels-and-why-we-should-care

[5] ACID Explained: Atomic, Consistent, Isolated & Durable  https://www.bmc.com/blogs/acid-atomic-consistent-isolated-durable/

[6] MYSQL 事务的底层原理 | 京东物流技术团队  https://juejin.cn/post/7300855272913240073#heading-0

[7] Write-Ahead Logging (WAL)   https://www.postgresql.org/docs/current/wal-intro.html 

[8] 理解Mysql InnoDB引擎中的锁 	https://wiyi.org/mysql-innodb-locking.html#gap-locks
