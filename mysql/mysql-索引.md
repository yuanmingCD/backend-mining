# mysql-索引-数据结构

## 什么是索引？



特点：有序

- 优点：
  - 索引最大的好处是提高查询速度，
- 缺点：
  - 影响插入时效率
  - 占用资源



# 索引数据结构

### Hash索引



## B-树

B-树（Balance tree，平衡树），读作B树，

## B+树





B+ Tree索引和Hash索引区别

- 哈希索引适合等值查询，但是无法进行范围查询 
- 哈希索引没办法利用索引完成排序 
- 哈希索引不支持多列联合索引的最左匹配规则 
- 如果有大量重复键值得情况下，哈希索引的效率会很低，因为存在哈希碰撞问题

## LSM树



参考文档

[[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

