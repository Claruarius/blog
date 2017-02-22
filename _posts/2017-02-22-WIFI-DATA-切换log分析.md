---
date: 2017-02-22
layout: post
title: WIFI-DATA切换log分析
categories: Android
tags: Wi-Fi
excerpt: Android系统使用评分机制对各个网络进行评分，优先使用分数高的网络，以保证机器的网络使用质量
---

Android系统使用评分机制对各个网络进行评分，优先使用分数高的网络，以保证机器的网络使用质量，由于最近在大陆地区使用android设备出现问题，所以先上一篇文章如何分析log。

（1）基础分：WIFI-60， DATA-50

（2）对于没有验证网络有效性的通过的网络会给予最严厉的惩罚，减掉40
![](/blog/assets/wifi/wifi-data-log-unvalidate-score.png)

网络有效性验证就是为了证明机器在使用的网络是可以通互联网的，它的原理是：
通过HTTP（andorid N之后优先使用HTTPS）访问Google的网站，然后通过HTTP返回值来判断是否可以访问互联网。
但是由于在中国大陆，不能访问到Google的网址，所以在android原生系统，此项是**必减分数**。

<div>（3）由于两个网络WIFI和DATA都验证不通过，所以两个分数都在基础分（60和50）减掉40，</div>
```
	Line 549: 02-22 18:16:54.854  3043  3109 D ConnectivityService: currentScore = 20, newScore = 10（WIFI和DATA）
	Line 1856: 02-22 18:17:28.071  3043  3109 D ConnectivityService: NetworkAgentInfo [MOBILE (EDGE) - 101] validation failed
	Line 2307: 02-22 18:17:51.787  3043  3109 D ConnectivityService: NetworkAgentInfo [WIFI () - 100] validation failed
```
<div>断开WIFI，只剩下DATA</div>
```
	Line 3740: 02-22 18:20:16.596  3043  3109 D ConnectivityService: NetworkAgentInfo [MOBILE (EDGE) - 101] validation failed
	Line 4226: 02-22 18:20:21.388  3043  3109 D ConnectivityService: currentScore = 0, newScore = 10（WIFI断开和DATA）
```
<div>关掉DATA，打开DATA，再打开WIFI</div>
```
	Line 5587: 02-22 18:21:21.499  3043  3109 D ConnectivityService: NetworkAgentInfo [MOBILE (EDGE) - 102] validation failed
	Line 8440: 02-22 18:21:34.180  3043  3109 D ConnectivityService: currentScore = 10, newScore = 20（DATA和WIFI）
	Line 8837: 02-22 18:21:47.259  3043  3109 D ConnectivityService: NetworkAgentInfo [WIFI () - 103] validation failed
```
所以在Google网站访问通的情况下，且WIFI信号正常的情况下，WIFI的分数还是比DATA的分数高。（WIFI如果信号太弱会减分），也就是说不会出现使用DATA的数据通路，出现E的图标。


（4）什么情况下会出现E或LTE图标
<div>如果在ConnectivityService的log中出现如下的log，Statusbar通常就会出现E或LTE图标（也有可能会根据需求被修改成永久显示，那就不能作为判断依据），表示正在使用数据通路上网。</div>
```
ConnectivityService: registerNetworkAgent NetworkAgentInfo{ ni{[type: MOBILE[EDGE], state: CONNECTED/CONNECTED, reason: connected, 
```
<div>（5）WIFI的一些细致评分在WifiScoreReport.java（andorid N之后）中（此部分还没有细看，待补充）</div>
```
（1）score -= BAD_LINKSPEED_PENALTY;（4分）

（2）if (wifiInfo.linkStuckCount > MIN_SUSTAINED_LINK_STUCK_COUNT（1次）) {（卡住超过2次就减分）
            // Once link gets stuck for more than 3 seconds, start reducing the score
            score = score - LINK_STUCK_PENALTY （2分）* (wifiInfo.linkStuckCount - 1);
        }

（3）if (mEnableWifiCellularHandoverUserTriggeredAdjustment
                        && currentConfiguration.numUserTriggeredWifiDisableNotHighRSSI > 0) {//如果用户曾由于WIFI不太好的RSSI手动切换过到DATA，减分
                    score = score - USER_DISCONNECT_PENALTY;（5分）
                    
```