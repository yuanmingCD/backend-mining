# mysql-数据库锁

[TOC]



## 锁分类

### 按类型划分

#### 排它锁

Exclusive Lock，也叫X锁，或者写锁、独占锁，大多为update,insert,delete以及select ... for update操作

#### 共享锁

Shared Lock 也叫S锁，或者读锁，select ... lock in share mode

通常加锁可以通过一个表格描述两种锁关系

|      | S      | X    |
| ---- | ------ | ---- |
| S    | 不冲突 | 冲突 |
| X    | 冲突   | 冲突 |

具体解释如下：

- 若数据已加写锁，另外一个写操作到来不能获取锁；另外一个读操作到来不能获取到锁；
- 若数据已加读锁，另外一个写操作到来不能获取锁；另外一个读操作到来可以获取到锁；

普通的select操作不加锁

意向锁

意向锁是表锁的一种，主要作用是处理行锁和表锁之间的矛盾。首先要明确的是，当表中存在行级写锁时，是不能加表级写锁的，这两个级别是互斥的。当一个事务想要去申请表锁时，需要遍历表中数据判断是否存在行锁，这个开销非常大。因而在表级锁添加标识，告知是否有行级锁，这个标识就是意向锁。



### 按粒度划分

表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高,并发度最低。

行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低,并发度也最高。

页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。

#### 表锁

Innodb和MyIsam均支持表锁，这里先主要介绍Innodb表锁

##### 表锁操作

- 加读锁：lock table tableName read;
- 加写锁：lock table tableName write;
- 批量解锁：unlock tables;

##### 查看加锁

``` sql
show open tables; 
```

结果解释

- 1 表示加锁，
- 0 表示未加锁。

##### 分析表锁定

```sql
show status like 'table_locks%'
```

结果解释

- table_locks_immediate：立即释放表锁数
- table_locks_waited：需要等待的表锁数，此值越高则说明存在着越严重的表级锁争用情况

##### 使用场景

InnoDB默认采用行锁，在未使用索引字段查询时升级为表锁

- **全表更新**。事务需要更新大部分或全部数据，且表又比较大。若使用行锁，会导致事务执行效率低，从而可能造成其他事务长时间锁等待和更多的锁冲突。
- **多表查询**。事务涉及多个表，比较复杂的关联查询，很可能引起死锁，造成大量事务回滚。这种情况若能一次性锁定事务涉及的表，从而可以避免死锁、减少数据库因事务回滚带来的开销。

#### 页锁

#### 行锁

只有Innodb才有行级锁，行级锁针对索引加锁，而非数据行加锁。



通过检查InnoDB_row_lock 状态变量分析系统上的行锁的争夺情况， show status like 'innodb_row_lock%'



如果索引失效，将会从行级锁升级为表级锁

##### 行锁分析

```sql
show status like 'innodb_row_lock%';
```

结果解释：

- innodb_row_lock_current_waits: 当前正在等待锁定的数量
- innodb_row_lock_time: 从系统启动到现在锁定总时间长度；非常重要的参数，
- innodb_row_lock_time_avg: 每次等待所花平均时间；非常重要的参数，
- innodb_row_lock_time_max: 从系统启动到现在等待最常的一次所花的时间；
- innodb_row_lock_waits: 系统启动后到现在总共等待的次数；非常重要的参数。直接决定优化的方向和策略。

show status like 'innodb_row_lock%';



间隙锁（next-key锁）

当我们用范围条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做"间隙(GAP)"。InnoDB也会对这个"间隙"加锁，这种锁机制就是所谓的间隙锁(Next-Key锁)。间隙锁可以有效防止幻读

- 

MySQL InnoDB支持三种行锁定方式：**InnoDB的默认加锁方式是next-key 锁。**

l   记录锁（Record Lock）:锁直接加在索引记录上面，锁住的是key。

l   间隙锁（Gap Lock）:锁定索引记录间隙，确保索引记录的间隙不变。间隙锁是针对事务隔离级别为可重复读或以上级别而已的。

l   邻键锁（Next-Key Lock ）：行锁和间隙锁组合起来就叫Next-Key Lock。 





| 等值查询所用的索引类型 | S      | X    |
| ---------------------- | ------ | ---- |
| S                      | 不冲突 | 冲突 |
| X                      | 冲突   | 冲突 |



`Next-Key Locks`：这个理解为`Record Lock`+索引前面的`Gap Lock`。锁住的是索引前面的间隙！比如一个索引包含值，10，11，13和20。那么，间隙锁的范围如下



只有在隔离级别为`Repeatable Read`和`Serializable`时，才会存在间隙锁。



意向锁

为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB存储引擎支持一种额外的锁方式，就称为**意向锁**，意向锁在 InnoDB 中是**表级锁**，按照锁类型分为意向共享锁（Intention Shard Lock，IS）和意向排它锁（Intention Excusive Lock，IX）

- 意向共享锁，在一个事务获取S锁之前，先在所在的表上加IS锁
- 意向排它锁，在一个事务获取X锁之前，先在所在的表上加IX锁



假设事务T1，用X锁来锁住了表上的几条记录，那么此时表上存在IX锁。那么此时事务T2要进行`LOCK TABLE … WRITE`的表级别锁的请求，可以直接根据意向锁是否存在而判断是否有锁冲突。

意向锁可以有效减少锁冲突



何时加锁，加什么样的锁









redo log 和binlog的一致性，为了防止写完binlog但是redo log的事务还没提交导致的不一致，innodb 使用了两阶段提交
大致执行序列为









# MVCC

MVCC（Multiversion concurrency control ，多版本并发控制）

读操作分为快照读和当前读

- **快照读：**简单的select操作，属于快照读，不加锁。(当然，也有例外，下面会分析)
  - select * from table where ?; 
- **当前读：**特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。
  - select * from table where ? lock in share mode;
  - select * from table where ? for update;
  - insert into table values (…);
  - update table set ? where ?;
  - delete from table where ?;





**对于快照读来说，幻读的解决是依赖mvcc解决。而对于当前读则依赖于gap-lock解决。**

MVCC的最大好处：读不加任何锁，读写不冲突，对于读操作多于写操作的应用，极大的增加了系统的并发性能；



**MVCC只是存在于两种事务级别下：Read Committed 与 Repeatable Read;**

因为：

(c)READ UNCOMMITTED 总是读取最新的数据，不符合当前事务版本的数据行，

(d)Serializable则会对所有的行加锁。





REPEATABLE READ隔离级别下普通的读操作即select都不加锁，使用MVCC进行一致性读取，这种读取又叫做snapshot read。
而update, insert, delete, select … for update, select … lock in share mode都会进行加锁，并且读取的是当前版本，也就是READ COMMITTED读的效果。[innodb-locks-set.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)中对各种操作会进行的锁操作有详细的说明，这里我简单总结下。
InnoDB中加锁的方法是锁住对应的索引，一个操作进行前会选择一个索引进行扫描，扫描到一行后加上对应的锁然后返回给上层然后继续扫描。InnoDB支持行级锁(record lock)，上述需要加锁的操作中，除了select … lock in share mode 是加shared lock（共享锁或读锁）外其他操作都加的是exclusive lock(即排他锁或写锁)。在加行级锁前，会对表加一个intention lock，即意向锁，意向所是表级锁，不会和行级锁冲突，主要用途是表明一个要加行级锁或正在加锁的操作。
另外InnoDB种除了record lock外还有一种gap lock，即锁住两个记录间的间隙，防止其他事务插入数据，用于防止幻读。当索引是主键索引或唯一索引时，不需要加gap lock。当索引不是唯一索引时，需要对索引数据和索引前的gap加锁，这种方式叫做next-key locking。
另外在插入数据时，还需要提前最插入行的前面部分加上insert intention lock, 即插入意向锁，插入意向锁之间不会冲突，会和gap锁冲突导致等待。当插入时遇到duplicated key错误时，会在要插入的行上加上share lock。





MySQL Innodb中存在多种日志，除了错误日志、查询日志外，还有很多和数据持久性、一致性有关的日志。
binlog，是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的, 另外通过解析binlog能够实现mysql到其他数据源（如ElasticSearch)的数据复制。
redo log记录了数据操作在物理层面的修改，mysql中使用了大量缓存，缓存存在于内存中，修改操作时会直接修改内存，而不是立刻修改磁盘，当内存和磁盘的数据不一致时，称内存中的数据为脏页(dirty page)。为了保证数据的安全性，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中, redo log是按照顺序写入的，磁盘的顺序读写的速度远大于随机读写。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。这样实现了事务的原子性、一致性和持久性。

Undo Log: 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC。

InnoDB的多版本并不是直接存储多个版本的数据，而是所有更改操作利用行锁做并发控制，这样对某一行的更新操作是串行化的，然后用Undo log记录串行化的结果。当快照读的时候，利用Undo log重建需要读取版本的数据，从而实现读写并发。



INNODB的MVCC通常是通过在每行数据后边保存两个隐藏的列来实现(其实是三列，第三列是用于事务回滚，此处略去)，

一个保存了行的创建版本号，另一个保存了行的更新版本号（上一次被更新数据的版本号）
这个版本号是每个事务的版本号，递增的。



这样保证了innodb对读操作不需要加锁也能保证正确读取数据。









参考文献

[MySQL 加锁处理分析](http://hedengcheng.com/?p=771#_Toc374698317)

[MySQL 表锁和行锁机制](https://juejin.im/entry/5a55c7976fb9a01cba42786f)

[select 加锁分析](https://www.cnblogs.com/rjzheng/p/9950951.html)

[Mysql 间隙锁原理，以及Repeatable Read隔离级别下可以防止幻读原理(百度)](https://www.cnblogs.com/aspirant/p/9177978.html)

[MySQL InnoDB锁机制全面解析分享](https://segmentfault.com/a/1190000014133576)

https://www.cnblogs.com/xiaoboluo768/p/7615823.html



https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html