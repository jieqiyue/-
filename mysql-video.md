

**undo log  ======>  原子性**

**redo log  =======> 持久性  =======> 使用redo log 提高性能**

Undo 记录某 **数据** 被修改 **前** 的值，可以用来在事务失败时进行 rollback；
 Redo 记录某 **数据块** 被修改 **后** 的值，可以用来恢复未写入 data file 的已成功事务更新的数据。
 即，

- Redo Log 保证事务的持久性
- Undo Log 保证事务的原子性（在 InnoDB 引擎中，还用 Undo Log 来实现 MVCC）

比如某一时刻数据库 DOWN 机了，有两个事务，一个事务已经提交，另一个事务正在处理。数据库重启的时候就要根据日志进行前滚及回滚，把已提交事务的更改写到数据文件，未提交事务的更改恢复到事务开始前的状态。即，当数据 crash-recovery 时，通过 redo log 将所有已经在存储引擎内部提交的事务应用 redo log 恢复，所有已经 prepared 但是没有 commit 的 transactions 将会应用 undo log 做 roll back。

redo/undo log 和 binlog

两者区别还是挺多的，大致如下，

- 层次不同。redo/undo 是 innodb 引擎层维护的，而 binlog 是 mysql server 层维护的，跟采用何种引擎没有关系，记录的是所有引擎的更新操作的日志记录。
- 记录内容不同。redo/undo 记录的是 每个页/每个数据 的修改情况，属于物理日志+逻辑日志结合的方式（redo log 是物理日志，undo log 是逻辑日志）。binlog 记录的都是事务操作内容，binlog 有三种模式：Statement（基于 SQL 语句的复制）、Row（基于行的复制） 以及 Mixed（混合模式）。不管采用的是什么模式，当然格式是二进制的，
- 记录时机不同。redo/undo 在 **事务执行过程中** 会不断的写入，而 binlog 是在 **事务最终提交前** 写入的。binlog 什么时候刷新到磁盘跟参数 `sync_binlog` 相关。



1. 可重复读（隔离级别中的一种）
   可重复读是指：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。

   ![2](https://yqfile.alicdn.com/b9222803629d82f3152b2544982ca5220b122c73.jpeg)

   

2. 幻读

   间隙锁

3. 快照读

   快照读指的是读取事务开始的时候的那份数据。

4. 当前读

   以数据库中最新的数据读取。比如说加了for update，或者是增删改的操作。

   select * from table where ? lock in share mode;
   select * from table where ? for update;
   insert into table values (…);
   update table set ? where ?;
   delete from table where ?;

5. MVCC

   通过增加一个版本号来控制。

6. 行锁和表锁

   在非唯一索引列的更新会使用表锁。如果以主键索引作为where条件会是行锁。两阶段锁，在需要时加锁，并不会在语句结束时释放锁，会在事务提交的时候去释放锁。

   ![3](https://yqfile.alicdn.com/3e6efccdbb340d9ddc07ec61aa175fa92eaf4007.jpeg)

