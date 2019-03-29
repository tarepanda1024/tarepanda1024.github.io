---
title: logback打印mybatis sql语句
date: 2018-01-24 10:34:07
tags: 问题记录
---
最近想在mybatis中打印一下sql语句，发现官网提供的方法都不适合，最终在os china找到了，现在记录一下，以作备份

#### 1. 在mybatis的configuration中增加setting配置
```
<settings>
        <setting name="logPrefix" value="dao."/>
</settings>
```
#### 2. 在logback配置文件中增加logger
```
<logger name="dao" level="DEBUG"/>
```
参考:
http://www.mybatis.org/mybatis-3/logging.html
https://my.oschina.net/u/2263802/blog/956588