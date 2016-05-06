---
layout: post
title: 安装chef
date: 2016-02-22 13:39:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
---

# 1 why chef
为什么选择chef,而不是puppet,ansible或salt.
说到配置管理大家首先想到应该是puppet, 为什么不用puppet有以下几点原因：
１　puppet的配置都集中在puppet master上，你只能上面编写module,class。这使得puppet与jenkins等CI/CD工具配合起来比较困难。

chef分为三个组件，chef server,chef workstation,node。其中chef workstation是日常管理常用的，你可配置多个chef workstation,与jenkins等CI/CD工具配合起来比较容易。
2 puppet的module和class使用的是puppet自己的一种语法，虽然跟ruby很相似，但毕竟不是一种语言，还是要有个学习的过程。
而chef直接使用了ruby语言来编写cookbook。所以chef更适合开发人员使用。

ansible和salt没有仔细了解，我看了一下配置文件是yaml的，语法看起来非常的难受，就没再研究了。

# 2 chef的架构
这是官方的图
![chef_overview](/assets/img/blog/2016/2/22/chef_overview.svg)

chef核心组件有三个，chef server,chef workstation,node。
chef workstation 用户在这上面编写、测试和维护cookbook，配置节点，分发配置到节点。chef的cookbook对应puppet的module,
用户在chef workstation上编写和测试cookbook完后push到chef server上。node节点从cher server上获取cookbook并运行。
这一点跟puppet不同，如果用puppet，用户大部分的时间可能都在puppet master上，因为配置都保存在puppet master上。直接在puppet　master上工作是有风险的。

因为workstation组件,我个感觉chef在架构比puppet要好一些。

chef server  官方把它比喻成信息港，它保存了cookbook，policy等信息，这些信息都workstation　push上来的。node节点访问chef server来获得配置信息。

node　就是chef管理的主机，它可以是物理机，虚拟机，云或者是网络设备。

# 3　安装　chef server
首先我们来安装chef server,chef官方提供了一个400多兆的.deb文件，你可以在[这里](https://downloads.chef.io/chef-server/)下载



#1 安装workstation

就是安装chef sdk包


# dpkg -i chefdk-xxxx.deb
# chef verify

# 2 安装chef node
就是安装chef client包

# chef generate repo ~/chef-repo


# Install chef workstation

去chef management console下载Starter.kit
```
$ chef generate repo ~/chef-repo
```



# install chef manage console.

这个应该是这样安装
$ chef-server-ctl install opscode-manage

因为无法直接。

只能下载离线包来安装
$  chef-server-ctl install chef-manage --path /root

$ chef-server-ctl reconfigure

$ chef-manage-ctl reconfigure


# Create the administrator account and an organization



Item 	Description 	Notes and examples
ADMIN_USER_NAME 	user name for the administrator account 	jsmith, admin
ADMIN_PASSWORD 	password for the administrator account 	Note your password somewhere safe!
ADMIN_FIRST_NAME 	administrator's first name 	Joe
ADMIN_LAST_NAME 	administrator's last name 	Smith
ADMIN_EMAIL 	administrator's email address 	joe.smith@example.com
ORG_FULL_NAME 	your organization's full name 	Fourth Coffee, Inc.
ORG_SHORT_NAME 	your organization's abbreviated name 	4thcoffee


```
#$ sudo chef-server-ctl user-create ADMIN_USER_NAME ADMIN_FIRST_NAME ADMIN_LAST_NAME ADMIN_EMAIL ADMIN_PASSWORD --filename ADMIN_USER_NAME.pem
$ sudo chef-server-ctl user-create admin Huailin Ren renhuailin@ecloud.com.cn zxcv5678 --filename admin.pem
```

查看生成的pem

```
$ ls *.pem
```


创建组织

```
# sudo chef-server-ctl org-create ORG_SHORT_NAME "ORG_LONG_NAME" --association_user ADMIN_USER_NAME

# sudo chef-server-ctl org-create chinaops "China OPS Inc." --association_user admin
```



# 现在我们切回到Workstation.

用浏览器打开Chef Server所在的主机的IP，如https://chefserver.china-ops.com

切到Administrator标签

然后下载Starter Kit.

然后把这个解压到workstation的`~/chef-repo`.

```
$ ls ~/chef-repo/.chef
admin.pem  chinaops-validator.pem  knife.rb
```



```
$ cd ~/chef-repo
$ knife ssl fetch
```

```
WARNING: Certificates from chefmaster.china-ops.com will be fetched and placed in your trusted_cert
directory (/root/chef-repo/.chef/trusted_certs).

Knife has no means to verify these are the correct certificates. You should
verify the authenticity of these certificates after downloading.

Adding certificate for chefmaster.china-ops.com in /root/chef-repo/.chef/trusted_certs/chefmaster_china-ops_com.crt
```


用`knife ssl check`来verify这个证书。

```
$ knife ssl check
	Connecting to host chefmaster.china-ops.com:443
	Successfully verified certificates from `chefmaster.china-ops.com'
```


### Test the connection to Chef server  测试Workstation到Chef server的连接。

```
$ knife client list
	chinaops-validator
```


接下来转到workstation上。
```
$ cd ~/chef-repo
$ chef generate cookbook cookbooks/hello_chef_server
```


```
# vi cookbooks/hello_chef_server/recipes/default.rb
```

输入以下内容

```
#
# Cookbook Name:: hello_chef_server
# Recipe:: default
#
# Copyright (c) 2016 The Authors, All Rights Reserved.

file "#{Chef::Config[:file_cache_path]}/hello.txt" do
  contenet "hello chef server"
end
```

上传Cookbook到chef server.

```
$ knife cookbook upload hello_chef_server
```
报错了。看看详细日志吧

```
$ knife cookbook upload hello_chef_server -VV
```

然后在Chef server 上看服务器端的日志
```
$ chef-server-ctl tail
```
结果如下：
```
2016-02-22 16:15:10.899 [error] <0.694.0> gen_server <0.694.0> terminated with reason: {empty,[{queue,get,[{[],[]}],[{file,"queue.erl"},{line,182}]},{epgsql_sock,command_tag,1,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,409}]},{epgsql_sock,on_message,2,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,684}]},{epgsql_sock,loop,1,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,334}]},{gen_server,try_dispatch,4,[{file,"gen_server.erl"},...]},...]}
2016-02-22 16:15:10.900 [error] <0.694.0> CRASH REPORT Process <0.694.0> with 1 neighbours exited with reason: {empty,[{queue,get,[{[],[]}],[{file,"queue.erl"},{line,182}]},{epgsql_sock,command_tag,1,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,409}]},{epgsql_sock,on_message,2,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,684}]},{epgsql_sock,loop,1,[{file,"/var/cache/omnibus/src/oc_bifrost/_build/default/lib/epgsql/src/epgsql_sock.erl"},{line,334}]},{gen_server,try_dispatch,4,[{file,"gen_server.erl"},...]},...]} in gen_server:terminate/7 line 804
```
说真的，这日志真心看不懂。

我突然想起来了，我的Chef server的hostname没改成chefserver.china-ops.com,可能跟这个有关。
于是到chef server上把hostname 改成 `chefserver.china-ops.com`,并在`/etc/hosts`里解析到本地`127.0.0.1`。然后再到workstation上upload cookbook,这次成功了。



# Bootstrap node1
```
$ knife bootstrap 192.168.50.58 --ssh-user ecloud --ssh-password 'qwer1011' --sudo --use-sudo-password --node-name node1 --run-list 'recipe[hello_chef_server]'


ERROR: Error connecting to https://chefmaster.china-ops.com/organizations/chinaops/clients, retry 1/5
```
看来node节点也要配置Chef server的解析。

配置完node节点的host,然后再运行OK了。

第一次bootstrap　node时，用`knife bootstrap`,它会配置节点的信息，如节点的名称，run-list等。
以后的更新用`knife ssh`.
```
$ knife ssh "name:node1" --ssh-user ecloud --ssh-password 'qwer1011' 'sudo chef-client'
```


# 安装 Chef Analytics


http://www.hurryupandwait.io/blog/multi-node-testing-with-test-kitchen-and-docker

https://medium.com/brigade-engineering/reduce-chef-infrastructure-integration-test-times-by-75-with-test-kitchen-and-docker-bf638ab95a0a#.16pstgp4a


这篇文章有讲到如何用docker容器来和kitchen来做开发。
http://www.timusg.com/blog/2013/10/15/testing-cookbook-with-docker-and-test-kitchen/

主要是使用了kitchen-docker
https://github.com/portertech/kitchen-docker


依据https://docs.chef.io/cookbook_repo.html
如果用git来管理cookbook,那么安装第三方cookbook的方式为

$  knife cookbook site install COOKBOOK_NAME

如果不使用git来管理cookbook，那么使用第三方cookbook的方式为：
$ knife cookbook site download COOKBOOK_NAME

以下命令用来查看 cookbook的元数据信息。
$ knife cookbook metadata
这个命令会生成元数据信息，并保存为json格式。


# Attributes
At the beginning of a chef-client run, all attributes are reset. 这个要注意。


# 关于Role
用chef management console 和knife或是直接编辑文件的方式来管理role是会相互影响的，文档里建议我们只用一种方式，不要几种方式共用，因为会相互覆盖导致数据丢失。


`why-run Mode` 这个概念puppet里有，就是输出执行过程并不真正执行命令。


实战：

规划


两个Node节点，

1.  Role: appserver, ENV: development
2.  Role: webserver, ENV： production

http://williamherry.blogspot.tw/2012/06/chef_23.html 这篇文章写得挺好的，虽然不长但真是能理解很多东西，：）


#　Chef Delivery

Push build artifacts to production servers in real time　　，这个很重要，是实时的？



＃　使用 docker cookbook
要在 ubuntu 14.04上使用docker cookbook,首先要安装ruby2.1以上的版本。

```
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:brightbox/ruby-ng
$ sudo apt-get update
```

然后在Node节点上安装docker.

Create a new cookbook on workstation.
# cd ~/chef-repo
# knife cookbook create COOKBOOK_NAME


Add `depends 'docker', '~> 2.0'` to metadata.rb


# Install third party cookbook

```
$ knife cookbook site install COOKBOOK_NAME
```

or just download a cookbook

```
$ knife cookbook site download COOKBOOK_NAME
```

要在node节点上把默认的rubygem源换成淘宝的,将来可以考虑把这一步也做成一个recipe.
```
$ gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/
$ gem sources -l
*** CURRENT SOURCES ***

https://ruby.taobao.org
# 请确保只有 ruby.taobao.org
$ gem install rails
```

**参考资料：**
http://stackoverflow.com/questions/31730319/how-to-push-docker-images-through-reverse-proxy-to-artifactory
