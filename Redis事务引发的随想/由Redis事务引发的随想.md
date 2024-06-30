# 由Redis事务引发的随想

## 前言

一般我们说起的事务，默认是数据库中的事务，数据库中的事务满足ACID四个特性:

- 原子性(**Atomicity**):  对于不可分割的操作，要么成功要么失败。
- 一致性(**Consistency**):  现实世界的一些约束到了软件世界也要予以保持，比如说人民币的最大币值等。如果数据库中的数据全部符合现实世界的约束，我们就说这些数据就是符合一致性的。
- 隔离性(**Isolation**):  对于现实世界的状态转换对应到某些数据库操作来说，不仅要保证这些操作以原子性的方式来执行完成，而且要保证其它的状态转换不会影响到本次状态转换，这个规则被称之为隔离性。

> 关于隔离性我的解读是事务在并发执行的时候，要有就像排队执行一样的效果

- 持久性(**Durability**): 当现实世界的一个状态完成后，这个转换的结果将永久保留，这个规则我们称之为持久性。当把现实世界的状态转换映射到数据库世界，持久性意味着转换对应的数据库操作所修改的数据都应该在磁盘上保留下来。

我们回忆一下Redis的事务:

> Redis Transactions allow the execution of a group of commands in a single step, they are centered around the commands  MULTI, EXEC, DISCARD and  WATCH. Redis Transactions make two important guarantees

Redis的事务允许在单个步骤中执行一组命令，它们主要围绕 MULTI、EXEC、DISCARD 和 WATCH 命令。

Redis 事务提供两个重要保证: 

- 事务中的所有命令都是串行化的，并按顺序执行。在事务的执行过程中，其他客户端发送的请求永远不会被服务。这保证了命令作为单个独立操作执行。
- EXEC 命令触发事务中所有的命令执行，因此如果客户端在调用EXEC命令之前在事务的上下文中失去了与服务器的连接，则不会执行任何操作。相反，如果调用了EXEC命令，则会执行所有的操作。当Redis使用AOF(append-only file )持久化机制的时候，Redis 会确保使用单个 write(2) 系统调用将事务写入磁盘。

 However if the Redis server crashes or is killed by the system administrator in some hard way it is possible that only a partial number of operations are registered。

但是如果 Redis  服务崩溃或 

## 





















