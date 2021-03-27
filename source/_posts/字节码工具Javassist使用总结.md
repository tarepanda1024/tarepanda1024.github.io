---
title: 字节码工具Javassist使用总结
tags:
  - 字节码
  - Java
categories: Java
date: 2021-03-27 10:48:23
---

## 前言

Javassist作为一个字节码生成工具，在链路追踪、性能分析等领域都有广泛的应用。不过[官方文档](https://www.javassist.org/tutorial/tutorial.html)只介绍了简单的用法，部分高级特性比如泛型显示、参数名称保留等没有详细介绍，同时网上的相关资料也比较缺乏。本文主要是结合开发过程中对Javassist的一些使用经验，总结了一下Javassist的基本用法和高级特性。


## 基本用法
Javassist对Java的各种概念进行了一层抽象，比如CtClass代表Class、CtMethod代表Method、CtField代表Field等。ClassPool是CtClass的容器，我们创建类、修改类都需要通过ClassPool。
### 新增类

#### 示例
```java
    @Test
    public void testCreateClass() throws Exception {
        //获取ClassPool实例
        ClassPool classPool = ClassPool.getDefault();
        //指定类名
        CtClass carCtClass = classPool.makeClass("com.demo.Car");

        //新增字段
        CtField ctField1 = CtField.make("private String brand = \"world\";", carCtClass);
        carCtClass.addField(ctField1);

        //创建方法形式1
        CtMethod helloMethod1 = CtMethod.make("public void run(){System.out.println(\"---run---\");}", carCtClass);
        carCtClass.addMethod(helloMethod1);

        //创建方法形式2
        CtMethod helloMethod2 = CtNewMethod.make("public void stop(){System.out.println(\"---stop---\");}", carCtClass);
        carCtClass.addMethod(helloMethod2);

        //创建方法形式3
        CtMethod helloMethod3 = CtNewMethod.make(Modifier.PUBLIC, CtClass.voidType, "slow", null, null, "{System.out.println(\"---slow---\");}", carCtClass);
        carCtClass.addMethod(helloMethod3);

        //添加构造器
        CtConstructor ctConstructor = CtNewConstructor.make(new CtClass[]{classPool.getCtClass(String.class.getName())}, null, carCtClass);
        ctConstructor.setBody("{ $0.brand = $1; }");
        carCtClass.addConstructor(ctConstructor);

        //添加默认构造器
        carCtClass.addConstructor(CtNewConstructor.defaultConstructor(carCtClass));

        //将方法写入文件
        carCtClass.writeFile("/home/lxd/test");
    }
```
执行上面的代码，我们就可以在```/home/lxd/test/com/demo/```目录下得到一个```Car.class```，将此文件拖入Idea，可以看到生成的代码如下：
![01.png](/images/20210327/01.png)


#### 说明

- 字段新增可以直接``` new CtField()```，也可以通过```CtField.make()```来进行操作，方法以及构造函数的操作新增类似
- 方法和构造函数还分别有```CtNewMethod```和```CtNewConstructor```，这两个类里面封装了很多有用的静态方法，可以简化操作，比如```CtNewMethod```内封装有```getter```以及```setter```的快捷方法，```CtNewConstructor```内封装有```defaultConstructor```等方法
- 在javassist中，```$0```代表this，```$1 $2 ...```分别对应方法的具体参数，这些信息可以从官方文档中看到。


### 编辑类

#### 示例

```java
    @Test
    public void modifyClass() throws Exception {

        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.get("com.example.Hello");
        CtMethod ctMethod = ctClass.getDeclaredMethod("sayHello", new CtClass[]{classPool.getCtClass(String.class.getName())});
        ctMethod.insertBefore("System.out.println(\"---before---\");");
        ctMethod.insertAfter("System.out.println(\"---after---\");");
        //完全替换method内容
        //ctMethod.setBody("{}");

        ctClass.writeFile("/home/lxd/test");
    }
```
通过上面的代码，我们可以为```sayHello```方法前后分别插入相关的日志打印功能。

![02.png](/images/20210327/02.png)

#### 说明

- ```CtMethod```有```insertBefore```和```insertAfter```方法，可以很方便的进行代码插桩。
- 将一个```class```类加载为```CtClass```后，我们就可以用CtField、CtMethod相关的抽象进行代码修改工作。
- ```ClassPool.getDefault()```默认和```jvm```有相同的类检索路径，我们可以通过```pool.insertClassPath("xxxx");```的形式新增加类检索路径。


## 高级特性
如果我们想用一些比较高级的特性，比如添加注解、泛型展示等，需要对类文件有一个大概的了解。通过对类文件结构的了解，我们可以很方便的从Javassist中找到对应的API。同时有一点需要注意：大部分高级操作都有相对来说比较简便、封装程度比较高的API，不用直接用Javassist的Bytecode类库。Bytecode相对来说较底层，操作的话对字节码了解水平要求较高。

![image.png](/images/20210327/03.png)

javassist.jar类文件结构如上图，相关class文件信息都可以从上面的两个包中找到相关抽象。

### 类文件
```java
ClassFile {
   u4             magic;
   u2             minor_version;
   u2             major_version;
   u2             constant_pool_count;
   cp_info        constant_pool[constant_pool_count-1];
   u2             access_flags;
   u2             this_class;
   u2             super_class;
   u2             interfaces_count;
   u2             interfaces[interfaces_count];
   u2             fields_count;
   field_info     fields[fields_count];
   u2             methods_count;
   method_info    methods[methods_count];
   u2             attributes_count;
   attribute_info attributes[attributes_count];
}
```
这里给出类文件的基本结构，详细各种属性，可以参考[jvm规范](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf)。
### 示例类
接下来的高级特性演示主要针对如下的```Hello```类：
```java
package com.example;

public class Hello {
    private String lang = "en";

    public void sayHello(String name) {
        System.out.println("Hello " + name);
    }

    public void setLang(String lang) {
        this.lang = lang;
    }
}
```

### 添加注解

```java
package com.example;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;


@Documented
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Log {
    String file();
}

```

假设我们想为上面的Hello类分别在类级别以及方法级别添加一个Log注解，我们需要创建出 ```AnnotationsAttribute``` ， 然后将这个```Attribute```添加到对应的```ClassFile```中以及```getMethodInfo```里面。

```java
    @Test
    public void addAnnotation() throws Exception {

        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.get("com.example.Hello");
        //获取到常量池
        ConstPool constpool = ctClass.getClassFile().getConstPool();
        
        //创建类级别的注解属性
        AnnotationsAttribute clzAnnotationAttr = new AnnotationsAttribute(constpool, AnnotationsAttribute.visibleTag);
        //类注解
        Annotation clzLogAnnotation = new Annotation(Log.class.getName(), constpool);
        clzLogAnnotation.addMemberValue("file", new StringMemberValue("CLASS_LOG", constpool));
        clzAnnotationAttr.setAnnotations(new Annotation[]{clzLogAnnotation});
        //添加到类属性中
        ctClass.getClassFile().addAttribute(clzAnnotationAttr);

        //创建方法级别的注解属性
        AnnotationsAttribute methodAnnotationAttr = new AnnotationsAttribute(constpool, AnnotationsAttribute.visibleTag);
        Annotation methodLogAnnotation = new Annotation(Log.class.getName(), constpool);
        methodLogAnnotation.addMemberValue("file", new StringMemberValue("METHOD_LOG", constpool));
        methodAnnotationAttr.setAnnotations(new Annotation[]{clzLogAnnotation});
        //添加到方法属性中
        CtMethod ctMethod = ctClass.getDeclaredMethod("sayHello");
        ctMethod.getMethodInfo().addAttribute(methodAnnotationAttr);

        //方法级别的注解类似
        //ctField.getFieldInfo().addAttribute();

        ctClass.writeFile("/home/lxd/test");
    }
```

执行后，我们就可以看到Hello类添加上了对应的注解。

![04.png](/images/20210327/04.png)

### 泛型显示
Java的泛型其实是一个语法糖，在运行态均是Object类型，然后强转为对应的类型。为了使class文件看起来相对有好一些，我们需要通过泛型签名来进行泛型展示。
```java
    @Test
    public void genericTest() throws Exception {
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.get("com.example.Hello");
        //添加字段的泛型签名
        CtField ctField = CtField.make("private java.util.List exts;", ctClass);
        ctField.setGenericSignature("Ljava/util/List<Ljava/lang/String;>;");
        ctClass.addField(ctField);

        //添加方法的泛型签名
        CtMethod ctMethod = CtNewMethod.make(" public void setExts(java.util.List exts) {$0.exts = $1;}", ctClass);
        ctMethod.setGenericSignature("(Ljava/util/List<Ljava/lang/String;>;)V");
        ctClass.addMethod(ctMethod);

        ctClass.writeFile("/home/lxd/test");
    }

```

#### 说明

- 我们创建方法或者字段，如果直接传入泛型，javassist会报错，只能通过添加泛型签名的形式进行泛型的展示。
- 泛型签名中类的全限定名需要用 ```/```代替```.```
- 我们还可以通过```SignatureAttribute.ClassSignature```以及```SignatureAttribute.MethodSignature```来简化泛型签名的拼写。

### 参数名称保留
```java
    @Test
    public void keepParamName() throws Exception {
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.get("com.example.Hello");
        ConstPool constPool = ctClass.getClassFile().getConstPool();

        //添加方法
        CtMethod ctMethod = CtNewMethod.make(" public void setExts(java.util.List exts) {}", ctClass);
        ctMethod.setGenericSignature("(Ljava/util/List<Ljava/lang/String;>;)V");

        String[] paramNames = new String[]{"exts"};
        int[] accessFlags = new int[]{0};

        //添加方法名称Attribute
        MethodParametersAttribute parametersAttribute = new MethodParametersAttribute(constPool, paramNames, accessFlags);
        ctMethod.getMethodInfo().addAttribute(parametersAttribute);

        ctClass.addMethod(ctMethod);

        ctClass.writeFile("/home/lxd/test");
    }
```
class文件中默认参数名称会变成```var$```的形式，原始参数名称不会保留，我们也需要通过添加属性到相关属性表中来保留参数名称。

## 总结
如果我们只是用来代码插桩或者不需要查看class文件的场景，那么javassist的官方文件已经满足大部分需要了。如果需要更深入的应用，则需要了解Javassist的源码。

