---
title: >-
  ApacheDoris异常案例：The statement has been forwarded to master FE(10.x.x.235) and
  failed to execute
tags:
  - doris
categories:
  - doris
date: 2023-05-15 19:33:01
updated:
---


> apache doris fe节点可以通过部署奇数个Follower构成高可用集群，集群中的master角色，会从这些Follower中选举产生。

> 如果你并不十分了解 FE 元数据的运行逻辑，或者没有足够 FE 元数据的运维经验，我们强烈建议在实际使用中，只部署一个 FOLLOWER 类型的 FE 作为 MASTER，其余 FE 都是 OBSERVER，这样可以减少很多复杂的运维问题！后面这句话，来自官方文档：[ApacheDoris元数据运维-最佳实践](https://doris.apache.org/zh-CN/docs/dev/admin-manual/maint-monitor/metadata-operation#%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5) 说的很中肯，最近连着遇到了两次元数据的问题😂

## 起因
业务方抛过来一张图：
![异常堆栈](/images/20230515/01/01.png)
表示在通过jdbc执行```alter table```语句时，发生了如下的异常。让我帮忙排查排查。

## 分析

因为```alter table``` 语句，涉及到元数据的修改，原则上只能在master节点上修改，难道是master节点出了问题？```show proc '/frontends'```看看：

![fronts](/images/20230515/01/02.png)

图中显示：
- 当前的master节点是243不是235
- fe节点状态都是正常的

看着应该是有节点的信息存在问题，识别错误master节点了。于是从其余的几个fe节点上排查日志，根据```The statement has been forwarded to master ```这个日志，进行查找，没有找到相同的问题。看了一下日志，从Error和Warn也没有看到日有用的信息。那么到底是谁转发的呢？求助与社区，社区建议将235节点drop掉，再加入。这。。。还是自力更生，从代码中看看有没有关键信息吧：
```java
//doris查询处理流程：
ConnectProcessor#dispatch -> ConnectProcessor#handleQuery -> StmtExecutor#execute
```
快到关键地方了：
```java
if (isForwardToMaster()) {
    if (isProxy) {
        // This is already a stmt forwarded from other FE.
        // If goes here, which means we can't find a valid Master FE(some error happens).
        // To avoid endless forward, throw exception here.
        throw new UserException("The statement has been forwarded to master FE("
                + Catalog.getCurrentCatalog().getSelfNode().first + ") and failed to execute" +
                " because Master FE is not ready. You may need to check FE's status");
    }
    forwardToMaster();
    if (masterOpExecutor != null && masterOpExecutor.getQueryId() != null) {
        context.setQueryId(masterOpExecutor.getQueryId());
    }
    return;
} else {
    LOG.debug("no need to transfer to Master. stmt: {}", context.getStmtId());
}
```
判断这条语句是否需要转发的master执行，那么alter语句是需要转到master的，同时本机不是正确的master。继续往下看，到```MasterOpExecutor#forward```方法：
```java
if (null != ctx.queryId()) {
    params.setQueryId(ctx.queryId());
}

LOG.info("Forward statement {} to Master {}", ctx.getStmtId(), thriftAddress);

boolean isReturnToPool = false;
```
出现了一条日志，很关键，按图索骥，一台一台机器查找，最终在150这台observer上见到了：
![dorisfe异常](/images/20230515/01/03.png)
可以看到这台机器将请求转发到了235，可是243才是master节点啊。说明这个节点记录的master信息有问题。

## 解决办法
执行重启大法。重启后，再看日志:
![dorisfe正常](/images/20230515/01/04.png)
转发正常了。

## 后续
此次解决了问题，但是具体为什么不一致，还没有排查出来。没想到，没过几天，又发生了类似的问题，预知后续如何，且待下回分解。。。
