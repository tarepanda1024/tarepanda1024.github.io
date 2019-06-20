---
title: 开源API网关Tyk简单介绍
date: 2019-01-15 21:11:38
tags:
categories: 网关
---

tyk是一款采用golang语言实现的API网关，具有API Gateway、Tyk Dashboard、 Tyk Pumpd和Tyk Identity Broker等几大组件。不过只有API Gateway的源代码是开放的。下面主要介绍一下API Gateway的功能与用法。

![gateway1.png](/images/2019011501.png)

#### 安装
我们以Ubuntu18.04为例讲一下安装步骤:

1. 安装redis-server
```
sudo apt-get install -y redis-server
```
2. 使用一键安装脚本进行安装
```
curl -s https://packagecloud.io/install/repositories/tyk/tyk-gateway/script.deb.sh | sudo bash
```
3. 配置服务
```
sudo /opt/tyk-gateway/install/setup.sh --listenport=8080 --redishost=<hostname> --redisport=6379 --domain=""
```
4. 启动服务
```
sudo service tyk-gateway start
```

#### 简单使用

在tyk中创建API分为两种，一种是在安装目录下的```apps```目录下创建一个```json```文件，另一种是使用tyk提供的```restful```接口```/tyk/apis```。我们采用第一种方式，在```apps```目录下创建一个```api1.json```文件:
```
{
    "name": "Tyk Test API",
    "api_id": "1",
    "org_id": "default",
    "use_keyless":true,
    "definition": {
      "location": "header",
      "key": "x-api-version"
  	},
    "version_data": {
      "not_versioned": true,
      "versions": {
        "Default": {
          "name": "Default",
          "use_extended_paths": true
        }
    	}
    },
    "proxy": {
        "listen_path": "/tyk-api-test/",
        "target_url": "http://httpbin.org",
        "strip_listen_path": true
    }
}
```

然后调用```curl -H "x-tyk-authorization: {your-secret}" -s https://{your-tyk-host}:{port}/tyk/reload/group```进行热加载配置文件(```x-tyk-authorization```必选带上，secret在配置文件中)。这样我们访问```/tyk-api-test```就会代理到```http://httpbin.org```

tyk的api支持的配置参数非常多，比如认证、版本号、频率控制和配额控制等，具体可参考[Tyk Api定义](https://tyk.io/docs/tyk-rest-api/api-definition-objects/```)


#### 插件机制
tyk的插件功能比较强大，一方面提供了IP黑白名单、参数提取和认证等诸多插件，另一方面也支持js、python、lua语言来自定义插件。插件的执行顺序如下图:

![gateway2.png](/images/2019011502.png)

我们用lua开发一个简单的插件测试一下:
需要安装依赖:
```
apt-get install libluajit-5.1-2
apt-get install luarocks
luarocks install lua-cjson
```

1. 创建一个```manifest.json```文件

```
{
  "file_list": [
    "mymiddleware.lua"
  ],
  "custom_middleware": {
    "pre": [
      {
        "name": "MyPreMiddleware"
      }
    ],
    "driver": "lua"
  },
  "checksum": "",   
  "signature": ""
}
```
2. 创建一个 ```mymiddleware.lua```文件
```
function MyPreMiddleware(request, session, spec)
  tyk.req.set_header("customheader", "customvalue")
  return request, session
end
```

然后我们需要使用```tyk-cli```(将```/opt/tyk-gateway/utils/```添加到```path```中)进行打包，切换到刚才的插件目录执行```tyk-cli bundle build -output bundle-test.zip -y```命令，将```bundle-test.zip```放到一个静态服务器中。
修改```tyk.conf```配置文件
```
"coprocess_options": {
  "enable_coprocess": true,
},
"enable_bundle_downloader": true,
"bundle_base_url": "http://my-bundle-server.com/bundles/",
```
执行
```
service tyk-gateway stop
service tyk-gateway-lua start
```
最后修改api1.json添加上:
```
"custom_middleware_bundle": "bundle-test.zip"
```
这样插件就生效了


#### 报表、监控和事件触发器

tyk的管理平台提供有一系列详细指标来监控各个API的访问情况，不过相关代码没有开源出来，相关指标的存储访问接口也没有文档中显示，如果想深入了解的话需要学习源代码。
![gateway4.png](/images/2019011503.png)

tyk也内置了两个事件触发器handler,一个是```eh_log_handler```(这个主要是debug使用)，另外一个是```eh_web_hook_handler```。内置的事件包括```QuotaExceeded```,```RatelimitExceeded```,```OrgQuotaExceeded```,```OrgRateLimitExceeded```等，这些事件大都需要启动API认证机制后才会生效。事件触发后后会发送类似如下的数据
```
{
  "event": "{{.Type}}",
  "message": "{{.Meta.Message}}",
  "path": "{{.Meta.Path}}",
  "origin": "{{.Meta.Origin}}",
  "key": "{{.Meta.Key}}"
}

```
内置的两个```event handler```相关来说比较鸡肋，不过也可以自己定义事件类型与```event handler```
#### 集群部署
tyk也支持集群部署，需要一个组件MDCB(Tyk Multi Data Centre)与多个slave节点。集群功能是需要付费的，这里就不详细介绍了。

![gateway3.png](/images/2019011504.png)


#### 协议支持情况

支持http、gRPC和websocket


#### 小结
tyk总的来说算是一个中规中矩的API网关，丰富的插件、强大的认证机制和多种语言的插件支持给人眼前一亮的感觉，不过开源版本的集群功能、日志监控和灰度发布等功能相对较弱。

