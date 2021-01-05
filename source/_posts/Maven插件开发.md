---
title: Maven插件开发
tags:
  - Java
  - Maven
categories:
  - Java
date: 2020-01-21 19:26:41
---


1. 创建maven-plugin工程

> 可以直接使用maven模板```maven-archetype-mojo```

```xml
     <!--打包类型-->
    <packaging>maven-plugin</packaging>
     <!--maven依赖-->
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-plugin-api</artifactId>
      <version>${maven.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-core</artifactId>
      <version>${maven.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-artifact</artifactId>
      <version>${maven.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.maven.plugin-tools</groupId>
      <artifactId>maven-plugin-annotations</artifactId>
      <version>3.6.0</version>
      <scope>provided</scope>
    </dependency>
```



2. 实现execute方法

```java
@Mojo(name="hello", defaultPhase = LifecyclePhase.INSTALL)
public class HelloMojo extends AbstractMojo {

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        getLog().warn("Hello world mojo");
    }
}
```
执行```mvn clean install```安装到本地

3. 引用自定义插件

在另外一个工程中引用此maven插件

```xml
<plugin>
    <groupId>org.example</groupId>
    <artifactId>hello-maven-plugin</artifactId>
    <version>1.0-SNAPSHOT</version>
    <executions>
       <execution>
        <phase>install</phase>
          <goals>
          <!-- 配置执行目标 -->
          <goal>hello</goal>
         </goals>
      </execution>
   </executions>
</plugin>
```
执行```mvn clean install```就可以看到打印出来的日志了。

```shell
[INFO] --- hello-maven-plugin:1.0-SNAPSHOT:hello (default) @ test ---
[WARNING] Hello world mojo
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS

```
