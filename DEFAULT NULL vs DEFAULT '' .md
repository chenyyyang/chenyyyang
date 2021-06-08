##### 创建一个Mysql表的时候
```
CREATE TABLE `t1_null` (
  `uuid` varchar(100) NOT NULL DEFAULT '',
  `Column1` varchar(100) DEFAULT NULL,  
  `Column2` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`uuid`),
  KEY `t1_null_Column1_IDX` (`Column1`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
```
CREATE TABLE `t2_not_null` (
  `uuid` varchar(100) NOT NULL DEFAULT '',
  `Column1` varchar(100) NOT NULL DEFAULT '',
  `Column2` varchar(100) NOT NULL DEFAULT '',
  PRIMARY KEY (`uuid`),
  KEY `t2_not_null_Column1_IDX` (`Column1`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
t1_null 和 t2_not_null 区别仅仅在于Column1，Column2是否 允许NULL。  

## what's better for disk space and performance ？


Mysql官方 [8.4.1 Optimizing Data Size](https://dev.mysql.com/doc/refman/5.7/en/data-size.html)
> Declare columns to be NOT NULL if possible. It makes SQL operations faster, by enabling better use of indexes and eliminating overhead for testing whether each value is NULL. You also save some storage space, one bit per column. If you really need NULL values in your tables, use them. Just avoid the default setting that allows NULL values in every column.

From 《High Performance MySQL, 3rd Edition》
>Avoid NULL if possible. A lot of tables include nullable columns even when the application does not need to store NULL (the absence of a value), merely because it’s the default. It’s usually best to specify columns as NOT NULL unless you intend to store NULL in them. It’s harder for MySQL to optimize queries that refer to nullable columns, because they make indexes, index statistics, and value comparisons more complicated. A nullable column uses more storage space and requires special processing inside MySQL. When a nullable column is indexed, it requires an extra byte per entry and can even cause a fixed-size index (such as an index on a single integer column) to be converted to a variable-sized one in MyISAM. The performance improvement from changing NULL columns to NOT NULL is usually small, so don’t make it a priority to find and change them on an existing schema unless you know they are causing problems. However, if you’re planning to index columns, avoid making them nullable if possible. There are exceptions, of course. For example, it’s worth mentioning that InnoDB stores NULL with a single bit, so it can be pretty space-efficient for sparsely populated data. This doesn’t apply to MyISAM, though.

翻译并简单来说就是，‘DEFAULT NULL’ 的字段会多使用1 bit的空间，查询时需要test each value是否为NULL

>As with most "what's better for disk space and performance" questions: why don't you just insert a million rows with NULL, test some queries, and check for disk space? Repeat with ""s, and once more with a relatively even mix. And the answer is much more reliable than what some random guy on SO says

那‘DEFAULT NULL’和 NOT NULL DEFAULT '' 对查询和存储的影响究竟有多大呢，来测试一下吧。

1.创建 t1_null 和 t2_not_null  

2.每个表写入 1600w行数据  

3.SELECT VERSION()   8.0.23 无查询缓存,重启Mysql Server  

4.打开Mysql 自带的SQL查询统计功能  
show variables like "%pro%";  
如果看到 profiling	OFF，set profiling = 1; 开启该功能  

##### 5.执行查询sql，count(column)的时候，column如果NULL，则不会被计数

show profiles;

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a606006b9e4c4e7680c31dfaa62d59dd~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52b34b15b90449f5bb5a662e4b1a6b3a~tplv-k3u1fbpfcp-watermark.image)  
- 总结是 select * 和 count(*)的时候 两者的遍历耗时差不多，且default null还会稍微快一点  
- select Column1 的时候，default ""有略微的优势。所以，DEFAULT NULL想用就用吧。
- 另一个角度讲，用default ""最大的优势就是,count(Column)时，可以选择这个列上的索引全索引扫描（否则就走主键索引），如果这一列size远小于主键的话，每次读取到buffer的列更多，减少磁盘IO次数，提高count()速度

explain分析一下查询计划
```
explain select count(Column1) from mytable.t1_null; 
1	SIMPLE	t1_null 	NULL	index	NULL	t1_null_Column1_IDX	403	NULL	15560054	100.00	Using index
explain select count(Column1) from mytable.t2_not_null;
1	SIMPLE	t2_not_null	NULL	index	NULL	t2_not_null_Column1_IDX	402	NULL	15929685	100.00	Using index

explain select count(*) from mytable.t1_null; 
1	SIMPLE	t1_null 	NULL	index	NULL	t1_null_Column1_IDX	403	NULL	15560054	100.00	Using index
explain select count(*) from mytable.t2_not_null;
1	SIMPLE	t2_not_null	NULL	index	NULL	t2_not_null_Column1_IDX	402	NULL	15929685	100.00	Using index
```

## COUNT()函数
版本 8.0 https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html  
For transactional storage engines such as InnoDB, storing an exact row count is problematic. Multiple transactions may be occurring at the same time, each of which may affect the count.  
InnoDB does not keep an internal count of rows in a table because concurrent transactions might “see” different numbers of rows at the same time. Consequently, SELECT COUNT(*) statements only count rows visible to the current transaction.
As of MySQL 8.0.13, SELECT COUNT(*) FROM tbl_name query performance for InnoDB tables is optimized for single-threaded workloads if there are no extra clauses such as WHERE or GROUP BY.  
InnoDB processes SELECT COUNT(*) statements by traversing the smallest available secondary index unless an index or optimizer hint directs the optimizer to use a different index. If a secondary index is not present, InnoDB processes SELECT COUNT(*) statements by scanning the clustered index.   
Processing SELECT COUNT(*) statements takes some time if index records are not entirely in the buffer pool. For a faster count, create a counter table and let your application update it according to the inserts and deletes it does. However, this method may not scale well in situations where thousands of concurrent transactions are initiating updates to the same counter table.**If an approximate row count is sufficient, use SHOW TABLE STATUS**.  
InnoDB handles SELECT COUNT(*) and SELECT COUNT(1) operations in the same way. There is no performance difference.  

版本 5.7 中额外还说明了  
Prior to MySQL 5.7.18, InnoDB processes SELECT COUNT(*) statements by scanning the clustered index. As of MySQL 5.7.18, InnoDB processes SELECT COUNT(*) statements by traversing the smallest available secondary index unless an index or optimizer hint directs the optimizer to use a different index. If a secondary index is not present, the clustered index is scanned.

#### 按照官方的说法，从5.7.18版本开始，SELECT COUNT(*)，会自动选择smallest 二级索引，也可以指定一个其他索引，如果没有二级索引，则会使用主键索引。如果可以接受模糊的count，可以使用SHOW TABLE STATUS来获取大致的rows数量。

解释了explain的结果为啥都是用的二级索引。  
但是为啥同样的数据行，每行数据内容也相同，count(*)理论上应该是t2_not_null_Column1_IDX 遍历效率会大于 t1_null_Column1_IDX（因为索引占用磁盘空间更小，便遍历过程中不再需要test is null）
```
select count(Column1) from mytable.t1_null force index(t1_null_Column1_IDX); 
16804155 18.2s
select count(Column1) from mytable.t1_null force index(t1_null_Column1_IDX); 
16820977  9.63s
```


## 用到的SQL
```
show variables like "%pro%";  
set profiling = 1;

explain select count(*) from mytable.t1_null; 
explain select count(*) from mytable.t2_not_null;

show profiles;

show profile for query 6;
show profile for query 7;


select count(*) from mytable.t1_null; 
select count(*) from mytable.t2_not_null;

select count(Column1) from mytable.t1_null force index(PRIMARY); 
select count(Column1) from mytable.t2_not_null force index(PRIMARY); 

SHOW TABLE STATUS;


select count(Column1) from mytable.t1_null force index(t1_null_Column1_IDX); 
select count(Column1) from mytable.t2_not_null force index(t2_not_null_Column1_IDX); 
```
