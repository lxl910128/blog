---
title: JAVA注解
date: 2019/3/8 18:36:10
categories:
- JAVA
tags:
- 注解
keywords:
- java, annotation , 注解
---
# 介绍
JAVA annotation（注解）的官方解释是:
>
Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。Java 注解是从 Java5 开始添加到 Java 的。
>
<!--more-->
JAVA注解个人理解就是给类、方法、变量等打标签，然后在编译或运行时对同类标签做有关处理。根据经验一般与反射机制一起使用。

# 定义
注解的定义方式和接口类似，但使用`@interface`定义。注解上可以再使用注解（元注解）。以lambda表达式的@FunctionalInterface注解为例，其定义如下。
```java
package java.lang;

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface FunctionalInterface {}
```
其中`@Documented` `@Retention(RetentionPolicy.RUNTIME)` `@Target(ElementType.TYPE)`是元注解，`@interface`是注解关键字，`FunctionalInterface`为注解名称。

# 元注解
元注解主要有：
* @Documented
* @Inherited
* @Target
* @Retention
* @Repeatable
* @Native
其中前4个是java1.5出现的，后两个是1.8新增的。他们都在`java.lang.annotation`包下。

## @Documented
被Documented标记的注解再标记实际元素后，该注解注解会包含到元素的 Javadoc 中去。

## @Inherited
被@Inherited标记的注解会自动被继承。即如果注解A被@Inherited标记，且注解A作用到了类A上，那么类A的子类B也默认被A注解标记。

## @Target
@Target注解指定了被标记注解的可运用的对象。其值包括：

* ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
* ElementType.CONSTRUCTOR 可以给构造方法进行注解
* ElementType.FIELD 可以给属性进行注解
* ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
* ElementType.METHOD 可以给方法进行注解
* ElementType.PACKAGE 可以给一个包进行注解
* ElementType.PARAMETER 可以给一个方法内的参数进行注解
* ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举
* ElementType.TYPE_PARAMETER 标记任何类型
* ElementType.TYPE_USE 标记任何类型

比如@FunctionalInterface被标记@Target(ElementType.TYPE)。则说明@FunctionalInterface可以标记类、接口、枚举。`ElementType`在`java.lang.annotation`包下。  


ElementType.TYPE_PARAMETER和ElementType.TYPE_USE是1.8新增的类型，在1.5之前注解只能在声明的地方使用，但1.8定义了这2个类型后注解可以应用在任何地方。比如:
```java
//创建类实例
 new @Interned MyObject();
 //类型映射
myString = (@NonNull String) str;
//implements 语句中
class UnmodifiableList<T> implements @Readonly List<@Readonly T> { ... }
//throw exception声明
 void monitorTemperature() throws @Critical TemperatureException { ... }
```
需要注意的是，类型注解只是语法而不是语义，并不会影响java的编译时间，加载时间，以及运行时间，也就是说，编译成class文件的时候并不包含类型注解。一般类型注解用来做程序中做强类型检查，跟插件式的check framework，可以在编译的时候检测出runtime error，以提高代码质量。  

## @Retention
表示注解可以保存的时间。
* RetentionPolicy.SOURCE //注解保留在源码阶段，被编译器丢弃
* RetentionPolicy.CLASS // 注解被编译器保存在字节码文件中，在运行时丢弃，默认的保留行为
* RetentionPolicy.RUNTIME //被虚拟机保存，可用反射机制读取

RetentionPolicy.CLASS主要是编译时做相关处理一般不用。一般编程中主要是RetentionPolicy.RUNTIME与反射技术结合实现逻辑。

## @Repeatable
被其标记的注解在使用时可以重复使用2次。
1.8之前：
```java
public @interface Authority { String role(); } 
public @interface Authorities { Authority[] value(); }

public class RepeatAnnotationUseOldVersion {
    @Authorities({@Authority(role="Admin"),@Authority(role="Manager")}) 
    public void doSomeThing(){ } 
}
```
1.8后：
```java
@Repeatable(Authorities.class) 
public @interface Authority { String role(); } 

public @interface Authorities { Authority[] value(); }

public class RepeatAnnotationUseNewVersion { 
@Authority(role="Admin") 
@Authority(role="Manager") 
public void doSomeThing(){ } 
}
```

## @Native
@Native表示定义常量值的字段可以被native代码引用，当native代码和java代码都需要维护相同的常量时，如果java代码使用了@Native标志常量字段，可以通过工具将它生成native代码的头文件。该注解与元注解放在一起，但看起来并不是元注解。

# 注解使用示例
个人认为根据JAVA注解存活的时间，注解可以分为方便其他程序分析源码的注解（RetentionPolicy.SOURCE），给JAVA编译器看的注解（RetentionPolicy.CLASS），以及运行时借助反射机制实现逻辑的注解（RetentionPolicy.RUNTIME）。本文着重介绍最后一种注解的应用。  


比如有这样一种业务场景，在处理业务请求时一般需要统一打印request和response日志。但是有诸如下载的请求时不应该打印response日志的。这时我们可以创建一个注解标注service的方法，从而达到白名单的效果。

# 参考
https://blog.csdn.net/falynn1220/article/details/50724648
https://blog.csdn.net/briblue/article/details/73824058
https://my.oschina.net/benhaile/blog/179642
https://my.oschina.net/benhaile/blog/180932
https://juejin.im/entry/5b286169e51d4558de5bcd80