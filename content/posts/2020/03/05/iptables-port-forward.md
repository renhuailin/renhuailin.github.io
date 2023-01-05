---
layout: post
title: 用iptable做IP转发
date: 2020-03-05 10:38:00
author: 任怀林
categories:
- linux
thumb: linux.png
isCJKLanguage: true
---

```bash
#!/bin/bash


sysctl -w net.ipv4.ip_forward=1

iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT

iptables -F
iptables -t nat -F

# 把某台server上的Port映射到本机的端口上，以便使用本机的所有的IP访问。
# $1 本机的网卡
# $2 本机的内网IP
# $3 要映射到本机的端口
# $4 内网server的IP
# $5 内网service的Port
function redirectPort {

    if [ $# -eq 0 ]  || [ $# -gt 5 ]
    then
        echo "bad parameter!"
        echo -1
    else
        interface=$1
        localIp=$2
        localPort=$3
        serverIP=$4
        servicePort=$5

        iptables -t nat -A PREROUTING -i $interface -p tcp --dport $localPort -j DNAT --to $serverIP:$servicePort

        # 要让包通过FORWARD链
        iptables -A FORWARD -p tcp -d $serverIP --dport $servicePort -j ACCEPT

        # 在这一步只所以要做dnat,是因为，如果不做dnat,源IP将是一个外网的IP，不是一个合法连接了。
        # 所以这一步要将源ip改为本机内网IP，让iptable把包回到这儿。
        iptables -t nat -A POSTROUTING -d $serverIP -p tcp --dport $servicePort -j SNAT --to $localIp

    fi
}


# 把内网的某台server上的Port映射到本机的端口上，以便使用本机的内网IP访问。
# $1 本机的内网IP
# $2 要映射到本机的端口
# $3 内网server的IP
# $4 内网service的Port
function redirectPort2 {

    if [ $# -eq 0 ]  || [ $# -gt 4 ]
    then
        echo "bad parameter!"
        echo -1
    else
        localIp=$1
        localPort=$2
        serverIP=$3
        servicePort=$4


        iptables -t nat -A PREROUTING -d $localIp -p tcp --dport $localPort -j DNAT --to $serverIP:$servicePort
        #iptables -t nat -A PREROUTING -i $interface -p tcp --dport $localPort -j DNAT --to $serverIP:$servicePort

        # 要让包通过FORWARD链
        iptables -A FORWARD -p tcp -d $serverIP --dport $servicePort -j ACCEPT


        # 在这一步只所以要做dnat,是因为，如果不做dnat,源IP将是一个外网的IP，不是一个合法连接了。
        # 所以这一步要将源ip改为本机内网IP，让iptable把包回到这儿。
        iptables -t nat -A POSTROUTING -d $serverIP -p tcp --dport $servicePort -j SNAT --to $localIp

    fi
}

# redirect redis port
redirectPort eth0 192.168.174.14 6379 172.21.0.4 6379

# MySQL    10.224.8.85:3306    33306
redirectPort2 192.168.174.14 33306 10.224.8.85 3306
```
