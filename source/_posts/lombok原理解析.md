---
title: lombok原理示例
tags:
  - Java
  - Lombok
categories:
  - Java
date: 2020-01-21 20:10:00
---


lombok采用的是编译期注解，依赖jsr269特性，修改的是AST抽象语法树

![jsr269](/images/20200121/01.png)

1. 添加依赖声明注解

```xml
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.8</version>
    <scope>system</scope>
    <systemPath>${java.home}/../lib/tools.jar</systemPath>
</dependency>
```

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.SOURCE)
public @interface Expose {
}
```

2. 定义处理器

```java
@SupportedAnnotationTypes({"org.example.Expose"})
public class ExposeProcessor extends AbstractProcessor {
    private int r = 0;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        System.out.println("begin process ... ");
        Set<? extends Element> elements = roundEnv.getRootElements();
        for (Element element : elements) {
            System.out.println("输入的类:" + element.getSimpleName());
        }
        Set<? extends Element> genElements = roundEnv.getElementsAnnotatedWith(Expose.class);

        for (Element genElement : genElements) {
            System.out.println("开始处理：" + genElement.getSimpleName());
        }
        System.out.println("第 " + (++r) + " 次循环");
        return true;
    }
}
```

3. 声明清单文件

在Resources目录下创建META-INF目录以及services子目录，并创建javax.annotation.processing.Processor文件。

```
├── java
│   └── org
│       └── example
│           ├── Expose.java
│           └── ExposeProcessor.java
└── resources
    └── META-INF
        └── services
            └── javax.annotation.processing.Processor

```
清单文件内容为
```
org.example.ExposeProcessor
```

4. 打包

```xml
<build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <excludes>
                    <exclude>META-INF/**/*</exclude>
                </excludes>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>2.6</version>
                <executions>
                    <execution>
                        <id>process-META</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>target/classes</outputDirectory>
                            <resources>
                                <resource>
                                    <directory>${basedir}/src/main/resources/</directory>
                                    <includes>
                                        <include>**/*</include>
                                    </includes>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```

5. 测试

新创建一个项目，并添加注解：

```xml
<dependency>
      <groupId>org.example</groupId>
      <artifactId>jsr269demo</artifactId>
      <version>1.0-SNAPSHOT</version>
</dependency>
```

```java
public class Test {

    @Expose
    public void hello() {

    }

    @Expose
    public void world() {

    }
}
```

执行mvn clean package时可以看到输出

```
begin process ... 
输入的类:App
输入的类:Test
开始处理：hello
开始处理：world
第 1 次循环
begin process ... 
第 2 次循环


```