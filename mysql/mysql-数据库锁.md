# mysql-数据库锁





## 按类型划分

### 排它锁

Exclusive Lock，也叫X锁，或者写锁、独占锁，大多为update,insert,delete操作

### 共享锁

Shared Lock 也叫S锁，或者读锁，大多为select操作

通常加锁可以通过一个表格描述两种锁关系

|      | S      | X    |
| ---- | ------ | ---- |
| S    | 不冲突 | 冲突 |
| X    | 冲突   | 冲突 |

具体解释如下：

- 若数据已加写锁，另外一个写操作到来不能获取锁；另外一个读操作到来不能获取到锁；
- 若数据已加读锁，另外一个写操作到来不能获取锁；另外一个读操作到来可以获取到锁；

## 按粒度划分

表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高,并发度最低。

行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低,并发度也最高。

页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。

### 表锁

Innodb和MyIsam均支持表锁，这里先主要介绍Innodb表锁

#### 表锁操作

- 加读锁：lock table tableName read;
- 加写锁：lock table tableName write;
- 批量解锁：unlock tables;

#### 查看加锁

``` sql
show open tables; 
```

结果解释

- 1表示加锁，
- 0表示未加锁。

#### 分析表锁定

```sql
show status like 'table_locks%'
```

结果解释

- table_locks_immediate：立即释放表锁数
- table_locks_waited：需要等待的表锁数，此值越高则说明存在着越严重的表级锁争用情况

#### 使用场景

InnoDB默认采用行锁，在未使用索引字段查询时升级为表锁

- **全表更新**。事务需要更新大部分或全部数据，且表又比较大。若使用行锁，会导致事务执行效率低，从而可能造成其他事务长时间锁等待和更多的锁冲突。
- **多表查询**。事务涉及多个表，比较复杂的关联查询，很可能引起死锁，造成大量事务回滚。这种情况若能一次性锁定事务涉及的表，从而可以避免死锁、减少数据库因事务回滚带来的开销。

### 页锁



### 行锁

只有Innodb才有行级锁，行级锁针对索引加锁，而非数据行加锁。



通过检查InnoDB_row_lock 状态变量分析系统上的行锁的争夺情况， show status like 'innodb_row_lock%'



如果索引失效，将会从行级锁升级为表级锁

#### 行锁分析

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

当我们用范围条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做"间隙(GAP)"。InnoDB也会对这个"间隙"加锁，这种锁机制就是所谓的间隙锁(Next-Key锁)。



`Next-Key Locks`：这个理解为`Record Lock`+索引前面的`Gap Lock`。锁住的是索引前面的间隙！比如一个索引包含值，10，11，13和20。那么，间隙锁的范围如下



只有在隔离级别为`Repeatable Read`和`Serializable`时，才会存在间隙锁。

意向锁

按照锁类型分为意向共享锁（Intention Shard Lock，IS）和意向排它锁（Intention Excusive Lock，IX）

- 意向共享锁，在一个事务获取S锁之前，先在所在的表上加IS锁
- 意向排它锁，在一个事务获取X锁之前，先在所在的表上加IX锁



假设事务T1，用X锁来锁住了表上的几条记录，那么此时表上存在IX锁。那么此时事务T2要进行`LOCK TABLE … WRITE`的表级别锁的请求，可以直接根据意向锁是否存在而判断是否有锁冲突。

意向锁可以有效减少锁冲突



何时加锁，加什么样的锁

# MVCC



参考文献

[MySQL 加锁处理分析](http://hedengcheng.com/?p=771#_Toc374698317)

[MySQL 表锁和行锁机制](https://juejin.im/entry/5a55c7976fb9a01cba42786f)

[select 加锁分析](https://www.cnblogs.com/rjzheng/p/9950951.html)