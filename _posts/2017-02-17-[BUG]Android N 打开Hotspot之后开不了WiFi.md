---
date: 2017-02-17
layout: post
title: Android N 打开Hotspot之后开不了WiFi
categories: Android
tags: Wi-Fi
excerpt: Android 在WiFi模块的一大创造就是状态机的运用，WifiController和WifiStateMachine就是两大核心状态机，但是有时候也会出现两个状态机状态不一致的情况。
---
## **BUG描述**
（1） 关闭WifiSettings中的WIFI开关，打开WIFI hotspot，回到WifiSettings界面手动打开WIFI发现开了之后就**自动被关了！**

（2） 打开WifiSettings中的WIFI开关，打开USB Tethering,打开WIFI hotspot，回到WifiSettings界面手动打开WIFI发现开之后就**自动被关了！**

**- - - -还有一种跟（1）很相似的操作但是是正常的。**

（3） 打开WifiSettings界面的WIFI开关，打开WIFI hotspot，回到WIFISettings界面打开WIFI，可以**手动正常打开WIFI开关,显示WIFI列表**

<div>对比以上三种情况的log发现惊人的相似但是为什么会有这么大的差异，令吾辈很费解。</div>

```
（1）WIFI-OFF && WIFI-Tether
	Line 5963: 01-01 09:24:33.336  3025  3076 D WifiController:  StaDisabledWithScanState !CMD_WIFI_TOGGLED
	Line 5964: 01-01 09:24:33.336  3025  3076 D WifiController: StaEnabledState enter
	Line 5965: 01-01 09:24:33.337  3025  3076 D WifiController: DeviceActiveState enter
	Line 6662: 01-01 09:24:35.084  3025  3076 D WifiController:  DeviceActiveState !CMD_WIFI_TOGGLED
	Line 6663: 01-01 09:24:35.084  3025  3076 D WifiController:  StaEnabledState !CMD_WIFI_TOGGLED
	Line 6664: 01-01 09:24:35.085  3025  3076 D WifiController: StaDisabledWithScanState enter
	Line 6940: 01-01 09:25:18.286  3025  3076 D WifiController:  StaDisabledWithScanState !CMD_SET_AP
	Line 6941: 01-01 09:25:18.286  3025  3076 D WifiController: ApStaDisabledState enter
	Line 6942: 01-01 09:25:18.287  3025  3076 D WifiController:  ApStaDisabledState !CMD_SET_AP
	Line 6943: 01-01 09:25:18.287  3025  3076 D WifiController: ApEnabledState enter
	Line 8200: 01-01 09:25:30.871  3025  3076 D WifiController:  ApEnabledState !CMD_SET_AP
	Line 8204: 01-01 09:25:30.875  3025  3076 D WifiController:  ApEnabledState !CMD_WIFI_TOGGLED
	Line 8283: 01-01 09:25:33.720  3025  3076 D WifiController:  ApEnabledState !CMD_AP_STOPPED
	Line 8284: 01-01 09:25:33.720  3025  3076 D WifiController: StaEnabledState enter----------------打不开WIFI列表
```
```
（2）WIFI-ON && USB-Tether && WIFI-Tether
	Line 16961: 01-01 08:14:54.084  2964  3020 D WifiController:  DeviceActiveState !CMD_WIFI_TOGGLED
	Line 16962: 01-01 08:14:54.084  2964  3020 D WifiController:  StaEnabledState !CMD_WIFI_TOGGLED
	Line 16963: 01-01 08:14:54.085  2964  3020 D WifiController: StaDisabledWithScanState enter
	Line 18102: 01-01 08:14:59.080  2964  3020 D WifiController:  StaDisabledWithScanState !CMD_SET_AP
	Line 18103: 01-01 08:14:59.080  2964  3020 D WifiController: ApStaDisabledState enter
	Line 18104: 01-01 08:14:59.081  2964  3020 D WifiController:  ApStaDisabledState !CMD_SET_AP
	Line 18105: 01-01 08:14:59.081  2964  3020 D WifiController: ApEnabledState enter
	Line 19316: 01-01 08:15:09.884  2964  3020 D WifiController:  ApEnabledState !CMD_SET_AP
	Line 19320: 01-01 08:15:09.901  2964  3020 D WifiController:  ApEnabledState !CMD_WIFI_TOGGLED
	Line 19401: 01-01 08:15:12.282  2964  3020 D WifiController:  ApEnabledState !CMD_AP_STOPPED
	Line 19402: 01-01 08:15:12.282  2964  3020 D WifiController: StaEnabledState enter-----------------打不开WIFI列表
```
```
（3）WIFI-ON && WIFI-Tether
	Line 9295: 01-01 08:02:53.304  3095  3146 D WifiController:  DeviceActiveState !CMD_SET_AP
	Line 9296: 01-01 08:02:53.305  3095  3146 D WifiController:  StaEnabledState !CMD_SET_AP
	Line 9297: 01-01 08:02:53.307  3095  3146 D WifiController: ApStaDisabledState enter
	Line 9302: 01-01 08:02:53.316  3095  3146 D WifiController:  ApStaDisabledState !CMD_SET_AP
	Line 9303: 01-01 08:02:53.317  3095  3146 D WifiController: ApEnabledState enter
	Line 10934: 01-01 08:03:06.689  3095  3146 D WifiController:  ApEnabledState !CMD_SET_AP
	Line 10939: 01-01 08:03:06.692  3095  3146 D WifiController:  ApEnabledState !CMD_WIFI_TOGGLED
	Line 11030: 01-01 08:03:08.912  3095  3146 D WifiController:  ApEnabledState !CMD_AP_STOPPED
	Line 11031: 01-01 08:03:08.912  3095  3146 D WifiController: StaEnabledState enter---------------可以打开WIFI列表
```

对比以上三种情况的log，WifiController最后的状态都是StaEnabledState，但是为什么就会有不同的表现！

**那就只能是中间状态转换时候出现了问题，因为状态转换时候会进行一些enter函数和exit函数的调用，这些函数是用于设置状态机状态的环境。**

## **WifiController中间状态切换**
将上面三份log的状态转换进行梳理如下面图片所示（最左边一列的函数调用情况是正常现象的，中间和最右边都是有问题现象的函数调用）

![](/blog/assets/wifi/wificontroller-bug.jpg)

对比以上的状态切换，就能够很明显的看出，出现bug的两个情况都有
StaDisabledWithScanState::enter()函数的调用。

![](/blog/assets/wifi/wificontroller-stadisabledwithscanstate-enter.png)

其他都不会对WIFI关闭造成什么影响，唯独以下的一个函数调用：
**mWifiStateMachine.setOperationalMode(WifiStateMachine.SCAN_ONLY_WITH_WIFI_OFF_MODE);**


## **WifiController与WifiStateMachine的状态不一致**
该函数将会影响WifiStateMachine的运行。好，现在进入WifiStateMachine
![](/blog/assets/wifi/wificontroller-wifistatemachine-set-operational-mode.png)

该消息将会在WifiStateMachine的DisconnectedState状态中被处理，可以看到该消息会将WifiStateMachine状态置为**ScanModeSate**，这是一种什么状态，这就是一种**关闭WIFI才有的状态**啊，至此可见**WifiController与WifiStateMachine的状态不一致！**WifiController的状态是StaEnabledState，而WifiStateMachine的状态是ScanModeSate,就是说WifiStateMachine已经处于WIFI关闭状态，但是WifiController还处于WIFI打开状态。

如下的log也证明了我们的想法。
情景1----------------打不开WIFI列表
![](/blog/assets/wifi/wificontroller-wifistatemachine-log1.png)

情景2----------------打不开WIFI列表
![](/blog/assets/wifi/wificontroller-wifistatemachine-log2.png)

情景3---------------可以正常打开WIFI列表
![](/blog/assets/wifi/wificontroller-wifistatemachine-log3.png)

至此已经有必要修改该状态机的状态。

## **修改思路**
在WifiStateMachine为**DisconnectedState状态**时，WifiController的亮屏状态应该是**DeviceActiveState**。

在WifiController状态机的ApEnabledState状态处理CMD_WIFI_TOGGLED 消息时候
![](/blog/assets/wifi/wificontroller-bug-fix.png)

将等待的状态修改为mDeviceActiveState。

这个时候又有另一个问题，为什么情况（3）没有接收到CMD_WIFI_TOGGLED,而其他两种情况都会处理到这种消息呢？还有待研究。


