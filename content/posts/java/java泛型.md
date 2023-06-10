---
layout: post
title: "java泛型"
date: 2022-10-2
categories: 
    - "java"
tags: [java]
author: "pan"
---

# java 泛型

​很多朋友对java的泛型不是很理解，很多文章写的已不是很清楚，这篇博客对java泛型进行 一个总结。

## 1.泛型的转换

List<Number> foo1 = new ArrayList<Integer>();//illegal

很多朋友会写出上面的代码，但会报如下错误：Type mismatch: cannot convert from ArrayList<Integer> to List<Number>

尽管Interge是Number的子类，但是ArrayList<Integer>不是List<Number>的子类，所以报错。下图可以很好解释这个问题。

![image.png](/java泛型/8596800-1cf03585c59b7679.png)

## 2.java泛型的通配符?

这里可以分为两类（1）? extends T (2) ? super T.

很多朋友对这两个不是很理解，也不知道上面时候用，我们知道java中提供泛型技术，是为了提供安全检查的，使得我们写的代码更加的健壮。

### 2.1 ? extends T

```java
public static void print_e(List<? extends Number> list){
    for(Number n : list){
        System.out.println(n);
    }
}
```

上面一个函数，我们可以传递如下的参数

```java
List<Integer> list_i = new ArrayList<Integer>();
    for(int i=0;i<10;i++){
        list_i.add(i);
    }
    List<Double> list_d = new ArrayList<Double>();
    for(int i=0;i<10;i++){
        list_d.add(i+0.0);
    }
    print_e(list_i);
    print_e(list_d);
```

使得我们写的代码即具有通用型有可以提供必要的安全检查，当然print_e你可以写出如下形式，这里就不具有安全检查的效果了。

```java
 void print_e(List list)
```

但是经常有的朋友写出如下的代码，我们举一个stackoverflow上的一个例子：

```java
List<? extends Number> foo3 = new ArrayList<Number>();  // Number "extends" Number (in this context)
List<? extends Number> foo3 = new ArrayList<Integer>(); // Integer extends Number
List<? extends Number> foo3 = new ArrayList<Double>();  // Double extends Number
```

上面的代码都是可以通过的，但是如果你向foo3中添加一个元素，比如

```java
foo3.add(new Integer(1))
```

将会报如下错误：

```java
The method add(capture#1-of ? extends Number) in the type List<capture#1-of ? extends Number> is not applicable for the arguments (Integer)
```

很多人觉得奇怪，Integer明明是Number的子类，为什么添加不进去了。

```java
List<? extends Number> foo3 = new ArrayList<Number>();  // Number "extends" Number (in this context)
```

stackoverflow上一个朋友是这样解释的，解释的很好，借用他的解释如下：

```java
Reading - Given the above possible assignments, what type of object are you guarenteed to read from List foo3:

You can read a Number because any of the lists that could be assigned to foo3 contain a Number or a subclass of Number.
You can't read an Integer because foo3 could be pointing at a List<Double>.
You can't read a Double because foo3 could be pointing at a List<Integer>.
Writing - Given the above possible assignments, what type of object could you add to List foo3 that would be legal for all the above possible ArrayList assignments:

You can't add an Integer because foo3 could be pointing at a List<Double>.
You can't add a Double because foo3 could be pointing at a List<Integer>.
You can't add a Number because foo3 could be pointing at a List<Integer>.
You can't add any object to List<? extends T> because you can't guarantee what kind of List it is really pointing to, so you can't guarantee that the object is allowed in that List. The only "guarantee" is that you can only read from it and you'll get a T or subclass of  T.
```

读：**可以以Number类型去读**，因为绑定到foo3的List肯定是存放Number类型或者Number类型子类(Integer、Double等)的List; **但是不能以Integer、Double类型去读**，因为你以Integer类型去读的时候，foo3可能是绑定List<Double>，有的同学可能会说我写代码的肯定是知道foo3是绑定的List<Double>还是List<Integer>的，但是java存在泛型擦除的问题（编译时检查，运行是相当于List<Object>, 类型丢失）

写： **因为 List<? extends Number> foo3 中你既可以添加一个Integer的有可以添加一个Double的，所以编译器不知道你具体添加到的是哪一种类型，所以编译器不允许你添加元素**

### 2.2 ? super T

再来看一下? super T的例子：

```java
List<? super Integer> foo3 = new ArrayList<Integer>();  // Integer is a "superclass" of Integer (in this context)
List<? super Integer> foo3 = new ArrayList<Number>();   // Number is a superclass of Integer
List<? super Integer> foo3 = new ArrayList<Object>();   // Object is a superclass of Integer
```

stackoverflow的解释如下：

```java
Reading - Given the above possible assignments, what type of object are you guaranteed to receive when you read from List foo3:

You aren't guaranteed an Integer because foo3 could be pointing at a List<Number> or List<Object>.
You aren't guaranteed an Number because foo3 could be pointing at a List<Object>.
The only guarantee is that you will get an instance of an Object or subclass of Object (but you don't know what subclass).
Writing - Given the above possible assignments, what type of object could you add to List foo3 that would be legal for all the above possible ArrayList assignments:

You can add an Integer because an Integer is allowed in any of above lists.
You can add an instance of a subclass of Integer because an instance of a subclass of Integer is allowed in any of the above lists.
You can't add a Double because foo3 could be pointing at a ArrayList<Integer>.
You can't add a Number because foo3 could be pointing at a ArrayList<Integer>.
You can't add a Object because foo3 could be pointing at a ArrayList<Integer>.
foo3你是不能读的，读出报如下类似错误Type mismatch: cannot convert from capture#2-of ? super Integer to Integer
```

因为foo3中的元素有可能是Integer，Number或者是Object的，所以编译器不知道读出的是什么类型，所以不允许你读出元素。

**总的来说，你要读就用? extends T，你要写就就用 ? super T,你既要读，又要写，你就不用泛型直接定义List list**，这样就好了，如果没还没有看明白，直接看stackoverflow上的这篇文章，解释的很好

参考文献
1. [Difference between <? super T> and <? extends T> in Java](http://stackoverflow.com/questions/4343202/difference-between-super-t-and-extends-t-in-java)

2. [java泛型－类型擦除](http://blog.csdn.net/caihaijiang/article/details/6403349)


​