---
title: 在M芯片Mac下安装mame模拟器
date: 2024-11-21T09:24:59+08:00
slug: installing-mame-on-m1-mac
categories:
  - mac
tags:
  - Mac
---



最近老刷到街机游戏的短视频，突然想起自从换了Mac的笔记本还没安装过街机模拟器。还好我之前收藏过开源街机模拟器，还知道软件的名字，所以收藏是个好习惯。

这款软件叫mame，源码地址是： https://github.com/mamedev/mame 
不过官方只提供windows版本的下载。

要下载mac版本的还需要移步到这个网站：[https://sdlmame.lngn.net/ ](https://sdlmame.lngn.net/ )  
在这个网站上要先下载[SDL运行时库](https://github.com/libsdl-org/SDL/releases)
因为这两个软件是开源软件，没有通过App Store分发，所以在Mac上首次运行会提示未验证的软件，所以要在隐私里点一下确认。

![](/assets/img/2024/11/20/2251101870.png)
如果遇到问题，能访问外网的可以参考这个视频：https://youtu.be/ipfdCzWXVGs?si=u94LaFJ0OxoLD16j

安装完后，就是下载ROM了，我在网上下载了很多ROM都不能运行，总提示缺少文件。奇怪怎么会缺文件呢？

![](/assets/img/2024/11/20/2251213090.png)

查了一下网上，原来是我的mame版本太新了，是最新版本。在不同版本中，ROM其实是不兼容的。

所以要找到对应mame版本的ROM。不是随便下载一个就能用的。

后来找到了这个网站 ：[https://www.retroroms.info/](https://www.retroroms.info/) ，这里有mame所有的rom， 而且是跟当前版本的mame相对应的。

有个问题，这个网站上的ROM都是英文的，不知道该下哪个文件。还好有热心网友已经为我们整理了街机游戏名字的中英文对照表，还放在了github上，真是值得我们点个赞。地址是： https://github.com/yingw/rom-name-cn
你可以在根据游戏的中文名找到对应的英文ROM的名称。

![](/assets/img/2024/11/21/0801480040.png)
假如我们想下载三国志II 吞食天地，对应的文件名是`wofj`,如果你直接下载这个文件 ，打开后还是提示缺少文件。
这是因为`wofj`这个版本是基于母版`wof`开发的，你往上看，可以看到`wof`这个母版，把它也下载下来，就可以开心地玩了。
