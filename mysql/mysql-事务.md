# mysql 事务



[TOC]



## 什么是事务

通俗来说，数据库事务就是一组sql作为一个原子查询操作，要么全都执行，要么全都不执行（失败的时候回滚到原来的状态），本文主要介绍单机事务。

## 事务的特性

事务的四个特性ACID，即Atomic（原子性）、Consistency（一致性）、Isolation（隔离性）和Durability（持久性）。

- 原子性：只有使据库中所有的操作执行成功，才算整个事务成功；事务中任何一个SQL语句执行失败，数据库应该退回到执行事务前的状态。
- 一致性：保证关系数据的完整性以及业务逻辑上的一致性。
- 隔离性：当不同的事务同时操纵相同的数据时，每个事务都有各自的完整数据空间。由并发事务所做的修改必须与任何其他并发事务所做的修改隔离。
- 持久性：一旦事务成功提交，它对数据库所做的更新就必须永久保存下来。即使发生系统崩溃，重新启动数据库系统后，数据库还能恢复到事务成功结束时的状态。Innodb通过redo log来实现数据持久化特性。

## 事务隔离级别

### 读未提交

`Read Uncommited(RU)`，一个事务可以读到另一个事务未提交的数据，因此会出现脏读的问题。

### 读已提交

`Read Committed (RC)`，一个事务读到的都是其他事务已经提交的数据，但是多次读取可能会出现数据不一致的问题，比如这条数据被另外一个事务修改了，导致两次读取不一致，因此有不可重复读的问题。

### 可重复读

`Repeatable Read (RR)`，一个事务多次读取同一条数据，会看到同样的数据行。是MySQL的默认事务隔离级别，

### 串行化

`Serializable`，该级别下读写串行化，且所有的`select`语句后都自动加上`lock in share mode`，即使用了共享锁。因此在该隔离级别下，使用的是当前读，而不是快照读。事务之间没有交叉

它通过强制事务排序，使之不可能相互冲突，假如两个事务都操作到同一数据行，那么这个数据行就会被锁定，只允许先读取/操作到数据行的事务优先操作，只有当事务提交了，数据行才会解锁，后一个事务才能成功操作这个数据行，否则只能一直等待,该隔离级别可能导致大量的超时现象和锁竞争

## 脏读、不可重复读和幻读

我们以一个账户余额表展示一下具体细节，（关于表结构和操作逻辑可以不必在意）

``` sql
CREATE TABLE account (
  id int(11) unsigned NOT NULL AUTO_INCREMENT,
  user_id int(11) NOT NULL,
  card_id varchar(100) NOT NULL COMMENT '卡ID',
  balance int(11) NOT NULL COMMENT '余额',
  update_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (id),
  KEY idx_user_id (user_id),
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 脏读：

无效的数据读出，一事务未提交的中间状态的更新数据 被其他会话读取到。

| 步骤 | 事务1                                   | 事务2                                              |
| ---- | --------------------------------------- | -------------------------------------------------- |
| 1    | BEGIN;                                  |                                                    |
| 2    | SELECT * FROM account WHERE user_id =1; |                                                    |
|      |                                         | BEGIN;                                             |
|      |                                         | UPDATE account SET balance = 100 where user_id =1; |
|      | SELECT * FROM account WHERE user_id =1; |                                                    |
|      |                                         | ROLLBACK;                                          |
|      | SELECT * FROM account WHERE user_id =1; |                                                    |
|      | COMMIT;                                 |                                                    |

事务1第二次读取读到了事务2还没有提交的数据，而事务2回滚后会导致这次读取到的数据是无效的，如果这是个操作是UPDATE account SET balance = balance -1000；操作将导致非常严重的错误，

脏读本质上是读到了未提交的事务，因而可能在的RU（读未提交）隔离级别中出现。

**如何解决脏读呢？**

- RU级别下，对写操作加排它锁，直到提交或者回滚之后才进行释放，这样就不会其他事务就不会读到未提交的事务，
- 隔离级别提升到RC。

### 不可重复读：

一个事务中读取的数据可能发生变化

**示例**

| 步骤 | 事务1                                   | 事务2                                              |
| ---- | --------------------------------------- | -------------------------------------------------- |
| 1    | BEGIN;                                  |                                                    |
| 2    | SELECT * FROM account WHERE user_id =1; |                                                    |
|      |                                         | BEGIN;                                             |
|      |                                         | UPDATE account SET balance = 100 where user_id =1; |
|      |                                         | COMMIT;                                            |
|      | SELECT * FROM account WHERE user_id =1; |                                                    |
|      | COMMIT;                                 |                                                    |

事务1两次的读取的结果不一致，原因是事务1读到事务2已经提交的事务，可能发生在RR和RU级别，两个事务为嵌套关系

**脏读和不可重复读的区别**

- 对应脏读的例子中，事务1第二次读操作发生的是脏读，而第三次操作发生的是不可重复读
- 脏读时，两个事务为交叉关系，不可重复读为嵌套关系

**如何解决不可重复读？**

- RC级别下，对读操作加共享锁，这样其它事务只能读而不能修改，保证可重复读
- 将隔离级别提升为RR

### 幻读

| 步骤 | 事务1                                       | 事务2                                                        |
| ---- | ------------------------------------------- | ------------------------------------------------------------ |
| 1    | BEGIN;                                      |                                                              |
| 2    | SELECT * FROM  account WHERE balance > 100; |                                                              |
| 3    |                                             | BEGIN;                                                       |
|      |                                             | INSERT INTO account (user_id,card,balance) VALUES(1,"1",110) |
|      |                                             | COMMIT;                                                      |
| 6    | SELECT * FROM account WHERE balance > 100;  |                                                              |
| 7    | COMMIT;                                     |                                                              |
|      |                                             |                                                              |

- 步骤2 事务1读取了账户余额大于100的记录，数量记为n
- 步骤3 事务2此时又插入一条这个范围内的一条记录
- 步骤6 事务1再去查询这个范围内的记录发现有n+1条记录



**不可重复读与幻读异同**

- 不可重复读发生在对记录的更新操作，幻读主要发生在插入或者删除操作
- ，而幻读不可以，还需要记录之间也要加间隙锁，间隙锁会锁住访问数据边界

**如何解决不可重复读？**

- RR级别下通过间隙锁解决
- 设置为`Serializable`级别

## 事务隔离级别查看与设置

```sql
--查看系统隔离级别：
select @@global.tx_isolation;
--查看当前会话隔离级别
select @@tx_isolation;
--设置当前会话隔离级别
SET session TRANSACTION ISOLATION LEVEL serializable;
--设置全局系统隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

## 事务的实现

　因为事务需要在失败的时候进行回滚，的通过记录undo日志来实现
