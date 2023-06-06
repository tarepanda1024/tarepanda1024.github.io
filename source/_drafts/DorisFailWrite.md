---
title: 记一次doris异常处理过程：failed to init delta writer. version count: 1001, exceed limit: 500
date: 2023-05-16 20:01:36
updated:
tags:
  - doris
categories:
  - doris
---

> apache doris fe节点可以通过部署奇数个Follower构成高可用集群，集群中的master角色，会从这些Follower中选举产生。

> 如果你并不十分了解 FE 元数据的运行逻辑，或者没有足够 FE 元数据的运维经验，我们强烈建议在实际使用中，只部署一个 FOLLOWER 类型的 FE 作为 MASTER，其余 FE 都是 OBSERVER，这样可以减少很多复杂的运维问题！后面这句话，来自官方文档：[ApacheDoris元数据运维-最佳实践](https://doris.apache.org/zh-CN/docs/dev/admin-manual/maint-monitor/metadata-operation#%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5) 说的很中肯，最近连着遇到了两次元数据的问题😂

## 背景
早晨收到大量的```failed to init delta writer. version count: 1001, exceed limit: 1000```的报警，影响到数据的导入，不少实时任务跑失败了，急需进行处理。
![/images/20230529/01.png](报警图片)
![/images/20230529/02.png](报警图片)

## 分析
针对这种异常，首先看一下tablet各个副本的状态：```show tablet xxx```，再执行tablet详情中的链接
![/images/20230529/03.png](报警图片)



## 
