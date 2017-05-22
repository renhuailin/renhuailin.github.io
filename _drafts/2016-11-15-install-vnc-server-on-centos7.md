---
layout: post
title: Cent OS 7安装VNC server.
date: 2016-11-15 12:30:00
author: 任怀林
categories:
- blog
- linux
thumb: linux.png
---

# 1 创建vnc用户

```
$ sudo useradd -c "User xcvnc Configured for VNC Access" xcvnc
$ sudo passwd xcvnc
```




#参考：　
[https://github.com/openstack/keystone/blob/master/tools/sample_data.sh](https://github.com/openstack/keystone/blob/master/tools/sample_data.sh)      
[https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst](https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst)
