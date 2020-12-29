---
title: logrus日志按照时间分割
date: 2017-12-11 21:49:44
tags:
categoris: Golang
---

最近学习了一下go语言，发现和Java的语法差别好大啊，不过用起来的确很方便。
golang官方的日志库不太好用，就从github上找了一个比较火的logrus日志库，这个库和java的库用法还是有不少区别的，今天主要想做的是将日志文件按照时间进行切割，logrus借助了```go-file-rotatelogs```　这个库，所以首先我们应该安装相关依赖
##### 1. 安装依赖库
```shell
go get github.com/sirupsen/logrus
go get github.com/lestrrat/go-file-rotatelogs
go get github.com/rifflock/lfshook
```
##### 2. 添加hook
```go
package log

import (
	"github.com/lestrrat/go-file-rotatelogs"
	"github.com/pkg/errors"
	"github.com/rifflock/lfshook"
	log "github.com/sirupsen/logrus"
	"path"
	"time"
)

func init() {
	log.SetLevel(log.InfoLevel)
	log.AddHook(newRotateHook("", "stdout.log", 7*24*time.Hour, 24*time.Hour))
}

func newRotateHook(logPath string, logFileName string, maxAge time.Duration, rotationTime time.Duration) *lfshook.LfsHook {
	baseLogPath := path.Join(logPath, logFileName)

	writer, err := rotatelogs.New(
		baseLogPath+".%Y-%m-%d",
		rotatelogs.WithLinkName(baseLogPath),      // 生成软链，指向最新日志文
		rotatelogs.WithMaxAge(maxAge),             // 文件最大保存时间
		rotatelogs.WithRotationTime(rotationTime), // 日志切割时间间隔
	)
	if err != nil {
		log.Errorf("config local file system logger error. %+v", errors.WithStack(err))
	}
	return lfshook.NewHook(lfshook.WriterMap{
		log.DebugLevel: writer, // 为不同级别设置不同的输出目的
		log.InfoLevel:  writer,
		log.WarnLevel:  writer,
		log.ErrorLevel: writer,
		log.FatalLevel: writer,
		log.PanicLevel: writer,
	}, &log.TextFormatter{DisableColors: true, TimestampFormat: "2006-01-02 15:04:05.000"})
}
```
##### 3. 使用
> logrus默认自带了一个log实例，步骤二给默认的实例添加了日志轮转，因此我们应该在其它包里面使用默认的log实例，而不应该直接使用```  logrus.New()```

```go
log "github.com/sirupsen/logrus"
```
