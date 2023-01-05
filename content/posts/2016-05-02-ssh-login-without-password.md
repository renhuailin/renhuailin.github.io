---
layout: post
title: 实现SSH无密码登录远程主机
date: 2016-05-02 15:35:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
isCJKLanguage: true
---


两台主机,运行ubuntu 14.04
192.168.56.1     本机
182.168.56.102   远程主机

远程主机的登录用户为ubuntu,我们希望用这个用户登录182.168.56.102时，不提示密码，主要是为了实现一些自动化运维的操作。

请在本机执行下面的命令：

```
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/harley/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/harley/.ssh/id_rsa.
Your public key has been saved in /home/harley/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:iTJopd0aS59/PPFWHcpYHV5/NHvYGqDeWdYDIGF418E harley@harley-ThinkPad-T420
The key's randomart image is:
+---[RSA 2048]----+
|        .+..+..  |
|       ......E oo|
|    .   . .. .+=*|
|   = . . ..  .=**|
|  + * o S. .++.+=|
| . . B .  o.ooo .|
|    o o  . o .   |
|       .  + o    |
|        .. o     |
+----[SHA256]-----+
```

copy public key to remote host.

```
$ scp ~/.ssh/id_rsa.pub  ubuntu@182.168.56.102:/home/ubuntu
```

接下来请登录到远程主机上完成下面的操作。

```
$ cat id_dsa.pub >> ~/.ssh/authorized_keys
```
如果目录不存在，请自行创建相关目录。

你不是感觉这样操作太麻烦，还需要登录到行程的主机上，你可以在本机用一条命令搞定。

```
$ ssh-copy-id -i ubuntu@192.168.56.101
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/harley/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
ubuntu@192.168.56.102's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'ubuntu@192.168.56.102'"
and check to make sure that only the key(s) you wanted were added.
```

**参考资料：**
* http://www.linuxproblem.org/art_9.html
* [Linux/UNIX下使用ssh-keygen设置SSH无密码登录](http://blog.csdn.net/leexide/article/details/17252369)
