---
title: kong中的trusted_ips、real_ip_header和real_ip_recursive
tags: ['Kong','Nginx']
categories: 网关
date: 2018-10-28 16:42:03
---

> 最近在将一个业务流量接入网关时遇到一个问题，本来应该是https请求，经由网关到应用服务器得到重定向url时变成了http请求。问了相关同事，发现他们是根据```X-Forwarded-Proto```头来判断协议进行转换的。流量从入口nginx到网关再到应用服务器时，入口nginx的XFP头为https，网关把```X-Forwarded-Proto```协议头覆盖掉了。于是就看了一下kong的XFP头处理逻辑。

kong的nginx_template中发往upstream的location中有这么一句话
```
proxy_set_header   X-Forwarded-Proto $upstream_x_forwarded_proto;
```
说明网关传往应用服务器时带了XFP头，不过带的是http,然后再看一下kong是怎么修改upstream_x_forwarded_proto变量的，在kong/core/hander.lua中，
```lua
     local realip_remote_addr = var.realip_remote_addr
     local trusted_ip = singletons.ip.trusted(realip_remote_addr)
      ......
      if trusted_ip then
        var.upstream_x_forwarded_proto = var.http_x_forwarded_proto or
                                         var.scheme

        var.upstream_x_forwarded_host  = var.http_x_forwarded_host  or
                                         var.host

        var.upstream_x_forwarded_port  = var.http_x_forwarded_port  or
                                         var.server_port

      else
        var.upstream_x_forwarded_proto = var.scheme
        var.upstream_x_forwarded_host  = var.host
        var.upstream_x_forwarded_port  = var.server_port
      end

```
kong先判断realip_remote_addr是否在可信ip范围内，是的话才会将XFP头设置为从上方带过来的XFP头，否则的话就用var.scheme，由于网关这里没有设置trust_ip，而入口nginx到网关采用的协议为http，从而导致网关将XFP设置为了http。找到了地方，那么我们只需要设置一下trust_ip就ok。trust_ip是在kong.conf文件中设置的。由于kong用的nginx原生的可信Ip模块，所以我们可以参照nginx原生的配置,
http://nginx.org/en/docs/http/ngx_http_realip_module.html#real_ip_recursive 这个是nginx的real_ip模块文档。

```
#kong.conf文件
trusted_ips = 192.168.0.1/16, 10.0.0.1/8, 127.0.0.1                  # Defines trusted IP addresses blocks that are

#real_ip_header = X-Real-IP      # Defines the request header field whose value


#real_ip_recursive = off         # This value sets the ngx_http_realip_module

```
- trusted_ips 决定kong是否会从XFF头或者 X-Real-IP头中获取客户端ip(也决定了是否使用传过来的XFP头)，为了安全，只会从可信IP中传过来的请求才会取这个头

- real_ip_header 这个决定从哪里获取客户端ip

- real_ip_recursive real_ip_recursive 是否递归地排除直至得到用户ip（默认为off） 

首先，real_ip_header 指定一个http首部名称，默认是X-Real-Ip，假设用默认值的话，nginx在接收到报文后，会查看http首部X-Real-Ip。

（1）如果有1个IP，它会去核对，发送方的ip是否在set_real_ip_from指定的信任ip列表中。如果是被信任的，它会去认为这个X-Real-Ip中的IP值是前代理告诉自己的，用户的真实IP值，于是，它会将该值赋值给自身的$remote_addr变量；如果不被信任，那么将不作处理，那么$remote_addr还是发送方的ip地址。

（2）如果X-Real-Ip有多个IP值，比如前一方代理是这么设置的：proxy_set_header X-Real-Ip $proxy_add_x_forwarded_for;

得到的是一串IP，那么此时real_ip_recursive 的值就至关重要了。nginx将会从ip列表的右到左，去比较set_real_ip_from 的信任列表中的ip。如果real_ip_recursive为off，那么，当最右边一个IP，发现是信任IP，即认为下一个IP（右边第二个）就是用户的真正IP；如果real_ip_recursive为on，那么将从右到左依次比较，知道找到一个不是信任IP为止。然后同样把IP值复制给$remote_addr。   

在本次的问题中，只需要给trusted_ips设置一下信任Ip列表即可。

