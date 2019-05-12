---
title: git常用命令
date: 2017-10-11 10:08:39
tags:
categories: 工具
---

#### 更新分支信息
```git
git fetch origin --prune
```
#### 设置远端分支(远端分支存在)
```git
git branch -u origin/[远程分支]
```

#### 设置远端分支(远端分支不存在)
```git
git push origin [远程分支]
```