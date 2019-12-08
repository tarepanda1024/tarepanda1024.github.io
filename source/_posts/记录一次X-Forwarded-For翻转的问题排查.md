---
title: 记录一次X-Forwarded-For翻转的问题排查
categories: 网关
date: 2019-12-08 12:36:20
tags:
---


## 背景

最近严选的服务在上云，上云后由于IP是不固定的，那么通过IP黑白名单鉴权的手段已经不太能够满足现实情况了。  

打算通过在Header中放入服务编码的方式来进行鉴权。如果是云内的服务A调用云内的服务B，那么服务A的sidecar(istio-envoy)就可以向请求中注入服务编码信息；如果是从云外访问云内的话，这件事情由边缘网关（基于kong）来做比较合适。

网关小组为此开发了一个插件，插件做的事情是拿到downstream的IP，然后从CMDB中查到这个IP对应的服务编码，最终把服务编码信息放到Header中传到业务方。  

插件运行了一段时间，有一个业务反馈拿不到服务编码。网关小组就是在此情况下开始了问题的排查。


## 排查过程

我们先看一下请求的流程：

![请求流程](/images/2019120700.png)


然后看一下边缘网关的accesslog：  

![边缘网关accesslog日志](/images/2019120701.jpg)

可以看到：remote address是：10.xx.xx.114 XFF中的信息为：10.xx.xx.120 。那么业务方收到的XFF是什么呢？是：10.xx.xx.120,10.xx.xx.114

到这里可以看到问题有三个：
1. 为什么网关收到到remote address是：10.xx.xx.114，而不是预想的10.xx.xx.120 ？
2. 为什么网关收到的XFF是10.xx.xx.120 ，而不是预想的：客户端IP，10.xx.xx.114 ？
3. 为什么业务方收到的XFF不对并且是翻转的呢？

感觉这三个问题都比较诡异。因为有不少业务都已经上云了，所以第一反应边缘网关应该没有问题。同时根据全链路ID的组成来看：```10.xx.xx.36(yanxuan-ianus)^xxxxx^9313374```，此ID是由网关生成的，前面的节点没有接入全链路监控。怀疑node层（consul-nginx作为sidecar）和边缘网关之间可能还有一层。  

假设两者中间还经过了10.xx.xx.114这台入口nginx，那么第一个和第二个问题勉强说的通了（不过xff还有点问题，没有客户端IP）。可是边缘网关收到的Host头是：online.xxx.mail.saas，这种Host应该是不经过入口nginx的。还是先找sa查一下吧。      
SA很快给出了回复，首先这种域名是不经过入口nginx的，其次查了一下这个点的日志，可以看到有这条请求是从客户端直接到node层的，客户端IP是：183.xx.xx.131 。

看来node层和边缘网关中间应该没有其他节点了。  

为了确认猜测，需要看一下node机器上的consul-nginx日志，consul-nginx作为sidecar，对所有的请求头都是透传的，应该没有修改。没有node机器的权限，委托另外一个同事查了一下，如下：  
![consul-nginx日志](/images/2019120702.jpg)

consul-nginx显示：下一跳直接到边缘网关了，没有经过其他节点。同时consul-nginx收到的xff是：10.xx.xx.120 。这个日志说明:
1. 网关获取remote address的方式有问题
2. node层可能对xff头做了覆盖操作，不然consul-nginx显示对xff应该是：183.xx.xx.131。

边缘网关日志中打印的是：```$remote_addr```，仔细查一下官方文档，可以看到这个变量的意思是客户端IP http://nginx.org/en/docs/http/ngx_http_core_module.html , 没有明确说明一定是上一跳的地址。  
查了相关资料后发现：real_ip_modul可能会更改client address。同时网关是基于kong的二次开发，kong默认开启了real_ip模块，对比之后可以看出很有可能是real_ip_module修改了remote_addr这个变量的值。

> The ngx_http_realip_module module is used to change the client address and optional port to those sent in the specified header field.

在real_ip_module的文档中，可以看到http://nginx.org/en/docs/http/ngx_http_realip_module.html 

> Syntax:	real_ip_header field | X-Real-IP | X-Forwarded-For | proxy_protocol;  
> Default:	real_ip_header X-Real-IP;  
> Context:	http, server, location  

默认是从X-Real-IP中取值赋值到remote_addr中的。为了取到原始的remote_addr值，可以用realip_remote_addr这个变量。这个点以前没有注意到。

到这里脉络基本清晰了：应该是网关从x-real-ip头中取到了了一个remote_addr，然后追加到xff中，导致应用服务收到了xff是翻转的。同时Node层可能追加了一个x-real-ip，并且对xff做了覆盖重写。

Node层的日志看着很简单，只打印了简单的请求URL以及目标服务的consul-nginx地址。后来问了一个前端大佬，从pm2工具隐藏的日志中看到了采样的部分请求的完整请求头，从中明确的看到了，入口nginx传了一个X-real-ip，node层追加了入口nginx的地址到x-real-ip中同时对xff做了清除，赋值本机IP到xff中。现在问题已经完全明了了。

## 结论

node层传到网关的xff为本机IP，x-real-ip为：客户端IP，入口nginxIP ；边缘网关根据x-real-ip中拿到了入口nginxIP作为客户端IP，然后追加到xff中；导致应用看到的xff好像是翻转的。

解决方法的话就是网关需要采用realip_remote_addr变量来获取和网关直连的节点IP；Node层需要明确自身的地位：如果node作为代理层，就需要正确追加xff；如果node层把自己作为应用服务器，就需要完全清除代理服务器相关的Header。

另外就是排查过程中也暴露了一些问题：
1. 对nginx对相关变量和参数对了解程度还不够，其他业务没有反应这个问题主要是因为链路不太一致，也有可能没有看到这个问题，不能靠巧合编程。
2. node层接入的全链路sdk应该有问题，导致链路追踪不全面；同时node层打印的accesslog日志也不够清晰，关键信息丢失不完整。

