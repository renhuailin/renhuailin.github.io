---
title: "Java 反编译器"
date: 2014-11-02 10:50:00
author: 任怀林
categories: 
- blog
- Java
tags:
- java
- decomplier
thumb: java.png
isCJKLanguage: true
---

前一段时间我因为项目的原因要用到反编译工具，就下了jd-gui，结果反编译出来的代码错误很多。想来是这些反编译器不思进取，对一些java的新语法支持的不好。于是在网找新的反编译工具，被我找到两个比较不错的，今天把它备录在这里。

<!--more-->

# procyon

项目地址：[https://bitbucket.org/mstrobel/procyon/wiki/Java%20Decompiler](https://bitbucket.org/mstrobel/procyon/wiki/Java%20Decompiler)

{% highlight sh %}

# 反编译单个类

$ java -jar decompiler.jar java.lang.String

# 反编译myJar.jar里的所有类，并把结果保存在outdir这个目录里

$ java -jar decompiler.jar -jar myJar.jar -o outdir
{% endhighlight %}

# CFR

这是另外一个反编译器，procyon的wiki里有提到过它。

主页：[http://www.benf.org/other/cfr/](http://www.benf.org/other/cfr/)    

{% highlight sh %}

# 反编译单个类

$ java -jar cfr_0_87.jar /home/harley/Test.class toString

# 反编译myJar.jar里的所有类，并把结果保存在/home/harley/java这个目录里

$ java -jar cfr_0_87.jar /home/harley/myJar.jar --outputdir /home/harley/java
{% endhighlight %}

好吧，这不算是一个好文章，只是我用来做备忘的，：）。
