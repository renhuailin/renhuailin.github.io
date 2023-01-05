---
layout: post
title: DCEVM+HotSwapAgent实现java类热加载
date: 2017-08-16 10:00:00
author: 任怀林
categories:
- java
thumb: java.png
isCJKLanguage: true
---

# 1.  安装DCEVM

DCEVM主页: https://dcevm.github.io/   

写此文时，支持的JDK 1.8的版本是：`Java 8 update 112, build 9`   
因为这个版本已经不是最新版jdk，所以需要去[Oracle Java Archive](http://www.oracle.com/technetwork/java/archive-139210.html)这个页面下载，你需要有oracle的账号。

下载完后安装好。

然后下载DCEVM的patch,是个jar包，从DCEVM主页上下载，我下文件名为：`DCEVM-light-8u112-installer.jar`

运行`java -version`确认您的jdk是`8u112`.

安装patch

```
$ sudo java -jar DCEVM-light-8u112-installer.jar
```

![dcevm-install.png](/assets/img/blog/2017/08/16/dcevm-install.png)

1. 选择安装目录   
   这个目录就是`Java 8 update 112`的安装目录，在Mac下，请运行`/usr/libexec/java_home`这个命令找到java_home.然后点击`Add installation directory...`这个按钮，选择java home下的jre目录。
2. 点击`Install DCEVM as altjvm`这个按钮安装。

# 2.   IntelliJ IDEA  配置

打IDEA的配置，选择左侧的`plugin`,搜索`HotSwapAgent`,然后安装它。

![config-idea.png](/assets/img/blog/2017/08/16/config-idea.png)
重启IDEA后，此plugin就生效了。

如果你的系统上安装了多个JDK，请确认你的项目用的是`Java 8 update 112`

下面配置HotSwapAgent plugin.

![config-hotswapagent-plugin.png](/assets/img/blog/2017/08/16/config-hotswapagent-plugin.png)

这样就行了。

打开你的项目，以debug的方式运行它（__一定要是debug模式__）。

IDEA有个问题，就是在debug模式下不是自动编译的。所以每次修改完代码，要按'cmd + shift + F9'来编译，然后class才能reload，这个挺烦人的，你会发现reload的速度并不是很快，不过总比每次点stop & run要快了不少。![](/Users/harley/Downloads/WX20201208-145543@2x.png)
