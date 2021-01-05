---
title: Java获取文件ContentType/MimeType
tags:
  - Java
categories:
  - Java
date: 2021-01-05 16:36:18
---


工作中需要用到根据文件后缀名称判断MimeType的场景，找到几种方式，记录一下。

## 判断ContentType

### 方式一 MimetypesFileTypeMap

```java
 MimetypesFileTypeMap.getDefaultFileTypeMap().getContentType("xxx.ext");
```
此类在javax.activation包下，java11已经移除了javax，不建议使用，同时部分文件无法正确获取到Content-Type。
可通过```addMimeTypes(String mime_types)```方式添加解析。

### 方式二 URLConnection

```java
URLConnection.getFileNameMap().getContentTypeFor("xxx.ext");
```
部分后缀获取不到，比如svg格式的图片，无法正确获取到ContentType。

### 方式三 ApplicationContext

```java
//HttpServletRequest request
request.getServletContext().getMimeType("xxx.ext")
```

此种方式依赖Spring，如果是Spring环境，建议使用此种方式。此方式适配性较好。

如果部分后缀不识别，可通过以下方式进行添加：
```java
@Configuration
public class MimeMappingConfig implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
    @Override
    public void customize(ConfigurableServletWebServerFactory factory) {
        MimeMappings mappings = new MimeMappings(MimeMappings.DEFAULT);
        mappings.add("xxx", "application/xml; charset=utf-8");
        factory.setMimeMappings(mappings);
    }
}
```

