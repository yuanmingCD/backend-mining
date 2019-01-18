# mysql 事务



[TOC]



## 什么是事务

通俗来说，就是一组sql作为一个原子查询操作，要么全都执行，要么全都不执行（失败的时候回滚到原来的状态）

本文主要介绍单机事务

## 事务的特性

事务的四个特性，ACID是Atomic（原子性）、Consistency（一致性）、Isolation（隔离性）和Durability（持久性）。

- 原子性：指整个数据库事务是不可分割的工作单位。只有使据库中所有的操作执行成功，才算整个事务成功；事务中任何一个SQL语句执行失败，那么已经执行成功的SQL语句也必须撤销，数据库状态应该退回到执行事务前的状态。
- 一致性：指数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。例如对银行转帐事务，不管事务成功还是失败，应该保证事务结束后ACCOUNTS表中Tom和Jack的存款总额为2000元。
- 隔离性：指的是在并发环境中，当不同的事务同时操纵相同的数据时，每个事务都有各自的完整数据空间。由并发事务所做的修改必须与任何其他并发事务所做的修改隔离。事务查看数据更新时，数据所处的状态要么是另一事务修改它之前的状态，要么是另一事务修改它之后的状态，**事务不会查看到中间状态的数据**。
- 持久性：指的是只要事务成功结束，它对数据库所做的更新就必须永久保存下来。即使发生系统崩溃，重新启动数据库系统后，数据库还能恢复到事务成功结束时的状态。

## 事务隔离级别

### `Read Uncommited(RU)`：读未提交

一个事务可以读到另一个事务未提交的数据！

### `Read Committed (RC)`：读已提交，

保证一个事务读数据的时候其它事务都是已经提交的。

### `Repeatable Read (RR)`:可重复读

在RC基础上，保证所有的数据读操作不能被修改，如果事务再次读到相同的数据，将

### `Serializable`：串行化

该级别下读写串行化，且所有的`select`语句后都自动加上`lock in share mode`，即使用了共享锁。因此在该隔离级别下，使用的是当前读，而不是快照读。事务之间没有交叉



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

更新覆盖

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

脏读本质上是读到了未提交的事务，因而可能在的RU（读未提交）隔离级别中出现



### 不可重复读：

一个事务中读取的数据可能发生变化

| 步骤 | 事务1                                   | 事务2                                              |
| ---- | --------------------------------------- | -------------------------------------------------- |
| 1    | BEGIN;                                  |                                                    |
| 2    | SELECT * FROM account WHERE user_id =1; |                                                    |
|      |                                         | BEGIN;                                             |
|      |                                         | UPDATE account SET balance = 100 where user_id =1; |
|      |                                         | COMMIT;                                            |
|      | SELECT * FROM account WHERE user_id =1; |                                                    |
|      | COMMIT;                                 |                                                    |



事务1两次的读取的结果不一致

事务1读到事务2已经提交的事务，



可能发生在RR和RU级别，两个事务为嵌套



脏读和不可重复读的区别

对应脏读的例子中，事务1第二次读操作发生的是脏读，而第三次操作发生的是不可重复读



### 幻读

| 步骤 | 事务1                                                  | 事务2                                                        |
| ---- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | BEGIN;                                                 |                                                              |
| 2    | UPDATE  account SET balance = 100 WHERE balance > 100; |                                                              |
| 3    |                                                        | BEGIN;                                                       |
|      |                                                        | INSERT INTO account (user_id,card,balance) VALUES(1,"1",110) |
|      |                                                        | COMMIT;                                                      |
| 6    | SELECT * FROM account WHERE balance > 100;             |                                                              |
| 7    | COMMIT;                                                |                                                              |
|      |                                                        |                                                              |

- 步骤2 事务1更新了账户余额大于100的记录
- 步骤3 事务2此时又插入一条这个范围内的一条记录
- 步骤6 事务1再去查询这个范围内的记录发现居然刚才有一条数据没有更新上



不可重复读与幻读异同

幻读主要针对插入或者删除范围操作

不可重复读可以通过记录加锁解决，而幻读不可以，而是要在记录之间也要加间隙锁





总结



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

　事务的（ACID）特性是由关系数据库管理系统（RDBMS，数据库系统）来实现的。数据库管理系统采**用日志来保证事务的原子性、一致性和持久性**。日志记录了事务对数据库所做的更新，如果某个事务在执行过程中发生错误，就可以根据日志，撤销事务对数据库已做的更新，使数据库退回到执行事务前的初始状态。

mysql 通过undo log实现事务

undo log