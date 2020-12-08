---
layout: post
title: 使用PowerDNS admin来管理PowerDNS
date: 2020-12-08 13:39:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
---

编辑PowerDNS的配置文件，打开API Server.

```
vi pdns.local.gmysql.conf
```



```
api=yes
api-key=changeme
# Needed before 4.1.0
webserver=yes


# When running multiple instances you might want to specify on 
# which address the web server should run:

# IP Address of web server to listen on
webserver-address=127.0.0.1
# Port of web server to listen on
webserver-port=8081
# Web server access is only allowed from these subnets
webserver-allow-from=0.0.0.0/0,::/0
```





```
dig admin.apic.faw-vw.com @127.0.0.1
```

下面是结果：

```
; <<>> DiG 9.11.3-1ubuntu1.8-Ubuntu <<>> admin.apic.faw-vw.com @127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47368
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1680
;; QUESTION SECTION:
;admin.apic.faw-vw.com. IN A

;; ANSWER SECTION:
admin.apic.faw-vw.com. 60 IN A 192.168.23.54

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Tue Dec 08 15:44:33 CST 2020
;; MSG SIZE rcvd: 66
```









# **参考资料：**
