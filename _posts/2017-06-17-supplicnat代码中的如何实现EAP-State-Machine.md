---
date: 2017-06-17
layout: post
title: 关于supplicant代码中的EAP State Machine的实现
categories: Android
tags: Wi-Fi
excerpt: 最近刚好有一个关于EAP的需求，所以就简单梳理了supplicant的源码，看看是否支持，并以此文做简要的笔记。
---

最近刚好有一个关于EAP的需求，所以就简单梳理了supplicant的源码，看看是否支持，并以此文做简要的笔记。

### **一、 术语解释**

|名词|解释|
|-|-|
|EAP Authenticator|EAP 认证服务端|
|EAP Peer|EAP 认证客户端|
|EAP|可扩展的身份验证协议（Extensible Authentication Protocol）的简称|
|EAP Method|EAP 方法，EAP 认证细节的实现|
|EAP-SIM|一种EAP 方法。使用GSM在EAP框架下进行鉴权，在RFC4186 中有做规范|
|EAP State Machine|EAP状态机，RFC4137 规定的EAP状态机规范，涉及状态间如何转换|

### **二、 如何看懂wap_supplicant源码中的EAP State Machine**
**1、 流程图如下**
![](/blog/assets/wifi/eap-state-machine-1.png) 

**2、流程介绍**

（1） 每一次的eap exchange都是 从eapol_sm_step(struct eapol_sm *sm)进入的。

（2） 然后在eap_peer_sm_step(struct eap_sm *sm)函数中有一个循环，该循环是用来处理EAP State Machine状态转换用的。

（3） 宏SM_STEP_RUN(EAP)与宏SM_STEP(EAP)翻译之后函数名称相同

（4） eap_peer_sm_step_local函数用于根据当前EAP State Machine的状态和当前状态处理数据的结果进行状态切换。

（5） 状态切换用到的宏是SM_ENTER

（6） 宏SM_ENTER(machine, state)与宏SM_STATE(machine, state)展开后相同

（7） 每进入一个状态将会调用SM_ENTRY(machine, state)判断当前的状态是否发生变化，如果发生变化表示还有数据需要继续处理。

（8） 如果状态没有发生变化，代表所有数据处理完成，退出 eapol_sm_step函数。

### **三、 一次成功的EAP-SIM连接**

EAP-SIM是EAP Method中的一种，使用SIM卡的数据进行鉴权，快速建立连接。

1、一次成功的EAP-SIM连接的状态切换的log
![](/blog/assets/wifi/eap-state-machine-2.png) 
从log中可以看到主要经过的几个阶段为IDENTITY, GET_METHOD, METHOD, SUCCESS。 

2、 整个EAP鉴权阶段所处的阶段是在WIFI连接的ASSOCIATED到4WAY_HANDSHAKE之间。

### **四、 EAP中的主要阶段**

这里可以参考RFC4137，该规范文档是用于指导如何代码书写EAP State Machine。对于EAP Peer端的状态转换，有详细，明确的说明。【A.1 EAP Peer State Machine (Figure 3)】

**1、 IDENTITY**

（1）此阶段主要是获取peer段的身份认证ID。这里身份ID在RFC4186中			规定有三种。

re-authentication id（快速重连ID）

pseudonyms id（混淆的ID）

permanent id（永久性的ID）

前两种ID是由EAP 给到客户端（EAP-Peer）的，前两中ID的生成基于最后一个ID。也就是说，当服务器和客户端双方都不认识谁的时候，先进行full authentication（包含permanent id）进行认证，然后在challenge阶段生成pseudonyms id和re-authentication id给到EAP-Peer。

（2） 这里我们考虑permanent id怎样获取。

permanent id是包含MccMnc组织成的。一个典型的例子就是`123415112200003@wlan.mnc015.mcc234.3gppnetwork.org`. Android M之后WifiConfiguration中一般不包含该ID，所以supplicant会向WifiStateMachine发消息索取该ID（MTK 平台）。
![](/blog/assets/wifi/eap-state-machine-3.png) 
在WifiStateMachine的ConnectionMode状态中接收该消息并进行处理
![](/blog/assets/wifi/eap-state-machine-4.png) 
在framework中取得permanent id之后就通过WifiNative下发给supplicant. 至此，获取permanent id过程结束。

**2、 GET_METHOD**

在进行IDENTITY阶段之后会进行GET_METHOD，在EAP框架中，没有规定具体地认证方法，只规定了一问一答的模式。所以，如果要进行认证，就需要有一种具体的eap-method，比如EAP-SIM， EAP-AKA， EAP-TLS等，这里GET_METHOD就是获取想要使用的eap-method。
![](/blog/assets/wifi/eap-state-machine-5.png) 
![](/blog/assets/wifi/eap-state-machine-6.png) 

sm变量是存储EAP State Machine的数据的结构，其中eap-method就是其中一项。其中sm->m是一个函数指针，如果我们在WifiConfiguration中配置EAP-SIM方法，这里sm->m被赋予的值就是eap_sim.c中定义的eap_sim_process()。接下来就可以进行下一步了使用该函数进行具体的EAP method进行认证。

**3、 METHOD**

METHOD阶段就是使用具体的EAP METHOD进行认证的阶段。该阶段因每一种METHOD的不同而需要收集的数据，回复的数据也不同。比如EAP-SIM的log如下：
![](/blog/assets/wifi/eap-state-machine-7.png) 
就需要收集三次数据并与EAP Server进行通信。


### **五、 一个需求的分析**
最近遇到一个运营商提出的需求，最后发现不支持，在这里将分析的过程分享一下。主要代码都在supplicant源码中，属于开源部分。需求如下:

**The terminal shall not delete assigned pseudonyms and/or fast re-authentication tokens in case the EAP exchange fails, with a notification from the network with the value of 1026 (0x0402 User has been temporarily denied access to the requested service) or 1031 (0x0407 User has not subscribed to the requested service) and MUST use the assigned pseudonyms or fast re-authentication token in subsequent retries to connect to the network.**

该需求大概讲的是如果EAP exchange中收到服务器发的消息码1026和1031，将不能删除pseudonyms id或者是fast re-authentication id，以便在下一次的网络连接中使用这些id进行连接，而不是使用permanent id。

对于该需求的评估结果：
经过check supplicant源码，目前不支持该需求。

0、 pseudonyms id或者是fast re-authentication id 都是EAP Server在进行chanllenge阶段（也就是METHOD阶段）给到EAP Peer的。

1、 首先在eap.c中SM_STATE(EAP, INITIALIZE)中会根据上一次是否失败对数据进行清除。

如果失败，将会在函数eap_deinit_prev_method()中清除pseudonyms id和fast re-authentication id.
![](/blog/assets/wifi/eap-state-machine-8.png) 

2、 在sm->prev_failure被赋值是在进入SM_STATE(EAP, FAILURE)
![](/blog/assets/wifi/eap-state-machine-9.png) 

3、 在eap_sim_common.h中定义这几个消息码
![](/blog/assets/wifi/eap-state-machine-10.png) 

4、 在eap_sim_process_notification函数中，data->state被赋值为FAILURE
![](/blog/assets/wifi/eap-state-machine-11.png) 
![](/blog/assets/wifi/eap-state-machine-12.png) 

5、 在eap_sim_proces函数中，根据data->state判断成功与否

并将一个至关重要的返回值ret赋值，decision为FAIL， methodState为DONE，代表METHOD阶段结束。
![](/blog/assets/wifi/eap-state-machine-13.png) 
![](/blog/assets/wifi/eap-state-machine-14.png) 

6、 在eap.c中SM_STATE(EAP, METHOD)函数, 这里的process函数就是eap_sim_proces()
![](/blog/assets/wifi/eap-state-machine-15.png) 
执行process()函数之后，会将ret中的值赋值给sm

7、 在eap.c中由于循环将进入eap_peer_sm_step_local()函数，可见State Machine将进入FAILURE状态。
![](/blog/assets/wifi/eap-state-machine-16.png) 

8、 最终将导致前文所述的2，1的发生，即re-auth id 和 pseudonyms id将被清除。

### **六、 参考文档**

|**1、 RFC4186**|Extensible Authentication Protocol Method for Global System for Mobile Communications (GSM) Subscriber Identity Modules (EAP-SIM)|
|**2、 RFC4137**|State Machines for Extensible Authentication Protocol (EAP)Peer and Authenticator|