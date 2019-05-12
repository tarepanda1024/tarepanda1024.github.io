---
title: Redis4.0安装及集群搭建
date: 2017-11-29 19:25:34
tags:
categories: Redis
---

环境:Deepin15 (Debian8)

## Redis4.0安装

###  1. 下载redis
```
wget http://download.redis.io/releases/redis-4.0.2.tar.gz
```

### 2.  编译安装
```
cp ~/redis-4.0.2.tar.gz  /usr/local/
tar -xzvf redis-4.0.2.tar.gz
cd redis-4.0.2
make

```
编译完成后会提示执行 ```make  test``` 命令，执行后可能报错，提示缺少tcl，那么我们需要安装一下
```
apt-get install tcl
```
不报错之后，执行 ```make install``` 命令;这样就安装好了
### 3.  测试
执行 ```redis-server```命令,就采用默认配置启动了,而后，我们执行```redis-cli```进入交互命令行，可以通过```ping```命令看看是否安装成功，通过```shutdown```命令关闭redis服务

## 集群配置
我们在单机上采用三主三从配置
###  1.  在redis4.0目录下建立cluster文件夹，并建立６个redis-conf文件夹
```
mkdir cluster
cd cluster
mkdir 7000 7001 7002 7003 7004 7005
cp ../redis.conf 7000/
cp ../redis.conf 7001/
cp ../redis.conf 7002/
cp ../redis.conf 7003/
cp ../redis.conf 7004/
cp ../redis.conf 7005/
```
### 2 . 对每个redis.conf文件，修改以下几个条目，注意端口号和nodes文件名称要和文件夹保持一致
```
port 7000
cluster-enabled yes
cluster-config-file nodes_7000.conf
cluster-node-timeout 5000
appendonly yes
daemonize yes
```
修改完成后切换到每个目录中，执行
```
redis-server redis.conf
```
命令（可以写个bash文件批量处理）
### ３. 安装ruby环境
```
apt-get install ruby
gem install redis
```
### ４.　切换到redis-4.0.2的src目录中，执行以下命令  
```bash
./redis-trib.rb  create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```
执行```./redis-trib.rb check 127.0.0.1:7000 ```来判断测试是否正常
最后我们可以通过``` redis-cli -c -p 7000``` 来连接集群