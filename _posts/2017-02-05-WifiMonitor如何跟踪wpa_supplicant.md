---
date: 2017-02-05
layout: post
title: WifiMonitor如何跟踪wpa_supplicant
categories: Wi-Fi
tags: Wi-Fi
excerpt: WifiMonitor是负责跟踪wps_supplicant，那是如何跟踪？
---
<div>
(1)在WifiStateMachine中InitialState状态收到CMD_START_SUPPLICANT消息时候</div>

{% highlight java %}
if(mWifiNative.startSupplicant(mP2pSupported)) {
    setWifiState(WIFI_STATE_ENABLING);
    if (DBG) log("Supplicant start successful");
    mWifiMonitor.startMonitoring(mInterfaceName);
transitionTo(mSupplicantStartingState);
} else {
...
}
{% endhighlight %}

(2)startMonitoring这是一个同步函数
![](/blog/assets/wifi/WifiMonitor-startMonitoring.png)

(3)关注ensureConnectedLocked函数
![](/blog/assets/wifi/WifiMonitor-ensureConnectedLocked.png)

<div>(4)一个跟踪的线程类MonitorThread</div>

{% highlight java %}
    private class MonitorThread extends Thread {
        private final LocalLog mLocalLog = mWifiNative.getLocalLog();

        public MonitorThread() {
            super("WifiMonitor");
        }

        public void run() {
            if (DBG) {
                Log.d(TAG, "MonitorThread start with mConnected=" + mConnected);
            }
            //noinspection InfiniteLoopStatement
            for (;;) {
                if (!mConnected) {
                    if (DBG) Log.d(TAG, "MonitorThread exit because mConnected is false");
                    break;
                }
                String eventStr = mWifiNative.waitForEvent();

                // Skip logging the common but mostly uninteresting events
                if (!eventStr.contains(BSS_ADDED_STR) && !eventStr.contains(BSS_REMOVED_STR)) {
                    if (DBG) Log.d(TAG, "Event [" + eventStr + "]");
                    mLocalLog.log("Event [" + eventStr + "]");
                }

                if (dispatchEvent(eventStr)) {
                    if (DBG) Log.d(TAG, "Disconnecting from the supplicant, no more events");
                    break;
                }
            }
        }
}
{% endhighlight %}

<div>(5)然后事件被dispatchEvent函数分发。
dispatchEvent函数有两个，一个是一个参数，一个是两个参数的。</div>
{% highlight java %}
private boolean dispatchEvent(String eventStr, String iface)
private synchronized boolean dispatchEvent(String eventStr)
{% endhighlight %}
