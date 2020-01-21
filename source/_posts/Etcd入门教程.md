---
title: Etcd学习笔记
date: 2020-01-13 20:08:27
tags:
---



Etcd是CoreOS开发的一个高可用、强一致性的分布式键值对数据库。采用Raft算法。可以用于服务发现。目前k8s的后端存储就用的Etcd。  
相对于Zookeeper来说，具有以下特点：    

- 简单：安装部署相对简单，用户友好支持http协议、支持grpc协议 
- 安全：支持tls认证
- 快速：1万次每次的写入速度

### 安装
- 环境：macOS Catalina
- 版本：etcd v3.4.3

为了快速体验Etcd，可以直接使用官方编译好的可执行文件：[Github下载页面](https://github.com/etcd-io/etcd/releases)。下载后解压可以看到Etcd以及Etcdctl两个可执行文件，前者是Etcd服务端，后者是一个命令行客户端。

启动一个本机单节点的etcd集群，直接执行```./etcd ```即可：

```
2020-01-04 17:10:16.654046 I | etcdmain: etcd Version: 3.3.18
2020-01-04 17:10:16.654162 I | etcdmain: Git SHA: 3c8740a79
2020-01-04 17:10:16.654166 I | etcdmain: Go Version: go1.12.9
2020-01-04 17:10:16.654170 I | etcdmain: Go OS/Arch: darwin/amd64
2020-01-04 17:10:16.654175 I | etcdmain: setting maximum number of CPUs to 8, total number of available CPUs is 8
2020-01-04 17:10:16.654181 N | etcdmain: failed to detect default host (default host not supported on darwin_amd64)
2020-01-04 17:10:16.654187 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
2020-01-04 17:10:16.655724 I | embed: listening for peers on http://localhost:2380
2020-01-04 17:10:16.655864 I | embed: listening for client requests on localhost:2379
2020-01-04 17:10:16.670602 I | etcdserver: name = default
2020-01-04 17:10:16.670645 I | etcdserver: data dir = default.etcd
2020-01-04 17:10:16.670655 I | etcdserver: member dir = default.etcd/member
2020-01-04 17:10:16.670659 I | etcdserver: heartbeat = 100ms
2020-01-04 17:10:16.670663 I | etcdserver: election = 1000ms
2020-01-04 17:10:16.670669 I | etcdserver: snapshot count = 100000
2020-01-04 17:10:16.670681 I | etcdserver: advertise client URLs = http://localhost:2379
2020-01-04 17:10:16.670689 I | etcdserver: initial advertise peer URLs = http://localhost:2380
2020-01-04 17:10:16.670700 I | etcdserver: initial cluster = default=http://localhost:2380
2020-01-04 17:10:16.779154 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
2020-01-04 17:10:16.779652 I | raft: 8e9e05c52164694d became follower at term 0
2020-01-04 17:10:16.779719 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
2020-01-04 17:10:16.779744 I | raft: 8e9e05c52164694d became follower at term 1
2020-01-04 17:10:16.850324 W | auth: simple token is not cryptographically signed
2020-01-04 17:10:16.888801 I | etcdserver: starting server... [version: 3.3.18, cluster version: to_be_decided]
2020-01-04 17:10:16.889057 I | etcdserver: 8e9e05c52164694d as single-node; fast-forwarding 9 ticks (election ticks 10)
2020-01-04 17:10:16.889634 E | etcdserver: cannot monitor file descriptor usage (cannot get FDUsage on darwin)
2020-01-04 17:10:16.890936 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
2020-01-04 17:10:17.086142 I | raft: 8e9e05c52164694d is starting a new election at term 1
2020-01-04 17:10:17.086212 I | raft: 8e9e05c52164694d became candidate at term 2
2020-01-04 17:10:17.086271 I | raft: 8e9e05c52164694d received MsgVoteResp from 8e9e05c52164694d at term 2
2020-01-04 17:10:17.086305 I | raft: 8e9e05c52164694d became leader at term 2
2020-01-04 17:10:17.086322 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
2020-01-04 17:10:17.087050 I | etcdserver: setting up the initial cluster version to 3.3
2020-01-04 17:10:17.097255 N | etcdserver/membership: set the initial cluster version to 3.3
2020-01-04 17:10:17.097308 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2020-01-04 17:10:17.097350 I | embed: ready to serve client requests
2020-01-04 17:10:17.097428 I | etcdserver/api: enabled capabilities for version 3.3
2020-01-04 17:10:17.100090 N | embed: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
```

日志中有几个关键的地方：

- name：etcd的名称，如果有多个etcd集群，需要修改名称
- data dir：etcd数据存储的地方
- heartbeat：心跳检测时间
- election：触发投票选举leader的时间

Etcd有两个版本的API，v2和v3，v3相对v2有很大的性能提升，建议直接使用v3版本。Etcdctl3.4.0版本采用v3版本的API和服务端交互，Etcdctl3.3及更早的版本默认采用v2版本的API和服务端交互，可以通过设置：  

```shell
export ETCDCTL_API=3
```

来采用v3版本的API和服务端交互。

本地测试可以通过

```shell
etcdctl put foo bar
```
设置一个key和value。

通过

```shell
etcdctl get fool
```
来获取value

### 基本操作

 Etcd作为一个数据库，基础的增删改查是少不了的
 
- 新增

```shell
etcdctl put foo bar
etcdctl put foo1 bar1
etcdctl put foo2 bar2
```

- 查找

```shell
# 打印出key和value
etcdctl get foo
#只打印出value
etcdctl get foo --print-value-only
#获取多个从foo到foo2
etcdctl get foo foo2
#根据前缀获取
etcdctl get --prefix foo

```

- 删除

删除key和查找key类似，支持各种前缀匹配以及范围匹配
```shell
etcdctl del foo
```

- 监控数据变化

```shell
etcdctl watch foo
```

### 总结
Etcd使用起来相对非常简单，下次要学习一下Etct 各个语言Client的使用。



### 参考文档
[Etcd官方教程](https://etcd.io/docs/v3.4.0/)  
[Etcd简介](https://blog.csdn.net/gaowenhui2008/article/details/95960873)  
[Etcd使用入门](https://www.jianshu.com/p/f68028682192)