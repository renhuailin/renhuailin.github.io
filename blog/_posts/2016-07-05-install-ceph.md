---
layout: post
title: 安装Ceph
date: 2016-07-05 12:35:21
author: 任怀林
categories:
- blog
- linux
thumb: linux.png
---

# 系统配置

4台主机,运行ubuntu 16.04
192.168.99.50    Ceph Monitor    
192.168.99.51    Ceph OSD1
192.168.99.52    Ceph OSD2
192.168.99.53    Ceph Deploy

选用的Ceph版本：

V10.2.2 JEWEL RELEASED

## Ceph Deploy节点192.168.99.53的安装
For Ubuntu ,Debian.
Add the release key:

```
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
```

```
echo deb http://download.ceph.com/debian-jewel/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
```

更新源，安装`ceph-deploy`
```
sudo apt-get update && sudo apt-get install ceph-deploy
```


## Ceph node设置

安装ntp server.  包括monitor和两个osd节点。
```
sudo apt-get install ntp
```

安装 SSH Server,要确保所有节点都安装了SSH Server。
```
sudo apt-get install openssh-server
```

创建一个用于部署ceph的用户
ceph-deploy这个工具必须能登录到节点上，而且要可以无需密码地运行sudo。最新版的ceph-deploy 支持`--username`这个选项，可以指定一个用户，也可以是`root`(not recommended),为了方便我们就用`harley`吧。


在每个节点上执行这个命令，让harleyr无需密码地运行sudo。
```
echo "harley ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/harley
sudo chmod 0440 /etc/sudoers.d/harley
```

## Ceph Deploy节点配置Login to nodes without password
 ceph-deploy没有用expect,所以不能输入密码，要求ceph-deploy用harley这个用户登录到各个节点时，不能提示输入密码。
 我们要在Ceph Deploy节点上生成证书然后分发到相关的节点上。
 
 add to ceph deploy node
 
``` 
vi /etc/hosts
```
添加下列内容：
```
192.168.99.50 ceph_monitor
192.168.99.51 ceph_osd1
192.168.99.52 ceph_osd2
```

复制ssh key到各个节点：

```
ssh-copy-id harley@ceph_monitor
ssh-copy-id harley@ceph_osd1
ssh-copy-id harley@ceph_osd2
```




ENSURE CONNECTIVITY
要能Ping通相关的节点？？？

开放相关的端口
如果你用的CentOS等，他默认的防火墙是很严格的，你需要配置防火墙打开相关的端口。


# 开始创建一个集群！

在ceph-deploy节点上
```
mkdir my-cluster
cd my-cluster

```

## CREATE A CLUSTER

ceph-deploy --user harley new  ceph_monitor

[ceph_deploy][ERROR ] RuntimeError: connecting to host: harley@ceph_monitor resulted in errors: IOError cannot send (already closed?)

原因是ubuntu 16.04上没有python这个命令，默认是python3. 登录到各个节点，cd到/usr/bin做个软链接就行了。




**参考资料：**
