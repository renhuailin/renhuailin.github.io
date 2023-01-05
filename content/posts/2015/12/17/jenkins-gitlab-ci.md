---
layout: article
title: Jenkins + gitlab做持续集成
date: 2015-12-17 10:43:06
author: 任怀林
tags:
- gitlab
- jenkins
thumb: gitlab.png
isCJKLanguage: true
---

jenkins  2核  2G  系统 ubuntu 14.04
registry
chef server 1 核  2G
gitlab 1核  2G

node1 最终部署节点　1 核  2G

# 安装Jenkins

## 配置jdk

# 安装 ant,maven

# 配置Jenkins的安全策略

#插件管理

在填好了git地址后，就开始build了，结果失败了，因为我们的gitlab https用的是自签名的证书，提示我们无法clone。这个其实简单，有二个解决办法，一是把git config里的http.sslverify设置为false。可以用一条命令来实现：

```
$ git config --global http.sslverify false
```

另一个办法就是设置环境变量

```
GIT_SSL_NO_VERIFY=1
```

我在安装Jenkins的Server上运行了`git config --global http.sslverify false`，然后运行Jenkins构建还是出错。我又试着重启了一下Jenkins,构建还是出错。看来Jenkins在运行git命令时没有读取这个全局的配置，挺奇怪的。

那就用第二种方法吧，如何设置某个项目构建时的环境变量呢？

在项目的“配置”里勾先上“参数化构建过程”，然后添加一个字符型的参数，把这环境变量添加上。

GitLab 8.3以後，GitLab Hook Plugin 已經不推薦了"deprecated".推薦使用'Jenkins CI' project 。

#错误处理

我一不小心点击了反选，把overall/read权限弄没有了，登录上来什么用不了。

搜了一下网上，简单的做法是把config.xml里的useSecurity设置为false,把`authorizationStrategy`，完全删除。再重启Jenkins就可以了。

#

在源码管理下勾选上`git`,然后从gitlab里复制项目的clone地址过来，填到Repository URL文本框里。

![git_repo_config.png](/assets/img/blog/2016/03/01/git_repo_config.png)

下一步`Additional Behaviours`中选择`Merge before build`.

`Name of repository`中填入`origin`

`Branch to merge to`中填入`${gitlabTargetBranch}`
![Additional_Behaviours.png](/assets/img/blog/2016/03/01/Additional_Behaviours.png)

接下来配置`构建触发器`，配置`Rebuild open Merge Requests`为　`On push to source or target branch`.
![build_trigger.png](/assets/img/blog/2016/03/01/build_trigger.png)

下面是配置mvn build时的参数，这个大家应该比较熟悉了，不多说。

接下我们要配置一下gitlab里项目，为项目添加一个webhook,这个webhook是Jenkins的gitlab插件触发build的URL.

在gitlab你的项目主页里点击`Settings`,然后点击左侧菜单的`Web hooks`.

在URL文本框里，我们要填写上你的项目在Jenkins里URL.

假设你的项目在Jenkins里的构建job的URL地址为：

```
http://192.168.50.55:8080/job/edesktop-manager
```

把URL中的`/job`换成`/project/`就是这个job的build触发URL，把它填到gilab项目设置的Web hooks　URL里。

```
http://192.168.50.55:8080/project/edesktop-manager
```

![gitlab_webhooks.png](/assets/img/blog/2016/03/01/gitlab_webhooks.png)

保存配置，然后点击`TEST HOOK`按钮，测试一下这个hook,如果系统显示没有问题，系统会提示：hook已经成功执行。
你在Jenkins里会看到有一个新build执行了。

要想让Jenkins能运行docker,我们要jenkins用户加到docker这个组里来。

```
# usermod -a -G docker jenkins
```

然后需要reboot jenkins那台主机,否则jenkisn无法访问docker daemon.

```
Cannot connect to the Docker daemon. Is the docker daemon running on this host?
```

可以参考这个[issue](https://github.com/docker/docker/issues/5314)

# 参考文档

http://www.vogella.com/tutorials/Jenkins/article.html#installation_ubuntu

http://hyhx2008.github.io/li-yong-jenkinsgitlabda-jian-chi-xu-ji-cheng-cihuan-jing.html
