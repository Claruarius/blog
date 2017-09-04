---
date: 2017-09-03
layout: post
title: Android O HIDL初印象
categories: Android
tags: Wi-Fi
excerpt: Android O不仅将AIDL进行扩展（Service端可以使用C++写并且注册到系统Service中），还在frameweork层与HAL层的沟通上运用了之前在AIDL的那一套IPC机制——Binder通信。
---

Android O不仅将AIDL进行扩展（Service端可以使用C++写并且注册到系统Service中），还在frameweork层与HAL层的沟通上运用了之前在AIDL的那一套IPC机制——Binder通信。

## **一、如果不熟悉Binder通信，可以理解为：**
（1） 现在存在一个服务实体，

```
Class Service1 { //服务， 一个服务实体
Public :
Void interface1() { do-something;  }
Int inteface2(classParam para) { do-something; }
}
```

（2） Binder调用我本人理解是一种remote调用，也就是说Service负责计算， 只要通过某种方式告诉远端Service要调用**哪个函数**来进行计算，**传入的参数**是什么？然后让Service将**返回值**以同样的方式返回给近端就可以。当然这种方式可以有很多种，使用socket等跨进程通信的方式都是可以的。Google为了方便开发者使用这种通信方式，将它进行封装，对开发者透明，也就是说普通开发者可以不用知道这种通信机制的实现原理，只要**按照特定的步骤和格式书写代码**就可以方便地使用这种功能。

（3） AIDL 和 HIDL都是为了实现Binder通信而需要的声明文件。因为近端至少需要知道该Service有什么函数可以调用。
```
Interface IService1 {
Void interface1()
Int inteface2(classParam para)
}
```

这里可以讨论一下这个“IService1”, 其实加一个I为了方便阅读代码的人知道这是一个远端的服务，近端的代理。

（4） Service注册到系统服务
远端的Service需要注册到系统服务才能被其他模块（所谓的近端）得到。一般形式如下：
```
Service1 service = new Service1();
ServiceManager.registService(service , “example-service”);
//将特定的service实体和名称进行注册。
```

（5） 近端根据特定的名称get到service的近端代理并进行通信
```
IService service = ServiceManager.getService(“example-service”);
```

## **二、 AIDL/HIDL 特定的参与元素**

从以上的说明中大概可以知道AIDL和HIDL特定的参与元素如下：

（1） 一个Service实体（实现的地方）

（2） Binder通信机制（Google 已经实现好）

（3） AIDL/HIDL声明文件

（4） 将Service注册到系统服务中

（5） 近端根据特定的服务名称get到Service的近端代理

## **三、 说明**

Android O扩展了AIDL的范围，远端Service的实现可以使用C++编写，例子WificondControl中的IWificond。

而HIDL作为替代framework与hal之间的native的调用，远端Service的实现自然是要使用C++编写，例子HalDeviceManager中的IWifi。

## **四、 例子说明如何看懂HIDL代码**

以SupplicantStaIfaceHal中的ISupplicant为例。
![](/blog/assets/wifi/wifi-android-O-hidl.png)

（1） 在SupplicantStaIfaceHal.java中listInterfaces函数调用如何从近端调用到远端，如何从java调用到C++。
![](/blog/assets/wifi/wifi-android-O-hidl-1.png)

备注：
![](/blog/assets/wifi/wifi-android-O-hidl-2.png)

图中标蓝色的部分是一个匿名内部类的实现的的写法。Java一个比较奇葩和节省笔墨的写法。也就是说不需要使用new关键字，也不需要赋值给一个变量之后再使用这个变量。只需要直接实现该类的**抽象未实现的方法**。实现该方法的写法是

**（参数）-> {方法实体}**

所以就不用管类名，需要知道的关键信息是该函数有哪些参数（可以从.hidl文件中找到）， 先卖个关子，为什么要有这个callback函数（其实这些参数是远端计算结果的返回值）：
![](/blog/assets/wifi/wifi-android-O-hidl-3.png)

（2） HAL文件

ISupplicant的hal文件放在/hardware/interfaces/wifi/supplicant/1.0
基本上hal文件放置的两个地方是 /hardware/interfaces/模块名
和 /vendor/厂商名/hardware/interface

（3） Android会自动为该hal文件生成一份C++端的代码，和一份java端的代码
由于是远端调用，所以需要规范和声明接口，hal文件就起到这个作用。Android会自动生成一份C++端规范，和java端的规范。C++端的Service实体需要继承该C++端的接口规范， 并实现该规范中没有实现的接口函数。

A） out目录中编译HIDL文件之后生成的java文件在out/target/common/gen/JAVA_LIBRARIES/android.hardware.wifi.supplicant-V1.0-java_intermediates/android/hardware/wifi/

![](/blog/assets/wifi/wifi-android-O-hidl-4.png)

B） Out目录中编译HIDL文件之后生成的c++文件在./out/soong/.intermediates/hardware/interfaces/wifi/supplicant/1.0/android.hardware.wifi.supplicant@1.0_genc++_headers/gen/android/hardware/wifi/supplicant/1.0/ISupplicant.h
![](/blog/assets/wifi/wifi-android-O-hidl-5.png)

ISupplicant类

![](/blog/assets/wifi/wifi-android-O-hidl-6.png)

（4） C++端需要继承实现ISupplicant类

ISupplicant服务的实现是在/external/wpa_supplicant_8/wpa_supplicant/hidl/1.0

![](/blog/assets/wifi/wifi-android-O-hidl-7.png)

继承实现ISupplicant类
![](/blog/assets/wifi/wifi-android-O-hidl-8.png)

由于java是作为近端，所以java端的代码直接由Android生成就够用了。

（5） 在Suplicant类中找到listInterfaces函数
![](/blog/assets/wifi/wifi-android-O-hidl-9.png)

调用listInterfacesInternal函数，同时**在（1）中说的callback函数就是图中的_hidl_cb**
![](/blog/assets/wifi/wifi-android-O-hidl-10.png)

注意观察listInterfacesInternal函数的返回值，跟我们在java端写的callback函数的参数是一样的。

所以由此可以猜测HIDL调用的**返回值**以**callback函数的参数**的形式返回给调用者。

（6） 最后我们再来看一下近端java代码中如何得到该Service的近端代理
![](/blog/assets/wifi/wifi-android-O-hidl-11.png)
通过getSupplicantMockable函数取得该对象
![](/blog/assets/wifi/wifi-android-O-hidl-12.png)

但是你会发现在ISupplicant.hal文件中没有getService函数。函数不会凭空调用，所以应该是Android自动生成了。可以查看java端android自动生成的代码：
out/target/common/gen/JAVA_LIBRARIES/android.hardware.wifi.supplicant-V1.0-java_intermediates/android/hardware/wifi/
![](/blog/assets/wifi/wifi-android-O-hidl-13.png)

可见Android已经替你写好了。
