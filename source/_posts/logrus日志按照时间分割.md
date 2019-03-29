---
title: logrus日志按照时间分割
date: 2017-12-11 21:49:44
tags:
categoris: 问题记录
---

最近学习了一下go语言，发现和Java的语法差别好大啊，不过用起来的确很方便。
golang官方的日志库不太好用，就从github上找了一个比较火的logrus日志库，这个库和java的库用法还是有不少区别的，今天主要想做的是将日志文件按照时间进行切割，logrus借助了　go-file-rotatelogs　这个库，所以首先我们应该安装相关依赖
##### 1. 安装相关依赖库
```
go get github.com/sirupsen/logrus
go get github.com/lestrrat/go-file-rotatelogs
go get github.com/rifflock/lfshook
```
##### 2. 为了全局使用，我做了一个简单的封装(可以通过调整时间使日志按照天进行轮转),注意时间格式化非常诡异，据说是go诞生的时间
```
package utils

import (
	"github.com/sirupsen/logrus"
	"github.com/lestrrat/go-file-rotatelogs"
	"github.com/rifflock/lfshook"
	"time"
	"path"
	"github.com/pkg/errors"
)
var Log *logrus.Logger
func init() {
	Log = logrus.New()
	Log.SetLevel(logrus.InfoLevel)
	ConfigLocalFilesystemLogger("", "std.log", time.Hour*24, time.Second*20)
}

func ConfigLocalFilesystemLogger(logPath string, logFileName string, maxAge time.Duration, rotationTime time.Duration) {
	baseLogPaht := path.Join(logPath, logFileName)
	writer, err := rotatelogs.New(
		baseLogPaht+".%Y%m%d%H%M%S",
		//rotatelogs.WithLinkName(baseLogPaht),      // 生成软链，指向最新日志文件
		rotatelogs.WithMaxAge(maxAge),             // 文件最大保存时间
		rotatelogs.WithRotationTime(rotationTime), // 日志切割时间间隔
	)
	if err != nil {
		Log.Errorf("config local file system logger error. %+v", errors.WithStack(err))
	}
	lfHook := lfshook.NewHook(lfshook.WriterMap{
		logrus.DebugLevel: writer, // 为不同级别设置不同的输出目的
		logrus.InfoLevel:  writer,
		logrus.WarnLevel:  writer,
		logrus.ErrorLevel: writer,
		logrus.FatalLevel: writer,
		logrus.PanicLevel: writer,
	},&logrus.TextFormatter{DisableColors: true, TimestampFormat: "2006-01-02 15:04:05.000"})
	//注意上面这个 and &amp;符号被转义了
	Log.AddHook(lfHook)
}
```
##### 3. 我们使用时就可以导入包进行使用啦
```
utils.Log.Info("Hello")
```
