# MySQL 5.7

### 背景需求
生产环境加字段是一个非常常见的操作。  
但是现在的场景是，单表有大约数亿行数据，磁盘占用数百G，1秒中内会有大量的select delete操作，且耗时敏感，本着不影响生产环境的原则，先调研一下，谨慎操作。

### 资料汇总
[MySQL OSC(在线更改表结构)原理](https://www.cnblogs.com/chinesern/p/7677379.html)  
这个文章介绍了OSC(Online Schema Change)的多种方案，写的非常好。  
分析了1.MySQL5.6 OnlineDDL（内置功能）
2.Percona公司的pt-online-schema-change。3.OAK Openark – kit。三种方式的实现步骤和可能存在的问题。

MySQL官方文档：  
英文版：https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html#online-ddl-index-operations  
中文版本：https://www.docs4dev.com/docs/zh/mysql/5.7/reference/innodb-online-ddl-operations.html#online-ddl-column-syntax-notes  

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a55974843d04f63a6e7abf1d1167c74~tplv-k3u1fbpfcp-watermark.image)

截图上可以看到，官方文档中在线DDL添加列的同时 允许并发DML，也就是允许select,update,delete...除非是增加的auto-increase的列不允许。

---
看一下《MySQL实战45讲》中的内容
> 因此，在MySQL 5.5版本中引入了MDL，当对一个表做增删改查操作的时候，加MDL读锁；当要对表做结构变更操作的时候，加MDL写锁。  
> 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。  
> 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。  
> 虽然MDL锁是系统默认会加的，但却是你不能忽略的一个机制。比如下面这个例子，我经常看到有人掉到这个坑里：给一个小表加个字段，导致整个库挂了。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4673cdbd1494864b0a9118c48991c16~tplv-k3u1fbpfcp-watermark.image)
```
我们可以看到session A先启动，这时候会对表t加一个MDL读锁。由于session B需要的也是MDL读锁，因此可以正常执行。

之后session C会被blocked，是因为session A的MDL读锁还没有释放，而session C需要MDL写锁，因此只能被阻塞。

如果只有session C自己被阻塞还没什么关系，但是之后所有要在表t上新申请MDL读锁的请求也会被session C阻塞。前面我们说了，所有对表的增删改查操作都需要先申请MDL读锁，就都被锁住，等于这个表现在完全不可读写了。

如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新session再请求的话，这个库的线程很快就会爆满。

你现在应该知道了，事务中的MDL锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。

基于上面的分析，我们来讨论一个问题，如何安全地给小表加字段？

首先我们要解决长事务，事务不提交，就会一直占着MDL锁。在MySQL的information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。如果你要做DDL变更的表刚好有长事务在执行，要考虑先暂停DDL，或者kill掉这个长事务。

但考虑一下这个场景。如果你要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁，而你不得不加个字段，你该怎么做呢？

这时候kill可能未必管用，因为新的请求马上就来了。比较理想的机制是，在alter table语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者DBA再通过重试命令重复这个过程。

MariaDB已经合并了AliSQL的这个功能，所以这两个开源分支目前都支持DDL NOWAIT/WAIT n这个语法。

ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 

```

可以知道，alter table会加上MDL，阻塞其他DML，这看起来好像跟MySQL官方的表格说法不同。
但是避免长事务的方式可以借鉴，业务中本来也没有长事务的case，都是单行执行完就提交的事务，不会出现长时间等待的问题。

### 验证和测试
MySQL 5.7 主从配置，模拟生产环境，往单表插入670w行数据。  
运行业务逻辑，对该表进行频繁读写，200 select/s,200 delete /s, 100 insert/s  
此时执行预先准备好的DDL语句
```
ALTER TABLE table_name ADD app_name varchar(64) NOT NULL DEFAULT '' ,
ADD INDEX APP_NAME_IDX(app_name) USING BTREE ;
```
观察DML的执行均未收到影响，DDL语句执行耗时150s。  
查日志也没有发现 MySQLTransactionRollbackException，一切正常。

### 总结
在MySQL 5.7版本 大表增加字段和索引的操作中不会阻塞DML语句（也有可能是阻塞非常短的时间），使用MySQL内置的功能即可，无需使用其他工具。  
但是要注意的是，避免长事务或者加上wait关键词,多条alter语句可以用逗号拼接。

# 其他MySQL版本

* MySQL 8.0 默认支持秒加字段  
https://jishuin.proginn.com/p/763bfbd2c9a7

* AWS Amazon Aurora MySQL  
https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FastDDL.html

*  PolarDB MySQL 阿里云原生数据库   
https://help.aliyun.com/document_detail/198913.html
