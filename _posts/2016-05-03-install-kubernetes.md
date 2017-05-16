---
layout: post
title: 安装kubernetes
date: 2016-05-03 15:35:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
---

# 系统配置
两台主机,运行ubuntu 14.04

|-----------------+------------|
| IP | 说明 |
|-----------------|:-----------|
| 192.168.99.60  | kube-minion2 |
| 192.168.99.61  | master/minion 一体机 |
| 192.168.99.62  | kube-minion1 |

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

```
$ ssh-copy-id harley@192.168.99.60
$ ssh-copy-id harley@192.168.99.61
$ ssh-copy-id harley@192.168.99.62
```


在每个节点上执行这个命令，让harleyr无需密码地运行sudo。
```
$ echo "harley ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/harley
$ sudo chmod 0440 /etc/sudoers.d/harley
```

因为重新配置docker需要用到`brctl`命令。需要先安装相关的包。
`apt-get install bridge-utils`.

kubernetes版本：1.2.0

从github上下载kubernetes 1.2.0的二进制安装包，上传到kubernetes包到kube-master节点上并解压。

```
 $ sudo cp kubernetes/server/kubernetes-server-linux-amd64.tar.gz /opt
 # tar xzvf kubernetes-server-linux-amd64.tar.gz
```
然后把`/opt/kubernetes/server/bin`加入到PATH里。

You will run docker, kubelet, and kube-proxy outside of a container, the same way you would run any system daemon, so you just need the bare binaries. For etcd, kube-apiserver, kube-controller-manager, and kube-scheduler, we recommend that you run these as containers, so you need an image to be built.

```
# KUBERNETES_PROVIDER=ubuntu ./kube-up.sh 
```


`kubelet`,`kube-proxy`不放在容器里。 `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` 放在容器里运行。

```
$ docker pull quay.io/coreos/etcd:v2.2.1
```

The hyperkube binary is an all in one binary
hyperkube kubelet ... runs the kubelet, hyperkube apiserver ... runs an apiserver, etc.

```
$ export KUBE_VERSION=1.3.2
$ export FLANNEL_VERSION=0.5.5
$ export ETCD_VERSION=2.2.1
```


modify `~/kubernetes/cluster/ubuntu/download-release.sh`      

``` bash
#!/bin/bash

# Copyright 2015 The Kubernetes Authors All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Download the etcd, flannel, and K8s binaries automatically and stored in binaries directory
# Run as root only

# author @resouer @WIZARD-CXY
set -e

function cleanup {
  # cleanup work
  rm -rf flannel* kubernetes* etcd* binaries
}

# 把下面的这一行注释掉，在国内我们经常会因网络原因导致下载进程卡死，当你手动中断时，你之前辛辛苦苦下载的东西会全部删除，就是因为这条命令。
#trap cleanup SIGHUP SIGINT SIGTERM

pushd $(dirname $0)
mkdir -p binaries/master
mkdir -p binaries/minion

# flannel
FLANNEL_VERSION=${FLANNEL_VERSION:-"0.5.5"}
echo "Prepare flannel ${FLANNEL_VERSION} release ..."
grep -q "^${FLANNEL_VERSION}\$" binaries/.flannel 2>/dev/null || {
  curl -L  https://github.com/coreos/flannel/releases/download/v${FLANNEL_VERSION}/flannel-${FLANNEL_VERSION}-linux-amd64.tar.gz -o flannel.tar.gz
  tar xzf flannel.tar.gz
  cp flannel-${FLANNEL_VERSION}/flanneld binaries/master
  cp flannel-${FLANNEL_VERSION}/flanneld binaries/minion
  echo ${FLANNEL_VERSION} > binaries/.flannel
}

# ectd
ETCD_VERSION=${ETCD_VERSION:-"2.2.1"}
ETCD="etcd-v${ETCD_VERSION}-linux-amd64"
echo "Prepare etcd ${ETCD_VERSION} release ..."
grep -q "^${ETCD_VERSION}\$" binaries/.etcd 2>/dev/null || {
  curl -L https://github.com/coreos/etcd/releases/download/v${ETCD_VERSION}/${ETCD}.tar.gz -o etcd.tar.gz
  tar xzf etcd.tar.gz
  cp ${ETCD}/etcd ${ETCD}/etcdctl binaries/master
  echo ${ETCD_VERSION} > binaries/.etcd
}

# k8s
KUBE_VERSION=${KUBE_VERSION:-"1.2.0"}
echo "Prepare kubernetes ${KUBE_VERSION} release ..."
grep -q "^${KUBE_VERSION}\$" binaries/.kubernetes 2>/dev/null || {

  # 不要再去下载了，我们已经下载完了，请直接
  #curl -L https://github.com/kubernetes/kubernetes/releases/download/v${KUBE_VERSION}/kubernetes.tar.gz -o kubernetes.tar.gz
  #tar xzf kubernetes.tar.gz

  # 直接copy过来
  cp ../../server/kubernetes-server-linux-amd64.tar.gz .
  cp ../../server/kubernetes-salt.tar.gz .

  #pushd kubernetes/server
  tar xzf kubernetes-server-linux-amd64.tar.gz
  tar xzf kubernetes-salt.tar.gz .
  #popd
  cp kubernetes/server/bin/kube-apiserver \
     kubernetes/server/bin/kube-controller-manager \
     kubernetes/server/bin/kube-scheduler binaries/master
  cp kubernetes/server/bin/kubelet \
     kubernetes/server/bin/kube-proxy binaries/minion
  cp kubernetes/server/bin/kubectl binaries/
  cp -r kubernetes/saltbase .


  echo ${KUBE_VERSION} > binaries/.kubernetes
}

rm -rf flannel* kubernetes* etcd*

echo "Done! All your binaries locate in kubernetes/cluster/ubuntu/binaries directory"
popd
```

Download easy rsa package.   
这时还是在cluster这个目录。
```
$ curl -L -O https://storage.googleapis.com/kubernetes-release/easy-rsa/easy-rsa.tar.gz > /dev/null 2>&1
```

把`kubernetes-salt.tar.gz`copy 到 cluster目录。
cp ../../server/kubernetes-salt.tar.gz .
把里面的目录解压出来，`kubernetes/saltbase` mv到 cluster目录。


登录到kube-master这台机器上,创建相关目录。

```
$ mkdir -p ~/kube/default
```

Back to workstation,and copy files to kube-master.

```
$ cd kubernetes/cluster/ubuntu
$ pwd
/home/harley/Software/kubernetes/cluster/ubuntu
$ scp -r \
    saltbase/salt/generate-cert/make-ca-cert.sh \
    easy-rsa.tar.gz \
    config-default.sh \
    util.sh \
    minion/* \
    master/* \
    reconfDocker.sh \
    binaries/master/ \
    binaries/minion \
    "192.168.56.101:~/kube"
```

我们要用flannel来实现overlay的网络，要把相关的配置copy到kube-master上。

```
$ scp -r minion-flannel/* master-flannel/* "192.168.56.101:~/kube"
```

我们回到kube-master上，看看kube目录有哪些内容了。

```
$ tree kube
kube
├── config-default.sh
├── default
├── easy-rsa.tar.gz
├── init_conf
│   ├── etcd.conf
│   ├── flanneld.conf
│   ├── kube-apiserver.conf
│   ├── kube-controller-manager.conf
│   ├── kubelet.conf
│   ├── kube-proxy.conf
│   └── kube-scheduler.conf
├── init_scripts
│   ├── etcd
│   ├── flanneld
│   ├── kube-apiserver
│   ├── kube-controller-manager
│   ├── kubelet
│   ├── kube-proxy
│   └── kube-scheduler
├── make-ca-cert.sh
├── master
│   ├── etcd
│   ├── etcdctl
│   ├── flanneld
│   ├── kube-apiserver
│   ├── kube-controller-manager
│   └── kube-scheduler
├── minion
│   ├── flanneld
│   ├── kubelet
│   └── kube-proxy
├── reconfDocker.sh
└── util.sh
```
看来flannel的配置文件只保留了一分 master-flannel下的内容。


Create `~/kube/default/etcd`。

```
ETCD_OPTS="
 -name infra
 -listen-client-urls http://127.0.0.1:4001,http://192.168.56.101:4001
 -advertise-client-urls http://192.168.56.101:4001"
```

Create `~/kube/default/kube-apiserver` with following contents.

```
KUBE_APISERVER_OPTS="
 --insecure-bind-address=0.0.0.0
 --insecure-port=8080
 --etcd-servers=http://127.0.0.1:4001
 --logtostderr=true
 --service-cluster-ip-range=192.168.3.0/24
 --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,SecurityContextDeny
 --service-node-port-range=30000-32767
 --advertise-address=192.168.56.101
 --client-ca-file=/srv/kubernetes/ca.crt
 --tls-cert-file=/srv/kubernetes/server.cert
 --tls-private-key-file=/srv/kubernetes/server.key"
```

Create `~/kube/default/kube-controller-manager` with following contents.

```
KUBE_CONTROLLER_MANAGER_OPTS="
 --master=127.0.0.1:8080
 --root-ca-file=/srv/kubernetes/ca.crt
 --service-account-private-key-file=/srv/kubernetes/server.key
 --logtostderr=true"
```

Create `~/kube/default/kubelet` with following contents.

```
KUBELET_OPTS="
 --hostname-override=192.168.56.101
 --api-servers=http://192.168.56.101:8080
 --logtostderr=true
 --cluster-dns=192.168.3.10
 --cluster-domain=cluster.local
 --config="
```

Create `~/kube/default/kube-proxy` with following contents.

```
KUBE_PROXY_OPTS="
 --hostname-override=192.168.56.101
 --master=http://192.168.56.101:8080
 --logtostderr=true"
```

Create `~/kube/default/flanneld` with following contents.

```
FLANNEL_OPTS="--etcd-endpoints=http://127.0.0.1:4001
 --ip-masq
 --iface=192.168.56.101"
```

现在文件树结构是这样的：


```
.
├── kube
│   ├── config-default.sh
│   ├── default
│   │   ├── etcd
│   │   ├── flanneld
│   │   ├── kube-apiserver
│   │   ├── kube-controller-manager
│   │   ├── kubelet
│   │   └── kube-proxy
│   ├── easy-rsa.tar.gz
│   ├── init_conf
│   │   ├── etcd.conf
│   │   ├── flanneld.conf
│   │   ├── kube-apiserver.conf
│   │   ├── kube-controller-manager.conf
│   │   ├── kubelet.conf
│   │   ├── kube-proxy.conf
│   │   └── kube-scheduler.conf
│   ├── init_scripts
│   │   ├── etcd
│   │   ├── flanneld
│   │   ├── kube-apiserver
│   │   ├── kube-controller-manager
│   │   ├── kubelet
│   │   ├── kube-proxy
│   │   └── kube-scheduler
│   ├── make-ca-cert.sh
│   ├── master
│   │   ├── etcd
│   │   ├── etcdctl
│   │   ├── flanneld
│   │   ├── kube-apiserver
│   │   ├── kube-controller-manager
│   │   └── kube-scheduler
│   ├── minion
│   │   ├── flanneld
│   │   ├── kubelet
│   │   └── kube-proxy
│   ├── reconfDocker.sh
│   └── util.sh
└── kubernetes.tar.gz
```




```
$ sudo cp ~/kube/default/* /etc/default/
$ sudo cp ~/kube/init_conf/* /etc/init/
$ sudo cp ~/kube/init_scripts/* /etc/init.d/

$ sudo groupadd -f -r kube-cert

$ sudo ~/kube/make-ca-cert.sh "192.168.56.101" "IP:192.168.56.101,IP:192.168.3.1,DNS:kubernetes,DNS:kubernetes.default,DNS:kubernetes.default.svc,DNS:kubernetes.default.svc.cluster.local"

$ sudo mkdir -p /opt/bin/
$ sudo cp ~/kube/master/* /opt/bin/
$ sudo cp ~/kube/minion/* /opt/bin/


$ sudo service etcd start

#因为我们使用的是flannel + VxLAN,所以我们需要重新配置docker.
$ sudo FLANNEL_NET="172.16.0.0/16" KUBE_CONFIG_FILE="config-default.sh" DOCKER_OPTS=""  ~/kubeeconfDocker.sh ai
```
接下来验证一下我们的安装是否成功，从`uitl.sh`来看，验证方法就是ssh登录上去，然后查看指定的进程有没有运行起来。

kube master上要运行这些进程："kube-apiserver" "kube-controller-manager" "kube-scheduler"。

kube minion上要运行这些进程："kube-proxy" "kubelet" "docker"。


vi `~/.kube/config` ,input following contents.

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://192.168.56.101:8080
  name: ubuntu
contexts:
- context:
    cluster: ubuntu
    user: ubuntu
  name: ubuntu
current-context: ubuntu
kind: Config
preferences: {}
users:
- name: ubuntu
  user:
    password: P9Ez40tyITBdJnPf
    username: admin
```

验证我们的安装是否成功。

```
$ cd cluster/ubuntu
$ binaries/kubectl get nodes
NAME             STATUS    AGE
192.168.56.101   Ready     2h
```

问题1： 不同节点上的两个Pod不能相互Ping通；
仔细研究发现在一个节点的flannel的子网配置和docker的启动参数里的`--bip`不一样，应该是这个原因了。
```
# docker的启动参数为
/usr/bin/docker daemon -H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock --bip=172.16.3.1/24 --mtu=1450 --raw-logs

# cat  /run/flannel/subnet.env 显示为：
FLANNEL_NETWORK=172.16.0.0/16
FLANNEL_SUBNET=172.16.45.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
应该是在安装节点时`reconfDocker.sh`,我们重新配置一下。
```
$ cd ~/kube
$ sudo FLANNEL_NET="172.16.45.1/24" KUBE_CONFIG_FILE="config-default.sh" DOCKER_OPTS=""  ~/kubeeconfDocker.sh i 
```
结果还是不通，这是我的路由表。eth0是net出去，eth1是virtualbox,host-only,那flannel向etcd注册的是哪个IP？

```
harley@k8smasterminion:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    0      0        0 eth0
10.0.2.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 flannel.1
172.16.30.0     0.0.0.0         255.255.255.0   U     0      0        0 docker0
192.168.99.0    0.0.0.0         255.255.255.0   U     0      0        0 eth1
```

**参考资料：**
1. http://dockone.io/article/622
2. http://aclisp.github.io/blog/2015/08/20/kubernetes-scratch.html
3. http://linux.die.net/man/1/bash
4. [Linux/UNIX下使用ssh-keygen设置SSH无密码登录](http://blog.csdn.net/leexide/article/details/17252369)
