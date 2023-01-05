---
layout: post
title: 在Ubuntu 16.04上安装PowerDNS并配置主从复制
date: 2017-05-09 17:26:00
author: 任怀林
categories:
- blog
- Linux
thumb: linux.png
isCJKLanguage: true
---

# 1. DNS的基本知识

## 1.1 理解 SOA and GLUE Record

首先要明白什么是SOA，NS,请参见[1]和[2]：
如果用户注册了一个域名,把这个域名的name server设置为你的dns server, 你的这个dns server就是这个域名的权威解析服务器. 就要在配置里添加这个域名的SOA记录,

那什么是GLUE记录呢?
假设你注册了一个域名`yourname.com`
同时你建了一个DNS用来解析你这个域名的,假设这个DNS你设置为`ns1.yourname.com`.

你在你的域名注册商那儿,要把你的域名的dns设置`ns1.yourname.com`.

这里就有一个问题: yourname.com下的所有的域名都要通过ns1.yourname.com这个域名所指向的服务器来解析,但时要知道ns1.yourname.com的IP就要先解析这个域名啊,死循环了.

要想破这个死循环,`ns1.yourname.com`就不能是通过`ns1.yourname.com`来解析的. 它必须是上级DNS能知道的.

如何让上级的DNS知道呢? 这就需要在上级的DNS里添加一个GLUE记录,把ns1.yourname.com和它的IP填入GLUE记录.

这里假设ns1.yourname.com的IP为 `192.168.88.1`.

假设我们添加了一个二级域名`test.yourname.com`,设置A记录为`192.168.88.30`.

解析过程是这样的:

1. `test.yourname.com`上级DNS,也就是解析`com`域的DNS接收到查询`test.yourname.com`请求后,查找`yourname.com`这个域名的DNS,发现设置为`ns1.yourname.com`。

2. `ns1.yourname.com`和`test.yourname.com`它们在同个域里,需要查询它的GLUE记录,发现`ns1.yourname.com`的IP为`192.168.88.1`,于是它向`192.168.88.1`查询`test.yourname.com`的记录。

3. `192.168.88.1`接到查询请求后,查找本地的记录,把`192.168.88.1`这个A记录返回给上级DNS,完成域名到IP的解析。

## 1.2 如何查看GLUE记录呢?

我们用一个实际的例子来看一下.我们看一下百度的域名

```
$ dig NS baidu.com
...

;; QUESTION SECTION:
;baidu.com.            IN    NS

;; ANSWER SECTION:
baidu.com.        1    IN    NS    ns7.baidu.com.
baidu.com.        1    IN    NS    ns2.baidu.com.
baidu.com.        1    IN    NS    ns4.baidu.com.
baidu.com.        1    IN    NS    ns3.baidu.com.
baidu.com.        1    IN    NS    dns.baidu.com.

...
```

我们可以看到百度也是自己解析域名的.所以肯定是有GLUE记录添加到`com`的DNS上的.
接下来我们看看`com`的DNS

```
dig NS com

...

;; QUESTION SECTION:
;com.                IN    NS

;; ANSWER SECTION:
com.            164153    IN    NS    e.gtld-servers.net.
com.            164153    IN    NS    a.gtld-servers.net.
com.            164153    IN    NS    l.gtld-servers.net.
com.            164153    IN    NS    g.gtld-servers.net.
com.            164153    IN    NS    k.gtld-servers.net.
com.            164153    IN    NS    c.gtld-servers.net.
com.            164153    IN    NS    f.gtld-servers.net.
com.            164153    IN    NS    j.gtld-servers.net.
com.            164153    IN    NS    m.gtld-servers.net.
com.            164153    IN    NS    b.gtld-servers.net.
com.            164153    IN    NS    h.gtld-servers.net.
com.            164153    IN    NS    d.gtld-servers.net.
com.            164153    IN    NS    i.gtld-servers.net.

...
```

我们来查询一下baidu.com的NS记录.

```
$ dig NS baidu.com @b.gtld-servers.net

...

;; QUESTION SECTION:
;baidu.com.            IN    NS

;; AUTHORITY SECTION:
baidu.com.        172800    IN    NS    dns.baidu.com.
baidu.com.        172800    IN    NS    ns2.baidu.com.
baidu.com.        172800    IN    NS    ns3.baidu.com.
baidu.com.        172800    IN    NS    ns4.baidu.com.
baidu.com.        172800    IN    NS    ns7.baidu.com.

;; ADDITIONAL SECTION:
dns.baidu.com.        172800    IN    A    202.108.22.220
ns2.baidu.com.        172800    IN    A    61.135.165.235
ns3.baidu.com.        172800    IN    A    220.181.37.10
ns4.baidu.com.        172800    IN    A    220.181.38.10
ns7.baidu.com.        172800    IN    A    119.75.219.82
...
```

其中的`ADDITIONAL SECTION`部分就是GLUE记录.

## 1.3 理解 txt Record

https://en.wikipedia.org/wiki/TXT_record

在kubernetes里有用到!     
這裏是google的解釋:

```
TXT记录的工作原理
TXT记录可包含任何文本值，具体取决于它的用途。
例如，Google使用TXT记录来验证您拥有希望用于我们企业服务的网域。
具体而言，我们会要求您输入TXT记录，其值为“google-site-verification=rXOxyZounnZasA8Z7oaD3c14JdjS9aKSWvsR1EbUSIQ”。
之后，我们会查询您的网域以确定该TXT记录是否已生效。
如果TXT记录已生效，那么我们会知道您有权访问该网域的DNS设置，那么您一定拥有该网域。
```

## 1.4 理解PowerDNS的Authoritative Server 和 Recursor

The Authoritative Server will answer questions about domains it knows about, but will not go out on the net to resolve queries about other domains. However, it can use a recursing backend to provide that functionality

Authoritative Server只回答它知道的域名的信息,它不知道的域名,不会去问网上的其它name server. 不过它可以通过遍历后端(recursing backend)来实现这个功能.
你可以理解为它是这样的一个专家:我知道什么我就答什么,不知道的我也不问别的专家. 不过我有一个后端团队(backend),如果他们知道答案,也可以代我回答.

Recursor 可以理解为一个域名的缓存,本身没有域名信息,你问我,我就问别人(DNS)去.     
很多dns是把这两部分的功能融合在一起的,PowerDNS把它们分开提供.

## 1.5 规划

在VirtualBox里启动两台Ubuntu 16.04的虚机,安装系统自带的PowerDNS,一台设置为master,另一台设置为Slave.
`172.16.69.70` 

| First Header | Second Header | Second Header | Second Header | Second Header |
| ------------ | ------------- | ------------- | ------------- | ------------- |
| Content Cell | Content Cell  | Content Cell  | Content Cell  | Content Cell  |
| Content Cell | Content Cell  | Content Cell  | Content Cell  | Content Cell  |

## 1.6 安装 PowerDNS

安装非常简单:

```
$ sudo apt install mysql-server 
```

一定要先安装mysql然后再安装powerdns,而且一定要用安装程序的提示来配置mysql。

```
$ sudo apt install pdns-server  pdns-backend-mysql
```

在弹出的对话框`Configuring pdns-backend-mysql`里填入一个密码,我在这里输入了`powerdns`.

首先`/etc/powerdns/pdns.d`目录请只保留`pdns.local.gmysql.conf`这个文件。

之前我保留了`pdns.simplebind.conf`这个文件，powerdns就报错。

```
powerdns Trying to set unexisting parameter 'gmysql-host'
```

```
mysql> insert into supermasters values ('111.111.111.111', 'ns2.example-dns.com', 'admin');
```

## 1.7 错误处理

因为我是在实验环境搭建的这两个dns.

```
# /etc/hosts

172.16.69.70 ns1.xiangcloud.com.cn
172.16.69.71 ns2.xiangcloud.com.cn
```

然后我添加一个测试的域名,它不同步.奇怪.
我把master的日志级别设置为

```
loglevel=7
```

然后在`/var/log/syslog`

```
Notification for hhhhh.com. to 192.157.208.178:53 failed after retries
```

这个IP不是slave的IP啊?

明明是已经在`/etc/hosts`里配置了,PowerDNS怎么不认呢?
我没办法又启动了两个ubuntu 14.04,安装了powerdns 3.3,同样的配置,它可以正常同步.
看来是master的问题, 16.04的PowerDNS是4.0的,可能已经不支持读取`/etc/hosts`里的配置了.
于是我把这两条NS记录

```
172.16.69.70 ns1.xiangcloud.com.cn
172.16.69.71 ns2.xiangcloud.com.cn
```

添加到14.04的PowerDNS里,把70的dns设置为72.

再用命令强制同步就可以了.

```
$ pdns_control notify hhhhh.com
```

这个问题在线上系统上应该是不会出现的,因为线上系统的ns记录一定是以GLUE记录的方式添加到上级DNS里的.

### 问题2: PowerDNS  SOA Serial are not updated automatically

虽然可以同步了,但是添加的域名在API更新了记录时, SOA Serial不自动更新,Slave发现 SOA Serial没变化就同步记录了.     
所以我们需要在记录变化时自动更新SOA Serial.我在网上找了N久终于找到了相关的设置.

在通过API创建zone的时候添加下面的参数:

```
"soa_edit_api" : "INCEPTION-INCREMENT",
```

http://jpmens.net/2015/01/09/a-look-at-the-powerdns-rest-api/

## Received NOTIFY for xiangcloud.com.cn. from 122.115.54.57 which is not a master

场景是这样的:
我从原来的dns里导出了域名解析的信息,然后通过powerdns的api插入到了主powerdns里. 不知道什么原因,这些记录没有同步过来,后来我重启了slave的pdns进程.
我发现没有同步后,我就手动把主powerdns的域名和解析记录导出mysqldump出来,导入到slave powerdns里了.
接下来我在master上添加记录,发现还是同步不了.就报上面的这个错误.

看了源代码:

```cpp
//packethandler.cc
  else if(!db->isMaster(p->qdomain, p->getRemote().toString())) {
    L<<Logger::Error<<"Received NOTIFY for "<<p->qdomain<<" from "<<p->getRemote()<<" which is not a master"<<endl;
    return RCode::Refused;
  }
```

db是一个DNSBackend,我们用的是sqlbackend,所以其实它是`GSQLBackend` 是去数据库里查询域名了.然后比对这个域名的master字段.如果字段与给 `p->getRemote()` 也是上面的122.115.54.57不相等,那就报这个错.
我看了slave上的pdns数据库里的domains表,记录里master字段为空.

于是新插了一个域名,发现同步是好使的,在slave上的pdns数据库里的domains表里,记录是这样的.

```
mysql> select * from domains where name = 'junaidong1.com.cn';
+-------+-------------------+---------------+------------+-------+-----------------+---------+
| id    | name              | master        | last_check | type  | notified_serial | account |
+-------+-------------------+---------------+------------+-------+-----------------+---------+
| 16266 | test.com.cn | 122.115.54.57 | 1497338783 | SLAVE |            NULL | admin   |
+-------+-------------------+---------------+------------+-------+-----------------+---------+
1 row in set (0.00 sec)
```

要想实现同步,slave上的数据必须是这样的.

原因找到就好办了,手动把slave上的数据库里的`domains`表的 `master`,`last_check`,`type`这三个字段加上内容就可以了.

# 参考:

[1] [SOA Records](https://support.dnsimple.com/articles/soa-record/)        
[2] [http://www.zytrax.com/books/dns/ch8/soa.html](http://www.zytrax.com/books/dns/ch8/soa.html)        
[3] [How To Install and Configure PowerDNS with a MariaDB Backend on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-powerdns-with-a-mariadb-backend-on-ubuntu-14-04)    
[4] [dns 里的glue records是什么意思，有什么作用？](https://www.zhihu.com/question/19691928)   
[5] [Circular_dependencies_and_glue_records](https://en.wikipedia.org/wiki/Domain_Name_System#Circular_dependencies_and_glue_records)
[6] [自建PowerDNS免费DNS服务器-PowerDNS和MariaDB安装配置教程](https://www.freehao123.com/powerdns/)      
[7] [搭建一套权威 DNS 服务架构](https://blog.phoenixlzx.com/2016/09/07/powerdns-admin-replacing-moedns/) 