---
title: electron安装卡node install.js的解决方法
date: 2017-12-10 19:01:33
tags:
categories: 前端
---
最近想用electron加上vue开发一个客户端软件，于是就打算采用simulatedgreg的脚手架，结果执行```npm install```时，一直卡在electron node install.js上；我把npm切换到taobao的源依然不行，后来查到一种方法，现在记录下来：
具体的操作是:
##### 1.  初始化脚手架
```javascript
vue init simulatedgreg/electron-vue my-project

# Install dependencies and run your app
cd my-project
```

##### 2. 删除package.json中的electron依赖，如果执行过```npm  install```的话，还需要删除　node_modules目录下的electron目录
##### 3.　执行```npm install```命令，这样会把除electron外的依赖全都安装好了
##### 4. 我们最后安装electron
```
ELECTRON_MIRROR=http://npm.taobao.org/mirrors/electron/ npm install --save-dev electron

```
因为electron的包比较大，所以还会卡install.js　不过我们看一下网速就发现，速度已经提上去了，20s左右就安装好了