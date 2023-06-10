---
layout: post
title: "mysql 杂七杂八记录"
date: 2022-10-6
categories: 
    - "数据库"
tags: [mysql]
author: "pan"
---

## mysql 命令

+ 加共享锁

```sql
select * from table where ? lock in share mode;
```

+ 排它锁
  
```sql
select * from table where ? for update;
```

+ 查询当前运行的所有事务

```sql
select * from information_schema.innodb_trx\G
```

+ 显示用户正在运行的线程

```sql
show processlist\G
```

+ 查看是否开启自动提交

```sql
show variables like 'autocommit';
```

+ 打开或关闭自动提交

```sql
set autocommit = 1;//打开
set autocommit = 0;//关闭
```

+ 查看数据库隔离级别

```sql
select @@tx_isolation;//当前会话隔离级别
select @@global.tx_isolation;//系统隔离级别
```

+ 设置数据库隔离级别(当前会话)

```sql
SET session transaction isolation level read uncommitted;
SET session transaction isolation level read committed;
SET session transaction isolation level REPEATABLE READ;
SET session transaction isolation level Serializable;
```

+ 查询bin_log 是否开启

```sql
show variables like 'log_bin';
```
