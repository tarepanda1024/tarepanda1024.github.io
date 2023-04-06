---
title: linux系统No space left on device的几种排查解决方法
tags: linux
categories: linux
date: 2023-04-06 19:21:15
updated:
---

在工作中会遇到No space left on device的异常，下面总结一下问题的排查和解决思路：

## 磁盘空间真的满了
执行```df -h```命令可以看到磁盘的占用情况：

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        36G  5.7G   28G  17% /
tmpfs           1.9G  209M  1.7G  12% /run
/dev/vdb         99G  5.7G   88G   7% /home
```
如果磁盘满了，use%会显示为100%。可以执行```du -h --max-depth=1```继续查看具体哪个目录占用空间大。执行``` find /home -type f -size +10M  -print0 |  xargs -0 du -h | sort -nr```命令查找出大文件，并排序。

找到大文件后，可以执行进行删除操作，如果文件删除后，空间还没有变小。需要检查文件是否还被进程持有(```lsof 文件名称```)，需要杀死进程，才会完全释放空间。

## 磁盘空间没有满，但是inode满了
执行```df -i```命令可以看到inode的使用情况，inode满了，说明可能存在大量的小文件，此时删除部分文件就可以解决
```
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/vda1      2359296 142821 2216475    7% /
tmpfs           485127     16  485111    1% /sys/fs/cgroup
/dev/vdb       6553600 666557 5887043   11% /home
```