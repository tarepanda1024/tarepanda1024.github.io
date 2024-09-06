---
title: ApacheDoris运维手册
date: 2023-02-06 20:11:47
updated: 2024-09-06 10:30:00
tags:
  - doris
categories:
  - doris
---


> 本手册用于快速排查Doris的问题，并进行紧急修复。

采用mysql客户端查询节点状态时，需要先执行 set forward_to_master=true;命令

# 关于重启
Doris正常跑一段时间后，突然出现严重的不可用问题，主要是数据无法写入、监控发现版本大量堆积。首要的处理办法
1. 重启fe的master节点
2. 如果没有解决，再逐个重启全部的fe节点
3. 如果重启后仍然无法解决，逐个重启所有的be节点

遇到过的三种异常：
1. 通过show backends;命令查看be节点的lastSuccessReportTabletsTime，如果lastSuccessReportTabletsTime不同节点相差较大，则需要重启fe和be
2. The statement has been forwarded to master xxx and failed to execute because Master FE is not ready
3. 事务堆积，无法写入，日志中出现：wait for publishing partition xxx

# 官方文档

- 官方的FQA文档：https://doris.apache.org/zh-CN/docs/faq/install-faq
- Github Issue文档：https://github.com/apache/doris/issues
- 社区问答集：https://ask.selectdb.com/
- 元数据运维文档（非常重要）：https://doris.apache.org/zh-CN/docs/admin-manual/maint-monitor/tablet-repair-and-balance 以及 https://doris.apache.org/zh-CN/docs/admin-manual/maint-monitor/metadata-operation 
- 分区分桶（非常重要）：https://doris.apache.org/zh-CN/docs/table-design/data-partition

# 微信社群

> selectdb基本是doris社区维护的主要力量，人员也比较友好，疑难问题可以联系他们

微信号：
- doris pmc：陈明雨 morningman-cmy
- selectdb：张彬华 yz-jayhua 

# 一些常用配置

## 基础设置

* 调整数据库容量

  ```sql
  ALTER DATABASE xxx SET DATA QUOTA 100T;
  ```

* 修改数据库链接

  ```sql
  SHOW PROPERTY FOR 'xxx' LIKE '%max_user_connections%';
  SET PROPERTY FOR 'xxx' 'max_user_connections' = '300';
  ```

## 查询

* 修改超时时间

  ```sql
  SET query_timeout = 1800;
  ```

## 导入

* 修改单次导入数据上限

  ```sql
  streaming_load_json_max_mb=10240
  ```

## 分区&副本

* 修改副本数目

  ```sql
  ALTER TABLE tb_xxx SET("dynamic_partition.replication_num" = "3");
  ```

* 查看分区状态

  ```sql
  SHOW DYNAMIC PARTITION TABLES;
  SHOW [TEMPORARY] PARTITIONS FROM [db_name.]table_name [WHERE] [ORDER BY] [LIMIT];
  ```

* 查看表全部的副本状态

  ```sql
  ADMIN SHOW REPLICA STATUS FROM db1.tbl1;
  ADMIN SHOW REPLICA STATUS FROM tbl1 PARTITION (p1, p2) WHERE STATUS = "VERSION_ERROR";
  ADMIN SHOW REPLICA STATUS FROM tbl1 WHERE STATUS != "OK";
  ```

* 动态分区删除副本

  ```sql
  ALTER TABLE tb_xxx SET ("dynamic_partition.enable" = "false");
  alter table tb_xxx drop partition p20211129
  ALTER TABLE tb_xxx SET ("dynamic_partition.enable" = "true");
  ```


# 问题处理

* 更改表结构时报错：Create replicas failed. Error: Error replicas: xxx=xxx

  原因：更改表结构超时 https://github.com/apache/doris/issues/2942

  解决办法：

  ```shell
  ADMIN SET FRONTEND CONFIG ("max_create_table_timeout_second" = "600");
  ```
* Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: errCode = 2, detailMessage = Incorrect table name 'xxxxx'

回答: 一般是表名过长，默认长度为64，需要调整表名长度。

* jdbc建表超时

回答: 默认查询超时是300s，如果建表语句分区过多，可能会触发超时。可以在session侧，单独配置超时时间，用以规避。配置参考

* 部分query异常，无法在日志里定位到 
回答: 经过排查，部分用户端显示的错误内容，只有在日志级别为debug是才会打印都日志文件。如果用户端显示错误内容已经很清晰的描述原因，直接以这个原因来判断即可。
‍

