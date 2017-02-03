---
layout: post
title: Netgear R6100安装OpenWRT 15.05
date: 2016-11-14 12:30:00
author: 任怀林
categories:
- blog
- linux
thumb: linux.png
---

# 1 下载固件
OpenWRT固件有两种：一种是factory固件（用于从原厂系统升级），一种是sysupgrade固件（用于从OpenWRT系统升级）。

先下载 [OpenWRT 15.05 for Netgear R6100 的固件](http://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/nand/openwrt-15.05-ar71xx-nand-r6100-ubi-factory.img)


开打管理页面：
http://www.routerlogin.net/adv_index.htm  admin/password


![openwrt_admin](/images/2016/11/14/5ECC2BA0-DE51-414C-B64C-F33141CA75DA.png)


# 2 安装shadowsocks
先删除所有的旧shadowsocks包
```
# opkg list_installed | grep shadowsocks
# opkg remove shadowsocks-*
```

下载安装shadowsocks-libev-spec
http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/

要先知道我的路由器是哪个平台的：
https://wiki.openwrt.org/toh/hwdata/netgear/netgear_r6100

```
Platform: Atheros AR9344
Target: ar71xx
Instruction Set: MIPS32
Sub Instruction Set: MIPS32 74K series
```

安装依赖包：

```
# opkg install ip ipset libopenssl iptables-mod-tproxy
```

网友`aa65535`大侠做了很多工作，他把所需要的包都整理好放在了sourceforge.net上，我们只要在OpenWRT的源里添加这个源就可以直接安装了。  
地址在这里：[http://openwrt-dist.sourceforge.net/](http://openwrt-dist.sourceforge.net/)。

到 [http://openwrt-dist.sourceforge.net/dist/base/ar71xx/](http://openwrt-dist.sourceforge.net/dist/base/ar71xx/) 这个页面下载： [ChinaDNS_1.3.2-4_ar71xx.ipk]()


到 [http://openwrt-dist.sourceforge.net/dist/luci/](http://openwrt-dist.sourceforge.net/dist/luci/) 这个页面下载： [luci-app-chinadns_1.6.0-1_all.ipk](http://openwrt-dist.sourceforge.net/dist/luci/luci-app-chinadns_1.6.0-1_all.ipk)、
[luci-app-shadowsocks_1.3.7-1_all.ipk](http://openwrt-dist.sourceforge.net/dist/luci/luci-app-shadowsocks_1.3.7-1_all.ipk)





然后，在Luci中切换至“网络”-“DHCP/DNS”设置，如下图，在”DNS转发”中填入127.0.0.1#35353


其中，35353是ChinaDNS的端口，如果你在之前设置界面里端口号不是35353，这里记得和前面保持一致。

然后切到HOSTS和解析文件选项卡，勾中“忽略解析文件”



开启dns转发：
开启shadowsocks的DNS转发




[下载ChinaDNS](https://github.com/aa65535/openwrt-chinadns/releases)

难道shadowsocks-libev和shadowsocks服务器端是不兼容的?我需要安装shadowsocks-libev 端？

#参考：     
　
https://cokebar.info/archives/664
