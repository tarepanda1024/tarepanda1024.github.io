---
title: Java自定义doclet
categories: Java
date: 2020-01-21 16:44:35
tags: Java
---


> 调研Doclet主要是因为公司项目中需要自定义接口文档并上传到平台，于是查到了几种方式，在这里记录一下。

javadoc是基于doclet API输出html javadoc文档的默认实现。我们可以通过自定义Doclet来实现自己的文档。在自定义Doclet的实现过程中，我们可以拿到类的完整信息，比如注解，方法等信息。

官方文档在这里：https://docs.oracle.com/javase/8/docs/technotes/guides/javadoc/doclet/overview.html

下面我们通过自定义Doclet来看看具体的流程

### 添加依赖包
首先我们需要引入jdk提供的tools包

```xml
<groupId>org.example</groupId>
<artifactId>OwnDoclet</artifactId>
<version>1.0-SNAPSHOT</version>
<!--省略其它配置-->
<dependency>
    <scope>system</scope>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.8</version>
    <systemPath>${java.home}/../lib/tools.jar</systemPath>
</dependency>
```

### 自定义Doclet

然后，我们自己编写一个简单的Doclet Demo。自定义Doclet需要继承自Doclet类，并实现start方法
```java
public class OwnDoclet extends Doclet {
    public static boolean start(RootDoc rootDoc) {
        System.out.println("Hello my doclet");
        for (ClassDoc classDoc : rootDoc.classes()) {
            System.out.println("ClassName:" + classDoc.qualifiedName());
        }
        return true;
    }
}
```
开发完毕我们需要执行```mvn clean install```安装到本地

备注：rootDoc中可以拿到类的完整信息，比如注解、方法、参数名称等信息。因此我们这里在这里做很多事情，比如根据特定的注解提起信息生成API文档并上传到平台中。

### 引用并测试

在另外一个工程中引入javadoc插件
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>3.1.1</version>
    <configuration>
        <encoding>UTF-8</encoding>
        <useStandardDocletOptions>false</useStandardDocletOptions>
        <doclet>org.example.OwnDoclet</doclet>
        <docletArtifacts>
             <docletArtifact>
                <groupId>org.example</groupId>
                <artifactId>OwnDoclet</artifactId>
                <version>1.0-SNAPSHOT</version>
            </docletArtifact>
        </docletArtifacts>
    </configuration>
</plugin>
```
执行```mvn javadoc:javadoc```命令，可以看到输出如下：
```
正在加载程序包org.example的源文件...
正在构造 Javadoc 信息...
Hello my doclet
ClassName:org.example.App
ClassName:org.example.Test
ClassName:org.example.RefTest
ClassName:org.example.Expose

```



### 参考文档
[Doclet Overview](https://docs.oracle.com/javase/8/docs/technotes/guides/javadoc/doclet/overview.html)

