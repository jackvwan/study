# about

* innodb 是事务型储存引擎；
* innodb 被设计用来处理大量短期的事务；
* innodb 的数据储存在表空间中，表空间由一系列数据文件组成。在mysql4.1版本以后，innodb可以将每个表的数据和索引存放在单独的文件中；
* innodb 采用mvcc来支持高并发，并实现了四个标准隔离级别，默认隔离级别是 REPEATTED READ，通过间隙锁(next-key locking) 来防止幻读出现；
* innodb 使用聚簇索引；
* 

## mvcc

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

