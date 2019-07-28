---
title: API网关Kong云端部署调研
date: 2019-07-10 19:28:49
tags:
categories: 网关
---
调研的版本如下：

- Kong: v1.2.1
- KongIngressController: v0.5.0
- Minikube: v1.2.0

## 设计思想

Kong主要通过KongIngressController来支持云端网关。KongIngressController总体设计思想是监听k8s的APIServer，将k8s的Ingress资源转换成Kong的配置。

### 无数据库模式

无数据库模式下，kongIngressController作为sidecar和Kong实例一起部署在数据平面。
![dbless-deployment](/images/2019071019/dbless-deployment.png)

#### 高可用
在此种模式下，通过增加副本数就可以达到高可用


### 依赖数据库模式
当有数据库时，KongIngressController单独部署在控制平面，监听ApiServer变更，将数据存储到数据库中。Kong实例依然在数据平面，从数据库中拉取数据。

![db-deployment](/images/2019071019/db-deployment.png)

#### 高可用

此种模式，Kong和KongIngressController分别部署在控制平面和数据平面，可以单独扩容。KongIngressController可以存在多个，但是同一时刻只有一个master节点。Kong实例可以部署成DaemonSet，也可以采用自动扩容的策略进行高可用。

## 安装使用

### 启动k8s集群
```bash
minikube start
minikube dashboard
```
### 创建KongIngressController
> 此种方式创建的是带有数据库模式
```bash
curl -sL https://bit.ly/kong-ingress | kubectl create -f -
```
![KongIngress](/images/2019071019/kong-ingress-k8s.png)

### 创建服务
```bash
curl -sL https://bit.ly/echo-server | kubectl apply -f -
```
### 创建Ingress
```bash
 echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo
spec:
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: echo
          servicePort: 80
" | kubectl apply -f -
```
### 查看IP
```bash
minikube service -n kong kong-proxy --url
```

![minikube-ip](/images/2019071019/minikube-ip.png)
第一个是Http端口，第二个是Https端口

### 测试echo服务
```bash
 curl -i http://192.168.99.100:32649/foo
```
此时就可以看到返回的集群信息


### 添加插件
> KongIngressController自定义了许多资源，现在添加一个request-id插件测试一下

```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-id
config:
  header_name: my-request-id
plugin: correlation-id
" | kubectl apply -f -
```

```bash
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-example-com
  annotations:
    plugins.konghq.com: request-id
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /bar
        backend:
          serviceName: echo
          servicePort: 80
" | kubectl apply -f -
```
此时访问
```bash
curl -i -H "Host:example.com" http://192.168.99.100:32649/bar
```
可以看到返回的消息体中就带有my-request-id了

![requestId](/images/2019071019/requestId.png)

plugin插件除了可以添加到ingress上面，还可以添加到service上面，这里就不细说了。

## CRD自定义资源
> KongIngressController自定义了一些资源，以便实现自己的功能。主要有KongPlugin、KongIngress、KongConsumer和KongCredential

### KongPlugin
> KongPlugin的含义和非Cloud环境下Kong的含义一样，都是Kong的插件

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: <object name>
  namespace: <object namespace>
  labels:
    global: "true" # 可选，是否为全局插件
consumerRef: <name of an existing consumer> #可选，指定Consumer
disabled: <boolean>  # 是否开启此插件
config: #插件本身的配置项
    key: value
plugin: <name-of-plugin> # 插件名称
```

### KongIngress

> KongIngresss在k8s原生的Ingress上面做了扩展，可以更细粒度的控制路由行为。

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: configuration-demo
upstream: #后端服务
  hash_on: none
  hash_fallback: none
proxy:
  protocol: http
  path: /
  connect_timeout: 10000
  retries: 10
  read_timeout: 10000
  write_timeout: 10000
route: #路由
  methods:
  - POST
  - GET
  regex_priority: 0
  strip_path: false
  preserve_host: true
  protocols:
  - http
  - https
```

KongIngress需要和Ingress结合使用，主要有两种方式：

- 创建和Ingress规则同样命名空间，同样名称的KongIngress规则。这种情况下，每个Ingress都需要一个KongIngress，不利于维护。

- 在Ingress配置中添加注解来进行关联：```configuration.konghq.com:<KongIngress-resource-name>``` 。 这种方式可以复用KongIngress配置。

## annotations

> KongIngressController在支持k8s原生ingress.class注解的基础上扩展了两个注解：```plugins.konghq.com```和```configuration.konghq.com```

### ```kubernetes.io/ingress.class```

如果一个集群中有多个IngressController，一个Ingress只想被指定的IngressController监听，就需要用到此参数

Ingress中添加：
```yaml
metadata:
  name: foo
  annotations:
    kubernetes.io/ingress.class: "kong"
```
KongIngressController中添加：
```yaml
spec:
  template:
     spec:
       containers:
         - name: kong-ingress-internal-controller
           args:
             - /kong-ingress-controller
             - '--election-id=ingress-controller-leader-internal'
             - '--ingress-class=kong-internal'
```

> 如果想在同一个k8s集群内部署多个Kong集群，需要同时调整election-id和ingress-class

### ```plugins.konghq.com```
这个注解主要用来配置插件

```
plugins.konghq.com: high-rate-limit, docs-site-cors
```

### ```configuration.konghq.com```

此注解主要用于配置KongIngress

## 和Istio结合
> 下图主要参考《IstioHandBook》,可以将Kong放到Cluster的前面，对所有的入口流量进行拦截处理。

![gateway-istio](/images/2019071019/gateway_istio.jpg)


## 优点
1. Kong在非云环境下的特性，KongIngressController基本都支持了，比如插件机制、负载均衡策略、Router、Service以及Consumer等概念

2. 水平扩展比较方便，很多细节都考虑到了

## 缺点

1. KongIngressController目前还没有发布正式1.0版本，后续版本可能有不兼容的情况存在（类比Kong1.0前和Kong1.0后经常有Breaking Changes）

2. 和服务网格Istio的结合思路还不是很清晰

## 参考文档

[为服务网格选择入口网关](https://www.servicemesher.com/istio-handbook/best-practices/how-to-implement-ingress-gateway.html)

[Ingress解析](https://jimmysong.io/kubernetes-handbook/concepts/ingress.html)

[Kubernetes Ingress Controller的使用介绍及高可用落地](https://www.servicemesher.com/blog/kubernetes-ingress-controller-deployment-and-ha/)

[Kong IngressController设计架构](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/design.md)

[Kong IngressController Document](https://github.com/Kong/kubernetes-ingress-controller/tree/master/docs)

[kubernetes 之 Ingress 使用总结](https://owelinux.github.io/2018/12/27/article43-k8s-Ingress/)

[kubernetes Handbook](https://jimmysong.io/kubernetes-handbook/)
