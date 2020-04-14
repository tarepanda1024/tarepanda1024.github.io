---
title: Javassist中文教程
date: 2020-04-05 18:10:23
tags:
categories: Java
---
> 本篇文章翻译自：[Javassit官方教程](http://www.javassist.org/tutorial/tutorial.html)

## 读写字节码
Javassist是一个字节码读写类库。Java的字节码存储在一个以.class为后缀的文件中。每个class文件包含一个Java类或者接口。
在Javassist中，```Javassist.CtClass```是类文件的抽象。每个```CtClass```实例代表一个class文件。下面是一个非常简单的例子：
```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("test.Rectangle");
cc.setSuperclass(pool.get("test.Point"));
cc.writeFile();
```
上面的程序首先获取到一个```ClassPool```对象，```ClassPool```主要用于字节码的修改，同时也是一系列```CtClass```的集合。它会读取class文件用于生成




## ClassPool

## ClassLoader

## 检视和定制

## 字节码级别的API 

## Vararages


## J2ME

## 装箱和拆箱

## Debug


