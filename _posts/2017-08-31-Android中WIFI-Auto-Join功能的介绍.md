---
date: 2017-08-31
layout: post
title: Android中WIFI Auto Join功能的介绍——以android N为例
categories: Android
tags: Wi-Fi
excerpt: 我是从android M开始接触WIFI自动连接这部分代码的。记得android M的这部分功能是一个名为WifiAutoJoinController的类在管理。 这个类里面有一个很重要的函数attemptAutoJoin()这个函数就是用来选择要连接哪个SSID的函数。 这个函数将近有500多行， 当时觉得这是很不人性化的，这个函数主要是运用过滤规则进行筛选合适的候选者。
---

我是从android M开始接触WIFI自动连接这部分代码的。记得android M的这部分功能是一个名为WifiAutoJoinController的类在管理。 这个类里面有一个很重要的函数attemptAutoJoin()这个函数就是用来选择要连接哪个SSID的函数。 这个函数将近有500多行， 当时觉得这是很不人性化的，这个函数主要是运用过滤规则进行筛选合适的候选者。

当然这么不人性化的代码书写，在以代码设计著称的Google里是存活不久的。在android N中这个功能被两个类替代。OK，接下来从代码结构组成开始。

## **一、 与该功能相关的类的结构**
主要的组成类的结构和关系如下， 类的属性和方法只是列出了关键的部分

![](/blog/assets/wifi/wifi-auto-join.png)

1、 WifiQualifiedNetworkSelector：顾名思义， 就是用来选择有“质量”的WIFI候选者进行连接。

2、 WifiConnectivityManager：android N对WIFI两大业务性功能进行打包管理，这两大业务性功能是“扫描”和“连接”。SingleScan？ PNOScan？ PeriodScan？怎么连接？自动连接？连哪个SSID？

3、 WifiStateMachine：对比之前的android版本，android N还是老任务不变，还是处于WIFI的核心位置，决定并说明WIFI处于什么状态，驱动加载完没有？连接状态？断开状态？处于打开热点的状态？


## **二、 原理介绍**

1、 自动连接都是WIFI返回扫描结果的时候触发的。

2、 WIFI的各种扫描管理由WifiConnectivityManager负责，在WifiConnectivityManager中有各种扫描的监听器，用来对各种扫描做出响应。

![](/blog/assets/wifi/wifi-auto-join-1.png)

3、 在每个监听器中都有一个onResults()函数，该函数将会在有扫描结果返回时候被调用，如果有需要，接着会调用handleScanResults函数处理扫描结果。这个函数就会把扫描结果交给WifiQualifiedNetworkSelector，由他来选出一个合适的候选者进行连接。然后交由WifiConnectivityManager进行连接。

![](/blog/assets/wifi/wifi-auto-join-2.png)

## **三、 如何选出有“质量”的网络**

这一部分的主要功能是在WifiQualifiedNetworkSelector的selectQualifiedNetwork函数中进行选择。

1、 如果扫描结果的SSID的是2.4GHz并且信号强度小于-85， 如果是5GHz并且信号强度小于-82，该候选者将会被抛弃。

2、 RSSI：RSSI越强加分越多，2.4GHz此项最多加100分，如果是5GHz将会加多50分.

3、 如果此次选择SSID与上次选择的SSID相同, 则需要判断是否为用户手动选择, 如果为用户手动选择, 则需要给该SSID加分, 跟据距离上一次时间的长短, 时间越长加分越少. 此项最多加分480分.

4、 如果候选者是目前连接的网络, 加分, 16分（漫游）.

5、 如果候选者的BSSID与目前连接的网络BSSID相同, 加分, 48分

6、 如果是passpoint 加分， 40分

7、 如果不是无需密码类型的网络，加分， 80分

8、 如果被记录“不能访问互联网”的次数大于0， 并且目前仍然不能访问互联网，扣分 (-60 + 85) * 4 + 40+ 16 + 24 + 80，260分
