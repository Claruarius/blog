---
date: 2017-09-03
layout: post
title: Android O 中WIFI代码初印象
categories: Android
tags: Wi-Fi
excerpt: Android O中对HAL层进行大刀阔斧的改变, 将HAL层抽象成类似于AIDL中的Service.
---


Android O中对HAL层进行大刀阔斧的改变, 将HAL层抽象成类似于AIDL中的Service.WIFI模块变化最大的应该就是WifiNative部分，从WifiNative看起。

## **一、取消Wifi.c和com_android_server_wifi_WifiNative.cpp**

对比之前的android 版本， WifiNative.java中的代码是通过com_android_server_wifi_WifiNative.cpp与wifi.c进行调用的。但是现在Google已经取消掉wifi.c这文件， 令人惊奇的是仍然保留了com_android_server_wifi_WifiNative.cpp， 只不过其中绝大部分的调用方法已经被取消掉，只留下一个读取kernel log的函数，这里提个问题，为什么要保留一个读取kernel log的方法呢？

![](/blog/assets/wifi/wifi-android-O-new-native.png)

## **二、 新增加的通讯方式**

WifiNative是用来与Hal层通信的对象，如果把相关的c文件和cpp文件废掉，那还如何发挥作用。在原来的Wifi.c文件中，有的函数是不通过supplicant直接跟驱动进行通信的函数，有的函数是通过操作supplicant来操作驱动层的代码。

在android O的WifiNative.java的代码中有以下三个对象：

![](/blog/assets/wifi/wifi-android-O-new-native-1.png)

**mWifiVendorHal, mSupplicantStaIfaceHal, mWificondControl**

## **三、 mWifiVendorHal、mSupplicantStaIfaceHal、mWificondControl**

**1、 mWifiVendorHal**

这个我认为是用于给厂商的驱动进行IPC通讯，这个也是HIDL实现的。将在第四部分中介绍。主要属性对象为mHalDeviceManager。mIWifiChip, mIWifiStaIface, mIWifiApIface都是从mHalDeviceManager得到的。

![](/blog/assets/wifi/wifi-android-O-new-native-2.png)

HalDeviceManager类的主要属性对象是mIWifi

![](/blog/assets/wifi/wifi-android-O-new-native-3.png)

A） HIDL代码在/hardware/interfaces/wifi

B） 实现在/hardware/interfaces/wifi/1.0/default

C） Out目录中编译HIDL文件之后生的java文件在./gen/JAVA_LIBRARIES/android.hardware.wifi-V1.0-java_intermediates/android/hardware/wifi/V1_0/IWifi.java

D） Out目录中编译HIDL文件之后生成的c++文件在./out/soong/.intermediates/hardware/interfaces/wifi/1.0/android.hardware.wifi@1.0_genc++_headers/gen/android/hardware/wifi/1.0/IWifi.h

**2、 mSupplicantStaIfaceHal**

（0）这个我认为是用于跟supplicant沟通用的，其中SupplicantStaIfaceHal使用了两个Service对象来进行IPC通讯。

![](/blog/assets/wifi/wifi-android-O-new-native-4.png)

ISupplicant对象和ISupplicantStaIface对象，从I开头可以看出，这两个对象都是远端Service对象。这一部分就是HIDL实现。

A） HIDL代码在/hardware/interfaces/wifi，

B） 实现在/external/wpa_supplicant_8/wpa_supplicant/hidl/1.0，

C） out目录中编译HIDL文件之后生成的java文件在out/target/common/gen/JAVA_LIBRARIES/android.hardware.wifi.supplicant-V1.0-java_intermediates/android/hardware/wifi/

D） Out目录中编译HIDL文件之后生成的c++文件在./out/soong/.intermediates/hardware/interfaces/wifi/supplicant/1.0/android.hardware.wifi.supplicant@1.0_genc++_headers/gen/android/hardware/wifi/supplicant/1.0/ISupplicant.h

（1） 首先来看它们如何被实例化

**A) mISupplicant**

![](/blog/assets/wifi/wifi-android-O-new-native-5.png)

**B) mISupplicantStaIface**

![](/blog/assets/wifi/wifi-android-O-new-native-6.png)
![](/blog/assets/wifi/wifi-android-O-new-native-7.png)

（2） 两个Service对象如何分工？

**A） mISupplicant**

个人理解，应该是作为supplicant本身在java层的代理。可以列举或得到各种interface，比如STA的。

**B）mISupplicantStaIface**
作为STA身份，用于操作supplicant相关的命令，比如addNetwork等。

**3、 mWificondControl**

1） 跟驱动直接沟通？

![](/blog/assets/wifi/wifi-android-O-new-native-8.png)

2） 如何得到IWificond的实例对象

![](/blog/assets/wifi/wifi-android-O-new-native-9.png)

3）IWificond这一部分代码的实现在/system/connectivity/wificond，这个是用AIDL实现的。

![](/blog/assets/wifi/wifi-android-O-new-native-10.png)

4） 由以上2）可以知道这个是从ServiceManager中得到的，所以之前也应该注册到ServiceManager中，这个注册的过程在/system/connectivity/wificond/main.cpp。是通过android::defaultServiceManager将Wificond注册到系统服务中的。

![](/blog/assets/wifi/wifi-android-O-new-native-11.png)

综上所述，Wificond的实现是使用AIDL实现的，但是不同于以往的AIDL，该Service端使用的是C++编写的，/system/connectivity/wificond/server.[h|c]

5） 函数调用

A）在WifiCondControl.java中调用Wificond的createClientInterface方法
![](/blog/assets/wifi/wifi-android-O-new-native-12.png)

B）./obj/JAVA_LIBRARIES/wifi-service_intermediates/dotdot/dotdot/dotdot/dotdot/dotdot/system/connectivity/wificond/aidl/android/net/wifi/IWificond.java
![](/blog/assets/wifi/wifi-android-O-new-native-13.png)

C）Server.cpp
![](/blog/assets/wifi/wifi-android-O-new-native-14.png)

但是可以看到在server.cpp中createClientInterface函数将要返回的ClientInterface对象变成指针传参数传进去。这个跟普通的AIDL还有区别。

就先写这么多吧,下一篇文章将会分析HIDL如何工作.