---
title: javaagent学习
categories: Java
date: 2019-12-04 22:01:06
tags:
---


## 什么是JavaAgent ?
JavaAgent叫做Java代理，也叫做Java探针。是JVM提供的一种动态修改增强字节码的机制。通过在启动时指定-javaagent参数，或者通过JVMTI（Java Virtual Machine Tool Interface）提供的接口，可以很方便的修改类的字节码。

## JavaAgent可以做什么？
1. 在类加载时对类做修饰，从而增强功能。比如lombok以及APM框架Pinpoint和Skywalking
2. 动态Attach到进程上面做一些性能分析，比如阿里巴巴的Arthas等

## 一个例子🌰

JavaAgent使用时需要声明一个```premain```方法，这个方法的签名有两个：
1. ```public static void premain(String agentArgs, Instrumentation inst)```
2. ```public static void premain(String agentArgs)```

JVM默认会优先加载第一个签名的方法，如果没有的话才会加载第二个签名的方法。

那么Instrumentation又是什么呢？  
Instrumentation是对类进行转换的核心，提供了
```java
void addTransformer(ClassFileTransformer transformer)
void addTransformer(ClassFileTransformer transformer, boolean canRetransform)
Class[] getAllLoadedClasses()
//提供了很多方法
```
现在我们尝试来写一个javaagent。

首先是一个普通的Java程序：
```java
//org.example.App
public class App {
    public static void main( String[] args ) {
        System.out.println( "Hello World!" );
    }
}
```
maven打包时需要注意打包为可执行文件
```xml
<plugin>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>org.example.App</mainClass>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

其次我们创建一个agent程序：
```java
public class PreApp
{
  public static void premain(String agentArgs, Instrumentation instrumentation){
    instrumentation.addTransformer(new ClassFileTransformer() {
      @Override
      public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        //此处需要指定类名，不然会打印多次before main。同时类名中的 . 需要替换为 /
        if("org/example/App".equals(className)){
          System.out.println("before main--->");
        }
        return classfileBuffer;
      }
    });
  }
}

```

maven POM中需要指定premain:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
    <archive>
        <manifest>
        <addClasspath>true</addClasspath>
        </manifest>
        <manifestEntries>
        <Premain-Class>
            org.example.PreApp
        </Premain-Class>
        </manifestEntries>
    </archive>
    </configuration>
</plugin>
```

然后我们就可以执行：
```shell
java -javaagent:agent-demo-1.0-SNAPSHOT.jar  -jar demo-1.0-SNAPSHOT.jar 
```
就可以看到控制台上打印出：
```
before main--->
Hello World!

```

说明类增强成功了








