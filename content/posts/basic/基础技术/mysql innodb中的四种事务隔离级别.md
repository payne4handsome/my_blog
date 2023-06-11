---
layout: post
title: "mysql innodb中的四种事务隔离级别"
date: 2022-10-06 21:28:47 +0800
categories: mysql 事务 隔离级别
---

# mysql innodb中的四种事务隔离级别

本文以实验的形式展示mysql Innodb引擎的四种事务隔离级别的影响。

## 四种隔离级别

| 隔离级别        | 脏读（Dirty Read）   |  脏读（Dirty Read）  |幻读（Phantom Read） |
| --------   | -----:  | :----:  |:----:  |
|未提交读（Read uncommitted）   | 可能|   可能     |可能    |
|已提交读（Read committed）        |  不可能|   可能     |可能    |
|可重复读（Repeatable read）      | 不可能|   不可能     |可能    |
|可串行化（Serializable ）   | 不可能|  不可能     |不可能    |

1. 未提交读(Read Uncommitted)：允许脏读，也就是可能读取到其他会话中未提交事务修改的数据
2. 提交读(Read Committed)：只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别 (不重复读)
3. 可重复读(Repeated Read)：可重复读。在同一个事务内的查询都是事务开始时刻一致的，InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读
4. 串行读(Serializable)：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞

## 详细说明

> 以下表（test）解释各个隔离级别，只有两个字段，一个id，一个account

![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-048285446d7c9fa6.png)


插入测试数据
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-530cc6e63eeb0cfe.png)
关闭mysql自动提交和设置隔离级别

+ 查看是否开启自动提交

```shell
show variables like 'autocommit';
```

+ 打开或关闭自动提交
  
```shell
set autocommit = 1;//打开
set autocommit = 0;//关闭
```

+ 查看数据库隔离级别

```shell
select @@tx_isolation;//当前会话隔离级别
select @@global.tx_isolation;//系统隔离级别
```

+ 设置数据库隔离级别(当前会话)

```shell
SET session transaction isolation level read uncommitted;
SET session transaction isolation level read committed;
SET session transaction isolation level REPEATABLE READ;
SET session transaction isolation level Serializable;
```

### 未提交读（Read uncommitted）

> 关闭自动提交、设置对应隔离级别，开启两个会话，下面不在赘述

![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-3a18f33cb3f342f6.png)

1. 会话A
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-afb7806b4c4f66de.png)
2. 会话B
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-3253ee508e9c5f85.png)
3. 会话A中插入一条记录，看B中情况

+ 会话A
![](/mysql%20innodb中的四种事务隔离级别/8596800-3f658fa9777e2b11.png)
+ 会话B
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-fe4f74a994c595a0.png)
+ 结论

我们发现会话A中事务并没有提交但是在会话B中却可以看到会话A中插入的记录，这种情况就是脏读。

### 已提交读（Read committed）

> 设置会话A、B隔离级别为已提交读, 目前会话A和会话B中都只有4条记录，如下：

![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-e6e5f51cfeb87fc3.png)

1. 会话A
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-5cb2a19166e59fb1.png)
2. 会话B
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-85216ece10dca65e.png)
3. 会话A commit以后，会话B的情况
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-e6a8ae7cba67fb2b.png)
4. 结论
当会话A中的事务没有提交的时候，会话B中是看不到A中插入的记录，不存在脏读的情况。但是当隔离级别为提交读(Read Committed)时候，会存在不可重复读的情况，实验如下：
5. 会话A和B开启事务，当A中插入一条记录并提交的情况中，会话B的事务中存在前后两次读取不一致的情况。

+ 会话A
![](/mysql%20innodb中的四种事务隔离级别/8596800-86a85a00a79bd7cb.png)
+ 会话B在A插入id=5这条记录的前后情况如下：

![A中没有commit之前](/mysql%20innodb中的四种事务隔离级别/8596800-f6d0f5d905fe0b95.png)

![A中commit之后](/mysql%20innodb中的四种事务隔离级别/8596800-1385a3a34769e085.png)

## 可重复读（Repeatable read）

> 会话A、B设置隔离级别为可重复读（Repeatable read）

1. 会话A
![会话A插入记录并提交前后情况](/mysql%20innodb中的四种事务隔离级别/8596800-42d94ae069ac7681.png)
2. 会话B
![会话A插入记录提交之前](/mysql%20innodb中的四种事务隔离级别/8596800-aa7c7f3626d6958f.png)
![会话A插入记录提交之后](/mysql%20innodb中的四种事务隔离级别/8596800-048ed6edc65bef2d.png)
3. 结论
我们发现无论是在会话A插入记录并提交之前还是提交之后，会话B中都看不到刚刚A插入的id=7的那条记录，既不存在在隔离级别为Repeatable read中的不可重复读的情况。无论A中插入、更新、删除，B中都是不可见的，即在Repeatable read级别下，B是可重复读的。我们都知道还有一个幻读的问题，为什么都可重复读了，还存在幻读的问题？mysql又是如何解决幻读的问题的呢？

### 幻读的定义

官网的定义：

```shell
The so-called phantom problem occurs within a transaction when the 
same query produces different sets of rows at different times. For 
example, if a [`SELECT`]
(https://dev.mysql.com/doc/refman/5.7/en/select.html "13.2.9 SELECT 
Syntax") is executed twice, but returns a row the second time that was
 not returned the first time, the row is a “phantom” row.
```

意思就是幻读指在同一个事务中，两次相同的查询结果集不同。那这个又和不可重复读有什么区别呢？**确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert**。

### 幻读问题

> 设置会话A和会话B的隔壁级别为可重复读（Repeatable read）

当前会话A和会话B的查询情况如下
会话A:
![session A](/mysql%20innodb中的四种事务隔离级别/8596800-ede1a586756dfcbd.png)
会话B：
![session B](/mysql%20innodb中的四种事务隔离级别/8596800-198954f38fe315fb.png)
下面我们复现一下幻读问题
会话A:
![会话A](/mysql%20innodb中的四种事务隔离级别/8596800-fe69486fc00fc342.png)
会话B(插入一条记录)：
![image.png](/mysql%20innodb中的四种事务隔离级别/8596800-7913c7536c5b465f.png)
我们再来看看会话A中情况，我们看看加锁读和不加锁读的区别：
会话A:
![不加锁读](/mysql%20innodb中的四种事务隔离级别/8596800-453eafe525858f50.png)
![加锁读](/mysql%20innodb中的四种事务隔离级别/8596800-3c1ca42b6f9ded0e.png)
我们发现在不加锁时候，是可以重复读的，加锁时候读到了额外的一条记录，这个我们就称之为幻读。那么mysql如何解决幻读的问题呢？答案是gap锁，确切的说是next-key lock。**nexy-key lock = record lock + gap lock**。比如上面的例子，我们在会话A中执行这条语句的时候（select * from test where account=300;）时候加锁lock，如下：（**select * from test where account=300 for update;**）。那么会话B在插入（4，300）时候会被阻塞，因为有gap锁。这里因为我们没有在account上加上索引，所以整个表都会被锁（准确的说是accout整个范围都会被锁）。那么mysql何时获取next-key lock？

### 何时获取next-key lock

官网描述如下：
```shell
+ For locking reads (SELECT with FOR UPDATE or LOCK IN SHARE 
MODE), UPDATE, and DELETE statements, locking depends on 
whether the statement uses a unique index with a unique search 
condition, or a range-type search condition.
+ For a unique index with a unique search condition, InnoDB locks only
 the index record found, not the gap before it.
+ For other search conditions, InnoDB locks the index range scanned, 
using gap locks or next-key locks to block insertions by other sessions 
into the gaps covered by the range. For information about gap locks and
 next-key locks, see Section 15.5.1, “InnoDB Locking”.
```

也就是locking reads，UPDATE和DELETE时，除了对唯一索引的条件外都会获取gap锁或next-key锁。 当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围

## 可串行化（Serializable ）

这个级别很简单，读加共享锁，写加排他锁，**读写互斥**。使用的悲观锁的理论，实现简单，数据更加安全，但是并发能力非常差。如果你的业务并发的特别少或者没有并发，同时又要求数据及时可靠的话，可以使用这种模式。**select在这个级别在Serializable这个级别，还是会加锁的！**

>**mysql 的隔离级别最难理解的地方在可重复读和幻读的区别，我虽然想尽力去把这里说明白，但是写的时候发现还是很难去描述清楚，这里我也看了很对的blog，也没有发现能把mysql的锁和隔离级别各个方面都讲的很明白的地方，所以要想搞明白这个问题，还是得都看一些资料，集众家之长，下面是我看的比较好的几篇blog**


[Innodb中的事务隔离级别和锁的关系-来自美团的技术团队](https://tech.meituan.com/2014/08/20/innodb-lock.html)

[mysql REPEATABLE READ对幻读的解决](https://my.oschina.net/hebaodan/blog/910443)

[官网-幻读](https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html)

[官网-事务隔离级别](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)

[官网-innodb锁](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks)

我想你把上面的几篇文章都看完了，应该就能理解了















