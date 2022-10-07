# MySQL事务学习笔记(一) 初遇篇

[TOC]

## 如何开启事务

在《数据库事务概论》中我们已经讨论了事务的基本概念、性质，那么在MySQL中如何将一系列操作当做一个事务呢。像下面这样: 

```mysql
BEGIN;  //代表开启事务

UPDATE  Student SET name = '张三三' WHERE id = '1';

COMMIT; // 代表将BEGIN和COMMIT之间的语句进行提交,也就是请求MySQL将结果刷新到磁盘上
```

如果只有BEGIN，没有COMMIT，那么代表BEGIN下面对数据的操作还未刷到磁盘上, 代表该事务处于部分提交的状态，在事务隔离级别为不可重读的情况下，其他Session 无法读到了这个更新，通过:

```mysql
 SHOW VARIABLES LIKE 'transaction_isolation';
```

可以查看MySQL中事务默认的隔离级别:

![MySQL事务的默认隔离级别](https://tva4.sinaimg.cn/large/006e5UvNly1gzjt5jeaqcj30ag03vjso.jpg)

可以看到当前我的数据库的隔离级别是可重复读，在这种情况下其他Session 是无法读到这个处于未提交状态的事务的更新的，我们来验证一下: 

![开启事务还未提交](https://tva4.sinaimg.cn/large/006e5UvNly1gzjtaolgozj30iq05uwg1.jpg)

现在我们再开启一个Session，也就是再开一个命令列界面，看能不能查询到上图对应Session的更新。

![不可重复读示例](https://tva1.sinaimg.cn/large/006e5UvNly1gzjtci9c1cj30di040jsc.jpg)

可以看到name还是张三三，现在我们执行Commit指令，提交事务，等价于请求MySQL将事务所做的改变刷新到磁盘上。然后看下另一个Session是否能查询到。

![提交事务](https://tvax1.sinaimg.cn/large/006e5UvNly1gzjtigfy83j30hg056wga.jpg)

![查询到了更新](https://tva3.sinaimg.cn/large/006e5UvNly1gzjtinkohoj30he07b40x.jpg)

## 自动提交

可能有同学会问，我平时直接执行单条UPDATE语句，在别的Session 中直接能查询到，那是不是不加UPDATE语句就不算事务了啊，那要不然为啥我执行了UPDATE，其他Session 立刻能查询到呢? 像下面这样: 

![隐式提交-1](https://tva1.sinaimg.cn/large/006e5UvNly1gzjtrg05rhj30iy03cgmt.jpg)

![隐式提交-2](https://tvax1.sinaimg.cn/large/006e5UvNly1gzjtrxqmaij30hx04a75a.jpg)

如果我们不显式的使用BEGIN或START TRANSACTION(比BEGIN更全面)来开启一个事务，那么每个DML语句(INSERT,UPDATE,DELETE )都算是一个独立的事务，自动提交，这种特性我们称之为事务的自动提交，在MySQL中可以通过:

```mysql
SHOW VARIABLES LIKE 'autocommit';
```

查看该特性是否开启：

![自动提交](https://tvax3.sinaimg.cn/large/006e5UvNly1gzju5hfr70j30df04a75h.jpg)

## 隐式提交

如果我们不想用自动提交可以用 BGEIN 和 COMMIT语句来手动提交，或者将autocommit关闭，但是即使我们使用了BGEIN，也不能百分之百的保证MySQL不会自动提交，在以下情况下:

- 如果BEGIN后跟了DDL语句(定义或修改数据库对象的数据定义语言 ,Data definition language )
- 隐式使用或修改mysql(这个是MySQL自带的数据库，就叫mysql)数据库中的表
- 事务控制或关于锁定的语句(BEGIN后面又来了个 BEGIN或者 START TRANSACTION)
- 或者在当前Session中打开autocommit的开关
- 或者使用LOCK TABLE、UNLOCK TABLES等关于锁定的语句也会隐式的提交前边语句所属的事务
- 加载数据的语句, 比如我们使用LOAD DATA语句来批量往数据库中带入数据时，也会隐式的提交前面语句所属的事务
- MySQL复制的一些语句，使用START  SLAVE，STOP SLAVE 、RESET SLAVE等语句时也会触发隐式提交
- ANALYZE TABLE [表名]、CACHE INDEX、CHECK TABLE [表名] 等语句也会触发前边语句所属的事务。

## 手动中止事务

那如果发生了意外，我想回滚或者手动中止事务呢，我们可以借助RollBack指令，像下面这样:

![ROLLBACK指令执行](https://tva1.sinaimg.cn/large/006e5UvNly1gzjuqofszrj30fv08dq66.jpg)

这里需要强调的是，ROLLBACK语句是MySQL用来需要手动的回滚事务时提供的语句，如果MySQL的事务在执行过程中遇到了某些错误而无法继续执行的时候，事务会自动的进行回滚。

## 保存点

如果我们在Begin之后，执行了好多条UPDATE 语句呢，我们只想回滚到某个点，再接着执行呢，请保存点同志出场说话，在事务对应的语句上当上几个点，调用ROLLBACK语句的时候，我们就能回滚到我们想回滚的点了。示例如下:

![保存点回滚演示](https://tva1.sinaimg.cn/large/006e5UvNly1gzjv5h3t33j30js0f9grw.jpg)

 语法示例:

```mysql
SAVEPOINT 保存点名称  // 产生保存点
RELEASE SAVEPOINT 保存点名称 // 删除保存点
```

## 另一种开启事务的方式

上面我们都是采取BEGIN语句来开启事务的，MySQL中还有另一种开启事务的语法:

```mysql
START TRANSACTION [修饰符],[修饰符],[修饰符];
```

 START TRANSACTION相对于BEGIN语句的不同之处就在于，START TRANCATION可以跟修饰符, 如下所示:

- READ ONLY (只读模式)

> 在该模式下只允许读数据，不允许写永久表的数据，可以写临时表的数据，临时表只存在于当前会话。

示例: 

![只读模式](https://tva1.sinaimg.cn/large/006e5UvNly1gzjvgwr1f7j30fm040q48.jpg)

-  WRITE ONLY 只写模式 
- WITH CONSISTENT SNAPSHOT 一致性读。

一个事务的模式是不能即使只读模式，又是只写模式，我们不加修改默认就是既允许读又允许读。