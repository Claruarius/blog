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

$$J(\theta) =  \frac {1}{2} \sum^m_{i=1}(h_\theta(x^{(i)})-y^{(i)})^2$$

现在需要求最小的$$J(\theta)$$

式子前面乘上1/2是为了在求导时候这个系数不见

如何调整$$\theta 似的J(\theta)取得最小值$$有很多方法,其中一种是**梯度下降法**. 梯度下降法的流程如下:
> 首先对$$\theta$$赋值, 这个值可以是随机的, 也可以让$$\theta$$是一个全零的向量
> 
>改变$$\theta$$的值, 使得$$J(\theta)$$按梯度下降方向进行减少

对函数$$J(\theta)$$求偏导数J

$$\frac{\delta}{\delta\theta}J(\theta) = \frac{\delta}{\delta\theta}\frac {1}{2} \sum^m_{i=1}(h_\theta(x^{(i)})-y^{(i)})^2$$
$$=(h_\theta{_j}(x^{(i)})-y^{(i)})x^{(i)}_j$$

**更新$$\theta$$的值**

$$\theta_i = \theta_i - \alpha\frac{\delta}{\delta\theta}J(\theta) = \theta_i - \alpha(h_\theta{_j}(x^{(i)})-y^{(i)})x^{(i)}_j$$

**2 疑问**
>(1)为什么要这样更新$$\theta$$可以达到全局或局部最小值
> 
> (2)什么是最小二乘法(数学描述方法)

明天加油!

## **2017-07-25**

**1 针对Gradient Descent, 为什么要这样更新$$\theta$$可以达到全局或局部最小值**

(1) 如果要求$$y = x^2$$,如果使用Gradient Descent方法来求该函数的最小值,那么首先求y对x的偏导数:$$\frac{\delta y}{\delta x} = 2x$$,分以下两种情况:
> 当x<0时候, 2x<0. 运算$$x = x - 2x$$时候, x将会向0靠近,而x=0,就是y的最小值
>
> 当x>0时候, 2x>0. 运算$$x = x - 2x$$时候, x将会向0靠近,而x=0,就是y的最小值

![](/blog/assets/kernel/tensorflow-4.png)

真不可思议.

能够得到的阶段性结论:**偏导数代表梯度, 是+梯度还是-梯度,看你要求最大值还是最小值.**

**2 什么是最小二乘法**

最小二乘法(generalized least squares)是一种数学的优化技术. [为什么我觉得英文让我更能理解这个词在讲什么? 哭笑脸.jpg]. 它通过`最小化误差的平方和`找到一组数据的`最佳`函数匹配.

考虑方程组:

$$\sum^{n}_{j=1}X_{ij}\beta_{j}=y_{i},(i=1,2,3,...m)$$

其中m代表有m个等式, n代表有n个未知数$$\beta$$, m>n, 将其进行向量化后为:

$$X\beta = y$$

$$X=
\left\{
\begin{matrix}
   X_{11} & X_{12} & ... & X_{1n} \\
   X_{21} & X_{22} & ... & X_{2n} \\
   ...& ...& ...& ...& \\
   X_{m1} & X_{m2} & ... & X_{mn}
  \end{matrix}
  \right\},
$$
$$\beta = \left\{
\begin{matrix}
\beta_1, \\
\beta_2, \\
... \\
\beta_n
\end{matrix}\right\},
$$
$$y= \left\{
\begin{matrix}
y_1, \\
y_2, \\
... \\
y_m
\end{matrix}\right\}
$$

为了选取最合适的$$\beta$$让等式"尽量成立", 引入**残差平方和函数S**

$$S(\beta)=\left|X\beta-y \right|^2$$
 -> $$S(beta) = \sum(X\beta - y)^2$$
通过对$$S(\beta)$$进行微分求最值, 可以得到

$$\sum_i(X_i\beta_j - y_i)X{_i}{^i} = 0$$
 -> $$\sum_i(X_iX{^j}{_i}\beta) = \sum_i(X{^j}{_i}y_i)$$ [备注: $$X{^j}{_i}$$代表X的第i行第j列, \beta参数的个数决定了最后结果的行数]

对以上的式子进行解读:**X的第j列的所有行($$\sum_i$$)乘上对应的y然后求和,这就是$$X^Ty$$, 同理可以得到X^T(X\beta)**

随后得到$$X^TX\beta = X^Ty$$, 就是$$\beta = (X^TX)^{-1}X^Ty$$

终于推导出求参数的公式,不得不说矩阵真是神奇和伟大!

**3 任务or疑问**

> 明天装个画图软件画图!
