---
title: Istio调研
date: 2019-07-28 17:42:05
tags: Istio
categories: SeviceMesh
---


单体应用迁移到微服务的过程中带了不少问题，比如服务注册、服务发现、熔断、鉴权等。Dubbo和SpringCloud虽然简化了微服务间的交互，但是和应用绑定太紧密了，升级和部署部署很方便。同时SpringCloud的一套东西太复杂了，完全掌握需要的成本比较高昂。  
因此最好是能把应用依赖的SDK下沉到基础设施中，降低微服务接入的复杂度，降低接入成本，使应用更加专注于业务本身。 SeviceMesh翻译过来即是服务网格，做的就是这样的事情。Istio是SeviceMesh的一种实现。本文主要简单介绍一下istio的基本概念和一个示例程序。

版本:  
- minikube: 1.2.0
- kubectl: 1.15
- istio: 1.2.2


## 设计

Istio主要分为两部分，一部分是数据平面，一部分是控制平面。数据平面主要是作为sidecar和应用部署在一起，拦截应用的流量，sidecar和sidecar进行通信。控制平面主要是控制各种路由策略，将配置下发到sidecar。

![Istio架构](/images/201907281742/arch.svg)

### Envoy
Istio将Envoy作为sidecar和服务部署在一个k8s pod中。Envoy代理应用的所有流量。并且提供负载均衡、服务发现等功能。

### Mixer
Mixer作为单独的进程部署在服务之外，负责在服务网格上执行访问控制和使用策略，并从 Envoy 代理和其他服务收集遥测数据。因为此种架构会对性能影响比较大。istio1.1版本已经将此功能默认关闭。

### Pilot

Pilot 为 Envoy sidecar 提供服务发现功能，为智能路由（例如 A/B 测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能。它将控制流量行为的高级路由规则转换为特定于 Envoy 的配置，并在运行时将它们传播到 sideca

### Galley
Gallery的作用主要是管理验证API配置。随着时间的推移，Galley 将接管 Istio 获取配置、 处理和分配组件的顶级责任。

## 概念
### DestinationRule
DestinationRule的含义相当于Envoy中cluster的概念。用于声明多台相同服务机器的集合。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

### VirtualService
VirtualService的含义相当于Envoy中Http Route Table。可以根据Host、Method、Uri等属性将请求路由到不同的服务。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1

```

### ServiceEntry
DestinationRule定义的是托管在云端集群内部的服务，ServiceEntry主要定义未托管在云端集群内部的服务

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-https
spec:
  hosts:
  - api.dropboxapi.com
  - www.googleapis.com
  - api.facebook.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
```

### Gateway
Gateway常部署在流量的入口。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
  namespace: some-config-namespace
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https-443
      protocol: HTTPS
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      mode: SIMPLE # enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
```



## 安装

### 启动minikube集群
```bash
minikube start --memory=8192 --cpus=4
```

### 下载安装包

https://github.com/istio/istio/releases

![文件目录](/images/201907281742/istio-dic.png)


### 安装CRD
> istio自定了50多个资源，包括各种组件以及VirtualService等

```bash
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```
## 安装istio
```bash
 kubectl apply -f install/kubernetes/istio-demo.yaml
```
## 启动istio自动注入
```bash
kubectl label namespace default istio-injection=enabled
```

![istio-system](/images/201907281742/istio-system.png)

## 示例

### book服务
```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### 应用缺省目标规则
```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

## 创建网关
```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
![book-demo](/images/201907281742/book.png)

## 获取host
```bash
minikube ip
```

## 获取port
```bash
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
```
访问http://ip:port/productpage即可看到页面

![book-page](/images/201907281742/bookpage.png)

上面的示例只是简单介绍了应用部署的流程，istio还提供有灰度发布、熔断、限速等其他功能。

## 总结
 Istio将业务逻辑和非业务逻辑分离，非业务逻辑下沉到基础设施，降低了应用接入的成本，带来的好处是巨大的。


## 参考文献

- [Istio官方文档](https://istio.io/docs/)
- [使用Istio治理微服务入门](https://www.cnblogs.com/williamjie/p/9442340.html)
- [Service Mesh：下一代微服务？](http://mini.eastday.com/mobile/171102113032674.html)
- [ServiceMesh发展趋势：云原生中流砥柱](https://www.servicemesher.com/blog/201905-servicemesh-development-trend/)
- [康威定律——这个50年前就被提出的微服务概念，你知多少？](https://segmentfault.com/a/1190000011118897)
- [Istio 庖丁解牛](https://cloud.tencent.com/developer/article/1409159)
- [浅谈Service Mesh体系中的Envoy](https://yq.aliyun.com/articles/606655)