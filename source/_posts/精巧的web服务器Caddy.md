---
title: 精巧的web服务器Caddy
date: 2019-01-22 21:11:38
tags:
categories: 网关
---

Caddy是一个开源的，使用Golang编写，支持HTTP/2的web服务器。第一个版本发布于2015年，至今在github上已经有超过2万+的stars。和apache、nginx一样，Caddy也提供基本的静态文件托管、反向代理和负载均衡等基本功能。同时主打易用性，配置比较简单，还具有很多看起来比较现代的特性，比如支持自动https(使用Let's Encrypt证书，并自动续期)、markdown文件托管、prometheus监控以及同步git代码生成个人博客(hexo、hugo)等等。

基于Golang语言的特性，Caddy还有以下特性：

- 依赖简单，只需要一个二进制文件就可以跑起来
- 跨平台，除了Windows, Mac, Linux, BSD, Solaris，还支持Android平台
- 天然多核支持

另外，之所以说Caddy比较精巧，是因为Caddy具有一个强大的插件系统，几乎Caddy提供的所有功能，全部是基于插件机制实现的。

下面主要介绍一下Caddy的安装、基本用法和插件开发。

### 安装

> Caddy采用的是Apache-2.0开源协议，如果我们直接从官网下载带有插件的二进制运行文件，商用的话是需要付费的。不过如果我们自己编译的话，是免费的。

Caddy的编译步骤非常简单，只需要执行如下四条命令:
```shell
go get github.com/mholt/caddy/caddy

go get github.com/caddyserver/builds

cd $GOPATH/src/github.com/mholt/caddy/caddy

go run build.go
```
在当前目录下会生成一个caddy二进制文件，我们可以把caddy放到环境变量里。执行```caddy```，服务就启动了，打开浏览器，输入```http://localhost:2015/```，会看到```404 Not Found```，说明服务启动正常。

### 使用

> Caddy的配置文件和指令和nginx很相似，不过配置起来会更简单一些。

一个简单的单站点caddyfile文件如下:
```nginx
localhost:8080
gzip
log ./access.log
```
或者
```nginx
localhost:8080 {
    gzip
    log ./access.log
}
```
caddy的指令也非常丰富
#### 指令

- proxy
> proxy指令用于代理
```nginx
proxy from to... {
	policy name [value]　　#负载均衡策略random, least_conn, round_robin, first, ip_hash, uri_hash, or header
	fail_timeout duration　
	max_fails integer #和上面的fail_timeout配合使用，失败多少次认为后端服务不可用，负载均衡时就不再向这台机器发请求
	max_conns integer
	try_duration duration
	try_interval duration
	health_check path　　　#健康检查
	health_check_port port
	health_check_interval interval_duration
	health_check_timeout timeout_duration
	fallback_delay delay_duration
	header_upstream name value
	header_downstream name value
	keepalive number
	timeout duration
	without prefix　　　　# 去除前缀
	except ignored_paths...
	upstream to
	insecure_skip_verify
	preset #websocket或者transparent
}
```

- rewrite
> 和nginx的rewrite含义一样，重写url
```nginx
rewrite [basepath] {
	regexp pattern
	ext    extensions...
	if     a cond b
	if_op  [and|or]
	to     destinations...
}
```

- errors
>　记录异常，支持自定义返回页面
```nginx
errors [logfile] {
	code     file
	rotate_size     mb
	rotate_age      days
	rotate_keep     count
	rotate_compress
}
```

- log
> 日志记录，支持文件分割
```nginx
log path file [format] {
	rotate_size     mb
	rotate_age      days
	rotate_keep     count
	rotate_compress
	ipmask          ipv4_mask [ipv6_mask]
	except          paths...
}
```


#### 插件

Caddy的插件分为```Server Types ```,```Directives```,```HTTP Middleware```,```Caddyfile Loader```,```DNS Provider```,```Cluster Support ```,```Listener Middleware```以及```Event Hook```共八种类型。

- Server Types (服务器类型)

这种类型的插件目前Caddy只有```HTTP```这一种。

- Directives (指令类型)

指令类型的插件用于自定义指令。具体步骤如下：
1. 新建一个go文件，在```init```方法中填写插件名和setup方法
```go
import "github.com/mholt/caddy"

func init() {
	caddy.RegisterPlugin("gizmo", caddy.Plugin{
		ServerType: "http",
		Action:     setup,
	})
}
```
```go
func setup(c *caddy.Controller) error {
	return nil
}
```
```go
for c.Next() {              
    if !c.NextArg() {      
        return c.ArgErr()   
    }
    value := c.Val()   //提取配置参数    
}
```
2. 在```run.go```的顶部导入即可:

```go
import _ "your/plugin/package/path/here"
```

- HTTP Middleware （HTTP中间件)
> HTTP Middleware类型的插件是在指令类型的插件上面实现了```httpserver.Handler```
```go
type MyHandler struct {
	Next httpserver.Handler
}
func (h MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) (int, error) {
	return h.Next.ServeHTTP(w, r)
}
```

```go
func setup(c *caddy.Controller) error {
	for c.Next() {
		// get configuration
	}
	handler := MyHandler{
		
	}

	httpserver.GetConfig(c).AddMiddleware(func(next httpserver.Handler) httpserver.Handler {
		handler.Next = next
		return handler
	})
}

```

- Caddyfile Loader (Caddy配置文件加载器)
> caddyfile loader类型的插件主要用于自定义配置文件加载器，比如从数据库中加载配置，通过http接口进行hot-reload等

```go
import "github.com/mholt/caddy"

func init() {
	caddy.RegisterCaddyfileLoader("myloader", caddy.LoaderFunc(myLoader))
}

func myLoader(serverType string) (caddy.Input, error) {
	return nil, nil
}
```

### 小结
从上面我们可以看到，作为一个web服务器，caddy完全可以满足日常使用的需求；同时Caddy可以自行扩展的插件类型非常丰富：从实现一个Server插件、集群插件到实现一个http中间件都是非常方便的。




