---
title: JAVA函数式接口及lambda表达式
date: 2019/3/8 18:18:10
categories:
- JAVA
tags:
- lambda
---
# 概要
本文从构造线程的新方式出发，首先介绍了函数式接口，然后介绍了lambda表达式，接着分析了JDK提供的函数式接口，着重介绍了`Function`接口。最后稍带介绍了闭包和方法引用。
<!--more-->

# 起因
故事是这样发生的。这天程序员小P正开开心心的写代码，在查询资料时忽然发现启动线程有种特别酷炫的方式，大体上是这个样子：
```java
String s = "just for fun";
new Thread(() -> {
    System.out.println(s);
}).start();
```

小P顿时感叹代码之精妙。当然做为混迹JAVA圈近3年的老菜鸟，小P知道这是用到了**Lambda表达式**，但自己还不甚理解，遂决定仔细研究一下，化为己用。

# FunctionalInterface注解
小P了解`Thread`对象在构造时是需要传用一个实现了`Runable`接口的实例的，莫非`Thread`还可以接收其它类型的参数来初始化么？通过查看`Thread`的构造函数，小P发现构造线程需要的主要变量是1.实现了`Runnable`接口的对象; 2.自定义线程名; 3.`ThreadGroup`线程组表示一个线程的集合，并没有其它类型的传参。  

在idea的帮助下，小P发现创建线程使用的构造函数接受的参数是`Runnabel`实例，那么就代表代码中的lambda表达是其实可以理解为是一个实现了`Runnable`接口的实例化对象。莫非所有接口都可以用`lambda表达式`实例化？这显然不可能，因为一个接口可以定义很多函数而`lambda表达式`显然仅表示1个函数。那么肯定是`Runnable接口`有什么不同之处。果然该接口上有个`@FunctionalInterface`注解，如下  
```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```
通过阅读`@FunctionalInterface`注解源(zhu)码(shi)可以了解到，它是用于注解`functional interface`接口(又称函数式接口)。从概念上讲`function interface`有且仅有1个抽象方法。但同时函数式接口里允许定义默认方法也可以定义静态方法，同样允许定义`java.lang.Object`里的public方法，这三种方法不算在那一个抽象方法内。*函数式接口的实例可以用lambda表达式表示*。根据源码的表述小P意识到如果函数需要的参数是个被`@FunctionalInterface`标记的接口，那么这个参数就可以使用lambda表达式，lambda表达是的结构应与接口唯一的抽象函数相符。那么lambda表达式是如何构成的呢？

# Lambda表达式
Lambda表达式由`()`,`->`,`{}`三部分组成。小括号内放传入参数的名称，如果仅有1个参数小括号可以省略。箭头表示连接符。大括号内是逻辑代码。比如接受一个字符串参数并返回其小写字符串，这个函数的lambda形式就是`x->{return x.toLowerCase();}`。如果函数是仅一个传参且无返回对象，那么lambda表达式的小括号和大括号都可以省略例如`x -> System.out.println(x)`。

# 常用函数式接口
从JAVA 1.8开始，JDK为我们提供了很多通用的函数式接口，当这些接口作为函数参数时都可以用lambda来代替。这些接口有：Function、Predicate、Supplier、Consumer和它们的子接口，这些接口都在`java.util.function`中。

## Function
直接上源码
```java
//Represents a function that accepts one argument and produces a result.
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     */
    R apply(T t);

    /**
     * Returns a composed function that first applies the {@code before}
     * function to its input, and then applies this function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     */
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    /**
     * Returns a composed function that first applies this function to
     * its input, and then applies the {@code after} function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     */
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    /**
     * Returns a function that always returns its input argument.
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```
正如源码所说，该接口就是接受1个值并返回一个值，两个值的类型可以不一样。其唯一的抽象方式是`R appliy(T t)`。值得注意的是该接口有2个默认函数和一个静态函数。  

两个默认函数都是为了实现嵌套，`compose`是接受一个`Function`返回一个新的`Function`，这个新`Function`的`apply`功能是先调用参数`Function`的`apply`再调用其自身的`apply`。而`andThen`与`composed`正好相反，返回`Funciton`的`apply`是先调用本函数的`apply`再调用参数的`apply`。从这里看出来，compose和andThen对于两个函数f和g来说，f.compose(g)等价于g.andThen(f)。而静态`identity`是返回一个`Function`，它的`apply`就是讲传入参数直接返回。

其实嵌套是十分有用的,下面举一个例子，假设有这样一个逻辑将传入的整数乘2再平方，或平方再乘2，则可以这么写
```java
Function<Integer,Integer> doubleMethod = x -> x * 2;
Function<Integer,Integer> squareMethod = x -> x * x;
// (3*2)^2 = 36
System.out.println(doubleMethod.andThen(squareMethod).apply(3));
System.out.println(squareMethod.compose(doubleMethod).apply(3));
// 3^2*2 = 18
System.out.println(doubleMethod.compose(squareMethod).apply(3));
System.out.println(squareMethod.andThen(doubleMethod).apply(3));
```

## Predicate
predicate是一个谓词函数，主要作为一个谓词演算推导真假值存在，其意义在于帮助开发一些返回bool值的Function。本质上也是一个单元函数接口，其抽象方法test接受一个泛型参数T，返回一个boolean值。等价于一个Function的boolean型返回值的子集。
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
    ...
}
```
其默认方法也封装了and、or和negate逻辑。

## Consumer
看名字就可以想到，这像谓词函数接口一样，也是一个Function接口的特殊表达——接受一个泛型参数，不需要返回值的函数接口。
```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
    ...
}
```
## Supplier
最后说的是一个叫做Supplier的函数接口，其声明如下：
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```
其简洁的声明，会让人以为不是函数。这个抽象方法的声明，同Consumer相反，是一个只声明了返回值，不需要参数的函数（这还叫函数？）。也就是说Supplier其实表达的不是从一个参数空间到结果空间的映射能力，而是表达一种生成能力，因为我们常见的场景中不止是要consume（Consumer）或者是简单的map（Function），还包括了new这个动作。而Supplier就表达了这种能力。

# 其它相关
小P在浏览相关技术博客时还发现了闭包和方法引用这两个比较意思的东西。这里简单记录，有空在深入了解。  

闭包的表现就是在使用lambda表达式时，变量的使用可以突破方法体。
```java
String str = "hello word!";
new Thread(()-> System.out.println(str)).start(); //lambda调用了外部的str
```
如上例，如果使用通常的传入Runnable实例化对象的方式创建线程的话，`str`必须要人为手动的传到对象中。闭包的价值在于可以作为函数对象或者匿名函数，持有上下文数据，作为第一级对象进行传递和保存。闭包广泛用于回调函数、函数式编程中。  

方法引用可以表示1个方法而不调用这个方法。具体做法就`类名::方法明`。一个方法引用可以跟lambda一样用于实例化函数式接口。
```java
Supplier lambdaMode = () -> new String();
Supplier other = String::new;
//获取一个空串
lambdaMode.get();
other.get();
```
如代码所示，两个`Supplier`获取的都是1个新的空串。

# 总结
小P今天了解了函数式接口，知道了lambda表达式和方法引用可以表示实例化的函数式接口，同时学习了JDK提供的几种函数式接口的作用。顿时小P觉得胸前的工牌更加光亮耀眼了。



# 参考
https://www.zybuluo.com/changedi/note/616132
https://blog.csdn.net/quanaianzj/article/details/81540677
