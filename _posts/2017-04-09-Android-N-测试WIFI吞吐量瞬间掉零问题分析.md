---
date: 2017-04-09
layout: post
title: Android N 测试WIFI吞吐量瞬间掉零问题分析
categories: Android
tags: WIFI
excerpt: 最近有一个问题是Android 7.1软件版本上测试WIFI吞吐量时候经常出现掉0情况, 后来跟高通一起排查, 是因为Google在Android 7.1 代码修改导致的. 
---

最近有一个问题是Android 7.1软件版本上测试WIFI吞吐量时候经常出现掉0情况, 后来跟高通一起排查, 是因为Google在Android 7.1 代码修改导致的.

## **问题描述**

在使用iperf软件进行WIFI吞吐量测试时候会出现掉零情况.如下图
![](/blog/assets/wifi/wifi-tput-1.png)

## **分析问题需要的log**

分析该问题需要的log: 机器的logcat log, iperf软件打印的log, sniffer log.

另外高通还提供了在驱动层面禁止WIFI scan的命令帮助排查问题, 只能在eng版本上执行.

## **分析**

如上图可以发现在48秒左右出现掉零情况. 此时看sniffer log,
![](/blog/assets/wifi/wifi-tput-2.png)

可以发现在47秒多有probe rsp帧,点击进去查看, 时间是19:26
![](/blog/assets/wifi/wifi-tput-3.png)

根据这个时间去查看logcat log,发现wpa_supplicant确实有切换到扫描状态
![](/blog/assets/wifi/wifi-tput-4.png)

查看Onimpeek的包类型统计, 发现有probe req和probe rsp的包, 这就意味着出现了WIFI scan行为.
![](/blog/assets/wifi/wifi-tput-5.png)

最后排查的原因是,在Android 7.1中, Google新增加一个叫WifiConnectivitymanager的类来管理WIFI的扫描和连接. 其中就对于WIFI在进行大吞吐量数据操作时候没有屏蔽掉WIFI扫描.
![](/blog/assets/wifi/wifi-tput-6.png)


## **解决问题**

如果判断在进行大吞吐量时候, 屏蔽WIFI亮屏常规扫描

```
if (mWifiState == WIFI_STATE_CONNECTED
   && (mWifiInfo.txSuccessRate > mConfigManager.MAX_TX_PACKET_FOR_PARTIAL_SCANS
   || mWifiInfo.rxSuccessRate > mConfigManager.MAX_RX_PACKET_FOR_PARTIAL_SCANS)) {
      localLog("Ignore scan due to heavy traffic, txSuccessRate=" + 
      mWifiInfo.txSuccessRate + " rxSuccessRate=" + mWifiInfo.rxSuccessRate);
} else {
       startSingleScan(isFullBandScan);
}
```

完