---
title: ApacheDoris快速运维手册
date: 2023-02-06 20:11:47
updated: 2023-02-06 20:11:47
tags:
  - doris
categories:
  - doris
---


> 本手册用于快速排查Doris的问题，并进行紧急修复。持续更新中。。。

# 基础设置

* 调整数据库容量

  ```sql
  ALTER DATABASE xxx SET DATA QUOTA 100T;
  ```

* 修改数据库链接

  ```sql
  SHOW PROPERTY FOR 'xxx' LIKE '%max_user_connections%';
  SET PROPERTY FOR 'xxx' 'max_user_connections' = '300';
  ```

# 查询

* 修改超时时间

  ```sql
  SET query_timeout = 1800;
  ```

# 导入

* 修改单次导入数据上限

  ```sql
  streaming_load_json_max_mb=10240
  ```

# 分区&副本

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

# 杂项

* 批量执行sql

  ```shell
  cat xxx.sql |xargs -d "\r\n" ./mysql -uxx -P9030 -e -h10.xx.xx.xx -pxxx
  ```

# 问题处理

* 更改表结构时报错：Create replicas failed. Error: Error replicas: xxx=xxx

  原因：更改表结构超时 https://github.com/apache/doris/issues/2942

  解决办法：

  ```shell
  ADMIN SET FRONTEND CONFIG ("max_create_table_timeout_second" = "600");
  ```

‍

