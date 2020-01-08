





[TOC]





## join

### join选型

![SQL JOINS](images/SQL JOINS.png)

- outer join
  - left join 左表全部符合条件的数据，即便对应右表数据为空
  - Right join
  - full outer join
- inner join 

笛卡尔积，交集



连接表还是子查询



### on 还是 where 

在使用left join时，on和where条件的区别如下：  
1、on条件是在生成临时表时使用的条件，它不管on中的条件是否为真，都会返回左边表中的记录。（实际上左连接中如果and语句是对左表进行过滤的，那么不管真假都不起任何作用。如果是对右表过滤的，那么左表所有记录都返回，右表筛选以后再与左表连接返回）  
2、where条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有left join的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉，on后的条件用来生成左右表关联的临时表，where后的条件对临时表中的记录进行过滤。

在使用inner join时，不管是对左表还是右表进行筛选，on and和on where都会对生成的临时表进行过滤。    

### 替代方案

通过子查询优化

## order by

排序操作，尽量使用索引排序

索引排序同样是利用最左前缀索引





## limit

