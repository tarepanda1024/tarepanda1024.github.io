---
title: SpringBoot部署war包到tomcat
date: 2017-09-26 07:37:17
tags:
categories: 问题记录
---

#### pom文件中打包方式由jar修改为war
```
<packaging>war</packaging>
```
#### 将内置tomcat设为provided

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
</dependency>

```

#### 需要集成SpringBootServletInitializer并重写configure方法

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.support.SpringBootServletInitializer;

@SpringBootApplication
public class App extends SpringBootServletInitializer {

	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
	
	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(App.class);
	}
}
```
