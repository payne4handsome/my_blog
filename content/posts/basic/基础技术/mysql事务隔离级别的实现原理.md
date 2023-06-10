---
layout: post
title: "mysql事务隔离级别的实现原理"
date: 2022-10-6
categories: 
    - "数据库"
tags: [mysql, 事务, 隔离级别]
author: "pan"
---

# mysql事务隔离级别的实现原理

[mysql innodb中的四种事务隔离级别](https://www.jianshu.com/p/1fc97ad4605d)上文主要以实验的形式的展示了四种隔离级别产生的读一致性问题，本文主要讨论一下mysql是如何实现这四种隔离级别的。

## 一、什么是事务的隔离级别

在数据库系统中，一个事务是指：由一系列数据库操作组成的一个完整的逻辑过程。具备ACID的特性。ACID分别指原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、永久性（Durability）。

事务隔离（Isolation）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，**隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致**。针对这种不一致的级别，产生了事务隔离的四个类别，包括未提交读（Read uncommitted）、提交读（read committed）、可重复读（repeatable read）和串行化（Serializable）。

可重复读(Repeated Read)是mysql的默认级别，本文以它为分析重点。

## 二、再看可重复读

可重复读：举例来说就是在一个事务内,如果先后发生了两次查询$Q_1, Q_2$，如果$Q_2$看到的内容只包含$Q_1$的内容和自已在本次事务中的内容，看不到其他事务操作的结果（无论其他事务对$Q_1$内容更新还是删除），那么这个就叫可重复读。

这里需要再次强调**不可重复读**和**幻读**的区别，不可重复读是针对删除和更新的，幻读是针对插入的。看起来幻读是属于不可以重复读的范畴的，但是为什么要分开呢？

**个人觉得是因为解决这两个的方式是不同的，对于不可重复读，可以直接用普通的锁来解决。但是对于幻读，由于不可能锁住不存在的记录，所以这里就分开了**，对于幻读其实是用的Next_Key锁（行锁+Gap锁）来解决的，这个上一篇文章有提到。

## 三 实验一（读-写操作）

**关闭自动提交、设置隔离级别为可重复读**

开始时刻，会话A,B查询到的结果如下：

```sql
mysql> select * from test;
+----+---------+
| id | account |
+----+---------+
|  1 |     400 |
|  2 |     500 |
|  3 |     600 |
+----+---------+
3 rows in set (0.00 sec)
```

1. 会话B插入一条记录并提交

    ```sql
    mysql> select * from test;
    +----+---------+
    | id | account |
    +----+---------+
    |  1 |     400 |
    |  2 |     500 |
    |  3 |     600 |
    +----+---------+
    3 rows in set (0.01 sec)

    mysql> insert into test values(4, 700);
    Query OK, 1 row affected (0.01 sec)

    mysql> select * from test;
    +----+---------+
    | id | account |
    +----+---------+
    |  1 |     400 |
    |  2 |     500 |
    |  3 |     600 |
    |  4 |     700 |
    +----+---------+
    4 rows in set (0.00 sec)

    mysql> commit;
    Query OK, 0 rows affected (0.00 sec)
    ```

2. 会话A中查询

    ```sql
    mysql> select * from test;
    +----+---------+
    | id | account |
    +----+---------+
    |  1 |     400 |
    |  2 |     500 |
    |  3 |     600 |
    +----+---------+
    3 rows in set (0.00 sec)
    ```

结论：A中没有读取到B中插入的那条记录，说明A中的读是可以重复读的，且不存在幻读问题。
A在整个过程中也没有加锁，那么mysql是如何实现呢？答案就是通过MVCC(Multiversion Concurrency Control--多版本并发控制).

### 3.1 MVCC

在InnoDB中，会在每行数据后添加两个额外的隐藏的值来实现MVCC，这两个值一个记录这行数据何时被创建，另外一个记录这行数据何时过期（或者被删除）。 在实际操作中，**存储的并不是时间，而是事务的版本号，每开启一个新事务，事务的版本号就会递增**。 在可重读Repeatable reads事务隔离级别下, MVCC的工作原理如下：

+ SELECT时，读取创建版本号<=当前事务版本号，删除版本号为空或>当前事务版本号。
+ INSERT时，保存当前事务版本号为行的创建版本号
+ DELETE时，保存当前事务版本号为行的删除版本号
+ UPDATE时，插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行

我们将上述会话A的读换一种读试一试。

```sql
mysql> select * from test lock in share mode;
+----+---------+
| id | account |
+----+---------+
|  1 |     400 |
|  2 |     500 |
|  3 |     600 |
|  4 |     700 |
+----+---------+
4 rows in set (0.00 sec)

mysql> select * from test for update;
+----+---------+
| id | account |
+----+---------+
|  1 |     400 |
|  2 |     500 |
|  3 |     600 |
|  4 |     700 |
+----+---------+
4 rows in set (0.00 sec)
```

我们发现A读到了B会话插入的记录，**那么是不是可以说明mysql的可重复读失效了？当然不是，只是我们用的不对而已**

### 3.2 mysql中的读

大部分的工作工程中，我们用的sql语句都都是不加锁的，我们称这种读为`快照读`。其他的如update、insert、delete都是`当前读`。总结一下

1. 快照读：就是select

```sql
select * from table ….;
```

2. 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，处理的都是当前的数据，需要加锁。

```sql
select * from table where ? lock in share mode;
select * from table where ? for update;
insert;
update ;
delete;
```

## 3.3 结论
所以MVCC可以解决不可重复读和幻读只是在**快照读-写**这种情况下。如果对于**当前读-写**，**写-写**这种情况需要通过两阶段锁协议


## 四 实验二（当前读-写操作）

根据上面的描述，实验一种的**读-写操作**，实际上是**快照读-写操作**。那么这么解决实验一的问题呢？即对**当前读-写操作**也要求是可以重复读且不存在幻读问题。

实验如下：

开始时刻，会话A、B状态如下

```sql
mysql> select * from test ;
+----+---------+
| id | account |
+----+---------+
|  1 |     400 |
|  2 |     500 |
|  3 |     600 |
|  4 |     700 |
+----+---------+
4 rows in set (0.00 sec)
```

1. 为了说明问题，会话A中，我们以当前读取两条记录

```sql
mysql> select * from test where account>=600 and account<=700 lock in share mode;
+----+---------+
| id | account |
+----+---------+
|  3 |     600 |
|  4 |     700 |
+----+---------+
2 rows in set (0.01 sec)
```

2. B 中插入一条记录

```sql
mysql> insert into test values(5, 650);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

**结论： 插入一条account=650的记录发现插入不进去，mysql会阻塞等待，直至超时。所以此时mysql是通过加锁不让你插入的方式来保证会话A中的可重复读的**

3. B 中试一下插入account为其他值的情况

```sql
mysql> insert into test values(5, 550);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into test values(5, 750);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql>
mysql> insert into test values(5, 450);
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+----+---------+
| id | account |
+----+---------+
|  1 |     400 |
|  5 |     450 |
|  2 |     500 |
|  3 |     600 |
|  4 |     700 |
+----+---------+
5 rows in set (0.00 sec)
```

**结论： 我们发现当account等于550,750,650的时候都是插入不进去的；但是account等于450的时候确插入进去了**

**原因解释：这个就是因为间隙锁的原因，会话A以共享锁的方式查询到了account等于600和700的记录，理论上只需要给这两条记录加行锁，但是为了避免幻读的问题给区间（600,700），[500, 600）,(600, +oo)都加上了锁，这个在一定程度下降低mysql的性能**

## 五 总结-MySQL隔离级别的实现

相对于传统隔离级别基于锁的实现方式，mysql通过mvcc和两阶段锁来实现事务的隔离级别

1. MySQL 是通过MVCC（Multiversion Concurrency Control--多版本并发控制）来实现**快照读-写**并发控制。MVCC是一种无锁方案，用以解决事务读-写并发的问题，能够极大提升读-写并发操作的性能。
2. 通过两阶段锁来实现写-写并发控制

补充说明
1. **只有在已提交读、可重复读两个隔离级别下才有MVCC**。
2. 通过传统的加锁（参见参考文献2）肯定也是可以实现4中隔离级别的，只不过我们的数据库在大部分时候都是select快照读这种查询，通过mvcc无锁这种方式大大提供了mysql的性能

## 补充说明

**注意上述实验account上是有索引的**，test表创建语句如下，大家可以自已验证

```sql
mysql> show create table test\G;
*************************** 1. row ***************************
       Table: test
Create Table: CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `account` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_id` (`id`) USING BTREE,
  KEY `idx_account` (`account`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=8 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

```

## 两段锁协议
将事务分成两个阶段，加锁阶段和解锁阶段

+ 加锁阶段：在该阶段可以进行加锁操作。在对任何数据进行读操作之前要申请并获得S锁（共享锁，其它事务可以继续加共享锁，但不能加排它锁），在进行写操作之前要申请并获得X锁（排它锁，其它事务不能再获得任何锁）。加锁不成功，则事务进入等待状态，直到加锁成功才继续执行。
+ 解锁阶段：当事务释放了一个封锁以后，事务进入解锁阶段，在该阶段只能进行解锁操作不能再进行加锁操作。

**所有遵守两段锁协议的事务，其并行执行的结果一定是正确的**

**注意**
1. 并没有一段锁协议，但是有一次封锁法，它是遵循两段锁协议的；一次封锁法是指一次性的将用到的数据全部加锁，但在数据库中不适用，因为在事务开始阶段，数据库并不知道会用到哪些数据


参考文献：
1. [Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)
2. [浅谈MySQL并发控制：隔离级别、锁与MVCC](https://juejin.cn/post/6844904096378404872)
3. [一条更新语句在MySQL是怎么执行的](https://gsmtoday.github.io/2019/02/08/how-update-executes-in-mysql/)
4. [mysql的binlog | redolog | undolog](https://juejin.cn/post/7033035066829701134)