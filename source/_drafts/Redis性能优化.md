---
title: Redis性能优化实践
date: 2021-05-28 14:41:14
tags: ['Redis']
categories: Redis
---
Redis作为一款优秀的NoSql数据库，在秒杀、分布式锁、社交网络等场景都有广泛的应用。严选也采用Redis作为风控系统标签的主要存储。

风控系统是一个对实时性要求很高的系统，大部分请求的耗时要求控制在150ms以内，对于部分链路，比如下单场景，常常要求50ms以内返回结果。那么该如何优化系统，来降低慢请求率呢？下面主要总结了一些常用的慢请求排查思路和优化办法。

## 检查Redis状态
### info

### slowlog

### cluster info

### --bigkeys


## 检查Key设计是否合理


## 检查Key过期策略的配置


## 检查持久化策略


## 宿主机检查


## 读写分离

## 抛弃Sentinel，拥抱Cluster

## Lettuce的使用

