---
title: 读书笔记：《MySql是怎样运行的》 事务及MVCC
tags:
  - 数据库
  - Mysql
  - 读书笔记
categories: Mysql
date: 2021-06-19 22:39:54
---


# 事务特性

ACID：原子性、一致性、隔离性和持久性

**原子性**：要执行的事务是一个独立的操作单元，要么全部执行，要么全部不执行

**一致性**：事务的一致性是指事务的执行不能破坏数据库的一致性，一致性也称为完整性。一个事务在执行后，数据库必须从一个一致性状态转变为另一个一致性状态。

**隔离性**：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。

**持久性（durability）**：事务一旦提交，则其所有的修改将会保存到数据库当做。即使此时系统崩溃，修改的数据也不会丢失。

# 事务并发执行的一些一致性问题

## 脏写

如果一个事务修改了另一个未提交事务修改过的数据，说明发生了脏写现象。

## 脏读

如果一个事务读到了另外一个未提交事务修改过的数据，说明发生了脏读现象。

1、在~~事务~~A执行过程中，事务A对数据资源进行了修改，事务B读取了事务A修改后的数据。

2、由于某些原因，事务A并没有完成提交，发生了RollBack操作，则事务B读取的数据就是脏数据。

![](https://tcs.teambition.net/storage/3126e9d12a3aaab471ec4ecc7581881d6b3d?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjZlOWQxMmEzYWFhYjQ3MWVjNGVjYzc1ODE4ODFkNmIzZCJ9.2aNLNNkv04C5IExBArq0Vmqrur-lQdCz8jxuRO5EBVo&download=image.png "")

## 不可重复读

事务B读取了两次数据资源，在这两次读取的过程中事务A修改了数据，导致事务B在这两次读取出来的数据不一致。

这种在同一个事务中，前后两次读取的数据不一致的现象就是不可重复读(Nonrepeatable Read)。

![](https://tcs.teambition.net/storage/3126e416e7528f61465eaaa6795777d1c06b?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjZlNDE2ZTc1MjhmNjE0NjVlYWFhNjc5NTc3N2QxYzA2YiJ9.3XxG61XMVdwdaLW6wmq8C9zFJY-RKzUbO0rYJM9oHco&download=image.png "")



## 幻读

事务B前后两次读取同一个范围的数据，在事务B两次读取的过程中事务A新增了数据，导致事务B后一次读取到前一次查询没有看到的行。

幻读和不可重复读有些类似，但是幻读强调的是集合的增减，而不是单条数据的更新。

![](https://tcs.teambition.net/storage/3126f7af624855b76eaabf01a901d5f1c937?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjZmN2FmNjI0ODU1Yjc2ZWFhYmYwMWE5MDFkNWYxYzkzNyJ9.rf_Bc-dp0uxr3o2KxJ83-GAaE1cTgPaK5sNJsF2lMC0&download=image.png "")

## 第一类更新丢失

事务A和事务B都对数据进行更新，但是事务A由于某种原因事务回滚了，把已经提交的事务B的更新数据给覆盖了。这种现象就是第一类更新丢失

![](https://tcs.teambition.net/storage/3126642479e16b16f324701e246ec2d9d40b?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjY2NDI0NzllMTZiMTZmMzI0NzAxZTI0NmVjMmQ5ZDQwYiJ9.xrnrFg8GR2LVKh7UQC6OGhTFGIzEdNLSXxO1wNUFCOw&download=image.png "")

## 第二类更新丢失

其实跟第一类更新丢失有点类似，也是两个事务同时对数据进行更新，但是事务A的更新把已提交的事务B的更新数据给覆盖了。这种现象就是第二类更新丢失。

![](https://tcs.teambition.net/storage/312643396a9d686394586eee7063e1aa11f4?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjY0MzM5NmE5ZDY4NjM5NDU4NmVlZTcwNjNlMWFhMTFmNCJ9.pCqTzQlTSIl3qGET2uclyMpBduVx1DPXodN8dsWcB9g&download=image.png "")




# 事务级别

![](https://tcs.teambition.net/storage/31261f82595d8e1c30b3ba07c53632fa946d?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTYyNDcxNjU3OSwiaWF0IjoxNjI0MTExNzc5LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMjYxZjgyNTk1ZDhlMWMzMGIzYmEwN2M1MzYzMmZhOTQ2ZCJ9._hdRry85S-fZSvrDa87xIrFahCbG5S5j3Q4UnWKwI6c&download=image.png "")

## Mysql默认事务级别

可重复读

# MVCC原理

> 针对读未提交和可重复读级别生效

## 版本链

聚簇索引记录中包含两个必要的隐藏列，一个是trx_id，另外一个是roll_pointer：

trx_id：一个事务每次对某条聚簇索引记录进行改动时，都会把该事务的事务id复制给trx_id隐藏列。

roll_pointer：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中。这个隐藏列相当于一个指针，通过它可以找到该记录修改前的信息。

每对记录进行一次修改，都会将旧值放到一条undo日志。每条undo日志都有一个roll_pointer属性。（insert操作对应的undo日志没有此操作，因为没有更早的版本）。通过这个属性将undo日志串成一个链表。称为版本链。同时每个版本还包含生成该版本时对应的事务Id。

## ReadView

读未提交级别：直接读取该记录的最新版本就好。

序列化级别：采用加锁方式来访问记录。

读已提交和可重复读级别：读取的都是已提交事务修改的记录。

MVCC用于读已提交和可重复度级别。核心问题是：需要判断版本链中的哪个版本是当前事务可见的。

ReadView主要包含四个重要的内容：

- m_ids：在生成ReadView时，当前系统中活跃的读写事务的事务id列表

- min_trx_id：在生成ReadView时，当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值。

- max_trx_id：在生成ReadView时，系统应该分配给下一个事务的事务Id值。

- create_trx_id：生成该ReadView的事务的事务Id。

判断某个版本是否可见的步骤：

1. 如果被访问版本的trx_id和ReadView中的create_trx_id相同，说明当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。

1. 如果被访问版本的trx_id比ReadView中的min_trx_id小，说明生成该版本的事务在当前事务生成ReadView前已提交，所以该版本可以被当前事务访问。

1. 如果被访问版本的trx_id属性大于或者等于ReadView中的max_trx_id，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。

1. 如果被访问版本的trx_id属性位于ReadView的min_trx_id和max_trx_id之间，则需要判断trx_id属性是否在m_ids列表中。如果在，说明创建ReadView时生成该版本的事务是活跃的，该版本不可以被访问。如果不在，说明创建ReadView时，该版本已提交，该版本可以被访问。

如果某个版本的数据对当前事务不可见，则沿着版本链找到下一个版本的数据，并沿用上面的步骤判断可见性。

### 读已提交生成ReadView的时机

每次查询开始都会生成一个独立的ReadView

### 可重复读生成ReadView的时机

只会在第一次查询的时候生成一个ReadView，之后的查询就不会重复生成ReadView了。

### 二级索引与MVCC

只有聚簇索引记录中才有trx_id和roll_pointer隐藏列，如果某个查询语句通过二级索引查询，需要通过如下步骤：

步骤一：

判断ReadView中min_trx_id的值，是否比二级索引页面PageHeader部分的PAGE_MAX_TRX_ID属性值大，如果是，说明可见；否则需要按照步骤二进行回表操作。

步骤二：

利用二级索引记录中的主键进行回表操作，得到对应的聚簇索引记录后再按照前面讲过的方式找到对该ReadView可见的第一个版本，然后判断相应的条件是否一致。

# 参考

[MySql是怎样运行的](https://book.douban.com/subject/35231266/)

[__https://blog.csdn.net/zl1zl2zl3/article/details/88542775__](https://blog.csdn.net/zl1zl2zl3/article/details/88542775)

[__https://zhuanlan.zhihu.com/p/347587789__](https://zhuanlan.zhihu.com/p/347587789)

[__https://zhuanlan.zhihu.com/p/117476959__](https://zhuanlan.zhihu.com/p/117476959)

