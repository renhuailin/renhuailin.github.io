---
layout: post
title: Linux uid备忘
date: 2015-12-30 12:02:06
author: 任怀林
categories:
- blog
- linux
thumb: linux.png
isCJKLanguage: true
---

最近在運行Jenkins容器時出現了權限問題，其原因是我掛載的volume的owner uid與容器裏的jenkins uid不一致,導致無法向掛載的卷寫數據。

我發現dockerfile裏指定jenkins的uid爲1000.爲什麼是1000？爲什麼不是其它值？我的host主機裏有uid爲1000的user嗎？


我查看了/etc/passwd,還真有uid爲1000的用戶。看來uid爲1000的用戶是除了root以外，第一個普通用戶的uid。真的是這樣嗎？問了Google後明白了一點，寫下來備忘。

uid爲0是root用戶;其實是所有uid爲0的用戶都是超級管理員。這意味着，你的系統裏可以有多個uid爲0的用戶，而且這個用戶的name不一定非要叫`root`. 但是請不要添加其它uid爲0的用戶，因爲這可能會引些系統混亂哦。

1~99的uid傳統是保留給特殊的系統用戶（有時稱做pseudo-users），比如www-data,mail,news,backup等。這些用戶也都是管理員，執行一些管理任務，但是它們不需要超級管理員那麼大的權限。


65534通常分配給nobody這個用戶。


接下來就是沒有特權的普通用戶了，有一些發行版是從100開始分配普通用戶的uid的，有一些發行版如redhat,是從500開始。debian是從1000開始的。

所以在ubuntu或debian系統裏的第一個普通用戶的uid肯定是1000，也是因爲這個，很多docker images裏要求的uid都是1000，因爲這個uid的用戶在host主機上應該是肯定存在的。

参考文档
http://www.linfo.org/uid.html
