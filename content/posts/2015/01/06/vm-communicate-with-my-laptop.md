---
layout: post
title: "从宿主机访问VirtualBox里的虚机"
date: 2015-01-06 14:30:00
author: 任怀林
categories: 
- blog
- other
thumb: linux.png
isCJKLanguage: true
---

我在VirtualBox里安装了OpenStack all in one，IP静态配置为192.168.234.160,公司银丰办公区使用192网段的IP，公司1+1主办公区的IP段是172的，当我到1+1主办公区时办公时，我就无法访问我的虚机了。我的虚机桥椄到宿主机的wlan0的网卡上。所以解决方案很简单，当我在1+1办公时，在我的wlan0上绑一个192段的IP就行了。

```
# ifconfig wlan0:0 192.168.234.249/24
```

回到银丰后，把这个配置删除掉：     

```
# ifconfig wlan0:0 192.168.234.249 down
```
