---
layout: article
title: 搭建一个邮件服务器(Mail Server)
date: 2020-12-23 12:38:00
author: 任怀林
tags:
- linux
thumb: linux.png
key: 2020-12-24-setup-a-mail-server
comment: true
isCJKLanguage: true
---

最近在安装公司的API Connect，安装完成后，在注册新用户时，需要发送激活邮件。需要在apic里配置一个smtp的服务。而目前所有的国内邮箱都开启了SMTP的验证码，不适合直接配置在apic里。于是我想那就自己搭建一个mail server吧。以前做云计算时，自己有云主机，搭建起来比较简单，没想到这次用Google的云主机来搭建时还挺麻烦的。把过程记下来，以备后查。

# 准备工作

## 域名设置

首先我们要把`mail.renhl.com`这个域名指向我们的邮件服务器的IP。
我们需要在godaddy的管理界面为mail.renhl.com添加一个A记录。

```
mail.renhl.com        <IP-address>
```

接下来我们要添加一个`MX`记录。
什么是`MX`记录？ `MX`记录全称为`mail exchanger record`,用于指定负责处理发往收件人域名的邮件服务器。

```
MX record    @           mail.renhl.com
```

## Add PTR Record

## 什么是PTR Record?

PTR是pointer的缩写，pointer在计算机行业里通常翻译为`指针`。它是一种与`A记录(A Record)`相反的一种记录。     
`A记录` 保存的是域名到IP的映射。    
`PTR Record` 保存的是IP到域名的映射。     

**为什么需要PTR Record?**

很多邮箱产品在收到邮件时，会根据发送邮件的IP，去反查一下`PTR Record`,如果这个IP的PRT没有指向你的mx记录的那个域名，这封邮件就会被认定为是垃圾邮件，而被转到垃圾箱或拒收。  

一句话：`PTR Record`是为了防止那些发垃圾邮件营销的人，用伪造的IP来发垃圾邮件。

`PTR Record`是由给你分配IP的公司来管理的，不是你的域名提供商。

比如你在Google Cloud上开了台云主机，IP是Google给你分配的，要设置你IP的PTR记录，你要去Google Cloud上找相关的文档。

## Google Cloud里设置PTR Record

我的mail服务器用的是Google Cloud的云主机，所以我需要到Google Cloud的管理Portal上找添加PTR Record的方法。

您可以使用 Google Cloud Console 更新访问配置或将访问配置添加到您的实例中，具体步骤如下所示：

1. 转到“虚拟机实例”页面。
2. 点击您要修改的实例。
3. 点击顶部菜单中的修改工具。
4. 点击主网络接口旁边的修改工具。
5. 点击外部 IP 下拉菜单。
6. 选中公开 DNS PTR 记录对应的启用框。
7. 输入您的域名。
8. 点击完成。
9. 点击页面底部的保存以保存您的设置。
   ![fdsf](/assets/img/blog/2020/12/24/WX20201224-151248@2x.png)

当你添加一个PTR记录时，google为了验证你真的拥有这个域名，会让你添加一个txt记录，记录的内容Google会提供给你。

![content_of_txt_record](/assets/img/blog/2020/12/24/WX20201224-154453@2x.png)

```
$ dig -t TXT mail.renhl.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t TXT mail.renhl.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17460
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;mail.renhl.com.            IN    TXT

;; ANSWER SECTION:
mail.renhl.com.        3599    IN    TXT    "google-site-verification=wBFC04AX_j-HwtQYBRWXtnV_HiATwg81uXQLWJF_o1U"

;; Query time: 32 msec
;; SERVER: 169.254.169.254#53(169.254.169.254)
;; WHEN: Thu Dec 24 16:10:24 CST 2020
;; MSG SIZE  rcvd: 124
```

添加完PTR记录后，我们查询一下，看看生效没有。

```
dig -x <IP> +short
```

如果返回结果为`mail.renhl.com`,那就说明生效了。

这样前期的准备工作就做好了，我们可以安装POSIX了。

# 安装Postfix

```
# apt-get install postfix -y
```

安装成功后，我们可以通过：

```
postconf mail_version
```

查看一下安装的版本。

# 配置SMTP Relay

如果你的云主机供应商没有封25端口，那么你可以跳过这一步直接发送测试邮件了。

因为Google Cloud封了25端口，所以我们必须配置SMTP Relay. 我这里用的是maijet. 首先，我们要注册Mailjet。

## mailjet配置

在注册完成后，mailjet会询问你使用它的方式。

![mailjet](/assets/img/blog/2020/12/24/WX20201225-155010@2x.png)

我选择`SMTP relay`。

接下来，mailjet会把用户名、密码和SMTP Server提供给我们，请保存好。

## mailjet domain 配置

你申请了`SMTP relay`以后，你可以要转发不同域名的邮件，比如你有两个域名:renhl.com和rhl.com，都配置了邮箱。你希望都通过mailjet来转发，这就需要mailjet的`域`的支持了。

### 添加一个发件人地址(Sender Address)

在mailjet的dashboard页面上点击`manage sender addresses`,添加一个新的发件人地址。这个地址可以是具体的一个邮箱，如：master@renhl.com;也可以是一个域,如：renhl.com。     

我就用域了。

mailjet要求你验证你的域，在你的dns里添加一条指定内容的txt记录。

![verify_your_domain](/assets/img/blog/2020/12/24/WX20201228-093829@2x.png)

### 配置域名的认证方式(Domain Authentication)

为了保证通过我们的邮件服务发出的邮件不被拒收，我们还要配置域名的认证方式： SPF和DKIM。

什么是SPF？

SPF全称为：Sender Policy Framework。

SMTP协议本身没有机制鉴别寄件人的真正身份，电子邮件的“寄件者”一栏可以填上任何名字，于是伪冒他人身份来网络钓鱼或寄出垃圾邮件便相当容易，而真正来源却不易追查。

SPF 记录允许域名管理员公布所授权的邮件服务器IP地址，接收服务器会在收到邮件时验证发件人在 SMTP 会话中执行 MAIL FROM 命令时的邮件地址是否与域名 SPF 记录中所指定的源 IP 匹配，以判断是否为发件人域名伪造。

邮件接收方的收件服务器在接受到邮件后，首先检查域名的SPF记录，来确定发件人的IP地址是否被包含在SPF记录里面，如果在，就认为是一封来自于被授权服务器的邮件，否则会认为是一封伪造的邮件并根据相关政策退回或放进收件人的杂件箱。

**什么时DKIM?**

DKIM全称 `Domain Keys Identified Mail`。

这是一种用公私钥加密来保证邮件安全的方式。
邮件服务在发送邮件时，会用私钥计算一个签名，然后放在header`DKIM-Signature`里，接收方在收到邮件后，通过查询域名的txt记录来获取公钥，然后用公钥来验证这个签名。没有通过验证的邮件就被会当成垃圾邮件。

下面是我的一些设置：
![spf_dkim](/assets/img/blog/2020/12/24/WX20201228-101242@2x.png)

完成这些以后，我们就可以使用mailjet来做relay了，我们要配置一下postfix.

## 配置postfix

```
sudo vi /etc/postfix/main.cf
```

补充上`relayhost`的内容。

```
relayhost = in-v3.mailjet.com:587
```

添加下面的内容到文件的结尾：

```
# outbound relay configurations
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = may
header_size_limit = 4096000
```

编辑文件`sasl_passwd`。

```
sudo vi /etc/postfix/sasl_passwd
```

加入下面的内容：

```
in-v3.mailjet.com:587  api-key:secret-key
```

请把`api-key`和`secret-key`换成你自己在mailjet上获取的。

用postfix提供的命令`postmap`，把这个文件转postfix能读的格式。

```
sudo postmap /etc/postfix/sasl_passwd
```

重启postfix就可以了。

```
sudo systemctl restart postfix
```

# 发送测试邮件

```
echo "test email" | sendmail your-account@gmail.com
```

---

[^1]: https://cloud.google.com/compute/docs/instances/create-ptr-record

[^2]: https://docs.oracle.com/en-us/iaas/Content/DNS/Reference/formattingzonefile.htm#Formatting_a_Zone_File