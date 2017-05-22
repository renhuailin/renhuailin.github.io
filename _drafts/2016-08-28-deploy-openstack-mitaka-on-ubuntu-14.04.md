---
layout: post
title: 部署OpenStack Mitaka
date: 2016-08-27 12:30:00
author: 任怀林
categories:
- blog
- OpenStack
thumb: openstack.png
---

# 1 规划

| 0:0 | 1:0 |
| -- | -- |
| 0:2 | 1:2 |


| Header1 | Header2 | Header3 | 
|:--------|:-------:|--------:| 
| cell1 | cell2 | cell3 | 
| cell4 | cell5 | cell6 |



192.168.234.202   Openstack controller node     
192.168.234.205   Openstack network node     
192.168.234.204   Openstack compute node     

networ node必须有三个NIC(network interface card),也就是网卡．

管理网段：10.0.0.0/24



#参考：　
[https://github.com/openstack/keystone/blob/master/tools/sample_data.sh](https://github.com/openstack/keystone/blob/master/tools/sample_data.sh)      
[https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst](https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst)
