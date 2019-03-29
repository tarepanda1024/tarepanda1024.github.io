---
title: kong0.12.x源码分析(一)----kong目录结构解析
date: 2018-04-30 09:22:44
tags:
categories: Kong
---

最近需要在kong的基础上做api网关，因此有不少功能需要定制，以便更符合业务需要。然后发现kong在0.12以后变化挺大的，网上现有的kong的源码解析文章比较老了，于是打算做一系列kong的源码解析文章，一方面可以对自己的理解做一下梳理，另一方面也可以给其他同事做一些参考。因本人水平有限，错误在所难免，欢迎各位在下面留言，也可以通过邮箱和我交流: liuxiaodong2017@gmail.com lxd2017@163.com
kong的目录结构如下

```shell
kong
|__api
|__cluster_events
|__cmd
|__core
|__dao
|__plugins
|__templates
|__tools
|__vendor
|__cache.lua
|__cluster_events.lua
|__conf_loader.lua
|__constants.lua
|__init.lua
|__meta.lua
|__mlcache.lua
|__singletons.lua
```
kong的目录主要就这些，看起来蛮清晰的。kong是基于OpenResty，而OpenResty又是基于nginx的，所以，kong的启动流程和nginx很相似，也有start，stop，reload等命令。
#### api目录
这个目录主要的作用是基于lapis开发的一个简单web服务，提供一系列restful api借口，供管理平台使用
```shell
├── api_helpers.lua 
├── crud_helpers.lua
├── init.lua
└── routes
    ├── apis.lua
    ├── cache.lua
    ├── certificates.lua
    ├── consumers.lua
    ├── kong.lua
    ├── plugins.lua
    ├── snis.lua
    └── upstreams.lua
```
#### cmd目录
cmd目录下主要关注一下init.lua 。这个文件负责根据不同的指令分别调用start，stop，reload等lua脚本，这些脚本中则通过调用nginx的start，stop等命令完成服务的启动。
```shell
├── check.lua
├── health.lua
├── init.lua
├── migrations.lua
├── prepare.lua
├── quit.lua
├── reload.lua
├── restart.lua
├── roar.lua
├── start.lua
├── stop.lua
├── utils
│   ├── kill.lua
│   ├── log.lua
│   ├── nginx_signals.lua
│   └── prefix_handler.lua
└── version.lua
```
#### core目录
core目录下面放的是一些kong的核心操作。router.lua负责创建到upstreams的路由，采用hash表存储，提高查找效率。reports文件是向udp logger发送日志。hander.lua负责集群事件的发布和订阅处理。plugins_iterator.lua负责加载插件。balancer.lua负责做负载均衡。
```shell
├── balancer.lua
├── certificate.lua
├── error_handlers.lua
├── globalpatches.lua
├── handler.lua
├── plugins_iterator.lua
├── reports.lua
└── router.lua
```
#### dao目录
dao目录下主要放置的是数据库访问相关的操作。kong目前只支持psql和cassandra两种数据库，这点比较坑。db目录下的文件主要是封装了对具体数据库访问的增删改查操作。外面的dao.lua文件通过传入的db类型，生成链接，负责增删改查操作。也就是说dao.lua不关心具体的数据库实现，只是对具体的数据库访问做了一层封装。factory.lua是一个工厂类，负责管理生成数据库实例，初始化，更新数据等操作。migrations目录下则是具体的创建更新表的动作，每次升级都需要执行一次migrations操作。schemas下面放的是model文件，类似java中的bean文件。
```shell
├── dao.lua
├── db
│   ├── cassandra.lua
│   ├── init.lua
│   └── postgres.lua
├── errors.lua
├── factory.lua
├── migrations
│   ├── cassandra.lua
│   ├── helpers.lua
│   └── postgres.lua
├── model_factory.lua
├── schemas
│   ├── apis.lua
│   ├── consumers.lua
│   ├── plugins.lua
│   ├── ssl_certificates.lua
│   ├── ssl_servers_names.lua
│   ├── targets.lua
│   └── upstreams.lua
└── schemas_validation.lua

```
#### cluster_events
这个目录下放的是集群管理相关的事件存储，kong0.12.x后的集群管理改变很大。kong的集群管理主要是采用的发布订阅模式，中间会以数据库为媒介，定时去poll一下，以此维持心跳，看看有没有事件更新。集群管理后面会进行详细的分析。
```shell
└── strategies
    ├── cassandra.lua
    └── postgres.lua

```

#### tools
tools目录下放置的是一些常用的工具类，比如加密、dns服务、ip服务等
```shell
├── ciphers.lua
├── dns.lua
├── ip.lua
├── printable.lua
├── public.lua
├── responses.lua
├── timestamp.lua
└── utils.lua

```
#### vendor
这个目录下只有一个文件，主要是给lua提供面向对象的能力。kong的插件机制会用到面向对象的继承思想，这个后面我们会详细分析。
```bash
.
└── classic.lua
```
#### templates
templates顾名思义就是模板啦，下面主要放一些模板文件，kong_defaults.lua下面放的是一些kong的默认配置文件；nginx.lua主要放置nginx的一些启动参数；nginx_kong.lua放的是nginx的配置文件模板，kong启动时会以此为模板生成nginx.conf文件
```shell
.
├── kong_defaults.lua
├── nginx_kong.lua
└── nginx.lua

```




