---
layout: post
title: Linux下用shadowsocks和Privoxy实现翻墙
date: 2017-02-03 15:30:00
author: 任怀林
categories:
- linux
thumb: linux.png
isCJKLanguage: true
---

安装Shadowsocks：

Debian / Ubuntu:

```
# apt-get install python-pip
# pip install shadowsocks
```


编辑文件`/etc/shadowsocks.json`，填写代理信息：


``` json
{
    "server":"shadowsocks的IP",
    "server_port":8388,
    "local_address": "0.0.0.0",
    "local_port":1080,
    "password":"fuuuuuuuuuuuuckgfw",
    "timeout":300,
    "method":"rc4-md5",
    "fast_open": false
}
```
其中local_address是本地绑定的IP。
method是加密码算法，这个必须跟shadowsocks的服务端的一致。
其它参数不解释，很容易理解。

接下来启动shadowsocks客户端。 

```
$ sudo sslocal -c /etc/shadowsocks.json -d start
```

要停止它，请用下面的命令：  

```
$ sudo sslocal -c /etc/shadowsocks.json -d stop
```


Shadowsocks使用的socks5协议,而终端很多工具目前只支持http和https等协议，所以我们要用工具把socks5转成http协议。
在linux下可以使用privoxy来实现这个转换。

```
$ sudo apt-get install privoxy
```

编辑配置文件`/etc/privoxy/config`,加入下面两行内容。

```
forward-socks5t   /               127.0.0.1:1080 .
listen-address  localhost:8118
```
需要注意：
1.  127.0.0.1：1080是shadowsocks的客户端的IP和端口，要与上面shadowsocks里的配置相符。
2.  请不要忽略这行配置后面的`.`,这可不是句号的意思，这个必须有。


启动服务：

```
$ sudo service privoxy restart
```

这样就配置好了。试一下

```
$ export http_proxy=http://localhost:8118
$ export https_proxy=http://localhost:8118
$ curl ip.cn
```

# 参考:
[http://www.jianshu.com/p/8e7d7f57bf59](http://www.jianshu.com/p/8e7d7f57bf59)
