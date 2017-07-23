---
date: 2017-07-23
layout: post
title: Linux内核-tensorflow学习笔记
categories: Linux
tags: Kernel
excerpt: 这个是一个特殊的笔记,因为还在学习中, 所以也就用了一些特殊的方法隐藏起来, 如果你看到了请不要笑话, 该文用来记录我学习python, tensorflow的笔记。
---

## **2017-7-23**

**1 今天晚上浏览了[Tensflow官方中文文档](http://www.tensorfly.cn/tfdoc/get_started/introduction.html)**

**2  关于python中* 和 dot的区别**
![](/blog/assets/kernel/tensorflow-2.png)
![](/blog/assets/kernel/tensorflow-3.png)
![](/blog/assets/kernel/tensorflow-1.png)

**3 Python xrange与range的区别**

[Python xrange与range的区别](http://www.nowamagic.net/academy/detail/1302446)

xrange做循环的性能比range好，尤其是返回很大的时候, **xrang不会返回一个list而是返回一个生成器,所以不用开辟一个很大的存储空间**。尽量用xrange吧，除非你是要返回一个列表。