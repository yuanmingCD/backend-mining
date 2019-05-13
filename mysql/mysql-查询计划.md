# mysql-查询计划explain

查询计划，可以更加详细知道sql的行为，是sql性能优化必修课

## explain命令

用法 在sql之前添加explain



### explain详解

- **id**：表的读取顺序，id列数字越大越先执行，如果说数字一样大，那么就从上往下依次执行。
- **select_type**：查询类型
- **table** 对应行正在访问哪一个表，表名或者别名
- **type**：查询类型，关键指标
- **partitions**：分区表命中的分区情况，非分区表该字段为空（null）
- **possible_keys**：可能用到的索引
- **key**：实际使用的索引。如果为`NULL`，则没有使用索引。很少的情况下，MySQL会选择优化不足的索引。这种情况下，可以在`SELECT`语句中使用`USE INDEX（indexname）` 来强制使用一个索引或者用`IGNORE INDEX（indexname）`来强制MySQL忽略索引。
- **key_len**：使用的索引的长度
- **ref**：ref列显示使用哪个列或常数与key一起从表中选择行。
- **rows**：rows列显示MySQL认为它执行查询时必须检查的行数。注意这是一个预估值。
- **filtered**： 一个百分比的值，这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例
- **extra**：附加信息，分析数据的整合过程



### select_type详解

- **simple** 简单子查询，不包含子查询和union，有连接查询时，外层的查询为simple，且只有一个
- **primary** 包含union或者子查询，最外层的部分标记为primary   
- **union** union连接的select查询，第二个及其以后的子查询被标记为union，第一个就被标记为primary
- **dependent union** 首先需要满足UNION的条件，及UNION中第二个以及后面的SELECT语句，同时该语句依赖外部的查询   subquery 子查询中第一个SELECT语句   dependent subquery 和DEPENDENT UNION相对UNION一样
- **union result**：包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null
- **subquery** 除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery  
- **dependent subquery**：与dependent union类似，表示这个subquery的查询要受到外部表查询的影响
- **derived** from字句中出现的子查询
- **materialized**：被物化的子查询
- **uncacheable subquery**：对于外层的主表，子查询不可被物化，每次都需要计算（耗时操作）
- **uncacheable union**：UNION操作中，内层的不可被物化的子查询（类似于UNCACHEABLE SUBQUERY） 

### type详解（性能由好到差排行）

- **system** 这是const连接类型的一种特例，表仅有一行满足条件。  

- **const** 当确定最多只会有一行匹配的时候，通过主键查找  

- **eq_ref** 使用唯一性索引或主键查找，最多只返回一条符合条件的记录。 

- **ref** 一种索引访问，它返回所有匹配某个单个值的行。此类索引访问只有当使用非唯一性索引或唯一性索引非唯一性前缀时才会发生。这个类型跟eq_ref不同的是，它用在关联操作只使用了索引的最左前缀，或者索引不是UNIQUE和PRIMARY KEY。ref可以用于使用=或<=>操作符的带索引的列。 

- **fulltext** 全文索引，优先级高于普通索引

- **ref_or_null** 与ref方法类似，增加了null值的比较。实际用的不多（is null）

- **index_merge** 表示查询使用了两个以上的索引，最后取交集或者并集，常见and ，or的条件使用了不同的索引，官方排序这个在ref_or_null之后，但是实际上由于要读取所个索引，性能可能大部分时间都不如range

- **unique_subquery** 用于where中的in形式子查询，子查询返回不重复值唯一值

- **index_subquery** 用于in形式子查询使用到了辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引将子查询去重。

- **range** 范围扫描，一个有限制的索引扫描。key 列显示使用了哪个索引。当使用=、 <>、>、>=、<、<=、IS NULL、<=>、BETWEEN 或者 IN 操作符,用常量比较关键字列时。需要注意的是，当数匹配数据超过表行数时，会退化为全表扫描，这是为了避免过多的 random disk。

- **index** 和全表扫描一样。只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大。如在Extra列看到Using index，说明正在使用覆盖索引，只扫描索引的数据，它比按索引次序全表扫描的开销要小很多   

- **All** 最坏的情况,全表扫描 ，通常为没有where语句，或者where语句中没有索引，或者索引范围查找且数量大于表内数据的一半会发生

   

### Extra详解

- **distinct** 优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作

- **select tables optimized away** 在没有GROUP BY子句的情况下，基于索引优化MIN/MAX操作，或者对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。  

- **impossible where** where子句的值总是false，不能用来获取任何元组    

- **Using join buffer** 使用了连接缓存：Block Nested Loop，连接算法是块嵌套循环连接;Batched Key Access，连接算法是批量索引连接  

- **Using where** 表示优化器需要通过索引回表查询数据，表示MySQL服务器将存储引擎返回服务层以后再应用WHERE条件过滤。 

- **Using index** 使用覆盖索引，表示直接访问索引就足够获取到所需要的数据，不需要通过索引回表，说明查询是覆盖了索引的，不需要读取数据文件，从索引树（索引文件）中即可获得信息。如果同时出现using where，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。这是MySQL服务层完成的，但无需再回表查询记录。    

- **Using index condition** MySQL 5.6新特性。Using index condition 会先条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行；简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上。 

    - ICP is used for the [`range`](https://dev.mysql.com/doc/refman/5.6/en/explain-output.html#jointype_range), [`ref`](https://dev.mysql.com/doc/refman/5.6/en/explain-output.html#jointype_ref), [`eq_ref`](https://dev.mysql.com/doc/refman/5.6/en/explain-output.html#jointype_eq_ref), and [`ref_or_null`](https://dev.mysql.com/doc/refman/5.6/en/explain-output.html#jointype_ref_or_null) access methods when there is a need to access full table rows.当type为。。。访问整个数据行

    - ICP can be used for [`InnoDB`](https://dev.mysql.com/doc/refman/5.6/en/innodb-storage-engine.html) and [`MyISAM`](https://dev.mysql.com/doc/refman/5.6/en/myisam-storage-engine.html) tables. (Exception: ICP is not supported with partitioned tables in MySQL 5.6; this issue is resolved in MySQL 5.7.)

    - For `InnoDB` tables, ICP is used only for secondary indexes. The goal of ICP is to reduce the number of full-row reads and thereby reduce I/O operations. For `InnoDB` clustered indexes, the complete record is already read into the `InnoDB` buffer. Using ICP in this case does not reduce I/O.

    - Conditions that refer to subqueries cannot be pushed down.子查询不能下推

    - Conditions that refer to stored functions cannot be pushed down. Storage engines cannot invoke stored functions.

    - Triggered conditions cannot be pushed down. (For information about triggered conditions, see [Section 8.2.2.3, “Optimizing Subqueries with the EXISTS Strategy”](https://dev.mysql.com/doc/refman/5.6/en/subquery-optimization-with-exists.html).)  

- **Not exists** MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了。

- **Using temporary** 用临时表保存中间结果，常用于GROUP BY 和 ORDER BY操作中，一般看到它说明查询需要优化了，就算避免不了临时表的使用也要尽量避免硬盘临时表的使用。   

  MySQL首先创建heap引擎的临时表，如果临时的数据过多，超过max_heap_table_size的大小，会自动把临时表转换成MyISAM引擎的表来使用。

- **Using filesort**  filesort主要用于查询数据结果集的排序操作，但注意虽然叫filesort但并不是说明就是用了文件来进行排序，只要可能排序都是在内存里完成的。首先MySQL会使用sort_buffer_size大小的内存进行排序，如果结果集超过了sort_buffer_size大小，会把这一个排序后的chunk转移到file上，最后使用多路归并排序完成所有数据的排序操作。常用于GROUP BY 和 ORDER BY操作中，这可能是一个CPU密集型的过程，可以通过选择合适的索引来改进性能，用索引来为查询结果排序。 

  MySQL filesort有两种使用模式:

  - 模式1: sort的item保存了所需要的所有字段，排序完成后，没有必要再回表扫描。
  - 模式2: sort的item仅包括，待排序完成后，根据rowid查询所需要的columns。

  很明显，模式1能够极大的减少回表的随机IO。

  filesort只能应用在单个表上，如果有多个表的数据需要排序，那么MySQL会先使用using temporary保存临时数据，然后再在临时表上使用filesort进行排序，最后输出结果。



参考文档

[MySQL · 答疑释惑· using filesort VS using temporary](https://www.kancloud.cn/taobaomysql/monthly/67180)