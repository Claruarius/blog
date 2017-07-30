---
date: 2017-07-23
layout: post
title: Linux内核-tensorflow学习笔记
categories: Linux
tags: Kernel
excerpt: 这个是一个特殊的笔记,因为还在学习中, 所以也就用了一些特殊的方法隐藏起来, 如果你看到了请不要笑话, 该文用来记录我学习python, tensorflow的笔记。
---

## **2017-07-30**

**1 什么是SWIG**

SWIG是个帮助使用C或者C++编写的软件能与其它各种高级编程语言进行嵌入联接的开发工具。SWIG能应用于各种不同类型的语言包括常用脚本编译语言例如Perl, PHP, Python, Tcl, Ruby and PHP。支持语言列表中也包括非脚本编译语言，例如C#, Common Lisp (CLISP, Allegro CL, CFFI, UFFI), Java, Modula-3, OCAML以及R，甚至是编译器或者汇编的计划应用（Guile, MzScheme, Chicken）。SWIG普遍应用于创建高级语言解析或汇编程序环境，用户接口，作为一种用来测试C/C++或进行原型设计的工具。SWIG还能够导出XML或Lisp s-expressions格式的解析树。SWIG可以被自由使用，发布，修改用于商业或非商业中。

简单说来就是C/C++用来与其他脚本语言沟通的工具

**2 使用SWIG编写C++模块的python接口**

1 安装SWIG, sudo apt-get install swig

2 编写C++代码

{% highlight C++ %}
#include <time.h>

double My_variable = 3.0;

int fact(int n) {
  if(n <= 1) return 1;
  else return n * fact(n-1);
}

int my_mod(int x, int y) {
  return (x%y);
}

char* get_time() {
  time_t ltime;
  time(&ltime);
  return ctime(&ltime);
}

{% endhighlight %}

3 编写swig接口文件

{% highlight swig %}
%module example
%{
  extern double My_variable;
  extern int fact(int n);
  extern int my_mod(int x, int y);
  extern char* get_time();
%}

extern double My_variable;
extern int fact(int n);
extern int my_mod(int x, int y);
extern char *get_time();

{% endhighlight %}

4 执行以下命令

```
swig -python example.i

gcc -c example.c example_wrap.c -I/usr/include/python2.7/ -fPIC

ld -shared example.o example_wrap.o -o _example.so 
```

5  接下来就可以在python文件中使用example模块

![](/blog/assets/kernel/tensorflow-9.png)


## **2017-07-28到2017-07-29**

**1 emacs显示目录, emacs快速切换窗口**

折腾了一个晚上终于弄好了emacs中关于窗口的焦点切换和cedet+ecb作为显示界面,这里做个笔记吧:

(1)切换窗口焦点

> 1 安装window-numbering.el,然后就可以执行下面的操作.
> 
> 2 焦点移动到下一窗口(常用). M-x other-window,快捷键是`C-x o`.
> 
> 3 水平/垂直切分当前窗口(常用). 快捷键`C-x 2/3`.
> 
> 4 将当前子窗口最大化(常用). M-x delete-other-windows,快捷键`C-x 1`.
> 
> 5 删除当前子窗口(偶尔用). M-x delete-window,快捷键`C-x 0`.

(2) 安装cedet+ecb

> 1 emacs 24.5已经有默认安装cedet
>
> 2 下载ecb, 然后放在.emacs.d文件下, 配置.emacs如下
>```
> ;;将ECB的路径添加到读取列表中
> (add-to-list 'load-path "/your-home-path/.emacs.d/elpa/ecb-master")
> ;;告诉emacs启用ecb
> (require 'ecb)
>```

(3) 最后来一个截图

![](/blog/assets/kernel/tensorflow-8.png)

**2 疑问or任务**

> SWIG是什么? bazel是什么?
> 
> 如何看懂tensorflow源码


## **2017-07-27**

**1 python代码中如何书写类?**
```
class Student(object):			#类名
    def __init__(self, name, score):	#类似于构造函数的函数
        self.name = name
        self.score = score

    def printScore(self):		#成员函数
        print'%s : %s' % (self.name, self.score)

```

具体的python类的书写可以参考[Python中的类(classes)](http://blog.csdn.net/on_1y/article/details/8640012)

**2 python中如何书写被其他模块引用的模块**

假设上文中的Student类被写在A/test/test.py中,在A/中有test1.py想要引用Student类.
```
from test.test import *

cla = Student('Claruarius', '95')
cla.printScore();

```
然后运行python test.py出现如下的错误:
```
Traceback (most recent call last):
  File "test1.py", line 1, in <module>
    from test.test import *
ImportError: No module named test
```
为了能够让其他模块引用到Student类,需要在Student类的同级目录下存在一个__init__.py的文件,即使这个文件是空的.

**3 import与 from XXX import XXX有什么区别**

区别如下:
![](/blog/assets/kernel/tensorflow-6.png)
![](/blog/assets/kernel/tensorflow-7.png)

**4 什么是.i文件**

.i就是SWIG的interface文件

**5 几个emacs常用的命令**

(1)M-x rgrep 输入要查找的单词

(2)M-x find-name-dired 输入查找文件名称

(3)
C-w剪切

M-w复制

C-y粘贴

**6 任务or疑问**

> SWIG是什么? bazel是什么?
> 
> 如何看懂tensorflow源码
> 
> emacs显示目录, emacs快速切换窗口

明天加油!

## **2017-07-26**

**1 画图**
![](/blog/assets/kernel/tensorflow-5.png)

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
 -> $$S(\beta) = \sum(X\beta - y)^2$$
通过对$$S(\beta)$$进行微分求最值, 可以得到

$$\sum_i(X_i\beta_j - y_i)X{_i}{^j} = 0$$
 -> $$\sum_i(X_iX{^j}{_i}\beta) = \sum_i(X{^j}{_i}y_i)$$ [备注: $$X{^j}{_i}$$代表X的第i行第j列, $$\beta$$参数的个数决定了最后结果的行数]

对以上的式子进行解读:**X的第j列的所有行($$\sum_i$$)乘上对应的y然后求和,这就是$$X^Ty$$, 同理可以得到$$X^T(X\beta)$$**

随后得到$$X^TX\beta = X^Ty$$, 就是$$\beta = (X^TX)^{-1}X^Ty$$

终于推导出求参数的公式,不得不说矩阵真是神奇和伟大!

**3 任务or疑问**

> 明天装个画图软件画图!

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

## **2017-07-23**

**1 今天晚上浏览了[Tensflow官方中文文档](http://www.tensorfly.cn/tfdoc/get_started/introduction.html)**

**2  关于python中* 和 dot的区别**
![](/blog/assets/kernel/tensorflow-2.png)
![](/blog/assets/kernel/tensorflow-3.png)
![](/blog/assets/kernel/tensorflow-1.png)

**3 Python xrange与range的区别**

[Python xrange与range的区别](http://www.nowamagic.net/academy/detail/1302446)

xrange做循环的性能比range好，尤其是返回很大的时候, **xrang不会返回一个list而是返回一个生成器,所以不用开辟一个很大的存储空间**。尽量用xrange吧，除非你是要返回一个列表。