---
date: 2017-02-16
layout: post
title: WifiTracker的设计及其问题
categories: Android
tags: Wi-Fi 设计模式
excerpt: WifiTracker的设计中有两个Handler，一个负责读取数据，一个负责更新数据。
---
### **WifiTracker 中两个handler**

**1、MainHandler**
![](/blog/assets/wifi/wifitracker-mainhandler.png)
看MainHandler中的消息可知道MainHandler是用来通知观察者状态发生变化需要过来取数据了。

**2、WorkerHandler**
![](/blog/assets/wifi/wifitracker-workerhandler.png)
从WorkerHandler消息看来主要是用来更新数据的

**3、观察者模式**

是的，WifiTracker是使用了观察者模式，类图大概如下（PS：没装viso，用exccel画类图不要见怪。捂脸.jpg）
![](/blog/assets/wifi/wifitracker-class-pic.png)

也就是说观察者可以是WifiSettings或者是SystemUI的WifiTile等

### **问题来了**
WifiTracker中有两个Handler，也就是有两个异步。
我们暂且可以理解为MainHandler是读取数据用的，WorkerHandler是用于更新数据用的。两个异步的线程同时操作一个数据块，但是一个读，一个写，好像可以很和谐，而且Google也未在WifiTracker中加任何的同步锁，可能是为了快速读取数据更新数据吧。

### **数据块--AccessPoint**
WifiTracker的两个Handler维护的是一个AccessPoint列表，查看AccessPoint数据结构
![](/blog/assets/wifi/wifitracker-accesspoint.png)
除了update函数用于给WifiTracker更新数据外，其他都是get函数。
根据`调用者需要检查被调用者返回值是否符合要求`的良好编码原则，基于AccessPoint这样的函数接口和结构特点，WifiTracker对于AccessPoint的读和写是没有问题的，这也是Google没有加锁的原因。
但这也埋下了一个坑，如果参与代码贡献的人没有意识到这种结构特点，就有可能造成读写冲突从而引`发空指针异常`的惨剧。
事实上已经存在这样的代码，如下
![](/blog/assets/wifi/wifitracker-accesspoint-getvisibilitystatus.png)

**getVisibilityStatus函数**是一个用于在打开wifi调试开关后，显示wifi调试信息的函数，但是在该函数中存在对与本结构中数据的引用，前面我们说过更新过程中，数据有可能被置 null，但是如果只是作为返回值对于这个数据结构没有什么太大问题，因为由调用者负责检查。但是如果在被两个线程异步操作的数据结构中去解引用一个有可能被置null的数据，而且**这种行为存在其成员函数中**就是这个数据结构本身的问题，是需要在该数据结构内部解决该问题。

很明显getVisibilityStatus函数打破了，AccessPoint和WifiTracker这种平衡，针对该函数中的这种写法，目前正在维持该结构特点的情况下寻找解决办法规避空指针
