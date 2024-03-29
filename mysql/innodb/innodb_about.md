# about

* innodb 是事务型储存引擎；
* innodb 被设计用来处理大量短期的事务；
* innodb 的数据储存在表空间中，表空间由一系列数据文件组成。在mysql4.1版本以后，innodb可以将每个表的数据和索引存放在单独的文件中；
* innodb 采用mvcc来支持高并发，并实现了四个标准隔离级别，默认隔离级别是 REPEATTED READ，通过间隙锁(next-key locking) 来防止幻读出现；
* innodb 使用聚簇索引；
* 

## 一、mvcc

> innodb的mvcc是通过在每行记录后保存两个隐藏列来实现的。这两列，一个是保存创建时间，一个保存行的过期时间。保存的并不是实际的时间戳，而是系统版本号。mvcc只在 REPEATABLE READ 和 READ COMMITTED 隔离级别下工作。另外两个隔离级别与mvcc不兼容，因为 READ UNCOMMITTED 总是读取最新数据行，而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行加锁。

### Select

> innodb会根据以下两个条件检查每行记录：

* innodb只查找创建版本早于当前事务版本的数据行。
* 行的删除版本要么未定义，要么大于当前事务版本。

### Insert

> innodb为新增的行保存当前系统版本号作为行创建版本

### Delete

> innodb为删除的行保存当前系统版本号作为行删除版本

### Update

> innodb插入一条新记录，存当前系统版本号作为行创建版本。同时，保存当前系统版本号到原来的行作为行删除版本

## 二、索引

> innodb 使用 B+Tree 索引，以及自适应哈希索引。其中，自适应哈希索引由innodb全权掌控

### B+Tree对以下索引有效

* 全值匹配
* 最左前缀匹配
* 匹配列前缀
* 匹配范围值
* 精确匹配最左列、范围匹配第二列
* 只访问索引的查询：覆盖索引

### 前缀索引

> 有时需要索引很长的字符列，这会让索引变得大且慢。这时，可仅使用字符列的前部分字符创建索引，从而节省索引空间，提高索引效率。例：ALTER TABLE sakila.city_demo ADD KEY (city(7))

> *注意* : mysql 无法使用前缀索引做 ORDER BY 和 GROUP BY，也无法使用前缀索引做覆盖扫描

### 索引选择性

> 索引的选择性是指，不重复的索引值（也称为基数，cardinality)和总记录数T的比值。范围从 1/T 到 1。选择性为 1 的索引是最好的。

### 三星索引

* 索引将相关记录放到一起则获得一星；
* 索引中的数据顺序和查找排列顺序一致则获得二星；
* 如果索引的列包含了查询所需的所有列，则获得三星！

### 聚簇索引

> 聚簇索引不是一种索引类型，而是一种数据储存方式。聚簇索引具体细节取决于实现，InnoDB的聚簇索引实际上在同一个结构中保存了 B+Tree索引和数据行。<br>
> <br>术语“聚簇”表示数据行和相邻的键值紧凑的储存在一起。因为无法同时将数据行存放在两个不同的地方，所以一个表只能有一个聚簇索引。<br>
> <br>InnoDB通过主键聚簇数据。如果没定义主键，InnoDB会选择一个唯一的非空索引代替。如果没有这样的索引，InnoDB会隐式定义一个主键来作为聚簇索引。<br>
> <br>聚簇索引每一个叶子节点都包含了主键值、事务ID、用于事务和MVCC的回滚指针以及剩余所有列。如果主键是一个列前缀索引，InnoDB页会包含完整的主键列。<br>

优点: 
* 可以把相关数据保存在一起；
* 数据访问更快；
* 使用覆盖索引扫描的查询可以直接使用叶节点中的主键值。

缺点：
* 聚簇索引最大限度地提高了IO密集型应用的性能，但如果数据全部都在内存中，则访问顺序就没那么重要了，聚簇索引也就没什么优势了；
* 插入速度严重依赖于插入顺序。
* 更新聚簇索引列的代价很高，因为会强制InnoDB将每个被更新的行移动到新的位置；
* 基于聚簇索引的列在插入新行或者主键被更新导致需要移动行时，可能面临页分裂问题；
* 聚簇索引可能导致全表扫描变慢，尤其是行比较稀疏，或者由于页分裂导致数据储存不连续时；
* 二级索引可能比想象的要更大，因为二级索引的叶子节点包含了主键列；
* 二级索引访问需要两次查找。

### 二级索引

> InnoDB中，非主键索引（聚簇索引）就是二级索引

优点：
    * InnoDB在移动行时，无需更新二级索引中的叶子节点值（主键）。

缺点：
    * 会导致二级索引占用更多空间；
    * 二级索引查询会导致两次查找。

### 覆盖索引

> 如果一个索引包含（或者说覆盖）所有需要查询的字段，我们就称之为 “覆盖索引”。

在 Mysql5.5以及更早的版本，即使索引完全覆盖了WHERE条件中的字段，但不完全覆盖查询涉及的字段，也总是会回表获取数据行再决定是否过滤数据行。
可以使用子查询来使用覆盖索引过滤数据行：子查询使用覆盖索引，主查询使用子查询的结果回表获取数据行。这种方式叫做延迟关联（deferred join）

Mysql5.6中新增的功能“索引条件下推（ICP）”可优化上述问题。启用索引条件下推功能后，Mysql服务层会将WHERE条件下推到储存引擎层，储存引擎可将不满足的数据行通过索引直接过滤掉。

ICP受以下条件限制：
    * ICP可用于InnoDB和MyISAM表。（例外：MySQL 5.6中的分区表不支持ICP; MySQL 5.7中已解决此问题。）
    * 对于InnoDB表，ICP仅用于二级索引。 ICP的目标是减少全行读取的数量，从而减少IO操作。对于InnoDB聚簇索引，完整记录已经读入InnoDB缓冲区。在这种情况下使用ICP不会降低IO.
    * 引用子查询的条件无法下推。
    * 无法推下涉及存储函数的条件。存储引擎无法调用存储的函数。
    * 触发条件无法下推。


## 锁

* 在 READ COMMITTED 事务隔离级别下，除了唯一性约束检查和外键约束的检查需要 gap lock，InnoDB不会使用 gap lock 的锁算法；
* 在 REPEATABLE READ 事务隔离级别下，使用 Next-Key Lock 锁算法；

### Next-Key Lock

Next-Key Lock是一种左开右闭区间的锁。在某些情况下，Next-Key Lock会退化成 Gap Lock 以及 Record Lock。

#### Next-Key Lock加锁分析

##### 唯一索引等值查询存在的数据

唯一索引等值查询存在的数据，Next-Key Lock会退化成 Record Lock。

##### 唯一索引等值查询不存在的数据

唯一索引等值查询不存在的数据，Next-Key Lock会退化成 Gap Lock，范围是包含查询值的区间。

##### 唯一索引范围查询

###### 大于的范围查询

大于的范围查询对查询的范围使用 Next-Key Lock。

###### 小于的范围查询

小于的范围查询对查询的范围使用 Next-Key Lock，最右侧的范围使用 Gap Lock，如果最右侧是 supremum pseudo-record，那么最右侧的范围也使用 Next-Key Lock。

注：即使小于的值是存在的数据的右边界值+1，(右边界，下一个记录) 也需要加锁，即：
| id |
| -- |
| 1  |
| 5  |
| 7  |
| 9  |

SELECT * FROM table WHERE id < 8 FOR UPDATE. 加锁：(1, 5]、(5, 7]、(7, 9).

###### 等值查询、范围查询混合

等值查询部分使用等值查询规则，范围查询部分使用范围查询规则。

##### 非唯一索引查询

非唯一索引可以有很多个，所以非唯一索引查询都是范围查询。对非唯一索引加锁还会对主键索引进行加锁，但只会对扫描到的已存在记录加 Record Lock。

非唯一索引上的锁使用 {ununique, primary} 表示。当 {u，p} 代表左边界时，{u, p-1} 不在范围中，{u, p+1} 在范围中；当 {u，p} 代表右边界时，{u, p-1} 在范围中，{u, p+1} 不在范围中。

###### 非唯一索引等值查询

非唯一索引等值查询，对等值使用 Next-Key Lock，对边界使用 Gap Lock，即：

| id | age |
| -- | --- |
| 1  |  10 |
| 5  |  20 |
| 7  |  30 |
| 9  |  40 |

1. Select * From table Where age = 20，锁定 ({10, 1}, {20, 5}]、({20, 5}, {30, 7}) 以及 Record Lock 5.
2. Select * From table Where age = 25，锁定 ({20, 5}, {30, 7})

###### 非唯一索引范围查询

非唯一索引范围查询总是使用 Next-Key Lock，不会退化成 Record Lock 和 Gap Lock。

| id | age |
| -- | --- |
| 1  |  10 |
| 5  |  20 |
| 7  |  30 |
| 9  |  40 |

1. Select * From table Where age > 20，锁定 ({20, 5}, {30, 7}]、({30, 7}, {40, 9}]、({40, 9}, {supremum pseudo-record}] 以及 Record Lock 7、9.

2. Select * From table Where age < 30，锁定 (infimum pseudo-record, {10, 1}]、({10, 1}, {20, 5}]、({20, 5}, {30, 7}] 以及 Record Lock 1、5.

3. Select * From table Where age < 31，锁定 (infimum pseudo-record, {10, 1}]、({10, 1}, {20, 5}]、({20, 5}, {30, 7}]、 ({30, 7}, {40, 9}] 以及 Record Lock 1、5、7.
