# Innodb锁机制：Next-Key Lock 浅谈



 数据库使用锁是为了支持更好的并发，提供数据的完整性和一致性。InnoDB是一个支持行锁的存储引擎，锁的类型有：共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）。为了提供更好的并发，InnoDB提供了非锁定读：不需要等待访问行上的锁释放，读取行的一个快照。该方法是通过InnoDB的一个特性：MVCC来实现的。

**InnoDB有三种行锁的算法：**

1，**Record Lock**：单个行记录上的锁。

2，**Gap Lock**：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。

3，**Next-Key Lock**：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

**测试一：默认RR隔离级别**

```mysql
root@localhost : test 10:56:10>create table t(a int,key idx_a(a))engine =innodb;
Query OK, 0 rows affected (0.20 sec)

root@localhost : test 10:56:13>insert into t values(1),(3),(5),(8),(11);
Query OK, 5 rows affected (0.00 sec)
Records: 5  Duplicates: 0  Warnings: 0

root@localhost : test 10:56:15>select * from t;
+------+
| a    |
+------+
|    1 |
|    3 |
|    5 |
|    8 |
|   11 |
+------+
5 rows in set (0.00 sec)

section A:

root@localhost : test 10:56:27>start transaction;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 10:56:29>select * from t where a = 8 for update;
+------+
| a    |
+------+
|    8 |
+------+
1 row in set (0.00 sec)


section B:
root@localhost : test 10:54:50>begin;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 10:56:51>select * from t;
+------+
| a    |
+------+
|    1 |
|    3 |
|    5 |
|    8 |
|   11 |
+------+
5 rows in set (0.00 sec)

root@localhost : test 10:56:54>insert into t values(2);
Query OK, 1 row affected (0.00 sec)

root@localhost : test 10:57:01>insert into t values(4);
Query OK, 1 row affected (0.00 sec)

++++++++++
root@localhost : test 10:57:04>insert into t values(6);

root@localhost : test 10:57:11>insert into t values(7);

root@localhost : test 10:57:15>insert into t values(9);

root@localhost : test 10:57:33>insert into t values(10);
++++++++++
上面全被锁住，阻塞住了

root@localhost : test 10:57:39>insert into t values(12);
Query OK, 1 row affected (0.00 sec)
```



**问题：**

**为什么section B上面的插入语句会出现锁等待的情况**？InnoDB是行锁，在section A里面锁住了a=8的行，其他应该不受影响。why？

**分析：**

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围（GAP）。上面索引值有1，3，5，8，11，其记录的GAP的区间如下：是一个**左开右闭**的空间（原因是默认主键的有序自增的特性，结合后面的例子说明）

（-∞,1]，(1,3]，(3,5]，(5,8]，(8,11]，(11,+∞）

特别需要注意的是，InnoDB存储引擎还会对辅助索引下一个键值加上gap lock。如上面分析，那就可以解释了。



```mysql
root@localhost : test 10:56:29>select * from t where a = 8 for update;
+------+
| a    |
+------+
|    8 |
+------+
1 row in set (0.00 sec)
```



该SQL语句锁定的范围是（5,8]，下个下个键值范围是（8,11]，所以插入5~11之间的值的时候都会被锁定，要求等待。即：插入5，6，7，8，9，10 会被锁住。插入非这个范围内的值都正常。

**################################### 

因为例子里没有主键，所以要用隐藏的ROWID来代替，数据根据Rowid进行排序。而Rowid是有一定顺序的（自增），所以其中11可以被写入，5不能被写入，不清楚的可以再看一个有主键的例子：



```mysql
会话1：
01:43:07>create table t(id int,name varchar(10),key idx_id(id),primary key(name))engine =innodb;
Query OK, 0 rows affected (0.02 sec)

01:43:11>insert into t values(1,'a'),(3,'c'),(5,'e'),(8,'g'),(11,'j');                                                                               
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0

01:44:03>select @@global.tx_isolation, @@tx_isolation;                                                                                                 +-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.01 sec)

01:44:58>select * from t;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
|    3 | c    |
|    5 | e    |
|    8 | g    |
|   11 | j    |
+------+------+
5 rows in set (0.00 sec)

01:45:07>start transaction;              

01:45:09>delete from t where id=8;
Query OK, 1 row affected (0.01 sec)


会话2：
01:50:38>select @@global.tx_isolation, @@tx_isolation;
+-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.01 sec)

01:50:48>start transaction; 

01:50:51>select * from t;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
|    3 | c    |
|    5 | e    |
|    8 | g    |
|   11 | j    |
+------+------+
5 rows in set (0.01 sec)

01:51:35>insert into t(id,name) values(6,'f');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:53:32>insert into t(id,name) values(5,'e1');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:53:41>insert into t(id,name) values(7,'h');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:54:43>insert into t(id,name) values(8,'gg');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:55:10>insert into t(id,name) values(9,'k');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:55:23>insert into t(id,name) values(10,'p');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

01:55:33>insert into t(id,name) values(11,'iz');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted

#########上面看到 id：5，6，7，8，9，10，11都被锁了。

#########下面看到 id：5，11 还是可以插入的
01:54:33>insert into t(id,name) values(5,'cz');
Query OK, 1 row affected (0.01 sec)

01:55:59>insert into t(id,name) values(11,'ja');
Query OK, 1 row affected (0.01 sec)
```



**分析：**因为会话1已经对id=8的记录加了一个X锁，由于是RR隔离级别，INNODB要防止幻读需要加GAP锁：即id=5（8的左边），id=11（8的右边）之间需要加间隙锁（GAP）。这样[5,e]和[8,g]，[8,g]和[11,j]之间的数据都要被锁。上面测试已经验证了这一点，根据索引的有序性，数据按照主键（name）排序，后面写入的[5,cz]（[5,e]的左边）和[11,ja]（[11,j]的右边）不属于上面的范围从而可以写入。

另外一种情况，把name主键去掉会是怎么样的情况？有兴趣的同学可以测试一下。

**##################################################**

**继续：**插入超时失败后，会怎么样？

超时时间的参数：innodb_lock_wait_timeout ，默认是50秒。
超时是否回滚参数：innodb_rollback_on_timeout 默认是OFF。



```mysql
section A:

root@localhost : test 04:48:51>start transaction;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 04:48:53>select * from t where a = 8 for update;
+------+
| a    |
+------+
|    8 |
+------+
1 row in set (0.01 sec)


section B:

root@localhost : test 04:49:04>start transaction;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 04:49:07>insert into t values(12);
Query OK, 1 row affected (0.00 sec)

root@localhost : test 04:49:13>insert into t values(10);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
root@localhost : test 04:50:06>select * from t;
+------+
| a    |
+------+
|    1 |
|    3 |
|    5 |
|    8 |
|   11 |
|   12 |
+------+
6 rows in set (0.00 sec)
```



经过测试，不会回滚超时引发的异常，当参数innodb_rollback_on_timeout 设置成ON时，则可以回滚，会把插进去的12回滚掉。

默认情况下，InnoDB存储引擎不会回滚超时引发的异常，除死锁外。

既然InnoDB有三种算法，那Record Lock什么时候用？还是用上面的列子，把辅助索引改成唯一属性的索引。

**测试二：**



```mysql
root@localhost : test 04:58:49>create table t(a int primary key)engine =innodb;
Query OK, 0 rows affected (0.19 sec)

root@localhost : test 04:59:02>insert into t values(1),(3),(5),(8),(11);
Query OK, 5 rows affected (0.00 sec)
Records: 5  Duplicates: 0  Warnings: 0

root@localhost : test 04:59:10>select * from t;
+----+
| a  |
+----+
|  1 |
|  3 |
|  5 |
|  8 |
| 11 |
+----+
5 rows in set (0.00 sec)

section A:

root@localhost : test 04:59:30>start transaction;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 04:59:33>select * from t where a = 8 for update;
+---+
| a |
+---+
| 8 |
+---+
1 row in set (0.00 sec)

section B:

root@localhost : test 04:58:41>start transaction;
Query OK, 0 rows affected (0.00 sec)

root@localhost : test 04:59:45>insert into t values(6);
Query OK, 1 row affected (0.00 sec)

root@localhost : test 05:00:05>insert into t values(7);
Query OK, 1 row affected (0.00 sec)

root@localhost : test 05:00:08>insert into t values(9);
Query OK, 1 row affected (0.00 sec)

root@localhost : test 05:00:10>insert into t values(10);
Query OK, 1 row affected (0.00 sec)
```



**问题：**

为什么section B上面的插入语句可以正常，和测试一不一样？

**分析：**

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围，按照这个方法是会和第一次测试结果一样。但是，当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。

注意：通过主键或则唯一索引来锁定不存在的值，也会产生GAP锁定。即： 



```mysql
会话1：
04:22:38>show create table t\G
*************************** 1. row ***************************
       Table: t
Create Table: CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `name` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

04:22:49>start transaction;

04:23:16>select * from t where id = 15 for update;
Empty set (0.00 sec)

会话2：
04:26:10>insert into t(id,name) values(10,'k');
Query OK, 1 row affected (0.01 sec)

04:26:26>insert into t(id,name) values(12,'k');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted
04:29:32>insert into t(id,name) values(16,'kxx');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted
04:29:39>insert into t(id,name) values(160,'kxx');
^CCtrl-C -- sending "KILL QUERY 9851" to server ...
Ctrl-C -- query aborted.
ERROR 1317 (70100): Query execution was interrupted 
```



如何让测试一不阻塞？可以显式的关闭Gap Lock：

1：把事务隔离级别改成：Read Committed，提交读、不可重复读。SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

2：修改参数：innodb_locks_unsafe_for_binlog 设置为1。

 

**总结：**

本文只对 Next-Key Lock 做了一些说明测试，关于锁还有很多其他方面的知识，可以查阅相关资料进行学习。

写完之后的几天刚好牛人写了一篇详细的文章：http://hedengcheng.com/?p=771