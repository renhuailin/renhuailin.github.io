---
layout: article
title: 使用PowerDNS admin来管理PowerDNS
date: 2020-12-08 12:38:00
author: 任怀林
tags:
- linux
thumb: linux.png
key: 2020-12-08-install-powerdns-admin
comment: true
isCJKLanguage: true
---

最近需要使用PowerDNS,因为之前安装过，所以很快就安装好了。之前我也用过PowerDNS admin,不过没有记录下来，这次记录下来，希望对大家有所帮忙。

## 1. 用docker来运行PowerDNS admin

运行PowerDNS admin最简单的方式就是用docker了，只需执行下面的命令就可以了。

```
docker run -d -v pda-data:/data -p 9191:80 ngoduykhanh/powerdns-admin:latest
```

`pda-data`是host上的目录，你在运行这个命令时，请换成你选择的目录。这是把PowerDNS运行时产生的数据存在本地目录，这样下次启动容器时，配置不会丢失。

## 2. 注册用户

在第一次登录时，需要注册用户，第一个用户会被PowerDNS admin设置为管理员。

![login-to-powerdns-admin.png](/assets/img/blog/2020/12/08/WX20201208-221328@2x.png)

## 3. 配置PowerDNS API

### 3.1   修改PowerDNS的启动配置

我们需要修改PowerDNS的启动配置，以enable API接口。

```
api=yes
api-key=1ba8e133e6d684805477f7002182e0ef
# Needed before 4.1.0
webserver=yes

webserver-address=192.168.152.8

# Port of web server to listen on
webserver-port=8081
# # Web server access is only allowed from these subnets
webserver-allow-from=0.0.0.0/0,::/0
```

需要注意的是`webserver-allow-from`这一行一定要加上，因为从4.1.1版本的默认值是127.0.0.1，如果不加上这一行，我们在容器里的PowerDNS admin是无法访问到这个API的。

修改完配置后重启PowerDNS.

```
service pdns restart
```

### 3.2  PowerDNS admin对接PowerDNS

在PowerDNS admin左侧的菜单里选择`Settings`->`PDNS`，打开API配置界面。

![api-config.png](/assets/img/blog/2020/12/08/WX20201208-223428@2x.png)

填入正确的信息，点击`Update`按键即可。

配置成功后，点击左侧的菜单"PDNS"，会看到PowerDNS的统计和配置信息，如果这个页面是空的，那就是API配置没有成功，请自行检查哪里配置错了。

## 4. 添加A记录

首先我们要添加一个域，这个是域不一定非要是一级域名，如`test.com`,也可以是二级域名。

![add-domain.png](/assets/img/blog/2020/12/08/WX20201208-145543@2x.png)

添加完域以后，就可以添加了A记录，A记录是我们最常的，毕竟我们使用NameServer就是为了解析域名嘛。

![add-a-record.png](/assets/img/blog/2020/12/08/WX20201208-230110@2x.png)

其它类型的记录就不一一解释了，你用到时候自然就明白了。
