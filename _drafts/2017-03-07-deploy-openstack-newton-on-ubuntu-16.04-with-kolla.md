---
layout: post
title: 在Ubuntu 16.04上部署OpenStack newton
date: 2017-03-07 12:30:00
author: 任怀林
categories:
- blog
- OpenStack
thumb: openstack.png
---

# 1 规划


| 主机名称 | 管理段IP | IP | 描述 |
|-----------------|:----:|:------------:|:-----------|
| controller1  | 172.16.69.235 |  192.168.56.110 | 控制节点 |
| compute1     | 172.16.69.236 |  192.168.56.111 | 计算节点 |
| network1     | 172.16.69.237 |  192.168.56.112 | 网络节点 |





* network_interface    
  While it is not used on its own, this provides the required default for other interfaces below.

* api_interface     
  This interface is used for the management network. The management network is the network OpenStack services uses to communicate to each other and the databases. There are known security risks here, so it's recommended to make this network internal, not accessible from outside. Defaults to `network_interface`.

* kolla_external_vip_interface     
  This interface is public-facing one. It's used when you want HAProxy public endpoints to be exposed in different network than internal ones. It is mandatory to set this option when `kolla_enable_tls_external` is set to yes. Defaults to `network_interface`.

* storage_interface   
  This is the interface that is used by virtual machines to communicate to Ceph. This can be heavily utilized so it's recommended to put this network on 10Gig networking. Defaults to `network_interface`.

* cluster_interface
  This is another interface used by Ceph. It's used for data replication. It can be heavily utilized also and if it becomes a bottleneck it can affect data consistency and performance of whole cluster. Defaults to `network_interface`.

* tunnel_interface     
  This interface is used by Neutron for vm-to-vm traffic over tunneled networks (like VxLan). Defaults to `network_interface`.
* neutron_external_interface    
  This interface is required by Neutron. Neutron will put br-ex on it. It will be used for flat networking as well as tagged vlan networks. Has to be set separately.

这些网络接口interface是在  `/etc/kolla/globals.yml`里定义的。







networ node必须有三个NIC(network interface card),也就是网卡．

管理网段：172.16.69.0/24

在所有节点的/etc/hosts里加入：

```
192.168.56.110  controller1 
192.168.56.111  compute1    
192.168.56.112  network1
```


# Kolla 
使用kolla快速部署openstack all-in-one环境
https://xiexianbin.cn/openstack/2016/10/23/use-kolla-to-deploy-openstack-all-in-one-env


九州云-生产环境中使用Docker自动化部署升级OpenStack的运维实践
http://doc.mbalib.com/view/b75d377a08dbe37ec0f42f5efbce5765.html

## Install OpenStack Newton with Kolla

openstack-controller1 172.16.69.226    ubuntu 16.04
openstack-network-node1 172.16.69.227  ubuntu 16.04
openstack-compute-node1 172.16.69.228  ubuntu 16.04
ansible-docker-registry 172.16.69.229  ubuntu 16.04  

### 安装pip
在ansible-docker-registry 172.16.69.229这台机器上安装pip

```
$ sudo apt-get install python-pip 
$ suod apt-get install python-dev libffi-dev gcc libssl-dev
```

### 安装ansible, kolla
On ansible-docker-registry 172.16.69.229

$ sudo apt install ansible

$ git clone https://github.com/openstack/kolla.git

运行一个私有registry服务。
$ tools/start-registry 


### Configure Docker on all nodes

在所有节点的/etc/hosts里加入：

```
172.16.69.226  controller1 
172.16.69.228  compute1    
172.16.69.227  network1
```


在其它所有的节点上配置docker,让它可以使用这个私有的registry.
编辑`/etc/default/docker`

```
DOCKER_OPTS="--insecure-registry 172.16.69.229:5000"
```

16.04 使用systemd了，所以根据Docker官方文档[Control and configure Docker with systemd](https://docs.docker.com/engine/admin/systemd/)

```
$ sudo cp /lib/systemd/system/docker.service /etc/systemd/system/docker.service
```
接下来，我们修改文件`/etc/systemd/system/docker.service`的`Service`小节：
``` ini
[Service]
MountFlags=shared
EnvironmentFile=-/etc/default/docker
ExecStart=/usr/bin/docker daemon -H fd:// $DOCKER_OPTS
```

重启docker服务

```
# systemctl daemon-reload
# systemctl restart docker
```

确认修改成功

```
$ docker info | grep Registries -C 10

....
Registry: https://index.docker.io/v1/
WARNING: No swap limit support
Insecure Registries:
 172.16.69.229:5000
 127.0.0.0/8
```
可以看到` 172.16.69.229:5000`加入到`Insecure Registries`中了。

編輯`/etc/rc.local`檔案，加入以下內容：

```
mount --make-shared /run
```

因为我们用的是Ubuntu 16.04,请卸载`lxd lxc`，因为`cgroup mounts`问题，在启动容器时，mounts会指数级地增长。

```
# apt remove lxd lxc
```

安装`pip` ,然后通过 `pip` 安装 `docker-py`

```
# apt-get install -y python-pip python-dev
# pip install -U pip docker-py
```



上面的操作要在所有的节点都操作一遍。


### 配置计算节点 
在`openstack-compute-node1 172.16.69.228`上

編輯`/etc/rc.local`檔案，加入以下內容：
```
mount --make-shared /var/lib/nova/mnt
```
保存后，执行下面的命令：

```
# mkdir -p /var/lib/nova/mnt /var/lib/nova/mnt1
# mount --bind /var/lib/nova/mnt1 /var/lib/nova/mnt
# mount --make-shared /var/lib/nova/mnt
```



配置时间同步
我的这三台机器都能上外网，所以直接配置ntp同步就行了。如果你的机器不能访问外网，就在其中的一台上安装ntp服务，然后让其它的机器从它同步时间，保证所有的机器时间一致就OK。

16.04 使用timedatect同步时间了。
https://help.ubuntu.com/lts/serverguide/NTP.html

```
# timedatectl status
```
请确认状态是对的。

Libvirt is started by default on many operating systems. Please disable libvirt on any machines that will be deployment targets. Only one copy of libvirt may be running at a time.
在很多系统下Libvrit是默认自动启动的，
```
service libvirt-bin stop
update-rc.d libvirt-bin disable
```

16.04默认是没有启动的，所以这一步可以略过。

### 重启所有节点
重启所有节点,以使配置生效。


# 编辑 ansible的`Inventory`文件


下面开始安装kolla，我发现必须如果要通过源码安装，一定要clone kolla repository.不能下载压缩包安装，否则会报下面的错：
```
 Complete output from command python setup.py egg_info:
    ERROR:root:Error parsing
    Traceback (most recent call last):
      File "/usr/local/lib/python2.7/dist-packages/pbr/core.py", line 111, in pbr
        attrs = util.cfg_to_args(path, dist.script_args)
      File "/usr/local/lib/python2.7/dist-packages/pbr/util.py", line 246, in cfg_to_args
        pbr.hooks.setup_hook(config)
      File "/usr/local/lib/python2.7/dist-packages/pbr/hooks/__init__.py", line 25, in setup_hook
        metadata_config.run()
      File "/usr/local/lib/python2.7/dist-packages/pbr/hooks/base.py", line 27, in run
        self.hook()
      File "/usr/local/lib/python2.7/dist-packages/pbr/hooks/metadata.py", line 26, in hook
        self.config['name'], self.config.get('version', None))
      File "/usr/local/lib/python2.7/dist-packages/pbr/packaging.py", line 725, in get_version
        raise Exception("Versioning for this project requires either an sdist"
    Exception: Versioning for this project requires either an sdist tarball, or access to an upstream git repository. Are you sure that git is installed?
    error in setup command: Error parsing /tmp/pip-AV4mIk-build/setup.cfg: Exception: Versioning for this project requires either an sdist tarball, or access to an upstream git repository. Are you sure that git is installed?
```

Installing Kolla and dependencies for development

To clone the Kolla repo:

git clone https://git.openstack.org/openstack/kolla
To install Kolla’s Python dependencies use:

pip install -r kolla/requirements.txt -r kolla/test-requirements.txt
Note This does not actually install Kolla. Many commands in this documentation are named differently in the tools directory.
Kolla holds configurations files in etc/kolla. Copy the configuration files to /etc:

cd kolla
cp -r etc/kolla /etc/
Install Python Clients
On the system where the OpenStack CLI/Python code is run, the Kolla community recommends installing the OpenStack python clients if they are not installed. This could be a completely different machine then the deployment host or deployment targets. Install dependencies needed to build the code with pip package manager as explained earlier.

To install the clients use:

yum install python-openstackclient python-neutronclient
Or using pip to install:


```
pip install -U python-openstackclient python-neutronclient

pip install .
```

```
kolla-build --base ubuntu --type source --registry 172.16.69.229:5000 --push
```

编辑`/etc/kolla/globals.yml`
```
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "4.0.0"
docker_registry: "172.16.69.229:5000"
network_interface: "eth0"
neutron_external_interface: "eth1"

# 这个IP绑定在网络节点的eth0上，要用ip addr这个命令才能看出来。
kolla_internal_vip_address: "172.16.69.240"  
network_interface: "eth0"
```




kolla-genpwd



```
# kolla-ansible prechecks -i ansible/inventory/multinode
```

```
172.16.69.226	control01
172.16.69.227	network01
172.16.69.228	compute01
```

control01 ansible_host=172.16.69.226 ansible_user=xcadmin ansible_ssh_pass="Xiangc10ud"
network01 ansible_host=172.16.69.227 ansible_user=xcadmin ansible_ssh_pass="Xiangc10ud"
compute01 ansible_host=172.16.69.228 ansible_user=xcadmin ansible_ssh_pass="Xiangc10ud"

```
TASK [prechecks : fail] ********************************************************
failed: [compute01] => (item={u'stdout': u'192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ', u'cmd': [u'getent', u'ahostsv4', u'controller1'], u'end': u'2017-01-13 15:28:12.477273', '_ansible_no_log': False, u'warnings': [], u'changed': False, u'start': u'2017-01-13 15:28:12.447745', u'delta': u'0:00:00.029528', 'item': u'control01', u'rc': 0, 'invocation': {'module_name': u'command', u'module_args': {u'creates': None, u'executable': None, u'chdir': None, u'_raw_params': u'getent ahostsv4 controller1', u'removes': None, u'warn': True, u'_uses_shell': False}}, 'stdout_lines': [u'192.157.208.178 STREAM controller1', u'192.157.208.178 DGRAM  ', u'192.157.208.178 RAW    '], u'stderr': u''}) => {"failed": true, "item": {"_ansible_no_log": false, "changed": false, "cmd": ["getent", "ahostsv4", "controller1"], "delta": "0:00:00.029528", "end": "2017-01-13 15:28:12.477273", "invocation": {"module_args": {"_raw_params": "getent ahostsv4 controller1", "_uses_shell": false, "chdir": null, "creates": null, "executable": null, "removes": null, "warn": true}, "module_name": "command"}, "item": "control01", "rc": 0, "start": "2017-01-13 15:28:12.447745", "stderr": "", "stdout": "192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ", "stdout_lines": ["192.157.208.178 STREAM controller1", "192.157.208.178 DGRAM  ", "192.157.208.178 RAW    "], "warnings": []}, "msg": "Hostname has to resolve to IP address of api_interface"}
failed: [control01] => (item={u'stdout': u'192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ', u'cmd': [u'getent', u'ahostsv4', u'controller1'], u'end': u'2017-01-13 15:28:12.405884', '_ansible_no_log': False, u'warnings': [], u'changed': False, u'start': u'2017-01-13 15:28:12.366799', u'delta': u'0:00:00.039085', 'item': u'control01', u'rc': 0, 'invocation': {'module_name': u'command', u'module_args': {u'creates': None, u'executable': None, u'chdir': None, u'_raw_params': u'getent ahostsv4 controller1', u'removes': None, u'warn': True, u'_uses_shell': False}}, 'stdout_lines': [u'192.157.208.178 STREAM controller1', u'192.157.208.178 DGRAM  ', u'192.157.208.178 RAW    '], u'stderr': u''}) => {"failed": true, "item": {"_ansible_no_log": false, "changed": false, "cmd": ["getent", "ahostsv4", "controller1"], "delta": "0:00:00.039085", "end": "2017-01-13 15:28:12.405884", "invocation": {"module_args": {"_raw_params": "getent ahostsv4 controller1", "_uses_shell": false, "chdir": null, "creates": null, "executable": null, "removes": null, "warn": true}, "module_name": "command"}, "item": "control01", "rc": 0, "start": "2017-01-13 15:28:12.366799", "stderr": "", "stdout": "192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ", "stdout_lines": ["192.157.208.178 STREAM controller1", "192.157.208.178 DGRAM  ", "192.157.208.178 RAW    "], "warnings": []}, "msg": "Hostname has to resolve to IP address of api_interface"}
failed: [network01] => (item={u'stdout': u'192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ', u'cmd': [u'getent', u'ahostsv4', u'controller1'], u'end': u'2017-01-13 15:28:12.505458', '_ansible_no_log': False, u'warnings': [], u'changed': False, u'start': u'2017-01-13 15:28:12.475810', u'delta': u'0:00:00.029648', 'item': u'control01', u'rc': 0, 'invocation': {'module_name': u'command', u'module_args': {u'creates': None, u'executable': None, u'chdir': None, u'_raw_params': u'getent ahostsv4 controller1', u'removes': None, u'warn': True, u'_uses_shell': False}}, 'stdout_lines': [u'192.157.208.178 STREAM controller1', u'192.157.208.178 DGRAM  ', u'192.157.208.178 RAW    '], u'stderr': u''}) => {"failed": true, "item": {"_ansible_no_log": false, "changed": false, "cmd": ["getent", "ahostsv4", "controller1"], "delta": "0:00:00.029648", "end": "2017-01-13 15:28:12.505458", "invocation": {"module_args": {"_raw_params": "getent ahostsv4 controller1", "_uses_shell": false, "chdir": null, "creates": null, "executable": null, "removes": null, "warn": true}, "module_name": "command"}, "item": "control01", "rc": 0, "start": "2017-01-13 15:28:12.475810", "stderr": "", "stdout": "192.157.208.178 STREAM controller1\n192.157.208.178 DGRAM  \n192.157.208.178 RAW    ", "stdout_lines": ["192.157.208.178 STREAM controller1", "192.157.208.178 DGRAM  ", "192.157.208.178 RAW    "], "warnings": []}, "msg": "Hostname has to resolve to IP address of api_interface"}
```


# kolla-ansible deploy -i ansible/inventory/multinode

如果没有错误，就可以运行下面的处理。
# kolla-ansible post-deploy


参考：
http://docs.openstack.org/developer/kolla/quickstart.html

