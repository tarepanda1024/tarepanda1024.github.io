---
title: 微服务网关kong编译安装教程
tags: 教程
categories: 网关
date: 2018-03-13 17:02:32
---



[Kong](https://github.com/Kong/kong)是一个高性能的微服务网关。依赖于OpenResty、lua、luarocks和postgresql或者Cassandra。在服务器上安装时可能没有sudo权限，因此相关依赖我们都需要从源码安装
#### 首先我们安装OpenResty
之前我写过[一篇博客](http://xiaodongliu.com/2017/11/27/OpenResty%E5%AE%89%E8%A3%85%E8%AE%B0%E5%BD%95/)进行OpenResty安装。这次主要是添加一些配置参数
```
wget https://openresty.org/download/openresty-1.13.6.1.tar.gz

tar -xzvf openresty-1.13.6.1.tar.gz

cd  openresty-1.13.6.1

./configure \
  --with-pcre-jit \
  --with-ipv6 \
  --with-http_realip_module \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --with-http_v2_module \
  --prefix=/home/webedit/openresty
  
  make
  
  make install
```
同时需要配置环境变量
```
export PATH="$PATH:/home/webedit/openresty/bin"
export PATH="$PATH:/home/webedit/openresty/luajit/lib"
export PATH="$PATH:/home/webedit/openresty/luajit/include/luajit-2.1"
```
#### 其次我们需要安装luarocks(OpenResty中已经包含有luajit了)
```
wget https://github.com/luarocks/luarocks/archive/2.4.3.zip

unzip luarocks-2.4.3.zip

cd uarocks-2.4.3

./configure \
  --lua-suffix=jit \
  --prefix=/home/webedit/luarocks \
  --with-lua=/home/webedit/openresty/luajit \
  --with-lua-include=/home/webedit/openresty/luajit/include/luajit-2.1
```
配置环境变量
```
export PATH="$PATH:/home/webedit/luarocks/bin"
```

#### 接下来我们需要下载kong 
```
wget https://github.com/Kong/kong/archive/0.12.3.zip

unzip kong-0.12.3.zip

cd kong-0.12.3

/home/webedit/luarocks/bin/luarocks make
```
配置环境变量
```
export LUA_PATH="/home/webedit/luarocks/share/lua/5.1/?.lua"
export PATH="$PATH:/home/webedit/kong-src/bin"
```

#### 然后下载postgresql tar.gz包
```

mkdir data
 ./pgsql/bin/initdb -D data/ --locale=en_US.UTF-8 -U postgres -W 
 ./pgsql/bin/pg_ctl -D data/ start
  ./pgsql/bin/psql -U postgres
 postgres=# Alter USER postgres WITH PASSWORD '***密码**';  //添加密码  
ALTER ROLE        //出现这个才算成功，第一次操作没成功，pgadmin连不上  
```
进入命令行工具创建user
```
CREATE USER kong; CREATE DATABASE kong OWNER kong;
```
#### 最后修改Kong的配置文件( /home/webedit/kong-src/kong.conf)如下
```
# -----------------------
# Kong configuration file
# -----------------------


prefix = /home/webedit/kong/       # Working directory. Equivalent to Nginx's
                                 # prefix path, containing temporary files
                                 # and logs.
                                 # Each Kong process must have a separate
                                 # working directory.


proxy_listen = 0.0.0.0:9000     # Address and port on which Kong will accept


admin_listen = 127.0.0.1:9001   # Address and port on which Kong will expose


pg_host = 127.0.0.1             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.
pg_user = kong                  # The username to authenticate if required.
pg_password =                   # The password to authenticate if required.
pg_database = kong              # The database name to connect to.

```
这样就可以启动kong了
```
kong migrations up -c /home/webedit/kong-src/kong.conf
kong start -c /home/webedit/kong-src/kong.conf
curl -i http://localhost:9001/
``` 


