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
| 172.16.69.231  | master/minion 一体机 |
| 172.16.69.232  | kube-minion1 |
| 172.16.69.233  | kube-minion2 |

为了安装k8s,这三台机器要能翻墙,具体的方法恕不奉告,😂.


# Prerequisites

下面的操作需要在所有的节点做一遍.

因为我是用privoxy代理翻墙的,所以我要在docker的配置添加代理信息.

**注意: 在命令行exoport http_proxy=xxx.xxx.xx.xx.:xxx的方式对docker不起作用**

$ mkdir -p /etc/systemd/system/docker.service.d
Create a file called `/etc/systemd/system/docker.service.d/http-proxy.conf` that adds the HTTP_PROXY environment variable:

```
[Service]
Environment="HTTP_PROXY=http://172.16.69.231:8118" "HTTPS_PROXY=http://172.16.69.231:8118" "KUBERNETES_HTTPS_PROXY=http://172.16.69.231:8118" "KUBERNETES_HTTP_PROXY=http://172.16.69.231:8118"
```

$ sudo systemctl daemon-reload

$ sudo systemctl restart docker

在`docker info`的输出中,你就可以看到代理信息了.


$ systemctl show --property=Environment docker
Environment=HTTP_PROXY=http://proxy.example.com:80/

```
# apt-get update && apt-get install -y apt-transport-https
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
# apt-get update

### Install docker if you don't have it already.
# apt-get install -y docker-engine
# apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```


# 初始化 kubernetes master

下载后的kube组件并未自动运行起来。在 /lib/systemd/system下面我们能看到kubelet.service：

kubeadm init --apiserver-advertise-address 172.16.69.231 --pod-network-cidr 10.244.0.0/16


[apiclient] Created API client, waiting for the control plane to become ready

这一步卡死了

```
root@k8smaster:/etc/kubernetes# docker logs -f k8s_kube-apiserver_kube-apiserver-k8smaster_kube-system_0d5f2da47764aad353c8d5deb5a3e692_0
E0525 06:09:27.034171       1 cacher.go:274] unexpected ListAndWatch error: k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/cacher.go:215: Failed to list *api.PodTemplate: etcdserver: not capable
E0525 06:09:27.051437       1 cacher.go:274] unexpected ListAndWatch error: k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/cacher.go:215: Failed to list *api.ServiceAccount: etcdserver: not capable
E0525 06:09:27.051637       1 cacher.go:274] unexpected ListAndWatch error: k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/cacher.go:215: Failed to list *api.PersistentVolume: etcdserver: not capable
E0525 06:09:27.051976       1 cacher.go:274] unexpected ListAndWatch error: k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/cacher.go:215: Failed to list *api.PersistentVolumeClaim: etcdserver: not capable
[restful] 2017/05/25 06:09:27 log.go:30: [restful/swagger] listing is available at https://172.16.69.231:6443/swaggerapi/
[restful] 2017/05/25 06:09:27 log.go:30: [restful/swagger] https://172.16.69.231:6443/swaggerui/ is mapped to folder /swagger-ui/
I0525 06:09:27.311252       1 serve.go:79] Serving securely on 0.0.0.0:6443
I0525 06:09:28.133020       1 trace.go:61] Trace "List /api/v1/componentstatuses" (started 2017-05-25 06:09:27.603509051 +0000 UTC):
[10.387µs] [10.387µs] About to List from storage
[528.777801ms] [528.767414ms] Listing from storage done
[528.992544ms] [214.743µs] Self-linking done
[529.453962ms] [461.418µs] Writing http response done (3 items)
"List /api/v1/componentstatuses" [529.455721ms] [1.759µs] END
E0525 06:09:57.123384       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *rbac.ClusterRoleBinding: Get https://localhost:6443/apis/rbac.authorization.k8s.io/v1beta1/clusterrolebindings?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.123384       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *rbac.ClusterRole: Get https://localhost:6443/apis/rbac.authorization.k8s.io/v1beta1/clusterroles?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.123590       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *api.LimitRange: Get https://localhost:6443/api/v1/limitranges?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.124149       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *api.ServiceAccount: Get https://localhost:6443/api/v1/serviceaccounts?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.124459       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *api.ResourceQuota: Get https://localhost:6443/api/v1/resourcequotas?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.124610       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *rbac.Role: Get https://localhost:6443/apis/rbac.authorization.k8s.io/v1beta1/roles?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.125015       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *rbac.RoleBinding: Get https://localhost:6443/apis/rbac.authorization.k8s.io/v1beta1/rolebindings?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.125299       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *storage.StorageClass: Get https://localhost:6443/apis/storage.k8s.io/v1beta1/storageclasses?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.125809       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *api.Namespace: Get https://localhost:6443/api/v1/namespaces?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.125901       1 reflector.go:201] k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go:70: Failed to list *api.Secret: Get https://localhost:6443/api/v1/secrets?resourceVersion=0: dial tcp 192.157.208.178:6443: i/o timeout
W0525 06:09:57.315970       1 storage_extensions.go:127] third party resource sync failed: Get https://localhost:6443/apis/extensions/v1beta1/thirdpartyresources: dial tcp 192.157.208.178:6443: i/o timeout
E0525 06:09:57.316339       1 client_ca_hook.go:58] Post https://localhost:6443/api/v1/namespaces: dial tcp 192.157.208.178:6443: i/o timeout
F0525 06:09:57.316401       1 controller.go:128] Unable to perform initial IP allocation check: unable to refresh the service IP block: Get https://localhost:6443/api/v1/services: dial tcp 192.157.208.178:6443: i/o timeout
```
跟这个issue里报的是一样的错误,我的localhost也没有解析到127.0.0.1
https://github.com/kubernetes/kubeadm/issues/226

网上的解释是nslookup和dig在解析的时候是不读/etc/hosts的.
解决方案是安装`dnsmasq`,然后把`127.0.0.1`加到`/etc/resolv.conf`里去.


这样终于安装成功了!


```
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.6.4
[init] Using Authorization mode: RBAC
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [k8smaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.16.69.231]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 16.035424 seconds
[apiclient] Waiting for at least one node to register
[apiclient] First node has registered after 3.537375 seconds
[token] Using token: 3ac7c9.9759b75a505007ee
[apiconfig] Created RBAC rules
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  sudo cp /etc/kubernetes/admin.conf $HOME/
  sudo chown $(id -u):$(id -g) $HOME/admin.conf
  export KUBECONFIG=$HOME/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 3ac7c9.9759b75a505007ee 172.16.69.231:6443

```

# 2  安装Pod network

```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

又报错了,:-(
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
8080是什么服务?为什么没有启动?

原因是我没有按上面的提示运行下面的这几条命令:
```
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```


# k8s minion1加入到cluster中

在minion节点上运行下面命令:
```
# kubeadm join --token 3ac7c9.9759b75a505007ee 172.16.69.231:6443
```


报错了,看看master上的日志:

Error syncing pod 50caf365-411c-11e7-b134-080027a6dade ("kube-dns-3913472980-cxw0k_kube-system(50caf365-411c-11e7-b134-080027a6dade)"), skipping: failed to "CreatePodSandbox" for "kube-dns-3913472980-cxw0k_kube-system(50caf365-411c-11e7-b134-080027a6dade)" with CreatePodSandboxError: "CreatePodSandbox for pod \"kube-dns-3913472980-cxw0k_kube-system(50caf365-411c-11e7-b134-080027a6dade)\" failed: rpc error: code = 2 desc = NetworkPlugin cni failed to set up pod \"kube-dns-3913472980-cxw0k_kube-system\" network: open /run/flannel/subnet.env: no such file or directory"




$ kubectl get pods -n kube-system -o=wide
 
```
NAME                                READY     STATUS                                                                                                                                                                                                                                                                                RESTARTS   AGE       IP              NODE
etcd-k8smaster                      1/1       Running                                                                                                                                                                                                                                                                               0          2h        172.16.69.231   k8smaster
kube-apiserver-k8smaster            1/1       Running                                                                                                                                                                                                                                                                               0          2h        172.16.69.231   k8smaster
kube-controller-manager-k8smaster   1/1       Running                                                                                                                                                                                                                                                                               0          2h        172.16.69.231   k8smaster
kube-dns-3913472980-cxw0k           0/3       rpc error: code = 2 desc = failed to start container "a7b94de2e6145d65c77b921a624ea835d06d0ca28b522a5f9a482a693c129a4b": Error response from daemon: {"message":"cannot join network of a non running container: 4af379073c0d5d2a638a0b87d7e1855cdc77678060e233730b379f84b55fbd7c"}   186        2h        <none>          k8smaster
kube-flannel-ds-2dxbh               1/2       CrashLoopBackOff                                                                                                                                                                                                                                                                      15         56m       172.16.69.231   k8smaster
kube-proxy-3xdxm                    1/1       Running                                                                                                                                                                                                                                                                               0          2h        172.16.69.231   k8smaster
kube-scheduler-k8smaster            1/1       Running                                                                                                                                                                                                                                                                               0          2h        172.16.69.231   k8smaster
```

这个问题跟[这个issue](https://github.com/kubernetes/kubernetes/issues/44029)是一样的.
也就是在安装Pod network 之前要先安装: `kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml`.


# 重新安装 Pod network

```
# kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

再看看相关的pods:
```
xcadmin@k8smaster:~$ kubectl get pods -n kube-system -o=wide
NAME                                READY     STATUS    RESTARTS   AGE       IP              NODE
etcd-k8smaster                      1/1       Running   0          2h        172.16.69.231   k8smaster
kube-apiserver-k8smaster            1/1       Running   0          2h        172.16.69.231   k8smaster
kube-controller-manager-k8smaster   1/1       Running   0          2h        172.16.69.231   k8smaster
kube-dns-3913472980-cxw0k           3/3       Running   234        2h        10.244.0.2      k8smaster
kube-flannel-ds-2dxbh               2/2       Running   18         1h        172.16.69.231   k8smaster
kube-proxy-3xdxm                    1/1       Running   0          2h        172.16.69.231   k8smaster
kube-scheduler-k8smaster            1/1       Running   0          2h        172.16.69.231   k8smaster
```



# minion1 再次加入到cluster中

```
# kubeadm join --token 3ac7c9.9759b75a505007ee 172.16.69.231:6443
```

又报错了....
```
[discovery] Failed to request cluster info, will try again: [Get https://172.16.69.231:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: net/http: TLS handshake timeout]
```
这个问题网上没有解决方案,看来是我自己的原因,研究了好久我才意识到可能是跟https_proxy有关,因为翻墙要用到它,所以我设置了`https_proxy`,关闭它.
然后再加入,成功了!



########   这是以前的
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
5. [Installing Kubernetes on Linux with kubeadm](https://kubernetes.io/docs/getting-started-guides/kubeadm/)

http://tonybai.com/2016/12/30/install-kubernetes-on-ubuntu-with-kubeadm/



