---
date: 2017-09-13
layout: post
title: Java中的Thread和String相关知识
categories: language
tags: Java
excerpt: 让人很激动, language分类迎来了一篇新的文章. 自己在大学时候曾经标榜是语言爱好者, 只是纯粹的喜欢各种语言的特性. 最近遇到几个有关Java相关的问题, 那就让我们来看一看. 
---

让人很激动, language分类迎来了一篇新的文章. 自己在大学时候曾经标榜是语言爱好者, 只是纯粹的喜欢各种语言的特性. 最近遇到几个有关Java相关的问题, 那就让我们来看一看.

## **1. String, StringBuilder和Stringbuffer的区别**

**1. String**

在Java的源代码中有如下的描述:
![](/blog/assets/language/java-string.png)

可以看到说明中提到, String是不可变的, 是常量.

**为什么String要设计成常量? **这个其实跟JVM的设计有关. JVM会维护一个"字符串池", 如果要创建的字符串已经在这个池中存在, 就直接将引用指向该字符串地址.

**为什么要维护一个字符串池?** 我们都知道字符串是每一种语言最常用的类型, 特别是像C++ 和 Java这种高级语言. 使用频率高, 如果String不是使用常量池来维护, 那么就会造成有很多具有相同的字符串变量存在, 造成内存空间使用紧张. 还有另一个优势, 由于String是常量, 只具有读取能力, 所以也是线程安全的.

```
String strA = "A";
String strA1 = "A";

String strC = strA + strA1;
```
strA 和 strA1 的地址应该是一样的, strC就是另一个字符串常量了. 通过如下的方法验证, 如果hashCode是一样的, 就证明是同一个字符串常量.

```
        String strA = "A";
        String strA1 = "A";

        System.out.println("strA hashCoode: " + strA.hashCode()
                        + ", strA1 hashCode: " + strA1.hashCode());

        String strC = strA + strA1;
        System.out.println("strC hashCOde: " + strC.hashCode());
```
![](/blog/assets/language/java-string-.1png)

可见, 结果是一样的.

**2. StringBuffer和StringBuiler**

![](/blog/assets/language/java-string-2.png)

从说明中可以看到, StringBuffer是可变的, 其长度可以改变, 只要调用恰当的方法. 如果涉及到进行大量的字符串拼接的时候, String对象将会频繁的进行对象创建, 而效率比较低. StringBuffer和StringBuilder就是创建字符串变量, 并在该变量原有的基础上进行拼接, 效率高于String.

StringBuffer区别于StringBuilder的另一个特点是, 说明中提到的StringBuffer是线程安全类型的. 经查看java对于StringBuffer的源码实现, 发现每一个方法都加上了**sychronized关键字**

![](/blog/assets/language/java-string-3.png)

