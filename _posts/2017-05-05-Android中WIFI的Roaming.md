---
date: 2017-05-05
layout: post
title: 源码解析Android中WIFI的Roaming
categories: Android
tags: Wi-Fi
excerpt: 从源码层面简单介绍Android中WIFI的Roaming现象。
---

## **Android Version： Android N**

* 1、 在WifiStateMachine中将状态转换为mRoamingState的地方有如下几处：

handleSupplicantStateChange(Message message)函数

在ConnectModeState状态收到CMD_AUTO_CONNECT消息

在ConnectedState状态收到CMD_AUTO_ROAM消息


* 2、 handleSupplicantStateChange(Message message)函数
该函数在DriverStartingState状态收到WifiMonitor.SUPPLICANT_STATE_CHANGE_EVENT消息。

在DriverStoppingState状态收到WifiMonitor.SUPPLICANT_STATE_CHANGE_EVENT消息

在ScanModeState状态收到WifiMonitor.SUPPLICANT_STATE_CHANGE_EVENT消息

在ConnectModeState状态收到WifiMonitor.SUPPLICANT_STATE_CHANGE_EVENT消息

* （1）从消息中获取supplicant现在的状态，这个消息也是从supplicant中上报上来的。

* （2）suplicant中上报上来的SSID为空，并且linkDebouncing已经为true就维持WifiStateMachine目前的状态。（return 返回）

* （3）如果supplicant处于Connecting状态（AUTHENTICATING，ASSOCIATING，ASSOCIATED，FOUR_WAY_HANDSHAKE，GROUP_HANDSHAKE，COMPLETED）就更新WifiStateMachine所持有的Wifi的WifiInfo的networkId

* （4）更新BSSID

* （5）如果上报上来的SSID和目前的SSID不一致，并且目前WifiStateMachine的状态是ConnectedState的状态，就将WifiStateMachine转换为RoamingState的状态。

* （6）更新WifiStateMachine所持有的WifiInfo的一些其他信息。

**所以从以上可以看出handleSupplicantStateChange函数当supplicant中的SSID和WifiStateMachine中的SSID不同时候进入RoamingState**


* 3、 在ConnectModeState状态收到CMD_AUTO_CONNECT消息
基本逻辑如下：
* （1） 在ConnectModeState状态收到CMD_AUTO_CONNECT消息附带需要自动连接的netId和BSSID

* （2） 使用wpa_supplicant的select命令选中该config，并且使用recnnect命令重新连接

* （3） 如果此时linkDebouncing为true，就将WifiStateMachine状态切换为RoamingState

* 4、 在ConnectedState状态收到CMD_AUTO_ROAM消息

* （1）  在ConnectedState状态中收到CMD_AUTO_ROAM消息，附带需要自动roam的netId和ScanResult的BSSID，决定使用哪个BSSID(netId取出的config还是ScanResult的BSSID)

* （2） 如果上次连接的netId跟这次的要连接的netId相同，使用wpa_supplicant的REASSOICATE命令进行重新关联，如果不同就使用selectNetwork命令进行连接

* （3） 将WifiStateMachine状态切换为RoamingState

* 5、 触发CMD_AUTO_ROAM消息被发送的位置有两处

* （1） 在setScanResults函数中
sendMessage(CMD_AUTO_ROAM, mLastNetworkId, 1, null);

* （2） 在WifiStateMachine的公开接口
public void autoRoamToNetwork(int networkId, ScanResult scanResult)
中

sendMessage(CMD_AUTO_ROAM, networkId, 0, scanResult);


<div>6、 针对5-（1）中函数，Android给出了如下描述：</div>
```
// If debouncing, we dont re-select a SSID or BSSID hence
// there is no need to call the network selection code
// in WifiAutoJoinController, instead,
// just try to reconnect to the same SSID by triggering a roam
// The third parameter 1 means roam not from network selection but debouncing
```
所以如果是linkDebouncing为true。那么，第三个参数将会被标注为1，并且连接的还是原来的SSID。

* 7、 那么linkDebouncing什么时候为true？

<div> （1）在ConnectedState状态中收到WifiMonitor.NETWORK_DISCONNECTION_EVENT消息，如果符合以下条件则会进行debouncing=true</div>
```
if (mScreenOn 如果亮屏
&& !linkDebouncing 并且不是处于debouncing
&& config != null 并且要连接的config不为null
&& config.getNetworkSelectionStatus().isNetworkEnabled() 并且该config是处于使能状态
&& !mWifiConfigManager.isLastSelectedConfiguration(config) 并且该config不是上次select的config
&& ((message.arg2 != 3 /* reason cannot be 3, i.e. locally generated */) 并且需要认证失败的原因不是reason 3
|| (lastRoam > 0 && lastRoam < 2000) /* unless driver is roaming */) 或者上一次roam的时间不超过2000毫秒
&& ((ScanResult.is24GHz(mWifiInfo.getFrequency())并且所持有的WifiInfo是2.4GHz的，信号强度大于-73dbm
&& mWifiInfo.getRssi() >
WifiQualifiedNetworkSelector.QUALIFIED_RSSI_24G_BAND)
|| (ScanResult.is5GHz(mWifiInfo.getFrequency())或者所持有的WifiInfo是5GHz的，信号强度大于-70dbm
&& mWifiInfo.getRssi() >
mWifiConfigManager.mThresholdQualifiedRssi5.get()))
```

* （2）在如下的情况中将linkDeboucing置为false
在L2ConnectedState中收到CMD_DELAYED_NETWORK_DISCONNECT将linkDebouncing置为false。

在ObtainingIpState和ConnectedState的enter函数中将linkDebouncing置为false

在handleNetworkDisconnect函数中将linkDebouncing置为false。

* 8、 针对5-（2）中autoRoamToNetwork接口

其被调用的地方是WifiConnectivityManager的handleScanResults()函数中，每次有扫描结果回来时候就会调用handleScanResults()函数，在handleScanResults函数中会调用connectToNetwork()函数，如果将要连接的候选网络的networkId和目前的networkId一致或者是候网络的WifiConfig与目前正在连接的WifiConfig是同一个则会调用到WifiStateMachine的autoRoamToNetwork接口。否则调用WifiStateMachine的autoConnectToNetwork接口。

## **总结成流程图**
![](/blog/assets/wifi/wifi-roaming.jpg)

