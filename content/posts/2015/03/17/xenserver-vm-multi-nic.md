---
layout: post
title: XenServer的虚拟机里安装配置两块网卡
date: 2015-03-11 12:30:00
author: 任怀林
categories: 
- blog
- OpenStack
thumb: linux.png
isCJKLanguage: true
---

在安装OpenStack时，我的network节点只配置了一个nic，所以当`配置Open vSwitch (OVS) 服务`时，在运行完命令
{% highlight sh %}
# service openvswitch-switch restart
# ovs-vsctl add-br br-ex
# ovs-vsctl add-port br-ex eth0
{% endhighlight %}
之后，我就无法通过eth0上的那个external IP来访问它了，管理IP也无法访问，所以官方的文档是要求network节点要有3个nic。

没有办法，我只能新建一个vm来安装network,这个过程还是碰到了问题。

因为我的XenServer有两卡网卡，我就创建了两个虚拟网卡，分别映射到这两块网卡上。vm启动后，我配置网络如下：

{% highlight sh %}
# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.234.205
netmask 255.255.255.0
gateway 192.168.234.1

#management interface
auto eth1
iface eth1 inet static
address 10.0.0.21
netmask 255.255.255.0
#gateway 10.0.0.1

#Instance tunnels interface.
auto eth1:1
iface eth1:1 inet static
address 10.0.1.21
netmask 255.255.255.0
{% endhighlight %}

也就是外网IP设置在eth0上，管理IP和tunnels IP设置在eth1上。 可是我发现，我无法通过eth1上的管理来访问vm，折腾了好久，我才意识到，XenServer上的网卡2应该没插网线。这提醒了我，如果宿主机上有两块网卡，一定要确定都能工作才行。

知道问题了就好解决，两块虚拟网卡都映射到同一块物理网卡就没问题了。

当然在运行完上面的命令后，你还是无法通过eth0上的IP来访问，
[这个帖子](http://www.gossamer-threads.com/lists/openstack/operators/34627)里讲了解决方案，那就是清除eth0的IP,把外网的IP绑定到br-ex上．

{% highlight sh %}
ifconfig eth0 0.0.0.0 
ifconfig br-ex 192.168.234.205 
{% endhighlight %}

你会问，我已经无法连接这台虚拟机了，怎么执行这个操作？

当然是用的network节点的管理段IP啊，它就是干这个用的，：）。
