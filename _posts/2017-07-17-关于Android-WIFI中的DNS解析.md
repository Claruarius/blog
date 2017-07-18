---
date: 2017-07-17
layout: post
title: 关于Android-WIFI中的DNS解析
categories: Android
tags: Wi-Fi
excerpt: 最近有一个同事不走寻常路, 在adb shell中启动WIFI驱动, 接着启动wap_supplicant进行扫描, 连接, 接下来可以在shell中ping通IP地址, 但是ping不通域名, 比如baidu.com。
---

最近有一个同事不走寻常路, 在adb shell中启动WIFI驱动, 接着启动wap_supplicant进行扫描, 连接, 接下来可以在shell中ping通IP地址, 但是ping不通域名, 比如baidu.com, 由于本人对此感兴趣就研究了一番, 在此做记录.

## **准备**

其实这些都在文章[wpa_cli与wpa_supplicant的交互命令](http://www.cnblogs.com/lidabo/p/4660213.html) 可以找到.

(1) 准备一个eng版本软件的机器

(2) adb shell进入shell, 
```
echo 1 > /dev/wmtWifi
```

(3)启动supplicant
```
wpa_supplicant -iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf
```

(4)打开另一个终端, adb shell, 输入
```
wpa_cli -iwlan0
```

(5)进行扫描,并获取扫描结果,如下两个命令
```
scan
scan_result
```
(6)连接一个扫描结果中的一个AP(假如SSID为Android-WIFI, 密码为12345678)
```
add_network//这个会返回一个数字, 表示可以用这个数字作为id创建network
set_network 数字 ssid "Android-WIFI"
set_network 数字 psk "12345678"
enable_network 数字
select_network 数字

status //查看当前的状态
```
(7)此时L2层已经连接上AP, 但是**L3层仍然未能连接上**(L2层是数据链路层, L3层是网络层). L3层顺利连接上的标志是获取到IP地址. 运行`ifconfig`命令可以看到wlan0未能获取到IPv4的地址. 
![](/blog/assets/wifi/wifi-dns-4.png)
所以运行如下的命令获取IP地址.
```
dhcptool wlan0
```
再次运行`ifconfig`命令可以看到已经获取IPv4地址.
![](/blog/assets/wifi/wifi-dns-5.png)

(8)使用`ping`命令可以ping通IP地址, 但是ping不通baidu.com等此类域名
![](/blog/assets/wifi/wifi-dns-6.png)

## **问题-为什么DNS解析会失效**

本人怀疑过以下的几个点

(1)ARP邻接路由表失效,网关丢失

**验证: 使用route命令查看之后没有丢失, 使用arping命令可以正常ping通局域网内的IP**

(2)iptables路由规则限制

**验证: 对比正常的机器的iptables路由规则和问题机器的路由规则, 没有实质性的区别**

(3)DNS地址配置错误

**验证: 使用getprop查看dns地址配置, 配置的ip地址可以正常ping通**

DNS系统是属于应用层, DNS报文使用UDP协议. 最后抓netlog, 用wireshark查看
![](/blog/assets/wifi/wifi-dns-1.png) 
可以看到DNS报文的源地址和目的地址都是127.0.0.1, 这不是回环地址吗?

### **确信是DNS解析出了问题**
经上网查找资料和对比源码发现其中的猫腻.

网上资料[Android4.3前后DNS解析简单研究](http://blog.csdn.net/insswer/article/details/17382535)

DNS解析的源码是在bionic中, 在android 4.2之前的源码的`android_getaddrinfo_proxy`函数中有如下的代码:
{% highlight  c %}
snprintf(propname, sizeof(propname), "net.dns1.%d", getpid()); 
if (__system_property_get(propname, propvalue) > 0) 
        return -1;  
    
// Bogus things we can't serialize.  Don't use the proxy
if ((hostname != NULL 
    strcspn(hostname, " \n\r\t^'\"") != strlen(hostname)) 
   (servname != NULL 
    strcspn(servname, " \n\r\t^'\"") != strlen(servname))) 
    return -1
} 
{% endhighlight %}

但由于我的现在的Android平台是 7.1, 所以经查证在`android_getaddrinfo_proxy`函数中没有通过系统属性获取DNS解析地址的源码, 在android 4.3开始DNS解析全部**由netd进程代理**.
![](/blog/assets/wifi/wifi-dns-2.png)

所以应该告诉netd我的DNS解析地址是多少.

网上资料[android 4.3以上修改DNS 及 流程（netd）](http://blog.csdn.net/ganyue803/article/details/51646284)主要介绍Android DNS的流程. 通过该文,可以提炼出以下两个步骤:

(1)设置iptables
```
iptables -t nat -A OUTPUT -p udp --dport 53 -j DNAT --to-destination DNS地址

取消设置的DNS：
iptables -t nat -L OUTPUT -n -v --line-numbers
iptables -t nat -D OUTPUT *linenumber*
```

(2)通过ndc命令可以配置DNS服务器：
```
ndc resolver setifdns wlan0 "" 8.8.8.8 8.8.4.4
```

ndc是用于与netd进行沟通的工具, 但是敲入上文的命令之后显示不识别.
但由于本人目前的Android 版本是7.1, 所以/system/netd源码有所不同, ndc所使用的命令是在CommandListener.cpp中处理的, grep dns关键字之后,发现如下
![](/blog/assets/wifi/wifi-dns-3.png)

遂修改命令如下:
```
ndc resolver setnetdns wlan0 "" DNS解析地址1 DNS解析地址2
```

成功. 随后也可以ping 域名的地址, 就表示DNS解析成功.

感谢我那位乱来的同事.
