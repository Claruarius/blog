---
date: 2017-07-23
layout: post
title: Linux内核-tensorflow学习笔记
categories: Linux
tags: Kernel
excerpt: 这个是一个特殊的笔记,因为还在学习中, 所以也就用了一些特殊的方法隐藏起来, 如果你看到了请不要笑话, 该文用来记录我学习python, tensorflow的笔记。
---

## **2017-07-23**

**1 今天晚上浏览了[Tensflow官方中文文档](http://www.tensorfly.cn/tfdoc/get_started/introduction.html)**

**2  关于python中* 和 dot的区别**
![](/blog/assets/kernel/tensorflow-2.png)
![](/blog/assets/kernel/tensorflow-3.png)
![](/blog/assets/kernel/tensorflow-1.png)

**3 Python xrange与range的区别**

[Python xrange与range的区别](http://www.nowamagic.net/academy/detail/1302446)

xrange做循环的性能比range好，尤其是返回很大的时候, **xrang不会返回一个list而是返回一个生成器,所以不用开辟一个很大的存储空间**。尽量用xrange吧，除非你是要返回一个列表。

## **2017-07-24**

**1 温习Gradient Descent算法**
如果使用$$X_1, X_2 ... X_n$$去描述feature的分量, 比如$$X_1=房间面积, X_2=房间的朝向$$我们可以计算函数:

$$h(x)=h_\theta(x)=\theta_0+\theta_1x_1+\theta_2x_2$$

$$\theta$$这里称为参数, 在这里的意思事调整每个feature 中的分量影响力. 如果让你x0 = 1,就可以表示为下面的式子:

$$h_\theta(x) = \theta^TX$$

为了评估$$\theta$$的好坏, 需要有一个评估机制. 一般这个函数称为**损失函数(loss function)** 如下:

$$J(\theta) =  \frac {1}{2} \Sigma^m_{i=1}(h_\theta(x^{(i)})-y^{(i)})^2$$

现在需要求最小的$$J(\theta)$$

式子前面乘上1/2是为了在求导时候这个系数不见

如何调整$$\theta 似的J(\theta)取得最小值$$有很多方法,其中一种是**梯度下降法**. 梯度下降法的流程如下:
> $$首先对\theta赋值, 这个值可以是随机的, 也可以让\theta是一个全零的向量$$
> 
>$$改变\theta的值, 使得J(\theta)按梯度下降方向进行减少$$

对函数$$J(\theta)$$求偏导数J

$$\frac{\delta}{\delta\theta}J(\theta) = \frac{\delta}{\delta\theta}\frac {1}{2} \Sigma^m_{i=1}(h_\theta(x^{(i)})-y^{(i)})^2$$
$$=(h_\theta{_j}(x^{(i)})-y^{(i)})x^{(i)}_j$$

**更新$$\theta$$的值**

$$\theta_i = \theta_i - \alpha\frac{\delta}{\delta\theta}J(\theta) = \theta_i - \alpha(h_\theta{_j}(x^{(i)})-y^{(i)})x^{(i)}_j$$

**2 疑问**
>(1)为什么要这样更新$$\theta$$可以达到全局或局部最小值
> 
> (2)什么是最小二乘法(数学描述方法)

明天加油!