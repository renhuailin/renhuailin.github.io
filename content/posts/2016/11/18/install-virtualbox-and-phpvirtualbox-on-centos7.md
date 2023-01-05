---
layout: post
title: Cent OS 7 VirtualBox 5.0和phpVirtualBox
date: 2016-11-18 12:30:00
author: 任怀林
categories:
- blog
- linux
thumb: linux.png
isCJKLanguage: true
---

在研发环境的CentOS 7上安装VirtualBox以便安装虚拟机使用。

参考：
http://tecadmin.net/install-oracle-virtualbox-on-centos-redhat-and-fedora/#

上面的文章里的`service vboxdrv setup`不能运行，应该用下面的命令:
`/usr/lib/virtualbox/vboxdrv.sh setup`

要关闭SELinux
```
setenforce 0
```


/lib/modules/3.10.0-327.el7.x86_64/build 

要下载 Extension Pack
http://download.virtualbox.org/virtualbox/5.0.28/Oracle_VM_VirtualBox_Extension_Pack-5.0.28-111378.vbox-extpack

$ sudo VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-5.0.28-111378.vbox-extpack 

启动：
/usr/lib/virtualbox/vboxwebsrv &

要想通过桌面连上去必须要在主机的设置里把主机的

# 安装phpVirtualBox



要安装 php-soap



#参考：　
[http://tecadmin.net/install-oracle-virtualbox-on-centos-redhat-and-fedora/#](http://tecadmin.net/install-oracle-virtualbox-on-centos-redhat-and-fedora/#)

