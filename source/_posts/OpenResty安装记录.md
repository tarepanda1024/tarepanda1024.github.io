---
title: OpenResty 安装记录
categories: 网关
date: 2017-11-27 17:29:39
tags:
---

最近风控项目需要用到OpenResty，于是打算在本地环境中搭建一套，然而搭建过程并没有那么顺利
官网的安装步骤很简单
### 1.安装相关依赖包
```
apt-get install libreadline-dev libncurses5-dev libpcre3-dev  libssl-dev perl make build-essential
```
### 2.下载源码并解压，然后编译安装
```
tar -xzvf openresty-VERSION.tar.gz
cd openresty-VERSION/
./configure
make
sudo make install
```
我的环境是Deepin系统，Deepin15采用的Debian8内核，按照官网的步骤时，系统会默认安装openssl1.1.0，然而OpenResty１.13版本暂时不支持openssl1.1.0，所以在安装过程中会报错，所以我们需要先安装openssl 1.0.2

###  0.  如果本机已经有openssl了，需要先卸载
```
openssl version
apt-get purge openssl
rm -rf /etc/ssl #删除配置文件 
```
###  1.  到openssl官网下载openssl1.0.2版本
[OpenSSl 1.0.2](https://www.openssl.org/source/) 
###  2.  编译安装
```
./config  --prefix=/usr/local --openssldir=/usr/local/ssl
make && make install
./config shared --prefix=/usr/local --openssldir=/usr/local/ssl
make clean
make && make install
```
###  3. 安装好之后需要将nginx放到环境变量中
```
PATH=/usr/local/openresty/nginx/sbin:$PATH
export PATH
```

